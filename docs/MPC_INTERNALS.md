# MPC OS internals (MPC One, firmware 3.9.1)

Observed live and read-only over SSH on 2026-10-01 (MPC One, armv7l, SSH-patched 3.9.1), by poloq.
Sources: `ps`, `/proc/<pid>/task/*/stat`, `/proc/<pid>/fd` and `maps`, `/proc/asound/*`, `/proc/net/*`,
systemd unit files, the device tree, `strings` on the binaries. Items marked **(inferred)** are reasoning
from that evidence, not confirmed. Other models (Live, Force) likely differ in card numbers and codec.

## System

- Rockchip RK3288, 4x Cortex-A17, 2 GB RAM, no swap. Linux 6.18 **PREEMPT_RT**, systemd, BusyBox userland
  (no `curl`, `ss`, `netstat`; `head -c`/`-5` unsupported, use `sed -n`).
- Root fs: read-only ext4 (~96% full). `/etc` and `/var` are overlays whose upper dirs live on `/data`
  (`/data/system/etc/overlay`, `/data/system/var/overlay`), so changes there persist across reboots
  without reflashing (e.g. a systemd drop-in under `/etc/systemd/system/acvs.service.d/`).
- Partitions: `mmcblk0p7` → `/media/acvs-content` (factory content); `mmcblk0p8` → `/data` =
  `/media/az01-internal` (projects, Settings, Synths/plugins). SD/USB mounted by `edisksd` at
  `/media/<LABEL>` (exFAT, `noexec`).
- Hardware watchdog `/dev/watchdog` exists but `RuntimeWatchdogSec` is off: a hang does not reboot.

## Processes

One monolithic app plus small daemons. None of the daemons is in the audio or MIDI path.

| Process | Unit | Role |
|---|---|---|
| `/usr/bin/MPC` (112 MB, JUCE) | `acvs.service`, `Restart=always`, via `/usr/bin/az01-launch-MPC` | Everything: sequencer, audio engine, UI, MIDI, plugins, files, Ableton Link, Splice |
| `crashpad_handler` | child of MPC | Minidumps to Sentry |
| `DeviceControlServer.bin` | `az01-network-midi.service` | Network device control / MIDI; mDNS 5353, UDP/TCP 51000, UDP 34804; holds `/dev/snd/seq` |
| `az0x-webserver` | `az0x-webserver.service` | HTTP :80 (see below) |
| `az01-script-runner` | `az01-script-runner.service` | Root `system()` on any datagram to UDP 127.0.0.1:8080, no auth. Caller unconfirmed (inferred: MPC, for commands that must outlive acvs.service) |
| `edisksd` | D-Bus `io.github.inmusicdev.edisksd` | Mounts USB/SD |
| `az01-unsupp-dev` | D-Bus `com.inmusicbrands.unsupported_devices` | Flags unsupported USB devices |
| connmand, wpa_supplicant, bluetoothd | | Network, BT |
| `az01-pwrbtn`, `az01-usbsata-fixer` | | Long-press power shutdown; SATA bridge config |

No X11/Wayland, no PulseAudio/PipeWire/JACK, no out-of-process plugin host.
The launcher also checks `dfu-util -l` for the panel MCU in DFU mode (`0x08000000`, STM32) and runs
`/usr/share/Akai/SME0/Firmware/update.sh` if so.

MPC's environment: `LD_PRELOAD=/usr/lib/libforce_cursor.so` (SSH/mouse patch), `MALLOC_ARENA_MAX=1`,
`GLIBC_TUNABLES=glibc.malloc.hugetlb=2`. The launcher comment says MPC is locked in memory, and caps
thread stacks at 1 MiB in its non-`systemd-inhibit` path.

## Threads inside MPC (43 total)

