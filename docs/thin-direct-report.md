# Thin "boot straight into macOS" without GNOME and without Wayland (2026-11-10, 15:00-16:10)

## What changed
The path from macOS to the panel is now: guest (DroidVMDisplay + the Metal/Vulkan bridge, unchanged) → virtio-gpu →
crosvm → **KMS directly** (crosvm is the DRM master of `/dev/dri/card1`, the DSI panel). Input goes **evdev → crosvm**
(the THP touchscreen, the cover keyboard, Bluetooth keyboards, the volume keys), grabbed so that the console never sees them.
There is no desktop, no Wayland compositor, no cage, no seatd and no separate presenter process.

New crosvm display backend (`crosvm-kms/`, applied with `crosvm-kms/apply.py`; binary before it: `~/vm/bin/crosvm-mmio-gpu.pre-kms`):
- `display_kms.c` (C, libdrm + GBM + EGL/GLES) and `gpu_display_kms.rs` (the Rust `DisplayT`, written like the Wayland one).
- Selected with the existing `--wayland-sock` argument: a path `kms:/dev/dri/card1` opens KMS instead of a compositor
  (`WL=kms:/dev/dri/card1` in run-macos.sh), so the display's virtio-input devices and scanout imports are reused as they are.
- **Why one GPU blit and not direct plane scanout yet:** the guest's scanout buffers are ABGR16161616F (half float,
  `fourcc=0x48344241`). No DPU plane scans that format out: modetest lists only 8/10-bit RGB and YUV on all planes. So each new
  guest frame is drawn once (scaled 1600x1068 → 3200x2136, aspect kept) into one of three linear XRGB8888 GBM buffers on the
  render node, then page-flipped. This is the same GPU work GNOME's compositor did for the VM window, minus the compositor.
  The blit context is high priority (`EGL_IMG_context_priority`). It does not block crosvm: the blit's native fence fd sits in
  the backend's epoll set, and the flip is queued when it signals. Only the newest frame is drawn; one flip is in flight.
- **Next step to true zero-copy:** an 8-bit scanout format from the guest (DroidVMDisplay/bridge, e.g. BGRA8 or 10-bit).
  The imported buffer could then go to a plane with `drmModeAddFB2` and no blit (plane scaling covers 2x). That is a guest
  render-format change, the `sheng-fb-bgra=1` path, which still had a red/blue issue; not done in this slice.
- The CPU path also works (the boot console before DroidVMDisplay: the simplefb surface → texture upload → same blit).
- Touch: raw MT protocol B, normalized by the device's ranges, then the device's udev `LIBINPUT_CALIBRATION_MATRIX` (the THP's
  Y flip), the panel letterbox undone, then guest pixels. Hotplug: inotify on `/dev/input`; SYN_DROPPED lifts all contacts.
