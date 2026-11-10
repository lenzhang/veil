# Gunyah guest/VM path: frozen state and how to switch back (2026-11-10)

The macOS-in-a-Gunyah-VM path (crosvm on the Ubuntu host, macOS 27.0.1 guest, direct KMS scanout) is **frozen** as of
2026-11-10 evening. The main line is now native EL1 boot (`el1-native-plan.md`). This file is enough to come back to
the VM path without rediscovering it. It points at the existing reports instead of repeating them.

No password appears in this file. Several tablet scripts (`run-macos.sh`, `macos-vm-stop.sh`, `rig/gssh.sh`, the
stage scripts) embed the tablet or guest credentials in plain text; the thin root images carry `boot-signing.pem`,
Wi-Fi PSKs and a password hash (`thin-direct-audit.md` §3 item 6). Keep all of that on the tablet and the private build host.
Scrub before anything is published.

---

## 1. What works today (state left on the tablet, 2026-11-10 ~18:00)

- **Direct mode is the daily driver:** macOS 27.0.1 full screen on the panel, no GNOME, no Wayland. crosvm is DRM master
  of the DSI panel and reads touch/keyboard/volume keys from evdev. Guest 1600x1068 (shown 2x on the 3200x2136 panel),
  8 GiB, 6 vCPUs (EEPPPP), virtio-gpu msm native context (Metal through the bridge), virtio-net (NAT), virtio-snd
  (PipeWire), virtio-media (Iris H.264/HEVC hardware decode), clean power-off and restart from the Apple menu.
- **Start:** app grid → **"macOS (direct, no desktop)"** (`~/vm/macos-direct.sh` via `macos-direct.desktop`: stops gdm,
  never `isolate`, makes sure `piano-charging`, `piano-thp` and `user@1000` run, starts unit `macos-vm` running
  `~/vm/thin/direct.sh`). GNOME alternative: app grid "macOS VM" (`~/vm/macos-vm-launch.sh`, crosvm Wayland window,
  fullscreen, touch works).
- **Way back to GNOME, A:** hold **Volume Up + Volume Down for 2 s** (crosvm `CROSVM_KMS_ESCAPE`): macOS shuts down
  cleanly, gdm comes back by itself in ~30 s (`direct-ended.sh`). **B:** Apple menu → Shut Down → GNOME returns.
  **By hand:** `~/vm/macos-gnome.sh`. Short power-button press = clean shutdown of the whole tablet (macOS first).
- **Never `systemctl isolate` on this tablet.** It stops `piano-charging.service` (battery drained at ~-1.8 A/cell for
  8 min on 11-10) and `piano-thp.service` (touchscreen dead until restarted). Both are now wanted by
  `multi-user.target`; check with `systemctl is-active piano-charging piano-thp` after any mode change.
- **Fast charging** works with the official Xiaomi adapter (MiPPS auth fixed: `/usr/libexec/xiaomi-mipps-auth` diverted,
  `dpkg-divert --list` shows it; diff: `xiaomi-mipps-auth-dfp-before-svid.diff`).
- Home acceptance checklist (12 steps, no passwords): `thin-direct-ACCEPTANCE.md`.

Measured in direct mode (`thin-direct-report.md`, `thin-direct-measurements.md`): 0 GPU recoveries, guest flush mean
4–16 ms, idle crosvm ~159 % CPU (guest vCPUs), v_gpu 0.1–5.8 %, MemAvailable 4.1–4.35 GB, ~6–7.6 W from USB at the login
window. 2-hour soak 13/13 (`soak-2026-11-09.log`).

---

## 2. Where everything is

