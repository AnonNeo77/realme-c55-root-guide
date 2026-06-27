# Tools Reference

A reference for every tool used across this repository. Organized by function.
Each entry covers what it does in the context of the RMX3710, where to get it,
and any device-specific notes.

---

## Partition Access and Flashing

### MTKClient

**Repo:** https://github.com/bkerler/mtkclient

The primary tool for all partition-level operations on the RMX3710. Communicates
with the device via the MediaTek Preloader or BootROM interface. Used for
bootloader unlock, individual partition backup and restore, and boot image
flashing.

```bash
git clone https://github.com/bkerler/mtkclient.git
cd mtkclient
pip install -r requirements.txt --break-system-packages
```

Common operations:

```bash
# Detect device
python mtk detect

# Read partition
python mtk r boot boot_backup.img

# Write partition
python mtk w boot magisk_patched.img

# Bootloader unlock
python mtk da seccfg unlock
```

**Device notes:**
- Requires Preloader/BootROM access — device must be powered off before
  connecting
- On Android 14 (≥1800 build) and Android 15, Preloader auth is patched;
  bypass_utility is required first
- Slower than SP Flash Tool for bulk partition writes

---

### SP Flash Tool

**Source:** Distributed via MediaTek and third-party mirrors. Search for
`SP Flash Tool Linux` or `SP Flash Tool Windows` for the appropriate platform
build.

GUI application for MediaTek firmware flashing. Faster than MTKClient for bulk
operations. Uses a scatter file to map partition names to eMMC offsets. Required
for recovery when the bootloader stages are corrupted and the device cannot
reach Preloader mode normally.

**Device notes:**
- Requires a Download Agent (DA) file
- The stock DA is rejected on the RMX3710 — use `MTK_all-in-one DA` or
  `NoAuth.bin` (see bypass_utility below)
- Always uncheck `nvram`, `nvcfg`, and `md1img` in the scatter file before
  flashing
- Preferred tool for full firmware restore from a scatter file

---

### bypass_utility

**Repo:** https://github.com/MTK-bypass/bypass_utility

Crashes the MediaTek Preloader to force a fallback to BootROM mode. Required
on the RMX3710 when Preloader authentication blocks MTKClient or SP Flash Tool
from connecting.

```bash
git clone https://github.com/MTK-bypass/bypass_utility.git
cd bypass_utility
pip install -r requirements.txt --break-system-packages

# Power off device, connect USB
python bypass_utility.py
```

**Device notes:**
- After a successful bypass, connect SP Flash Tool or MTKClient immediately —
  the BootROM window may be time-limited
- Required for recovery from hard brick states where Preloader does not respond
- Works at the BootROM level — unaffected by firmware version

---

## Root and Boot Image

### Magisk

**Repo:** https://github.com/topjohnwu/Magisk

Systemless root via boot image patching. Modifies the ramdisk within `boot.img`
to inject Magisk's init binary during early userspace startup. Does not touch
`system`, `vendor`, or any AVB-verified partition.

**Installation on RMX3710:**

1. Copy `boot.img` to the device
2. Open Magisk app → Install → Select and Patch a File
3. Flash the output `magisk_patched_*.img` via MTKClient

```bash
python mtk w boot magisk_patched.img
```

**Device notes:**
- The patched image must be built from the exact stock `boot.img` for the
  installed firmware build
- OTA updates overwrite `boot` — repatch and reflash after every update
- Ramdisk-based installation confirmed working on RMX3710

---

### mkbootimg / unpackbootimg

**Source:** Android Platform Tools / AOSP

Used to pack and unpack boot images when building a custom kernel or inspecting
boot image structure.

```bash
# Unpack — outputs base address, pagesize, ramdisk offset
unpackbootimg -i stock_boot.img

# Pack
mkbootimg \
    --kernel Image.gz-dtb \
    --ramdisk ramdisk.img \
    --base 0x40000000 \
    --pagesize 2048 \
    --output custom_boot.img
```

---

## Dynamic Partitions

### lpdump

**Source:** Android Platform Tools / AOSP (`lptools` package)

Reads and displays the logical partition metadata stored at the beginning of a
`super.img`. First step before any super image analysis — provides partition
names, offsets, sizes, group assignments, and block device geometry needed for
repacking.

```bash
lpdump super.img
```

---

### lpunpack

**Source:** Android Platform Tools / AOSP (`lptools` package)

Extracts logical partitions from a `super.img` as individual image files.

```bash
lpunpack super.img output_dir/
# Produces: system_a.img, vendor_a.img, product_a.img, etc.
```

**Device notes:**
- `_b` slot partitions are typically empty on a single-boot device — use `_a`
- Sparse images must be converted with `simg2img` first

---

### lpmake

**Source:** Android Platform Tools / AOSP (`lptools` package)

Reconstructs a `super.img` from individual partition images with updated
metadata. Parameters must exactly match the original geometry from `lpdump`
output.

```bash
lpmake \
    --metadata-size 65536 \
    --super-name super \
    --metadata-slots 3 \
    --device super:9126805504 \
    --group main_a:4563402752 \
    --partition system_a:readonly:0:main_a \
    --image system_a=system_a.img \
    --sparse \
    --output super_new.img
```

