# duchamp-kernel-addon

A kernel for the **Poco X6 Pro (duchamp, MediaTek MT6897)**, built from the same source as the kernel shipped with the AOSP custom ROMs for duchamp ([mt6897-devs](https://github.com/mt6897-devs)), with **KernelSU-Next**, **SUSFS** and **NoMount** added.

> Personal, unofficial project, shared as is. Flash at your own risk.

## Why this kernel exists

Every AOSP custom ROM for duchamp ships the kernel from `mt6897-devs`. That kernel includes no root manager support, no SUSFS and no NoMount.

The goal of this repo is to **build the kernel from the same base as the ROM and add only the needed features, with as few changes as possible and an open build process**, so every change can be inspected and reproduced.

## What is in it

| Component | Source |
|---|---|
| Kernel source (GKI 6.1 + MediaTek modules) | `mt6897-devs/kernel_manifest`, branch `lineage-23.2` |
| Root | [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) (`pershoot` fork, branch `dev-susfs`) |
| Hiding | [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) (`gki-android14-6.1`) |
| Path redirection | [NoMount](https://github.com/maxsteeel/nomount) (branch `dev`) |
| Flashable package | [AnyKernel3](https://github.com/osm0sis/AnyKernel3) |

Build settings:
- **No LTO** (`--lto=none`). The ROM kernel is also `CONFIG_LTO_NONE=y`.
- SUSFS logging is turned off (`CONFIG_KSU_SUSFS_ENABLE_LOG=n`) to avoid extra overhead.
- Only `Image` is built and flashed. Vendor modules (Wi-Fi, display, touchscreen, etc.) keep using the ones already in the ROM.

## Differences from the ROM kernel

This kernel is **not an exact copy** of the ROM kernel. Comparing `/proc/config.gz` shows:

- Intended additions: `CONFIG_KSU`, the `CONFIG_KSU_SUSFS_*` options and `CONFIG_NOMOUNT`.
- Xiaomi/duchamp-specific options that are present in the ROM kernel but missing from this build, because the driver sources are not part of the `mt6897-devs` manifest. Examples: `MI_DISP_LHBM`, `MI_DISP_VDO_AOD`, `XIAOMI_FINGERPRINT`, `TOUCHSCREEN_XIAOMI_TOUCHFEATURE`, `SND_SMARTPA_AW882XX`, `UFSFEATURE` and `MI_THERMAL_*`.
- The source commit follows the branch at build time, so it can be newer than the one the ROM was built from.

Test the related features (fingerprint, AOD, touch, audio, charging) after flashing.

## Status

- Builds and boots on VoltageOS 6.3 (Android 17).
- KernelSU-Next, SUSFS v2.4.0 and NoMount are detected by the manager.
- Long-term stability and other hardware functions have not been tested thoroughly.
- The KernelSU version inside the kernel shows as `v0.0.1`, because git info does not reach the build sandbox. The spoofed manager still accepts it.

## Build

1. Fork or copy this repo.
2. Open **Actions → Build duchamp kernel → Run workflow**. The default inputs are correct for duchamp.
3. Wait about 15-30 minutes. The result is in **Artifacts** as `duchamp-kernel-zip`.

| Input | Purpose | Default |
|---|---|---|
| `manifest_branch` | Branch of `mt6897-devs/kernel_manifest` | `lineage-23.2` |
| `kernel_dir` | GKI source folder after sync | `kernel-6.1` |
| `ksu_repo`, `ksu_branch` | KernelSU-Next source | `pershoot/KernelSU-Next`, `dev-susfs` |
| `ksu_commit` | KernelSU-Next commit, empty = latest | empty |
| `susfs_branch`, `susfs_commit` | SUSFS branch and commit | `gki-android14-6.1`, empty |
| `lto` | `none`, `thin`, `full` or `default` | `none` |
| `nomount_builtin` | Build NoMount in. If `false`, use the LKM from the NoMount zip | `true` |

The SUSFS patch can fail if the source changes. The workflow then stops and prints the rejected hunks.

## Flash

1. **Keep the original ROM boot image** in case of a bootloop.
2. Flash the zip from recovery, or extract `Image` and flash it manually.
3. Reboot, then open the KernelSU-Next manager.

## Verify

```sh
su -c "zcat /proc/config.gz | grep -E 'NOMOUNT|KSU|LTO'"
```

Expected: `CONFIG_NOMOUNT=y`, `CONFIG_KSU=y`, `CONFIG_KSU_SUSFS=y` and `CONFIG_LTO_NONE=y`.

## Repo layout

```
.github/workflows/build.yml   # build workflow
anykernel/anykernel.sh        # AnyKernel3 script (replaces Image only)
```

## Credits

[mt6897-devs](https://github.com/mt6897-devs), [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next), [pershoot](https://github.com/pershoot), [SUSFS (simonpunk)](https://gitlab.com/simonpunk/susfs4ksu), [NoMount (maxsteeel)](https://github.com/maxsteeel/nomount) and [AnyKernel3 (osm0sis)](https://github.com/osm0sis/AnyKernel3).
