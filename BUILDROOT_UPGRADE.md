# ANTSDR E200 Buildroot Upgrade to 2025.02 with SDR Stack

This documents the process of upgrading the ANTSDR E200 firmware buildroot
from the ADI fork (old, based on 2019.x) to mainline buildroot 2025.02 with
an internal toolchain and a full Python/GNU Radio/SDR software stack.

## Overview

- **Old setup**: ADI buildroot fork at commit `e783aadc`, external Linaro GCC 7.3 toolchain
- **New setup**: Mainline buildroot 2025.02 (commit `aa2d7ca5`), internal GCC 13.3 toolchain
- **New software**: Python 3.12, numpy, scipy, GNU Radio 3.10.11, liquid-dsp, fftw, and more
- **Root filesystem size**: ~72MB (was ~20MB)

## Prerequisites

- Xilinx Vivado 2023.2 installed at `/opt/Xilinx/Vivado/2023.2/`
- The `plutosdr-fw` submodule with HDL, linux, u-boot already patched for e200
- Host packages: `build-essential`, `cmake`, `git`, `wget`, `u-boot-tools`, etc.

## Step-by-Step Procedure

### 1. Apply the e200 patches (if starting fresh)

```bash
cd /home/siuser/Documents/antsdr-fw-patch-codex
./resetGit.sh
./patch.sh e200
```

### 2. Switch buildroot to mainline 2025.02

The `patch.sh` applies the old ADI buildroot patches. We need to replace that
with mainline buildroot while keeping the ADI custom packages.

```bash
cd plutosdr-fw/buildroot

# Save ADI custom packages (not in mainline buildroot)
for pkg in libad9361-iio libiio libini poll_sysfs ad936x_ref_cal jesd204b_status; do
    cp -r package/$pkg /tmp/adi_pkg_$pkg
done

# Switch to mainline buildroot 2025.02
git remote add upstream https://github.com/buildroot/buildroot.git 2>/dev/null || true
git fetch upstream --depth=1 2025.02
git checkout aa2d7ca5    # 2025.02 release tag

# Copy ADI packages back
for pkg in libad9361-iio libiio libini poll_sysfs ad936x_ref_cal jesd204b_status; do
    cp -r /tmp/adi_pkg_$pkg package/$pkg
done
```

### 3. Register ADI packages in Config.in

Edit `package/Config.in` and add after the `source "package/libiio/Config.in"` line:

```
source "package/libad9361-iio/Config.in"
source "package/libini/Config.in"
source "package/poll_sysfs/Config.in"
source "package/ad936x_ref_cal/Config.in"
source "package/jesd204b_status/Config.in"
```

### 4. Apply board files from old patch

The `board/e200/` directory (init scripts, post-build.sh, busybox config, etc.)
comes from the old buildroot patch. Apply it selectively, excluding the defconfig
and Config.in which we replace:

```bash
# From the outer repo's patch directory:
cd plutosdr-fw/buildroot
git apply ../../patch/e200/0001-add-support-buildroot.patch \
    --exclude='configs/zynq_e200_defconfig' \
    --exclude='package/Config.in'
```

Or if starting from a working tree that already has these files, just make sure
`board/e200/` is populated.

### 5. Apply required fixes

These fixes are needed for cross-compilation with the internal toolchain:

#### Fix 1: Remove stale libiio patch

The ADI libiio v0.26 already includes the libxml 2.12 compatibility fix that
mainline buildroot ships as a patch for v0.25. Remove it:

```bash
rm package/libiio/0001-xml-Fix-compatibility-with-libxml-2-12.patch
```

#### Fix 2: GNU Radio pybind11 cross-compilation

GNU Radio with Python bindings fails because pybind11 detects a pointer-size
mismatch (host Python is 64-bit, ARM compiler is 32-bit). Add
`PYBIND11_USE_CROSSCOMPILING=ON` to `package/gnuradio/gnuradio.mk`:

In the `BR2_PACKAGE_GNURADIO_PYTHON` section, change:

```makefile
GNURADIO_CONF_OPTS += -DPYBIND11_PYTHONLIBS_OVERWRITE=OFF
```

to:

```makefile
GNURADIO_CONF_OPTS += -DPYBIND11_PYTHONLIBS_OVERWRITE=OFF \
	-DPYBIND11_USE_CROSSCOMPILING=ON
```

#### Fix 3: numpy meson/lapack pkg-config variable

numpy's meson.build unconditionally queries `openblas_config` from the lapack
pkg-config file, but standalone lapack doesn't have this variable. Create the
patch file `package/python-numpy/0001-meson-fix-openblas_config-variable-for-standalone-lapack.patch`:

The patch changes line 203 of `numpy/meson.build` from:

```meson
conf_data.set(name + '_OPENBLAS_CONFIG', dep.get_variable('openblas_config'))
```

to:

```meson
conf_data.set(name + '_OPENBLAS_CONFIG', dep.get_variable('openblas_config', default_value: ''))
```

#### Fix 4: post-build.sh for internal toolchain

The original `board/e200/post-build.sh` sources `.config` directly (fails with
non-shell-compatible kconfig lines) and uses `BR2_TOOLCHAIN_EXTERNAL_PREFIX`
(doesn't exist with internal toolchain).

**Line 4** — change:
```bash
. ${BR2_CONFIG}
```
to:
```bash
eval $(grep -E '^BR2_' ${BR2_CONFIG} | grep -v '^\s*#' || true)
```

**Lines 26-28** — change the hardcoded external prefix to detect internal toolchain:
```bash
if [ -n "$BR2_TOOLCHAIN_EXTERNAL_PREFIX" ]; then
	TC_PREFIX=${BR2_TOOLCHAIN_EXTERNAL_PREFIX}
else
	TC_PREFIX=arm-buildroot-linux-gnueabihf
fi
GCC_VERSION=$(${TC_PREFIX}-gcc --version | head -1 | sed 's/.*(\(.*\))/\1/' 2>/dev/null || echo "unknown")
BIN_VERSION=$(${TC_PREFIX}-as --version | head -1 | sed 's/.*(\(.*\))/\1/' 2>/dev/null || echo "unknown")
GCC_TRIPLE=$(${TC_PREFIX}-gcc -v -c  2>&1 | sed 's/ /\n/g' | grep -e "--target" | awk -F= '{print $2}' || echo "arm-buildroot-linux-gnueabihf")
```

#### Fix 5: Missing LICENSE.html

The genimage config (`board/e200/genimage-msd.cfg`) references `LICENSE.html`
but only `LICENSE` exists in `board/e200/msd/`:

```bash
cp board/e200/msd/LICENSE board/e200/msd/LICENSE.html
```

### 6. Install the defconfig

The new defconfig lives at `configs/zynq_e200_defconfig`. Key differences from
the old one:

- `BR2_TOOLCHAIN_BUILDROOT=y` (internal toolchain, not external Linaro)
- `BR2_TOOLCHAIN_BUILDROOT_GLIBC=y` (glibc, not uclibc)
- `BR2_TOOLCHAIN_BUILDROOT_CXX=y` (C++ for GNU Radio, Boost)
- `BR2_TOOLCHAIN_BUILDROOT_FORTRAN=y` (Fortran for numpy/scipy/LAPACK)
- Full Python/SDR/GNU Radio package selection

### 7. Update scripts/e200.mk

The CROSS_COMPILE variable must match the internal toolchain tuple:

```makefile
# Override CROSS_COMPILE for internal buildroot toolchain
CROSS_COMPILE = arm-buildroot-linux-gnueabihf-
```

### 8. Build

```bash
cd plutosdr-fw
export TARGET=e200
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2023.2/settings64.sh
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export SKIP_LEGAL=1
make
```

**IMPORTANT**: Use `export TARGET=e200`, NOT `make TARGET=e200`. Command-line
make variables propagate to sub-makes via MAKEFLAGS and override target-specific
variable assignments in the HDL build system (`TARGET:=xilinx` in
`hdl/projects/scripts/project-xilinx.mk`), breaking the FPGA library builds.

**First build will take a long time** (~1-2 hours) because the internal toolchain
compiles GCC 13.3, binutils, and glibc from source. Subsequent rebuilds reuse
the cached toolchain and are much faster.

### 9. Generate SD card image

```bash
rm -rf build_sdimg    # Must remove first if it exists
make sdimg
```

Output files in `build_sdimg/`:

| File | Description |
|------|-------------|
| `BOOT.bin` | FSBL + FPGA bitstream + U-Boot |
| `uImage` | Linux kernel |
| `devicetree.dtb` | Device tree blob |
| `uEnv.txt` | U-Boot environment |
| `uramdisk.image.gz` | Root filesystem (~72MB) |

### 10. Write to SD card

Copy all files from `build_sdimg/` to a FAT32-formatted SD card.

## Software Included in the New Image

### Toolchain
- GCC 13.3.0 (ARM Cortex-A9, NEON, VFPv3, hard-float)
- glibc (with C++, Fortran)

### Python Stack
- Python 3.12.9
- numpy 1.25.0 (with OpenBLAS + LAPACK)
- scipy
- PyZMQ, PyYAML

### GNU Radio 3.10.11.0
- gnuradio-runtime, gr-blocks, gr-analog, gr-digital
- gr-filter, gr-fft, gr-fec, gr-network
- gr-channels, gr-trellis, gr-zeromq
- **gr-iio** (talks directly to AD9361 via libiio)
- Full Python bindings

### DSP Libraries
- liquid-dsp
- FFTW (single + double precision)
- OpenBLAS 0.3.29 (Cortex-A9 optimized)
- LAPACK 3.10.1

### ADI/IIO Stack
- libiio 0.26 + iiod (IIO daemon, USB backend)
- libad9361-iio (AD9361 multi-chip sync)
- libini, poll_sysfs, ad936x_ref_cal

### System
- Dropbear SSH
- Avahi (mDNS/DNS-SD)
- BusyBox
- wpa_supplicant (WiFi)
- ncurses, zlib, zstd

## Files Modified vs Mainline Buildroot 2025.02

**Modified existing files** (6):
- `package/Config.in` — added ADI package registrations
- `package/gnuradio/gnuradio.mk` — added `PYBIND11_USE_CROSSCOMPILING=ON`
- `package/libiio/Config.in` — ADI version with extra options (IIOD USB, etc.)
- `package/libiio/libiio.hash` — updated hash for v0.26
- `package/libiio/libiio.mk` — ADI version with v0.26 + extra features
- Removed: `package/libiio/0001-xml-Fix-compatibility-with-libxml-2-12.patch`

**New files** (74):
- `configs/zynq_e200_defconfig` — full SDR stack defconfig
- `board/e200/` — board support (init scripts, post-build.sh, busybox config, web UI, etc.)
- `package/libad9361-iio/` — ADI custom package
- `package/libini/` — ADI custom package
- `package/poll_sysfs/` — ADI custom package
- `package/ad936x_ref_cal/` — ADI custom package
- `package/jesd204b_status/` — ADI custom package
- `package/python-numpy/0001-meson-fix-openblas_config-...patch` — cross-compile fix

## Not Yet Done

- **SoapySDR**: Not in mainline buildroot. Would need a custom package recipe.
- **Patch generation**: The changes to buildroot should be captured as a new
  patch file in `patch/e200/` for the patch-based workflow. Run from
  `plutosdr-fw/buildroot/`:
  ```bash
  git add -A && git diff --cached HEAD > ../../patch/e200/0001-add-support-buildroot.patch
  ```
