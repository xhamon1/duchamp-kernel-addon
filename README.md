# Duchamp S1 Kernel

A custom kernel for the **Poco X6 Pro (duchamp, MediaTek MT6897)**. It is built from the same sources as the kernel shipped with the AOSP custom ROMs for duchamp ([mt6897-devs](https://github.com/mt6897-devs)), with **SUSFS**, **NoMount** and a choice of root manager (**KernelSU-Next** or **BakaSU**) added. The kernel release ends with `Duchamp-S1`, so it is easy to tell apart from the stock kernel.

> Personal, unofficial project, shared as is. Flash at your own risk.

## S1 and S2

There are two variants, each in its own repo. They share the same base, and S2 adds a few extra changes on top of S1.

| | S1 | S2 |
|---|---|---|
| Kernel release suffix | `-Duchamp-S1` | `-Duchamp-S2` |
| Base source | mt6897-devs `lineage-23.2` | Same |
| Root manager (selectable) | KernelSU-Next or BakaSU | Same |
| SUSFS and NoMount | Built in | Same |
| TCP congestion control | CUBIC (stock default) | **BBR** (default) |
| Queueing disciplines | Stock | Adds **fq_codel** and **cake** (`fq_codel` was already present in the stock config) |
| rtmutex fix ([CVE-2026-43499](https://nvd.nist.gov/vuln/detail/CVE-2026-43499)) | Not applied | **Applied** |
| Config differences from mt6897-devs | 16 options | 16 options plus the changes above |

- **Choose S1** if you want the smallest set of changes from the ROM kernel.
- **Choose S2** if you also want BBR, the extra qdiscs and the rtmutex security fix. BBR and the qdiscs only affect networking. They do not change CPU or GPU performance or temperature.

The rtmutex bug is fixed upstream in Linux 6.1.175. The mt6897-devs source is 6.1.134, so it is affected. The S2 patch applies the same change to `remove_waiter()` in `kernel/locking/rtmutex.c`. It uses plain lock calls instead of the `scoped_guard` helper from the upstream stable commit. I compared it with the upstream 6.1.175 code; I have not been able to confirm on a device that the bug can no longer be triggered.

## Why this kernel exists

Every AOSP custom ROM for duchamp ships the kernel from `mt6897-devs`. That kernel has no root manager support, no SUSFS and no NoMount.

This project builds the kernel from the same base as the ROM and adds only the needed features, with as few changes as possible and an open build process, so every change can be inspected and reproduced.

## What is in it

| Component | Source |
|---|---|
| Kernel source (GKI 6.1 and MediaTek modules) | `mt6897-devs/kernel_manifest`, branch `lineage-23.2` |
| Root manager (selectable) | [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) (`pershoot` fork, branch `dev-susfs`) or [BakaSU](https://github.com/Baka-SU/BakaSU) (SUSFS inline hook mode) |
| Hiding | [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) (`gki-android14-6.1`) |
| Path redirection | [NoMount](https://github.com/maxsteeel/nomount) (branch `dev`) |
| Flashable package | [AnyKernel3](https://github.com/osm0sis/AnyKernel3) |

Build settings:
- **No LTO** (`--lto=none`). The mt6897-devs kernel is also built without LTO (`CONFIG_LTO_NONE=y`).
- SUSFS logging is off (`CONFIG_KSU_SUSFS_ENABLE_LOG=n`) to avoid extra overhead.
- The Samsung-only KernelSU-Next options (`KSU_SAMSUNG_*`) are off, since this is not a Samsung device.
- Only `Image` is built and flashed. Vendor modules (Wi-Fi, display, touchscreen and so on) keep using the ones already in the ROM.

## Differences from the mt6897-devs kernel

Comparing `/proc/config.gz` of this kernel (KernelSU-Next build) with the mt6897-devs kernel shows **16 differing options out of about 7,000**:

- 14 intended additions: `CONFIG_KSU`, `CONFIG_KSU_MULTI_MANAGER_SUPPORT`, `CONFIG_KSU_FULL_NAME_FORMAT`, the `CONFIG_KSU_SUSFS_*` options and `CONFIG_NOMOUNT`.
- `CONFIG_LOCALVERSION="-Duchamp-S1"` (empty in the mt6897-devs kernel).
- `CONFIG_FRAME_WARN`: 0 here, 2048 in the mt6897-devs kernel. It only sets a compiler warning threshold.
- Every option enabled in the mt6897-devs kernel is also enabled here.

This comparison was made on the S1 config. It is a config comparison, not a binary one. The kernel is built from the same sources and config overlay as the mt6897-devs kernel, but it is not a bit-for-bit copy of the maintainer's prebuilt binary (the compiler, the `kernel_device_modules` commit and the build time can differ). More details:

- Xiaomi and duchamp-specific options come from the device overlay `kernel_device_modules-6.1/kernel/configs/duchamp.config`. The ROM build applies it, and so does this workflow (`defconfig_overlays`). Checked on device: `MI_DISP_LHBM=y`, `MI_DISP_VDO_AOD=y`, `UFSFEATURE=y`, `XIAOMI_FINGERPRINT=m`.
- The GKI source commit matches the mt6897-devs kernel (`g606f36c019fa` in AOSP ROMs on Duchamp). The mt6897-devs kernel has no `-dirty` suffix. This one does because of the added patches. The `kernel_device_modules` commit follows the branch at build time and was not compared.
- With an LKM root alone on the mt6897-devs kernel, the manager shows no SUSFS. SUSFS can be added there with an LKM module; see the comparison below.

Test the related features (fingerprint, AOD, touch, audio, charging) after flashing.

## Compared with the SUSFS LKM module

SUSFS can also be added to the unmodified mt6897-devs kernel without flashing a new kernel, using [susfs-lkm-module](https://github.com/matsuzaka-yuki/susfs-lkm-module/releases/tag/v2.3.0-r18.9-1). It is a KernelSU module that loads a SUSFS loadable kernel module (a port by [inforcqb](https://github.com/inforcqb/susfs4ksu-lkm)) at boot. NoMount can be used on the mt6897-devs kernel in the same way, as an LKM. Both approaches are valid, but they trade different things:

| | This kernel | mt6897-devs kernel + SUSFS LKM module |
|---|---|---|
| How SUSFS is added | Compiled into the kernel (SUSFS v2.4.0 patch set from simonpunk) | Loaded as a `.ko` by a module (`v2.3.0-r18.9-1`, a SUSFS port) |
| Kernel flash | Required | Not required, the kernel stays unmodified |
| Root manager | Built into the kernel (KernelSU-Next or BakaSU) | A KernelSU-compatible LKM root is needed |
| When it is active | From the start of the boot | At the module `post-fs-data` stage. The module offers an optional earlier stage (`early_load`) |
| Compatibility risk | Built from the same GKI source and config, so no runtime ABI matching | The loader relocates symbols and may adjust the module version check. Its own README says this is not proof of ABI compatibility and that vendor differences can still prevent loading |
| SUSFS interface | The original SUSFS kernel interface | `/proc/susfs_*` control nodes. It is a port, so behavior and available features can differ from the original SUSFS |
| Difference from the mt6897-devs kernel | A few config options (see above) | None, it is the same kernel |
| After a ROM update | Reflash this kernel if the update replaces the kernel | Nothing to reflash, the module keeps loading |
| Recovery | Flash the original boot image | Disable the module and reboot |
| Updating | Run the workflow and flash the zip | Update the module zip |

Do not use both at once. The LKM module skips loading when SUSFS already exists, and loading two SUSFS implementations is not supported.

I have not tested the LKM module on this device. Its README lists on-device loading evidence for only one kernel family (android15-6.6), so check its issues and notes for `android14-6.1` before relying on it.

Choose this kernel if you want SUSFS and the root manager built in and active from the start, and you accept flashing a kernel. Choose the LKM route if you want to keep the original kernel untouched.

## Status

- Builds and boots on an AOSP ROM on Duchamp (Android 17, tested on one ROM) with either root manager.
- KernelSU-Next: working (built in). SUSFS v2.4.0 and NoMount v2.3.1 are detected, and the Zygisk Next, LSPosed and Morphe modules run.
- BakaSU (`v4.2.0-rc3`, SUSFS inline hook): working (built in). SUSFS v2.4.0, NoMount and Zygisk Next are detected, and existing modules are still listed and running.
- The config shows the Xiaomi options from the device overlay (`MI_DISP_LHBM`, `UFSFEATURE`) and no LTO.
- S1 is the longer-tested variant.
- Some modules (for example `.BRENE - SUSFS`) can spoof the kernel version shown by `uname`. Check `/proc/config.gz` or `dmesg` to see the real kernel.
- Long-term stability and other hardware functions have not been tested thoroughly.
- The build sandbox has no git history, so KernelSU-Next would report `v0.0.1 (1)`. The workflow sets the version code and tag itself, so the manager shows the same kind of version as with the mt6897-devs kernel (for example `v3.4.2 (33554)`). By default it computes them on every build from the latest official KernelSU-Next `dev` branch (`30000 + commit count`, nearest tag). `ksu_version` and `ksu_tag` override that. These numbers are only labels and can be one commit ahead of your manager build.

## Build

1. Fork or copy this repo.
2. Open **Actions → Build Duchamp S1 Kernel → Run workflow**. The default inputs are correct for duchamp.
3. Pick `root_manager` and wait about 15 to 30 minutes.
4. When the build finishes, a **release** is created with two files:
   - `duchamp-s1-<root_manager>-<date>.zip`: an AnyKernel3 zip, ready to flash from recovery or a kernel flasher app.
   - `*.apk`: the matching root manager (the KernelSU-Next spoofed build, or BakaSU). It comes from the upstream manager CI for the same commit as the kernel source. If that commit has no manager build, the latest build is used and the log shows a warning. If no APK can be fetched, the release contains only the kernel zip.

   The release description lists the exact versions and options of that build. The same files are also kept as a workflow artifact named `duchamp-kernel`.

The workflow needs **Settings → Actions → General → Workflow permissions → Read and write permissions** so that it can create releases.

| Input | Purpose | Default |
|---|---|---|
| `manifest_branch` | Branch of `mt6897-devs/kernel_manifest` | `lineage-23.2` |
| `kernel_dir` | GKI source folder after sync | `kernel-6.1` |
| `root_manager` | `kernelsu-next` or `bakasu`. Selects which root solution is built in | `kernelsu-next` |
| `bakasu_branch`, `bakasu_commit` | [BakaSU](https://github.com/Baka-SU/BakaSU) source (only for `bakasu`; empty commit = latest) | `main`, empty |
| `ksu_repo`, `ksu_branch` | KernelSU-Next source | `pershoot/KernelSU-Next`, `dev-susfs` |
| `ksu_commit` | KernelSU-Next commit, empty = latest | empty |
| `ksu_version`, `ksu_tag` | Version code and tag shown by the manager. Empty = latest official KernelSU-Next `dev` (computed on every build) | empty |
| `susfs_branch`, `susfs_commit` | SUSFS branch and commit | `gki-android14-6.1`, empty |
| `defconfig_overlays` | Device config overlay applied by the ROM build (`kernel_device_modules-6.1/kernel/configs`). It enables the Xiaomi options | `duchamp.config` |
| `localversion` | Suffix appended to the end of the kernel release string. Empty = none | `-Duchamp-S1` |
| `lto` | `none`, `thin`, `full` or `default` | `none` |
| `susfs_log` | Enable SUSFS logging. Off by default; turn it on if the build fails on `SUSFS_LOGI_RL` | `false` |
| `nomount_builtin` | Build NoMount in. If `false`, use the LKM from the NoMount zip | `true` |

The SUSFS patch can fail if the source changes. The workflow then stops and prints the rejected hunks.

## Flash

1. **Keep the original ROM boot image** in case of a bootloop.
2. Flash the zip from recovery or a kernel flasher app, or extract `Image` and flash it manually.
3. Reboot, then install the manager APK from the release and open it. Use the manager that matches the root you built; the other one will show "Not installed".

## Verify

```sh
su -c "zcat /proc/config.gz | grep -E 'NOMOUNT|KSU|LTO|LOCALVERSION'"
su -c "dmesg | grep -m1 'Linux version'"
```

Expected: `CONFIG_NOMOUNT=y`, `CONFIG_KSU=y`, `CONFIG_KSU_SUSFS=y`, `CONFIG_LTO_NONE=y` and `CONFIG_LOCALVERSION="-Duchamp-S1"`. The `dmesg` line shows the real release, ending with `Duchamp-S1` (for example `6.1.134-android14-...-dirty-Duchamp-S1`).

## Repo layout

```
.github/workflows/build.yml   # build workflow
anykernel/anykernel.sh        # AnyKernel3 script (replaces Image only)
```

## Credits

[mt6897-devs](https://github.com/mt6897-devs), [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next), [pershoot](https://github.com/pershoot), [SUSFS (simonpunk)](https://gitlab.com/simonpunk/susfs4ksu), [NoMount (maxsteeel)](https://github.com/maxsteeel/nomount) and [AnyKernel3 (osm0sis)](https://github.com/osm0sis/AnyKernel3).
