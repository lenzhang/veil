# Direct mode (crosvm drives the panel through KMS, no GNOME/Wayland): passive idle measurements, 2026-11-10

All readings were passive. Nothing was started, stopped or injected, and nothing ran in the guest. sudo (password [REDACTED]) was used
only to read `/proc/<crosvm>/fdinfo` and `/sys/kernel/debug/dri/1/{state,clients}`.
VM: run `1110-161638-macos` (started 16:16:38), `--mem 8192 --cpus 6`, `--gpu ... displays=[[mode=windowed[1600,1068]]]`,
`--wayland-sock kms:/dev/dri/card1`, plus `--vnc-server ... screen=simplefb,view-only=true` and `--v4l2-proxy`.
The guest sat at the macOS login window with no user input.
Window: **16:21:51-16:35:53 (842 s, 48 intervals of about 17.5 s)**. The sampler ran a loop over all 99 crosvm threads and a sudo
call each round, so each round took about 17 s, not the 10 s that was planned. The sampler's own cost is included in "host non-crosvm".
Samples after 16:35:53 are left out: from about 16:36 a large host load that was not crosvm appeared (host CPU 491 %,
cpuss up to 101 °C, MemAvailable down to 2.6 GB, while crosvm stayed at 149 %). That was the start of the coordinator's crosvm build,
which ran from about 16:38. Those samples are listed separately at the end.

Panel: crtc-0 mode `3200x2136` at **144 Hz** (pixel clock 1 099 707 kHz, 3306x2310 total). One plane (plane-0, sspp_8,
multirect split into two 1600x2136 halves). The FB is `XR24` 3200x2136 and is allocated by `v_gpu`. crosvm rotates 3 FBs (ids 111/112/114).

## Results

The guest showed two clear phases. Phase A has frames on the panel. In phase B, from about 16:27:50, the guest stopped sending
frames, most likely macOS's display sleep at the login window.

| metric | A: frames (16:21:51-16:27:42, 351 s) | B: guest quiet (16:28:17-16:35:53, 456 s) | whole window (842 s) |
|---|---|---|---|
| crosvm total CPU (100 % = 1 core) | **169 %** | **152 %** | 159 % (per 17 s: 149-180) |
| vCPU0..5 | 6.6 / 4.3 / 57.3 / 53.7 / 25.9 / 11.0 | 6.1 / 3.1 / 56.9 / 52.8 / 20.9 / 6.1 | 6.3 / 3.7 / 57.1 / 53.2 / 23.0 / 8.2 |
| v_gpu | 5.8 % | 0.1 % | 2.5 % |
| simplefb_display (VNC/simplefb path) | 3.6 % | 4.7 % | 4.2 % |
| v_snd_common | 0.9 % | 1.3 % | 1.1 % |
| v_media_worker, v_input, vnc_h264_drain, virtio_blk, v_net, msm-sync (48) | < 0.1 % each | < 0.1 % each | < 0.1 % each |
| host CPU busy (of 800 %) | 225 % | 206 % | 214 % (27 % of 8 CPUs) |
| host CPU not used by crosvm (kworkers, gpu-worker, sshd, sampler) | 55 % | 54 % | 55 % |
| load average (1 min) | 2.74 | 2.27 | 2.48 (1.86-3.71) |
| GPU busy: sum of crosvm `drm-engine-gpu` | **15.7 %** | **0.14 %** | 6.7 % |
| GPU devfreq cur_freq (samples) | 160 MHz 19 times, 832 once, 1100 once | 160 MHz 26 of 26 | - |
| mdss (DPU) IRQ/s, a vblank/flip proxy | **136.9/s** (about 144 Hz vblank on) | **7.8/s** | 62.6/s |
| gpu-irq/s | 96.4/s | 0.9/s | 41.2/s |
| USB input (xiaomi-mca-usb), 8.90 V | 850 mA (541-1153), **7.6 W** | 668 mA (406-894), **5.95 W** | 746 mA, 6.6 W |
| battery cells (bq27z561-0/-1) | 98 %, "Not charging", about 0 mA | same | input ≈ whole-system draw incl. backlight |
| cpuss-0-0 / cpuss-1-0 (°C) | 67.8 / 55.9 | 61.2 / 50.3 | 64.1 / 52.8 |
| gpuss4 / ddr (°C) | 54.5 / 51.5 | 48.9 / 46.4 | 51.4 / 48.7 |
| skin-msm / skin-quiet / battery / display (°C) | 49.1 / 32.7 / 36.0 / 31.0 | 44.8 / 31.4 / 34.3 / 30.1 | 46.8 / 32.0 / 35.1 / 30.5 |
| MemAvailable | 4.15-4.35 GB | 4.10-4.18 GB | 4.1-4.35 GB of 15.1 GB |
| swap (zram, 7.4 GB) | 216 MB used | 216 MB | flat |
| PSI some avg10 cpu / memory / io | 2.7 / 0.0 / ≤0.17 | 2.2 / 0.0 / ≤0.4 | memory pressure 0 the whole time |