### Tablet (`the tablet host`, `~/vm/`)
| Path | What |
|---|---|
| `bin/crosvm-mmio-gpu` | the crosvm in use (KMS backend, wl_touch, idle inhibit, media). Backups: `.pre-kms`, `.pre-touch`, `.pre-idle` |
| `bin/crosvm-ghk`, `bin/crosvm-ghk.abi3` | kernel-first anchor variants (stage 3/4), not used by the macOS path |
| `bin/edk2-gunyah-cam-novgpu.fd` | guest UEFI (edk2-gunyah, `VIRTIO_GPU_ENABLE=FALSE`, `PCI_CAM_MODE=FALSE`) |
| `opt/virgl` | virglrenderer with the msm native-context patches |
| `kmod/{gunyah_qcom,gunyah_share,udmabuf_gki,msm_import_coherent,ghk}` | out-of-tree modules (ghk = kernel-first only) |
| `images/macos27/{boot.img,os.img,systemos.img,data-store.img,boot.env}` | boot disk (DarwinBoot + KC + ADT + Data seed), OS and SystemOS containers, Data store; `boot.env` records MEM/CPUS/SCREEN/MEDIA/NET/SND and feeds `run-macos.sh` defaults |
| `run-macos.sh` | the launcher (modules, hangcheck 10 s, vm watermarks, compaction, tap/dnsmasq/NAT, crosvm command line). `WL=kms:<card>` selects the KMS backend |
| `macos-direct.sh`, `macos-gnome.sh`, `thin/direct.sh`, `thin/direct-ended.sh` | direct mode in/out, supervisor (restart on guest reset, `DIRECT_ON_POWEROFF`) |
| `macos-vm-launch.sh`, `macos-vm-stop.sh`, `restart-vm-piano.sh` (Mac) | GNOME launcher, clean stop (ssh `shutdown -h`, waits for PSCI off), remote restart |
| `thin/` | thin-root tooling: `direct-thin-test.sh`, `dtest.sh`, `chord.py`, `uinput-type.py`, `kiosk*.sh` (cage era) |
| `src/crosvm-mmio` (c9e24ed + ~1.1k lines uncommitted + the two KMS files), `src/crosvm-ghk`, `src/virglrenderer`, `src/virtio-media`, `src/v4l2r`, `src/edk2-gunyah`, `src/minijail` | sources as built |
| `runs/<ts>-macos/{cmdline.txt,stderr.log,generated.dtb,guest.log,info.txt}` | one directory per VM run; `kms:` lines in stderr.log |
| `logs/direct.log`, `journalctl -u macos-vm` | direct-mode logs |
| `rig/` | host-side guest debugging: `gstacks.py`, `gdmesg.py`, `glog.sh`, `gssh.sh`, `soak.sh`, `asidtest`, `vcpu-chaos.sh`, `vtcap.py` |
| `/home/thin/` | thin root builds: `build-thin.sh`, `root-{2..5}`, `thin-{2..5}.sqfs` (root-5 built 17:46, bluez, never booted), `dracut/99thin` picker (not installed), repacked piano debs |
| `~/piano-kbuild/{linux,out/kernel-usb1}` | the exact kernel tree/out of 7.2.15-piano58usb1 (module builds) |

### Mac (`~/Downloads/XiaomiPad8Pro-Ubuntu/droidvm-build/`)
| Path | What |
|---|---|
| `piano-patches/` | every patch, script backup and report (this directory). Key trees: `crosvm-kms/` (`display_kms.c`, `gpu_display_kms.rs`, `apply.py`), `kernel-first/` (ghk ABI 1–3, anchor patches, tiny guest, live dmesg), `thin/`, `thin-direct/`, `netseed/` |
| `macOS-DroidVM/` | k27 kernel (`patches/xnu-12377/0001–0052`, `build-k27-piano.sh`), KC tooling (`k27/`), `boot-disk-piano.sh`, `iterate-k27-piano.sh`, `restart-vm-piano.sh`, overlays (`overlay/kext/DroidVM*`, `overlay/metal/NativeAdrenoMetalPlugin.mm`) |
| `UniXNU-turnip/` | Metal bridge frontend/bridge, Turnip build for macOS |
| `PROGRESS-thin-macos.md` | the running log of 2026-11-10 (all decisions, times, incidents) |
| `RESTORE-20261110.md`, `piano-backup-20261110/` | restore procedure and the boot-critical backup subset |

### The private build host
`~/droidvm/macOS-DroidVM` (boot-disk and stage scripts, `stage-piano.sh`, `stage-gpu-plugin.py`), `~/droidvm/work/macos27/`
(IPSW members, symbols/dyld cache for `gsym.py`), `~/kfirst/` (kernel-first build tree, tiny guest, ghk), `~/fdtools`
(freedreno decoders), `~/piano-backup-20261110/` (full backup, 135 partitions verified).