---

### simg2img

**Source:** Android Platform Tools / `android-tools` package

Converts Android sparse image format to raw image format. Required before
MTKClient write operations or mounting — MTKClient and mount do not handle
sparse format.

```bash
simg2img super_sparse.img super_raw.img
```

---

### MTK-SuperBuilder

**Repo:** https://github.com/AnonNeo77/MTK-SuperBuilder

Python CLI tool that wraps `lpdump`, `lpunpack`, and `lpmake` into a single
workflow for MediaTek super images. Handles RMX3710 geometry automatically.

```bash
git clone https://github.com/AnonNeo77/MTK-SuperBuilder.git
cd MTK-SuperBuilder
pip install -r requirements.txt --break-system-packages
python superbuilder.py
```

Built specifically for the MediaTek super image format used on devices like
the RMX3710.

---

## Filesystem

### erofsfuse

**Repo:** https://github.com/erofs/erofs-utils

Userspace FUSE driver for mounting erofs filesystem images without kernel erofs
support. The RMX3710's `system` and `vendor` partitions use erofs on newer
firmware builds.

```bash
# Arch Linux
yay -S erofs-utils

# Mount
erofsfuse system_a.img /mnt/system

# Unmount
fusermount -u /mnt/system
```

**Device notes:**
- Required if the host kernel does not have `CONFIG_EROFS_FS` built in
- Older firmware builds may use ext4 — check with `file system_a.img` first

---

## AVB and Partition Inspection

### avbtool

**Source:** AOSP / `android-tools` package

Inspects `vbmeta.img` structure — shows algorithm, flags, chained partition
descriptors, and hash tree metadata.

```bash
avbtool info_image --image vbmeta.img
```

---

### avbctl

Available on rooted device via ADB. Queries dm-verity state on running Android.

```bash
adb shell su -c "avbctl get-verity"
```

---

## Android Platform Tools

**Source:** https://developer.android.com/tools/releases/platform-tools

Provides `adb`, `fastboot`, and related utilities.

```bash
# Arch Linux
sudo pacman -S android-tools
```

ADB is used throughout for shell access, file transfer, and log inspection on
a running rooted device. Fastboot and Fastbootd are present on the RMX3710 but
do not provide bootloader unlock or physical partition flashing capability on
this device.

---

## Kernel and Module Building

### aarch64-linux-gnu-gcc

Cross-compiler for building the Android kernel and out-of-tree modules
targeting the RMX3710's ARM64 architecture.

```bash
sudo pacman -S aarch64-linux-gnu-gcc
```

Used in all kernel and module build commands:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

---

### scripts/sign-file

Kernel script for signing `.ko` module files. Located within the kernel source
tree at `scripts/sign-file`. Required when running a custom kernel with module
signature enforcement enabled.

```bash
scripts/sign-file sha256 \
    certs/signing_key.pem \
    certs/signing_key.x509 \
    module.ko
```

---

## Firmware Source

### realmefirmware.com

**URL:** https://realmefirmware.com/realme-c55-firmware/

Community-maintained repository of official Realme OTA and firmware packages
for the C55 (RMX3710). Provides firmware packages for all Android versions
including downgrade OTA packages needed to reach Android 13 for MTKClient
compatibility.

Download the package that exactly matches the target build number. Verify the
build number against the installed firmware before flashing any partition image.

---

## IMEI Recovery

### UnlockTool

**URL:** https://unlocktool.net

Paid commercial platform used by repair technicians for MediaTek IMEI write
operations. Available via online rental services — hourly or daily sessions,
no full subscription required for a one-time use.

Writes a supplied IMEI value to modem partitions via the MediaTek diagnostic
interface.

**Requirements:**
- Unlocked bootloader — hard requirement, tool will not proceed without it
- Active session with operation credits
- Device connected and recognized

**Limitations:**
- Writes a replacement IMEI — does not restore original factory calibration
- Paid service — cost varies by rental platform
- Not required if a backup of `nvram`, `nvcfg`, and `md1img` exists from
  before the data was lost

---

## Quick Reference

| Tool              | Purpose                                  | Interface  |
|-------------------|------------------------------------------|------------|
| MTKClient         | Partition R/W, bootloader unlock         | CLI        |
| SP Flash Tool     | Bulk firmware flash, BootROM recovery    | GUI        |
| bypass_utility    | Preloader crash → BootROM access         | CLI        |
| Magisk            | Systemless root via boot image patch     | App + CLI  |
| mkbootimg         | Pack/unpack boot images                  | CLI        |
| lpdump            | Inspect super.img metadata               | CLI        |
| lpunpack          | Extract logical partitions from super    | CLI        |
| lpmake            | Repack super.img                         | CLI        |
| simg2img          | Sparse → raw image conversion            | CLI        |
| MTK-SuperBuilder  | MediaTek super image workflow            | CLI        |
| erofsfuse         | Mount erofs partition images             | CLI        |
| avbtool           | Inspect vbmeta structure                 | CLI        |
| aarch64-linux-gnu-gcc | ARM64 cross-compilation             | CLI        |
| scripts/sign-file | Kernel module signing                    | CLI        |
| UnlockTool        | IMEI restoration (paid)                  | GUI/Web    |