GPU devfreq residency from 16:21:51 to 16:47:55 (trans_stat delta; this includes the build period): the GPU was powered only about 433 s
of the 1564 s (it is runtime-suspended the rest of the time). Of the powered time: 78 % at 160 MHz, 16 % at 342 MHz, 2.8 % at 1100 MHz.

DRM clients: all 7 open DRM files belong to crosvm. It is the master on card1 (KMS, 84 MiB of scanout BOs) and has 6 render-node
contexts. The main virgl context used 125 MiB, and a second context used 257 MiB of shared BOs.

## Host footprint ("thin")
- No compositor: no gnome-shell, mutter, Xwayland or gdm (gdm `inactive`). The only desktop-related user processes left are
  PipeWire and WirePlumber, kept for VM audio.
- 275 tasks in total, 68 of them userland. Biggest RSS after crosvm: systemd-journald 66 MB, mihomo 53 MB, snapd 36 MB,
  wireplumber 28 MB, tuned 28 MB, dockerd 25 MB, containerd 22 MB, udisksd 21 MB, polkitd 20 MB,
  snapd-desktop-integration 20+11 MB, NetworkManager 20 MB, pipewire 18 MB. Every one of them is at about 0 % CPU.
- Running services include several that are not needed for a tablet VM host: docker/containerd, snapd (+ desktop integration), cups/cups-browsed,
  ModemManager, avahi, unattended-upgrades, colord, accounts-daemon, udisks2. These are left over from the full install. The
  thin root would drop them.
- crosvm RSS reads 16.1 GB because the 8 GiB guest shmem is mapped twice (see PROGRESS-thin-macos.md). The real cost is about
  8.5 GB plus overhead.

## Comparison with GNOME mode (from thin-direct-report.md; that was a Safari/Finder workload, not idle)
| path | v_gpu thread | host MemAvailable |
|---|---|---|
| GNOME + Wayland window (before) | ~90 % | ~3.0 GB |
| direct KMS, full install, active workload | 55-80 % | ~4.4 GB |
| **direct KMS, idle login window (this run)** | **5.8 % (frames) / 0.1 % (quiet)** | **4.1-4.35 GB** |
No idle GNOME-mode numbers (CPU, power, temperatures) exist in the tree, and modes were not switched for this run. The MemAvailable
gain of about 1.2-1.4 GB is consistent with the GNOME session being gone.

## Interpretation
- The display path is cheap at idle. While the guest sends frames, v_gpu is under 6 % and the GPU is about 16 % busy at 160 MHz.
  When the guest goes quiet, both drop to about 0 (v_gpu 0.1 %, GPU 0.14 %, the GPU is mostly runtime-suspended), and DPU IRQs fall from about 137/s
  (vblank on at 144 Hz) to about 8/s. So crosvm does not redraw on its own when nothing changes. It only flips when the guest flushes.
- Most of crosvm's idle CPU, about 150 %, is in the **guest vCPUs**, mainly vCPU2 and vCPU3 (about 55 % each) and vCPU4 (about 21-26 %).
  This happens even with the display asleep, so the cause is in the guest: background work about 5-20 min after boot (Spotlight/mds etc.),
  or idle that does not reach WFI cheaply. The host display stack is not the cause. This is the largest idle cost and the next thing to look at,
  inside the guest at a later session (not done here).
- The `simplefb_display` thread (the view-only VNC/simplefb mirror) costs a steady **4-5 %** in both phases, more than v_gpu
  when idle. Dropping `--vnc-server`/`--simplefb` in direct mode would save that.
- Power: about 6.0 W from USB with the guest quiet and about 7.6 W with frames, panel lit (the last frame stays lit; there is no DPMS
  yet). The cells sit at 98 % "Not charging" (piano-charging is holding them).
  Temperatures are moderate: CPU clusters 50-68 °C, skin-quiet about 32 °C. Memory pressure is zero and swap is flat.

## Excluded samples (external load starting, the coordinator's crosvm build)
| time | host CPU | crosvm | cpuss-0-0 | MemAvailable | USB current |
|---|---|---|---|---|---|
| 16:36:12 | 218 % | 153 % | 85 °C | 3.52 GB | 1172 mA |
| 16:36:28 | 491 % | 149 % | 100 °C | 2.64 GB | 1356 mA |
Raw data: `/tmp/thinmeas/samples.txt` (with `s2.txt` for the first 50 samples), parsed with `an.py` and `ph.py` (on the Mac, temporary).