### Reports (read these, do not rewrite them)
`README-macos.md` (operating manual, load-bearing settings), `thin-direct-report.md`, `thin-direct-audit.md`,
`thin-direct-measurements.md`, `thin-direct-ACCEPTANCE.md`, `crosvm-source-review.md`, `kernel-first-vmm-plan.md`,
`kernel-first-stage0.md`, `kernel-first-stage3-anchor.md`, `kernel-first-stage4.md`, `native-display-gpu-research.md`,
`native-display-8bit-scanout-design.md`, `streamB-input-report.md`, `streamC-charging-report.md`,
`streamC2-auth-report.md`, `../PROGRESS-thin-macos.md`.

---

## 3. Load-bearing pieces (do not undo when returning)

From `README-macos.md` and the memory notes:
- vCPUs pinned 1:1 (`--cpu-affinity 0=0:1=1:2=2:3=3:4=6:5=7`): Gunyah does not flush a pCPU's TLB for a migrating vCPU;
  k27 patch 0050 (16-bit ASIDs) makes the guest safe anyway.
- Pseudo-unprotected VM with the shim; `--hugepages --prepare-lend-mthp-mode chunked`; shmem THP `advise`; host
  compaction before start (13 GiB needs 2 MiB chunks). Unprotected VMs fail at START; 4 KiB LEND hard-hangs the host.
- Board identity `VirtualMac2,1` (vma2); E cores in the ADT (EEPPPP) or login hangs (AMP scheduler).
- Disabled in the guest: fskitd, AppSSODaemon, cryptexd, Spotlight, msrpc; apsd and audiomxd enabled.
- `msm_import_coherent` kretprobe, virgl guest queues pinned to host ring 0, hangcheck 10 s, `vm.watermark_scale_factor=200`.
- Linger for leo (PipeWire without a session), leo in group `audio`, WirePlumber `support.logind=disabled`.
- oomd drop-in `90-piano-no-pressure-kill.conf`; mutter `check-alive-timeout` 60 s.
- `piano-charging` and `piano-thp` wanted by `multi-user.target`; the mipps-auth divert.

---

## 4. Kernel-first stages 0–4 (status, 2026-11-10 17:04–17:52, all on real Gunyah with the macOS VM stopped)

| Stage | Result |
|---|---|
| 0 tiny arm64 Linux guest on stock crosvm | PASS: serial, virtio-mmio net + blk, clean PSCI power-off |
| 1 vhost-net | PASS on stock crosvm (`vhost-<pid>` worker carried traffic). Flag is a bare `vhost-net`, not `vhost-net=true` |
| 2 `ghk.ko` skeleton (ABI 1) | PASS: loads/unloads, refuses non-VM fds |
| 3 crosvm-ghk anchor, vcpu0 in a ghk kthread, in-kernel 16550 (ABI 2) | PASS (`kernel-first/stage3-live-dmesg.txt`); fix: Gunyah `try_clone_descriptor` = dup |
| 4 unhandled MMIO forwarded from ghk to crosvm's Bus (ABI 3) | PASS: virtio blk + net visible, md5 and ping OK (`kernel-first/stage4-live-dmesg.txt`) |
| vhost-net **under ghk** | NOT RUN: the script had the `vhost-net=true` typo; fixed in both copies, rerun pending |
| 5–6 (macOS devices moved one by one; thin backend process) | not started |

Go/no-go criteria and the fallback thin-VMM design are in `kernel-first-vmm-plan.md` §3; the "what a thinner VMM still
needs" estimate (15–20 kLOC) is in `crosvm-source-review.md` §6.

---

## 5. Known gaps when frozen

- Cover **touchpad** does nothing in direct mode (no hover on the virtio touch device; needs a relative-pointer mapping).
- **No auto-blank / idle timeout**: the panel only turns off with the cover (SW_LID → DPMS); the guest's display sleep
  leaves the last frame lit. Sleep/suspend is blocked while the VM runs (logind inhibitor).
- **Native 3200x2136** renders but a real GPU hang (rb 0, >10 s) hits ~1–2 min after login (3/3); with `sheng-diag=1` 0
  hangs but ~13 fps. Default stays 1600x1068. Test harness: `hang3200.sh`.
