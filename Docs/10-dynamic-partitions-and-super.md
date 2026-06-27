# 10 - Dynamic Partitions

## Overview

The Realme C55 (RMX3710) uses Android's dynamic partition system, introduced
in Android 10. Instead of fixed-size partitions for `system`, `vendor`,
`product`, and `odm`, these are combined into a single large partition called
`super`.

The contents of `super` are managed at runtime by a userspace partition
manager (`fastbootd` / `dm-linear`), which maps logical partitions from the
physical `super` block device.

---

## Physical vs Logical Partitions

### Physical partitions (fixed, on eMMC)

These exist as real partitions in the GPT partition table:

boot        preloader       vbmeta

dtbo        recovery        persist

super       metadata        userdata

These are directly addressable by the bootloader and MTKClient.

### Logical partitions (inside super)

These exist only within the `super` physical partition and are managed by
`dm-linear`: 

system      system_ext

vendor      vendor_dlkm

product     odm

These are **not** visible as separate partitions to the bootloader. They are
mapped into the device-mapper at runtime by the init process.

---

## super.img Structure

The `super` partition uses the Android `liblp` (lib logical partitions) format.
The layout metadata lives at the beginning of the `super` block and describes
how logical partitions are arranged within it.

### Top-level structure

super.img

├── Metadata header (liblp)

│   ├── Partition table

│   │   ├── system  (offset + size)

│   │   ├── vendor  (offset + size)

│   │   ├── product (offset + size)

│   │   └── ...

│   └── Block device geometry

└── Raw partition data

├── system (ext4 or erofs)

├── vendor (ext4 or erofs)

└── ...

### Slot suffixes

The RMX3710 uses an A/B-like slot system for dynamic partitions even though
it is a Virtual A/B device. Logical partitions carry `_a` and `_b` suffixes
in the metadata:

system_a    system_b

vendor_a    vendor_b

product_a   product_b

Only one slot is active at a time. The other slot may be empty or contain a
previous firmware snapshot depending on the OTA state.

---

## Inspecting super.img

### Read the partition metadata

```bash
lpdump super.img
```

Output includes the metadata version, partition table, extents, and block
device geometry. This is the starting point for any super.img analysis.

Example output (truncated):  Metadata version: 10.2

Metadata size: 65536 bytes

Metadata max size: 65536 bytes

Metadata slot count: 3

Partition table:

Name: system_a

Attributes: readonly

Extents:

0 .. 2424831 linear super 2048

Name: vendor_a

Attributes: readonly

Extents:

0 .. 614399 linear super 2426880


### Extract logical partitions

```bash
lpunpack super.img output_dir/
```

This extracts each logical partition as a separate image file: output_dir/

├── system_a.img

├── vendor_a.img

├── product_a.img

└── ...

Each extracted image is an `ext4` or `erofs` filesystem image that can be
mounted or inspected directly.

---

## Filesystem Formats Inside super

The RMX3710 uses **erofs** (Enhanced Read-Only File System) for system
partitions on newer firmware builds. Earlier builds may use ext4.

### Identifying the format

```bash
file system_a.img
# erofs: Linux erofs filesystem data
```

### Mounting erofs

```bash
sudo mount -t erofs -o loop system_a.img /mnt/system
# Requires kernel erofs support or erofsfuse
```

Using `erofsfuse` (userspace, no root required):

```bash
erofsfuse system_a.img /mnt/system
```

---

## Repacking super.img

After modifying logical partition contents, `super.img` must be repacked with
updated metadata before it can be flashed.

### Using lpmake

```bash
lpmake \
  --metadata-size 65536 \
  --super-name super \
  --metadata-slots 3 \
  --device super:9126805504 \
  --group main_a:4563402752 \
  --partition system_a:readonly:0:main_a \
  --image system_a=system_a.img \
  --partition vendor_a:readonly:0:main_a \
  --image vendor_a=vendor_a.img \
  --sparse \
  --output super_new.img
```

Parameters must match the original metadata exactly. Use `lpdump` output as
the reference for block device size, group sizes, and slot count.

### MTK-SuperBuilder

For the RMX3710 specifically, the geometry and metadata slot configuration is
handled automatically by
[MTK-SuperBuilder](https://github.com/AnonNeo77/MTK-SuperBuilder), which wraps
`lpdump`, `lpunpack`, and `lpmake` into a single CLI workflow tailored for
MediaTek device super images.

---

## Flashing super.img

Super images can be large (6–9GB uncompressed). Flashing options on the
RMX3710:

### Via MTKClient (physical super partition)

```bash
python mtk w super super_new.img
```

This writes directly to the physical `super` partition. MTKClient handles the
size and offset automatically.

### Sparse images

Firmware distributed by Realme uses sparse image format to reduce file size.
Convert to raw before flashing via MTKClient, or use tools that handle sparse
format natively:

```bash
simg2img super_sparse.img super_raw.img
python mtk w super super_raw.img
```

---

## Relationship to Firmware Analysis

Dynamic partitions are the primary surface for Android firmware analysis on
the RMX3710:

| Partition  | Contains                                      |
|------------|-----------------------------------------------|
| `system`   | Android framework, core libraries, apps       |
| `vendor`   | MediaTek HALs, proprietary blobs, kernel mods |
| `product`  | OEM apps, Realme UI overlays                  |
| `odm`      | Device-specific vendor overlays               |

For RE work, `vendor` is typically the most interesting partition — it contains
MediaTek-specific binaries, init scripts, and proprietary service daemons that
are not part of AOSP.

```bash
# After mounting vendor_a.img
ls /mnt/vendor/lib/
ls /mnt/vendor/bin/
ls /mnt/vendor/etc/init/
```

---

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `lpdump` reports bad magic | Sparse image format | Convert with `simg2img` first |
| `lpunpack` produces 0-byte images | Inactive slot (empty `_b`) | Use `_a` suffix partitions |
| Mount fails on erofs | Kernel lacks erofs support | Use `erofsfuse` instead |
| `lpmake` output won't boot | Geometry mismatch | Re-run `lpdump` and match parameters exactly |
| MTKClient write fails on super | Image larger than partition | Verify partition size from `lpdump` |

---

## Summary

The Realme C55 uses Android's `liblp` dynamic partition system with `super`
as the physical container for all major Android partitions. The logical layout
is defined in metadata at the start of `super` and mapped at runtime via
`dm-linear`.

For firmware analysis, `lpunpack` → mount is the standard workflow to access
partition contents. For firmware modification, `lpmake` (or MTK-SuperBuilder)
reconstructs a valid `super.img` with updated metadata.

The `vendor` partition is the primary target for MediaTek-specific binary
analysis on the RMX3710.

---

## Next

`11-recovery-and-restore.md` — Recovery options available on the Realme C55,
how to restore a bricked or bootlooping device, and full partition backup
strategy using MTKClient.