- Keys: any device with A+Enter, plus the volume keys. The power key is never grabbed unless `CROSVM_KMS_POWER_KEY=1` (the C
  default is now also "leave it to logind": a short press = clean shutdown, the VM's unit stops the guest first).
- Lid: gpio-keys (grabbed by crosvm) also carries SW_LID. Closing the cover turns the panel off (DPMS), opening it turns it
  on again (added 16:12-17:00, see "Audit fixes" below).
- Escape without a desktop: **Volume Up + Volume Down held 2 s** runs `CROSVM_KMS_ESCAPE`. On the full install that stops the
  VM cleanly and GNOME comes back. On the thin root it restarts the VM.

## Scripts (tablet `~/vm`, backups in `thin-direct/`)
- `thin/direct.sh`: the VM straight on the panel (what the thin root's `kiosk.service` runs now, in place of cage). Since the
  audit it is a small supervisor (see "Audit fixes" below).
- `macos-direct.sh`: on the full install, leaves GNOME (`systemctl stop gdm`, never isolate), makes sure piano-charging,
  piano-thp and user@1000 (PipeWire) run, and starts the unit `macos-vm` with direct.sh. App grid: **"macOS (direct, no desktop)"**.
- `thin/direct-ended.sh` (ExecStopPost): when the direct VM ends (macOS shut down, the chord, a failure), gdm starts again.
- `macos-gnome.sh`: back to GNOME by hand (clean VM stop, gdm).
- `thin/direct-thin-test.sh [ROOT]`: the thin root's kiosk run from inside a thin root tree on the booted system.
- `thin/dtest.sh [MINS]`: acceptance test (synthetic login retried until a console login, workload, recoveries, frame stats).
- `thin/chord.py`: synthetic Volume Up + Volume Down hold (tests the escape).
- Thin root: `/home/thin/overlay/etc/systemd/system/kiosk.service` is now the direct unit (cage one kept as `.cage`).
  `/home/thin/build-thin.sh` no longer installs cage/seatd/grim/wlr-randr. It copies the piano `multi-user.target.wants`
  links (piano-thp and piano-charging are static units the enabled-list misses), plus the diverted mipps-auth fix.

## Fixed on the way
- **piano-thp (the real touchscreen) had been stopped since 11:03** by the `systemctl isolate` incident; nothing restarted it
  (static unit, udev start only). Restarted, and it is now also wanted by multi-user.target (like piano-charging).
- Linger enabled for leo, so the user's PipeWire (the VM's audio) keeps running with no desktop session.
- oomd drop-in (`90-piano-no-pressure-kill.conf`, from the 3200 work): no memory-pressure kill of the GNOME session.

## Measured (same workload: Safari / Finder / System Settings / apple.com, 1600x1068 guest, verified console login)
| path | GPU recoveries | guest flush mean / max | max frames/s | crosvm v_gpu thread | host MemAvailable |
|---|---|---|---|---|---|
| GNOME + Wayland window (before) | 0 | 26.7 ms / 283 ms | 55 | ~90 % | ~3.0 GB |
| direct KMS, full install | 0 | 15.6 ms / 248 ms | 45-65 | 55-80 % | ~4.4 GB |
| direct KMS, from thin root-3 | 0 | 15.4 ms / 213 ms | 73 | — | — |
| direct KMS, from the new thin root-4 (2 min) | 0 | 6.6 ms / 109 ms | 87 | — | — |
The first direct version (`glFinish` on crosvm's thread, normal priority) measured 24 ms; the fence + priority change brought
it to 15 ms. Audio (PipeWire stream "crosvm-mmio-gpu" into the Speaker sink), charging (piano-charging active, 100 %) and
clean shutdown (guest synced and halted, crosvm rc=0) were all checked in direct mode.

## Not done / next
- Boot chain: booting the thin root (the 99thin picker / signed boot image) still waits for the USB-C cable. Everything here
  runs on the booted full install, or from the thin root tree via `direct-thin-test.sh`.
- Sleep/wake: suspend stays blocked while the VM runs (logind sleep inhibitor). The panel goes dark only with the cover
  (SW_LID → DPMS off). There is no idle timeout: macOS's own display sleep stops new frames, but the last one stays lit.
  Next: DPMS off when the guest blanks, plus the guest sleep protocol.
- The cover's touchpad is not handled in direct mode (in GNOME it moves the pointer). The virtio touch device has no hover,
  so it needs a cursor/relative-pointer mapping.
- Direct plane scanout (zero copy) needs an 8-bit guest scanout format (above); native-display-gpu-research.md rates this
  the next high-value slice.
- Bluetooth: only keyboards already paired in GNOME (they reconnect by themselves); pairing needs GNOME. The thin root gets
  bluez from build-thin.sh (root-5, to be rebuilt).
- DroidVM stays: macOS still drives this panel and Adreno only through DroidVMDisplay + the bridge. Only the host-side
  compositor is gone.

## State left for the user (16:18)
Full install, in **direct mode**: the macOS login window on the panel, with GNOME stopped (gdm inactive). piano-charging and
piano-thp are active, and the charger is online (cells full). Thin root-4 (`/home/thin/thin-4.sqfs`, 403 MB) is built from
the new build-thin.sh, has no compositor, and was tested from the booted system; flashing and booting it waits for the cable.

## Audit fixes (2026-11-10 16:12-17:00; audit: thin-direct-audit.md)
Final crosvm build finished 16:59; the direct VM was restarted on it at 17:00. "Verified" = checked on the tablet; the rest is
implemented and deployed, and is being tested by the coordinator.

`crosvm-kms/display_kms.c`:
- Keys held on a device that disappears, or lost to SYN_DROPPED, are released for the guest. The kernel itself also releases
  held keys when an input device unregisters (**verified** with a disappearing uinput keyboard: no stuck key).
- Touch: the calibration (`LIBINPUT_CALIBRATION_MATRIX`) is re-read at the start of every gesture, so touch is no longer
  mirrored after piano-thp recreates its device; devices are rescanned after any device closes. MT slots beyond 10 are dropped
  instead of merged into slot 9.
- The power key is never grabbed unless `CROSVM_KMS_POWER_KEY=1` (default: logind's clean shutdown).
- Escape: the child is reaped, and the escape is retried if the spawn fails.
- DRM master / SetCrtc are retried. Page-flip failures are rate-limited, and EINVAL triggers a full mode set. A page-flip event
  that never arrives is given up after 1 s.
- Shader `highp`. The calling thread's EGL context (virglrenderer's) is saved and restored instead of released.
- **Lid → DPMS:** closing the cover (SW_LID on gpio-keys) turns the panel off; opening it turns it on again (**verified** with a
  synthetic lid device: panel off/on logged; the first version needed the EINVAL → mode-set fix, now included).

Scripts:
- `thin/direct.sh` is a small supervisor: macOS "Restart" (crosvm exit 32) restarts the VM instead of dropping to GNOME; stop
  signals are forwarded to crosvm; `DIRECT_ON_POWEROFF=poweroff` (thin root) powers the tablet off when macOS shuts down.
  The KMS device is opened by path (`/dev/dri/by-path/platform-ae01000.display-controller-card`), not as card1. `VNC=0` in
  direct mode (the view-only VNC mirror cost 4-5 % CPU).
- `macos-direct.sh`: the `macos-vm` unit is ordered `After=user@1000 network piano-charging`, so at poweroff it stops before
  PipeWire and the network. `direct-ended.sh` does not start gdm while the system is stopping.
- Thin root: `kiosk.service` no longer waits for network-online (no Wi-Fi no longer delays the VM by up to 2 min);
  `DIRECT_ON_POWEROFF=poweroff`; `build-thin.sh` adds bluez (root-5 to be rebuilt).

## Idle measurements (direct, login window, 14 min; thin-direct-measurements.md)
- crosvm ~159 % CPU, mostly guest vCPUs (macOS background work). v_gpu 5.8 % while frames flow, 0.1 % when the guest is quiet.
- GPU ~16 % busy at 160 MHz while drawing, ~0 when quiet.
- USB input ~7.6 W with frames, ~6 W quiet (panel lit).
- MemAvailable 4.1-4.35 GB, no memory pressure. No gnome-shell / mutter / Xwayland / gdm on the host.
- That run still had the VNC mirror (simplefb_display ~4 %); direct.sh now sets VNC=0.