| Policy / prio | Threads |
|---|---|
| SCHED_FIFO 20 | `Audio Processing` (drives ALSA), `AudioWorker0..3` (one per core) |
| SCHED_FIFO 10 | `MIDI`, `JUCE MIDI Input`, `MIDI Out: MPC One…`, `MIDI Out: JUCE…` |
| SCHED_FIFO 1 | `FileIO` (disk streaming), `JUCE Timer` |
| SCHED_OTHER | `MPC Main Thread` (UI), `JUCE Message Thread` x5, `LinuxPoll`, `DBusClient`, `AudioWatchdog`, `scripting`, `PHReporting`, `sentry-http` |
| niced | `Background 1/2`, `File Management`, `JUCE FileBrowser` x6, `DirectoryWatcher` x8, `Block Handler`, `Malloc Free` |

Plugin implications: a VST2 `process()` runs on an `AudioWorker` at FIFO 20 on an RT kernel, so a
blocking call there stalls the whole block. Plugins are in-process, so a plugin crash kills MPC (systemd
restarts it; the open project is lost). `Malloc Free` = MPC defers `free()` off the audio thread; do the same.

## Audio path

- Codec: inMusic "ACVA" (`inmusic,acva-audio-codec`) on the SoC I2S (`ff890000.i2s`), driver
  `snd_soc_inmusic_jp07`. ALSA **card 1 "ACVA"**, `hw:1,0`, one playback + one capture PCM.
- No ALSA mixer controls at all: gain is digital, in MPC.
- MPC opens `pcmC1D0p`/`pcmC1D0c` directly (`hw`, not `plughw`, no dmix). Observed config:
  S32_LE, 2 ch, 44100 Hz, period 128 frames (2.90 ms), playback buffer 384 (3 periods), capture 256 (2).
  These follow MPC's audio preferences.
- USB-B computer mode: MPC builds a configfs gadget `smexstream` (`uac2.audio` + `midi.midi`) on UDC
  `ff580000.usb` via libusbgx. Absent when no computer is attached.

## MIDI path

- The pads, knobs, buttons, LEDs **and the DIN MIDI jacks** sit behind a panel MCU (STM32) on an internal
  USB bus (`ff540000`), enumerating as USB-MIDI **card 0 "MPC One MIDI"**, `midiC0D0`, 3 ports:
  `MPC Studio Live Public` (0), `… Private` (1), `… MIDI Port` (2, the DIN jacks).
- MPC opens ports 0 and 1 as **rawmidi** directly (3 fds on `midiC0D0`), not via the sequencer.
  Counters at ~3 h uptime: Public out 804 KB; Private in 1.7 KB, out 0.7 KB. (inferred) Private carries
  panel events, Public carries LED/pad-colour updates. Protocol undocumented; port names suggest the
  MPC Studio/controller-mode protocol.
- ALSA sequencer: MPC is client 129 "MPC", connected to 16:2 (DIN) both ways. MPC also creates
  client 128 "JUCE" with `Virtual MIDI Input 1/2` and `Virtual MIDI Output 1/2`, all wired to 129.
  (inferred) Other processes can inject MIDI into MPC via `aconnect` to those ports.
- See NOTES.md "Open issues" for the plugin-side MIDI port workaround (`poc/midiport.c`).

## Display and input

- DRM/KMS `/dev/dri/card1` (VOP + MIPI panel), panfrost GPU, RGA blitter `/dev/video0`.
- Touch: ILI2117 `/dev/input/event0` (libinput). `gpio-keys` on `event1`.

## Network ports

| Port | Owner |
|---|---|
| 22 | sshd (patch) |
| 80 | az0x-webserver |
| 41401/tcp | MPC (not HTTP; purpose unknown) |
| 51000, 34804/udp, 5353/udp | DeviceControlServer |
| 127.0.0.1:8080/udp | az01-script-runner |

## az0x-webserver (port 80)

- `/` → 404 (no web UI; `/srv` empty). `/files/` → 401 (auth data `/data/az0x-webserver/auth-data.json`
  absent). `/software-update` → upload-only (`az01-update --file /data/az01-update.img`).
- `/testapp` → 200, **unauthenticated** upload form. On upload it runs
  `systemctl stop "<services>" && test-app-launcher --testapp-path="<file>"`, which stops MPC.
  Config would live in `/etc/az0x-webserver/az0x-webserver.conf` (absent; defaults used).
