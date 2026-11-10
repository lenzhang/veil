# veil

**FROZEN / ARCHIVED as of 2026-11-10.**

`veil` was the thin Linux + Gunyah host path for running a macOS guest (crosvm, direct KMS scanout) on the Xiaomi Pad 8 Pro (`piano`). That route is frozen. The **active main line** is native macOS at EL1 in [lenzhang/UniXNU-piano](https://github.com/lenzhang/UniXNU-piano).

This GitHub repository is a **curated public archive**: patches, design notes, and the switch-back document, with home-LAN addresses, machine hostnames, backup paths, and credentials removed. It is not a complete drop-in of the local worktree.

The full private snapshot remains on the development Mac in `droidvm-build` on branch `archive/gunyah-guest` (local commit `532b4ce`). That branch is not published: it contains operational host inventory and large or restricted artifacts. Do not push it here.

Project-wide conventions: [UniXNU-piano/docs/PROJECT-MAP.md](https://github.com/lenzhang/UniXNU-piano/blob/main/docs/PROJECT-MAP.md) (a copy is in [docs/PROJECT-MAP.md](docs/PROJECT-MAP.md)).

## 中文摘要

`veil` 是已于 2026-11-10 冻结的精简 Linux + Gunyah 客户机路线的公开归档。当前主线是 [`UniXNU-piano`](https://github.com/lenzhang/UniXNU-piano) 的 EL1 原生启动。本仓库只包含清洗过的补丁和说明；完整本地归档仍在本机 `droidvm-build` 的 `archive/gunyah-guest` 分支，请不要把那条分支推到 GitHub。

## What this archive contains

| Path | Contents |
| --- | --- |
| `docs/gunyah-guest-SWITCHBACK.md` | How the frozen guest path worked and how to switch back (scrubbed) |
| `docs/thin-direct-*.md` | Direct-mode acceptance, report, and measurements (the daily-driver guest path) |
| `patches/` | crosvm, virglrenderer, virtio-media, XNU guest, and board patches that belonged to this route |
| `archive/nested/` | HEAD / branch / status snapshots of the nested checkouts as of the freeze (no full trees) |

## What is not here (on purpose)

- Boot / OS images, partition dumps, screenshots, `out-*`, roots, IPSW-derived kernel collections
- Signing keys, Wi-Fi PSKs, password hashes, and any credential file
- Host restore runbooks that list private backup locations
- Operational scripts that targeted a private LAN (those stay in the local `droidvm-build` snapshot)
- Build logs with machine-local paths

## Hard rules

Even for archive fixes: RAM boot only if the tablet is involved; never flash or erase partitions; never `systemctl isolate`; never put secrets or vendor/Apple payloads in this repository. Do not add new Gunyah-guest features here — the route is frozen.
