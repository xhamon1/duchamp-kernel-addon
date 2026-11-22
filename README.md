# duchamp-kernel-enhanced

A kernel for the **Poco X6 Pro (duchamp, MediaTek MT6897)**, built from the same source as the kernel shipped with the AOSP custom ROMs for duchamp ([mt6897-devs](https://github.com/mt6897-devs)), with **SUSFS**, **NoMount** and a choice of root manager (**KernelSU-Next** or **BakaSU**) added. The kernel release carries an `Enhanced` tag, so it is easy to tell apart from the stock kernel.

> Personal, unofficial project, shared as is. Flash at your own risk.

## Why this kernel exists

Every AOSP custom ROM for duchamp ships the kernel from `mt6897-devs`. That kernel includes no root manager support, no SUSFS and no NoMount.

The goal of this repo is to **build the kernel from the same base as the ROM and add only the needed features, with as few changes as possible and an open build process**, so every change can be inspected and reproduced.

## What is in it

| Component | Source |
|---|---|
| Kernel source (GKI 6.1 + MediaTek modules) | `mt6897-devs/kernel_manifest`, branch `lineage-23.2` |
| Root (selectable) | [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) (`pershoot` fork, branch `dev-susfs`) or [BakaSU](https://github.com/Baka-SU/BakaSU) (SUSFS inline hook mode) |
| Hiding | [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) (`gki-android14-6.1`) |
| Path redirection | [NoMount](https://github.com/maxsteeel/nomount) (branch `dev`) |
| Flashable package | [AnyKernel3](https://github.com/osm0sis/AnyKernel3) |

Build settings:
- **No LTO** (`--lto=none`). The ROM kernel is also `CONFIG_LTO_NONE=y`.
- SUSFS logging is turned off (`CONFIG_KSU_SUSFS_ENABLE_LOG=n`) to avoid extra overhead.
- The Samsung-only KernelSU-Next options (`KSU_SAMSUNG_*`) are turned off, since this is not a Samsung device.
- Only `Image` is built and flashed. Vendor modules (Wi-Fi, display, touchscreen, etc.) keep using the ones already in the ROM.

## Differences from the ROM kernel

This kernel is built from the same sources and config overlay as the ROM kernel, but it is not a bit-for-bit copy of the maintainer's prebuilt binary. Differences:

- Intended additions: `CONFIG_KSU`, the `CONFIG_KSU_SUSFS_*` options and `CONFIG_NOMOUNT`.
- Xiaomi/duchamp-specific options come from the device overlay `kernel_device_modules-6.1/kernel/configs/duchamp.config`, which the ROM build applies and this workflow applies too (`defconfig_overlays`). Checked on device: `MI_DISP_LHBM=y`, `MI_DISP_VDO_AOD=y`, `UFSFEATURE=y`, `XIAOMI_FINGERPRINT=m`.
- The GKI source commit is the same as the ROM kernel (`g606f36c019fa` on VoltageOS 6.3). The ROM kernel shows no `-dirty` suffix, this one does because of the added patches. The `kernel_device_modules` commit follows the branch at build time and was not compared.
- With the ROM kernel and an LKM root, the manager shows no SUSFS. SUSFS is what this kernel adds on top; NoMount also works there as an LKM.

Test the related features (fingerprint, AOD, touch, audio, charging) after flashing.

## Status

- Builds and boots on VoltageOS 6.3 (Android 17) with either root manager.
- KernelSU-Next: working (built-in), SUSFS v2.4.0 and NoMount v2.3.1 detected, Zygisk Next, LSPosed and Morphe modules running.
- BakaSU (`v4.2.0-rc3`, SUSFS inline hook): working (built-in), SUSFS v2.4.0, NoMount and Zygisk Next detected, existing modules still listed and running.
- The config shows the Xiaomi options from the device overlay (`MI_DISP_LHBM`, `UFSFEATURE`) and no LTO.
- Some modules (for example `.BRENE - SUSFS`) can spoof the kernel version shown by `uname`, so check `/proc/config.gz` or `dmesg` to see the real kernel.
- Long-term stability and other hardware functions have not been tested thoroughly.
- The build sandbox has no git history, so KernelSU-Next would report `v0.0.1 (1)`. The workflow sets the version code and tag itself, so the manager shows the same kind of version as with the ROM kernel (for example `v3.4.2 (33554)`). By default it computes them on every build from the latest official KernelSU-Next `dev` (`30000 + commit count`, nearest tag); `ksu_version` and `ksu_tag` override that. These numbers are only labels and can be one commit ahead of your manager build.

## Build

1. Fork or copy this repo.
2. Open **Actions → Build duchamp kernel → Run workflow**. The default inputs are correct for duchamp.
3. Pick `root_manager` and wait about 15-30 minutes.
4. The **Artifacts** section has `duchamp-kernel`, which contains:
   - `duchamp-enhanced-<root_manager>-<date>.zip`: an AnyKernel3 zip for recovery or a kernel flasher app.
   - `manager/*.apk`: the matching root manager (KernelSU-Next spoofed build, or BakaSU). It comes from the upstream manager CI for the same commit as the kernel source. If that commit has no manager build, the latest build is used and the log shows a warning. If no APK can be fetched, the kernel zip is still produced.

| Input | Purpose | Default |
|---|---|---|
| `manifest_branch` | Branch of `mt6897-devs/kernel_manifest` | `lineage-23.2` |
| `kernel_dir` | GKI source folder after sync | `kernel-6.1` |
| `root_manager` | `kernelsu-next` or `bakasu`. Selects which root solution is built in | `kernelsu-next` |
| `bakasu_branch`, `bakasu_commit` | [BakaSU](https://github.com/Baka-SU/BakaSU) source (only for `bakasu`; empty commit = latest) | `main`, empty |
| `ksu_repo`, `ksu_branch` | KernelSU-Next source | `pershoot/KernelSU-Next`, `dev-susfs` |
| `ksu_commit` | KernelSU-Next commit, empty = latest | empty |
| `ksu_version`, `ksu_tag` | Version code and tag the manager displays. Empty = latest official KernelSU-Next `dev` (computed on every build) | empty |
| `susfs_branch`, `susfs_commit` | SUSFS branch and commit | `gki-android14-6.1`, empty |
| `defconfig_overlays` | Device config overlay the ROM build applies (`kernel_device_modules-6.1/kernel/configs`). It enables the Xiaomi options | `duchamp.config` |
| `localversion` | Suffix added to the kernel release, after the version (`6.1.134-Enhanced-android14-...`). Empty = none | `-Enhanced` |
| `lto` | `none`, `thin`, `full` or `default` | `none` |
| `susfs_log` | Enable SUSFS logging. Off by default; turn it on if the build fails on `SUSFS_LOGI_RL` | `false` |
| `nomount_builtin` | Build NoMount in. If `false`, use the LKM from the NoMount zip | `true` |

The SUSFS patch can fail if the source changes. The workflow then stops and prints the rejected hunks.

## Flash

1. **Keep the original ROM boot image** in case of a bootloop.
2. Flash the zip from recovery, or extract `Image` and flash it manually.
3. Reboot, then install the manager APK from the artifact and open it. Use the manager that matches the root you built; the other one will show "Not installed".

## Verify

```sh
su -c "zcat /proc/config.gz | grep -E 'NOMOUNT|KSU|LTO|LOCALVERSION'"
su -c "dmesg | grep -m1 'Linux version'"
```

Expected: `CONFIG_NOMOUNT=y`, `CONFIG_KSU=y`, `CONFIG_KSU_SUSFS=y`, `CONFIG_LTO_NONE=y` and `CONFIG_LOCALVERSION="-Enhanced"`. The `dmesg` line shows the real release, with `Enhanced` after the version.

## Repo layout

```
.github/workflows/build.yml   # build workflow
anykernel/anykernel.sh        # AnyKernel3 script (replaces Image only)
```

## Credits

[mt6897-devs](https://github.com/mt6897-devs), [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next), [pershoot](https://github.com/pershoot), [SUSFS (simonpunk)](https://gitlab.com/simonpunk/susfs4ksu), [NoMount (maxsteeel)](https://github.com/maxsteeel/nomount) and [AnyKernel3 (osm0sis)](https://github.com/osm0sis/AnyKernel3).
