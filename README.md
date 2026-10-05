# CachyOS Kernel for Surface Devices

This repository includes the files needed to build an optimized CachyOS kernel including custom patches for Microsoft Surface devices. The patches are based on the work from the linux-surface repository. The patches have been updated to ensure compatibility with CachyOS's patches.

This is a homebrew effort. CachyOS no longer ships a Surface kernel, and the official [linux-surface/linux-surface](https://github.com/linux-surface/linux-surface) project hasn't released patches for any 7.x kernel (its newest tag is `arch-6.19.8-3`). So this repo builds on community ports of the patches that haven't been merged upstream yet. There are two branches:

- **`7.2`** (this branch): CachyOS 7.2.9, with the community 7.2 patch set adapted for 7.2.9 and fixes for the Surface Pro 9 cameras. It builds and is verified as described below, but hasn't been tested on hardware yet.
- **`7.1`**: CachyOS 7.1.3 with the 7.1 patch set. It's tested daily on a Surface Pro 9 (see [Tested hardware](#tested-hardware-surface-pro-9-intel)) and is the fallback if 7.2 misbehaves. Linux 7.1 itself is end-of-life upstream.

- `linux-cachyos-surface` builds against the pre-patched kernel tarball published by CachyOS at [CachyOS/linux/releases](https://github.com/CachyOS/linux/releases) (e.g. `cachyos-7.2.9-2.tar.gz`). This tarball is vanilla Linux with the CachyOS base patches (BBR3, cachy tweaks, fixes, ntsync, t2, zstd, amd-cache-optimizer, …) already applied. It replaces the `0001-cachyos-base-all.patch` meta-patch, which was removed for 6.18+. It does not include the BORE scheduler, so the default `cachyos` scheduler setting gives EEVDF, just like upstream `linux-cachyos`. `_tagrel` selects the CachyOS packaging revision, independently of this PKGBUILD's `pkgrel`. The Surface patches and the `surface-${_major}.config` fragment come from a single pinned commit, set by `_surface_repo` and `_surface_commit`. On this branch that's [`Apiznel/linux-surface@6bcf30a`](https://github.com/Apiznel/linux-surface/commit/6bcf30a1f5387768f2a5068c7b4330a4f9eb3841), the source branch of upstream PR [linux-surface/linux-surface#2233](https://github.com/linux-surface/linux-surface/pull/2233), rebased for 7.2.4. You can drop extra fixes into the package directory: a file named `override-<surface patch name>` replaces that Surface patch, and `local-*.patch` files are applied after the Surface patches. Both are applied with zero fuzz, so a changed upstream context fails the build instead of silently mis-applying.
- `linux-cachyos-surface-lts` deliberately stays on a real kernel.org LTS line (currently Linux 6.12.x, supported by upstream LTS through approximately December 2026). It uses the stock kernel.org tarball plus the still-present `${_major}/all/0001-cachyos-base-all.patch` meta-patch from `cachyos/kernel-patches` — that meta-patch was retired for 6.18+ but remains available for 6.12. The PKGBUILD includes a commented-out template for the pre-baked tarball switch (along the lines used by the mainline variant) for the future kernel bump past 6.12. Its `_surface_ref` defaults to `master` because upstream's `arch_lts-*` tag series stopped at 4.19. Note: upstream `CachyOS/linux-cachyos`'s own `linux-cachyos-lts` variant has redefined "lts" to mean "previous stable cachyos kernel"; this repo intentionally keeps the original kernel.org-LTS meaning so Surface owners get the longest maintenance window per major bump.

To bump the mainline kernel:

1. Pick a release from [CachyOS/linux/releases](https://github.com/CachyOS/linux/releases) and set `_major`, `_minor` and `_tagrel`.
2. Point `_surface_commit` at a linux-surface commit that has `patches/${_major}/` and `configs/surface-${_major}.config` for that kernel series. Prefer one rebased for the same or a nearby point release; mismatched patch sets can break Surface hardware (touchscreen, Type Cover, sensors).
3. Run `makepkg -o` and read the `prepare()` output. Fuzz or rejects mean the patch set needs refreshing.
4. Check for duplicate quirk values (`MT_QUIRK_*`, `MT_CLS_*`, `BTUSB_*`). A clean `patch` run doesn't catch them, and a collision silently mixes up two devices' quirks.

Kernel options the base `config` doesn't set get their Kconfig defaults through `make olddefconfig`, so the build never stops to ask about new options.

_**NOTE:** The configuration files and prebuilt kernels are optimized for X86_64_v3 instruction sets, this should be fine for most Surface devices, but might not work on very old (1st or 2nd gen) devices._

## The 7.2 branch: what changed and how it was verified

Compared with the tested `7.1` build:

- **Base:** CachyOS `cachyos-7.2.9-2` and CachyOS's own 7.2 kernel config ([CachyOS/linux-cachyos@b11ba14](https://github.com/CachyOS/linux-cachyos/blob/b11ba14854d9748bdb3e3daf7a90e8d3a31004ec/linux-cachyos/config)). Options that are new in 7.2 now get CachyOS's settings instead of plain defaults; for example, BBR3 TCP congestion control was off in the 7.1 build and is now built.
- **Surface patches:** `Apiznel/linux-surface@6bcf30a` (PR #2233; touch, pen, Type Cover, SAM and cameras, as in 7.1), with three patches replaced by local `override-*` copies:
  - `override-0010-surface-typecover.patch` and `override-0015-hid-surface.patch`: stable 7.2.9 started using `BIT(26)` and `0x0117` in `hid-multitouch.c` for its own quirk and device class, the same values the Type Cover patches use. The patches now use `BIT(27)`, `BIT(28)` and `BIT(29)`, and classes `0x0118` and `0x0119`. The unmodified patches fail to apply on 7.2.9.
  - `override-0003-mwifiex.patch`: context refresh for `btusb.c`, so the patch applies without fuzz. No functional change.
- **Surface Pro 9 fixes** (`local-*` patches):
  - `local-0100-int3472-sp9-uninitialized-ret.patch`: the experimental "Add camera support for Surface Pro 9" patch leaves `ret` uninitialized for camera GPIO type `0x11`, which can make camera probing fail at random. This sets it.
  - `local-0101-ov13858-sp9-rear-rotation-flip.patch`: upstream PR [linux-surface/linux-surface#2227](https://github.com/linux-surface/linux-surface/pull/2227), ported to 7.2.9. The rear camera is reported as rotated 180°, and the sensor gets the flip controls libcamera needs to turn the picture the right way up. This fixes the upside-down rear camera.
  - `local-0102-ov5693-ipu6-mipi-ctrl00.patch`: upstream PR [linux-surface/linux-surface#2171](https://github.com/linux-surface/linux-surface/pull/2171), reduced to setting only bit 5 of `MIPI_CTRL00` (confirmed sufficient on a Surface Pro 9). This should let the front camera stream.

How it was verified (in a build container, not on the device):

- `prepare()` was run from this PKGBUILD on a fresh `cachyos-7.2.9-2` tree. Every patch applies, with no fuzz and no rejects, and there are no config prompts.
- On the patched tree, the `MT_QUIRK_*`, `MT_CLS_*`, `BTUSB_*` and int3472 GPIO type values have no duplicates.
- All the drivers the patches touch compile with clang (147 objects across `hid`, `ithc`, `ipts`, `platform/surface`, `int3472`, `ipu-bridge`, `ov5693`, `ov13858`, `mwifiex`, `btusb`, PCI quirks and more), and the touched drivers have no warnings under `W=1`.

Things to check on the device after installing it:

- the same checks as the 7.1 table below;
- the front camera (`cam -c 1 --capture=10`, and look for `stream stop time out` in the kernel log);
- the rear camera orientation;
- the microphone.

For the kernel log, use `journalctl -k -b --no-hostname` rather than `dmesg`; on this machine the `dmesg` buffer gets overwritten by firewall log lines. Keep the 7.1 kernel installed as a fallback.

## Tested hardware: Surface Pro 9 (Intel)

The `linux-cachyos-surface` kernel built from the `7.1` branch has been tested daily on one machine. The logs, the final kernel `.config`, the `prepare()` patch log and the test notes are in [`build-reports/2026-10-04-7.1.3-1-cachyos-surface`](https://github.com/jmacpolin/linux-cachyos-surface/tree/7.1-sp9-build-report/build-reports/2026-10-04-7.1.3-1-cachyos-surface) on the `7.1-sp9-build-report` branch.

- **Machine:** Surface Pro 9, 12th-gen Intel (Alder Lake-U, Iris Xe), SKU `Surface_Pro_9_2038`, UEFI 23.102.143, running CachyOS
- **Kernel:** `7.1.3-1-cachyos-surface`, built 2026-09-21, tested through 2026-10-04

| Build input | Value |
|---|---|
| Base kernel | CachyOS pre-patched tarball `cachyos-7.1.3-1` |
| Surface patches | [`Apiznel/linux-surface@df5430f`](https://github.com/Apiznel/linux-surface/commit/df5430f474d2923c4ff5da745aadbfd0fe4801d7), the source branch of the still-unmerged upstream PR [linux-surface/linux-surface#2178](https://github.com/linux-surface/linux-surface/pull/2178) |
| This repo | `fdab95a` |
| Toolchain / options | clang 22.1.8, full LTO, `-O3`, x86-64-v3, 1000 Hz, EEVDF scheduler |
| Config | `config` + `surface-7.1.config`, with every option that is new in 7.1 (168 prompts) left at its default |
| Userspace | iptsd 3.1.0, libcamera 0.7.2, linux-firmware 20260916, sof-firmware 2026.09.1 |
| Extra kernel parameters | `pci=hpiosize=0 i915.enable_psr=0 rcutree.enable_rcu_lazy=1` |

The scheduler is EEVDF, not BORE. The CachyOS tarball doesn't contain the BORE scheduler, so the PKGBUILD's `scripts/config -e SCHED_BORE` does nothing. Upstream `linux-cachyos` behaves the same way: its default kernel is "EEVDF", and BORE is a separate patch for the `bore` variants.

| Feature | Status | Notes |
|---|---|---|
| Touchscreen | ✅ Works | `ithc` (legacy mode) + iptsd. Still works after resume; iptsd restarts on each resume, which is expected |
| Pen | ✅ Works | Pressure and buttons |
| Type Cover | ✅ Works | Keys, touchpad, backlight, Fn keys |
| Tablet mode on detach | ✅ Works | |
| Wi-Fi / Bluetooth | ✅ Works | Intel CNVi (`iwlwifi`), `btusb` |
| Speakers | ✅ Works | |
| Microphone | ⚪ Not tested | |
| Screen brightness | ✅ Works | OS slider, Type Cover Fn keys, and auto-brightness from the ambient light sensor |
| Suspend / resume (s2idle) | ✅ Works | 5–10 cycles; longest captured sleep 6.7 h; roughly <10 % battery drain per day asleep (estimate) |
| Charging | ✅ Works | USB-C and Surface Connect |
| Rear camera (ov13858) | ⚠️ Partial | Streams, but the image is upside down. Fixed by upstream PR [linux-surface/linux-surface#2227](https://github.com/linux-surface/linux-surface/pull/2227), which the 7.1 build doesn't include but the `7.2` branch does |
| Front camera (ov5693) | ⚪ Not tested | Detected by libcamera. Streaming probably needs upstream PR [linux-surface/linux-surface#2171](https://github.com/linux-surface/linux-surface/pull/2171), which the 7.1 build doesn't include but the `7.2` branch does |
| IR camera | ⚪ Not detected | No IR sensor driver loads, and libcamera lists only the front and rear cameras |

These kernel log messages appear on this machine and are harmless:

- `ithc 0000:00:10.6: hid_input_report failed with -16`: about 30 lines right after resume, while the touch device is being recreated.
- `auxiliary intel_ipu6.psys.40: Failed to get runtime PM`: once per resume.
- `surface_serial_hub serial0-0: event: unhandled event (... cid: 0x1a ...)`.
- iptsd logs `Reading from file failed: Input/output error` and restarts on each resume.

To reproduce this build, check out the `7.1` branch and run `makepkg -si` in `linux-cachyos-surface/`. The PKGBUILD is pinned to `Apiznel/linux-surface@df5430f`, and the base `config` is the tested kernel's own `.config`, so the build doesn't ask any configuration questions. The build that was tested here was made before the pin: it pulled the tip of the same branch, which was `df5430f`, and its 168 new-option prompts were all answered with the defaults.

## Variants

### linux-cachyos-surface

This variant is as close to the original cachyos kernel as possible, it is build using Clang LTO mode `full` with llvm for maximum performance.

### linux-cachyos-surface-lts

This variant is based on the original cachyos lts kernel, but is build with gcc for better stability and support.

## Installation Instructions

### Build from source

To build the kernel and header files from source, run the following commands within CachyOS (or any Arch derivative):

```bash
sudo pacman -S base-devel
git clone -b 7.2 https://github.com/jmacpolin/linux-cachyos-surface
cd linux-cachyos-surface/linux-cachyos-surface
makepkg -si
```

Or alternatively using docker:

```bash
docker run --name kernelbuild -v $PWD:/pkg cachyos/docker-makepkg-v3
sudo pacman -U linux-cachyos-surface-*.pkg.tar.zst
```

_**NOTE:** Per default the linux-cachyos-surface kernel is configured in LTO mode `full`, this may take a bit longer to compile and requires more ram. It can be changed by updating the following line in the PKGBUILD:_
```bash
# Clang LTO mode, only available with the "llvm" compiler - options are "none", "full" or "thin".
# ATTENTION - one of three predefined values should be selected!
# "full: uses 1 thread for Linking, slow and uses more memory, theoretically with the highest performance gains."
# "thin: uses multiple threads, faster and uses less memory, may have a lower runtime performance than Full."
# "thin-dist: Similar to thin, but uses a distributed model rather than in-process: https://discourse.llvm.org/t/rfc-distributed-thinlto-build-for-kernel/85934"
# "none: disable LTO
: "${_use_llvm_lto:=full}"
```

### Install prebuilt packages

This fork doesn't publish prebuilt packages. The original project's [releases](https://github.com/jonpetersathan/linux-cachyos-surface/releases) are older kernels built with the official linux-surface patches. To install one of those, download the kernel and header packages and run:

```bash
sudo pacman -U linux-cachyos-surface-*.pkg.tar.zst
```

## Other sources of newer Surface kernels

- [linux-surface/linux-surface#2179](https://github.com/linux-surface/linux-surface/pull/2179): the Surface patches rebased to the 6.18 long-term kernel (6.18.55), if you'd rather run an LTS kernel.
- [Steefzar/linux-cachyos-surface-latest](https://github.com/Steefzar/linux-cachyos-surface-latest): another CachyOS + Surface package. It tracks the newest CachyOS release with a hand-rebased patch set and offers a signed pacman repository. It's tested on a Surface Laptop 4 AMD.
- [adam-adrian/linux-cachyos-surface](https://github.com/adam-adrian/linux-cachyos-surface) (`7.2` branch): another fork of this repo's lineage, on CachyOS 7.2.7 with `Apiznel/linux-surface@7.2` and GitHub Actions builds. It's tested on a Surface Pro 2017.

## Acknowledgements

- Maximilian Luz: [surface-linux/surface-linux](https://github.com/linux-surface/linux-surface)
- Peter Lung: [CachyOS/linux-cachyos](https://github.com/CachyOS/linux-cachyos)