- **Power sync not done**: no guest-blank → DPMS, no guest sleep protocol, no battery/charger state in the guest.
- **Thin root not bootable yet**: baking `99thin` into the initramfs and the signed boot image needs the USB-C cable;
  root-5 is built, never booted. Direct mode runs from the full Ubuntu install.
- 8-bit guest scanout / direct DPU plane (zero copy): designed, not built (`native-display-8bit-scanout-design.md`).
- Bluetooth pairing needs GNOME; already-paired keyboards reconnect in direct mode.
- Clipboard sharing, rotation (landscape only), VM autostart at login: not done.
- Guest idle CPU ~150 % (macOS background work on vCPUs 2–4): not investigated.

---

## 6. How to re-enable / rebuild the path

1. **Just run it:** boot Ubuntu, app grid → "macOS (direct, no desktop)". If the tablet shows GNOME, that is the fallback
   state; nothing else is needed. Check `systemctl is-active piano-charging piano-thp macos-vm`.
2. **Rebuild the boot disk** (Mac): `bash macOS-DroidVM/boot-disk-piano.sh` (env `MEM=8192 CPUS=6 SCREEN=1600x1068
   MEDIA=1 NET=1 SND=1`; `SCREEN=3200x2136 ... sheng-diag=1` for native). Writes `boot.env` next to `boot.img`.
3. **Rebuild the kernel + KC** (Mac): `macOS-DroidVM/build-k27-piano.sh` then `iterate-k27-piano.sh --no-boot`
   (KC only) or without `--no-boot` (KC + disk + hard restart of the VM). Recipe pitfalls: memory note
   `piano-macos-k27-build` (KDK without Apple ID, ld-classic, Tightbeam-less KDK, gnu-sed).
4. **Rebuild crosvm** (tablet): if `~/vm/src/crosvm-mmio` is fresh, `python3 crosvm-kms/apply.py ~/vm/src/crosvm-mmio`
   plus the `.py` patchers in this directory (`crosvm-rgha.py`, `virgl-fd-high.py`, `idle-inhibit.py`, the wl patches),
   then `~/vm/build-crosvm-gpu.sh` (nice/ionice, keep `-j2` while a VM runs). Compare with `crosvm-source-review.md` §1.
5. **Rebuild virglrenderer** (tablet): `msm-guest-alloc.py`, `msm-submit-all.py`, `msm-pipe.py`, `msm-iova-evict.py`,
   `msm-dump.py` on `~/vm/src/virglrenderer`, install to `~/vm/opt/virgl`.
6. **Restage the Data seed** (the private build host, sudo): `~/droidvm/stage-piano.sh` (12 GiB, netseed plists, time zone,
   disabled jobs, Metal plugin), then rebuild the boot disk.
7. **Stop cleanly:** `sudo systemctl stop macos-vm` or `~/vm/macos-vm-stop.sh` (guest halts, PSCI off, crosvm rc 0).
   Never `pkill -f` by command-line match, never two VMs at once, never start while `gunyah_share` still tracks parcels
   (`run-macos.sh` waits for `outstanding total=0`).
8. **If the VM breaks host state:** a reboot returns the stock stack (modules are loaded per launch). A hard host hang is
   ended by the ~30 s Gunyah watchdog; the tablet comes back to Ubuntu.

---

## 7. Switch-back procedure for a future session

1. Read `PROGRESS-thin-macos.md` from "2026-11-10 15:01" to the end, then `thin-direct-report.md` and `README-macos.md`.
2. Confirm the tablet still has `~/vm` intact (`sha256sum ~/vm/bin/crosvm-mmio-gpu`, `ls ~/vm/images/macos27`,
   `cat ~/vm/images/macos27/boot.env`). If the native work repartitioned LUN0, `/home` (linux_data) may have shrunk;
   `~/vm` lives there, so check free space before rebuilding.
3. Start direct mode (section 6.1). Run the 12-step acceptance (`thin-direct-ACCEPTANCE.md`).
4. Pick up the frozen next steps in this order: vhost-net under ghk rerun → 8-bit scanout T1 (guest BGRA on today's
   blit) → DPMS on guest blank → touchpad → thin root boot (cable) → native 3200 hang.