- `test-app-launcher` 2.2.0 (JUCE, takes DRM + libinput) accepts: V2 `.zip` with `manifest.yaml`
  (`testApps`: `ProductCodes`, `OSVersionID`, `Version`, `RelExePath`); V1 archives with a `*TestApp*`
  executable; signed `.taimg` mounted via `az01-signed-fs` under `/secure-media`. Executable must contain
  `testapp` in its name. Runs as root. Whether unsigned packages are accepted on this build is a runtime
  flag, not verified. MPC does not restart afterwards (`systemctl start acvs` or reboot).

## Replacing MPC with another app (assessment, not attempted)

- Easy: audio (`hw:1,0` direct), display (DRM/KMS) and touch once `acvs` is stopped; launching via
  `systemctl stop acvs && ./app`, `/testapp`, or persistently via an `/etc` drop-in with a fallback to MPC.
- Hard part: the panel MCU protocol (input mapping on Private, LED/pad colour output and any init
  handshake on Public). Plan: an `LD_PRELOAD` shim in MPC that logs `midiC0D0` reads/writes from boot.
  Check community prior art (TKGL mpcmapper / IamForce) first.

## CPU layout and plugin performance (measured 2026-10-01)

- CPU: Cortex-A17 (part 0xc0d) x4 at a fixed 1.608 GHz (`performance` governor, no cpuidle states).
  Features: NEON, VFPv4 (fused multiply-add), hardware integer divide. Per spec: 32 KB L1D per core,
  1 MB L2 shared by all four cores (sysfs doesn't expose sizes). Kernel HZ=1000.
- `isolcpus=2-3`. MPC pins every thread to one core:
  core 0 = UI main, JUCE Timer, browser threads, AudioWorker0, most IRQs (eth, GPU, touch, SD);
  core 1 = AudioWorker1, FileIO, dir watchers; core 2 = AudioWorker2, MIDI threads, panel-USB IRQ;
  core 3 = Audio Processing, AudioWorker3, audio DMA IRQ. System daemons are confined to cores 0-1.
  IRQ threads run at FIFO 50, above the audio workers (FIFO 20): after ~3 h, AudioWorker0 had 99
  involuntary context switches, AudioWorker1 had 2.
- Idle load: core 2 ~1%, the others 7-9%. Core 2 is the best home for a plugin's own background thread.
  Threads inherit the creator's affinity and (by default) its scheduling policy: a thread created from the
  editor/main thread ends up pinned to busy core 0; set its affinity explicitly.
- Plaits test: MPC keeps a track on one worker (Plaits stayed on AudioWorker1/core 1 across runs).
  A running sequence cost ~7% of a core on that worker plus ~4% on Audio Processing (~0.2-0.35 ms of the
  2.9 ms block). Plaits creates no threads.
- Memory: MPC is `mlockall`ed (VmLck ≈ VmRSS ≈ 650 MB), so plugin allocations are locked and resident
  too, and new mappings are faulted in up front: allocate in open/resume, never in process().
  ~1.2 GB available. Thread stacks default to 1 MiB.
- ABI on the device: glibc 2.39, libstdc++ GLIBCXX_3.4.32 / CXXABI_1.3.14 (shared with MPC).
- **Build flags leave NEON unused.** `tools/build_port.sh` compiles with `arm32v7/gcc:12` and plain `-O2`;
  that compiler defaults to `-march=armv7-a+fp` (VFPv3-D16, Thumb-2, no NEON). The deployed `plaits.so`
  reports `Tag_FP_arch: VFPv3-D16` and is almost all scalar VFP code. Candidate flags (not yet benchmarked):
  `-mcpu=cortex-a17 -mfpu=neon-vfpv4 -O3`. GCC won't auto-vectorise float maths onto ARMv7 NEON without
  `-funsafe-math-optimizations` (or `-ffast-math`), because NEON flushes denormals; check bit-exact tests.
- Denormals: scalar VFP honours them by default (FPSCR.FZ=0 on Linux); whether MPC sets FZ on its audio
  threads is unverified. Setting FZ in `process()` is cheap insurance for decaying filters/reverbs.
