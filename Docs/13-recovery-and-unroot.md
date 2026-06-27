# 13 - Recovery and Restore

## Overview

Recovery on the Realme C55 (RMX3710) is significantly more limited than most
Android devices. There is no dedicated recovery partition — the device uses a
combined boot/recovery model. The recovery menu itself provides only three
options. All meaningful partition-level recovery is handled externally via
MTKClient or SP Flash Tool through the MediaTek Preloader/BootROM interface.

---

## Recovery Menu

The Realme C55 does not have a separate `recovery` partition. Recovery is
embedded within the boot image itself.

Access: Power off → Hold Volume Up + Power

Available options:

| Option        | Function                              |
|---------------|---------------------------------------|
| Format Data   | Factory reset (wipes userdata)        |
| Reboot        | Reboots the device normally           |
| Power Off     | Powers down the device                |

There is no ADB sideload, no custom image flashing, no partition access, and
no Fastboot interface exposed from this menu. For any recovery beyond a factory
reset, external tools through the MediaTek interface are required.

---

## Recovery Tools

Two tools can access the device at the MediaTek Preloader/BootROM level for
partition backup, restoration, and firmware flashing.

### MTKClient

[MTKClient](https://github.com/bkerler/mtkclient) communicates with the device
via the Preloader or BootROM interface. It provides scripted, partition-aware
read/write operations and is the primary tool used throughout this
documentation.

<img width="2946" height="1646" alt="device-detected" src="https://github.com/user-attachments/assets/caa7d5ed-56ad-4936-88db-456baa3b8bb5" />

Best suited for:
- Individual partition backup and restore
- Bootloader unlock
- Scripted or automated workflows

### SP Flash Tool

SP Flash Tool is a MediaTek-provided GUI application for firmware flashing. On
the RMX3710 it is generally **faster than MTKClient** for full firmware
flashing operations, particularly when writing large images like `super`.

<img width="1920" height="1080" alt="sp-flash-tool" src="https://github.com/user-attachments/assets/c4cda626-6ef6-4882-ab51-6d1edf08b9cf" />

Best suited for:
- Full firmware restoration from a scatter file
- Faster bulk partition flashing
- Users who prefer a GUI workflow

SP Flash Tool requires a Download Agent (DA) file to communicate with the
device. On devices where the stock DA is rejected due to authentication
requirements, alternative DA files are used — see the bypass section below.

---

## Bypassing Preloader Authentication

MediaTek Preloader authentication can block SP Flash Tool
from connecting to the device. On the RMX3710, this is bypassed by crashing
the Preloader to force a fallback to BootROM mode, which has no authentication
requirement.

This is handled by:

**[bypass_utility](https://github.com/MTK-bypass/bypass_utility)**

```bash
git clone https://github.com/MTK-bypass/bypass_utility.git
cd bypass_utility
pip install -r requirements.txt --break-system-packages

# Power off device, connect USB with Volume Down held
python bypass_utility.py
```

The utility sends a crafted payload that crashes the Preloader, causing the
MediaTek SoC to fall back to BootROM. At the BootROM level, no authentication
is enforced, and both MTKClient and SP Flash Tool can connect freely.

if this tool not working(that happens to me) erase the lk_a or lk_b partition and then try again while pressing both volume buttons.

NOTE: after erasing lk partitions phone seems dead and only vibration works!

After a successful bypass, the device appears as a new USB device. Proceed
immediately with MTKClient or SP Flash Tool — the BootROM window is time-limited
on some configurations.

---

## SP Flash Tool — DA Files

SP Flash Tool requires a Download Agent binary to be loaded onto the device
before any operations can proceed. On authenticated devices, the stock DA will
be rejected. Two alternatives work after a bypass:

| DA File             | Notes                                        |
|---------------------|----------------------------------------------|
| `MTK_all-in-one DA` | Broad compatibility across MediaTek chipsets |
| `NoAuth.bin`        | Minimal DA, bypasses authentication checks   |

Load the DA file in SP Flash Tool under:

Options → Connection → DA File → Browse

Then proceed with scatter file-based flashing or individual partition writes.

---

## Firmware Source

Official Realme C55 firmware packages are available at:

**https://realmefirmware.com/realme-c55-firmware/**

These are official Realme OTA/firmware packages. Download the package that
exactly matches the target Android version and build number.

The firmware package contains a scatter file and individual partition images.
The scatter file is required by SP Flash Tool to map partition names to their
correct offsets on the eMMC.

---

## NVRAM, NVCFG, and MD1IMG — Critical Warning

Three partitions in the firmware package contain **device-specific data** that
is unique to each physical unit:

| Partition | Contains                                      |
|-----------|-----------------------------------------------|
| `nvram`   | RF calibration, Wi-Fi/BT MAC address         |
| `nvcfg`   | Network configuration, device parameters      |
| `md1img`  | Modem firmware including IMEI                 |

**These partitions must be excluded from any flashing operation.**

When flashing via SP Flash Tool with a scatter file, uncheck these three
partitions before starting the flash. When flashing via MTKClient, do not
include them in write commands.

Flashing these from a firmware package will overwrite the device's IMEI and
calibration data with generic or zeroed values.

### IMEI Recovery

If `nvram`, `nvcfg`, or `md1img` are overwritten:

- IMEI is lost and the device will show `null` or `000000` IMEI
- Cellular connectivity is broken
- The device cannot register on any network

IMEI restoration is possible but constrained:

- Tools such as **UnlockTool** can write IMEI back to the modem partitions
- These are **paid commercial services/tools** — not free
- The restored IMEI is written programmatically, not restored from the original
  calibration data, so it may not fully replicate factory modem behavior
- UnlockTool and similar tools **require an unlocked bootloader** to perform
  IMEI write operations

There is no free, reliable, fully-original method to recover IMEI once these
partitions are overwritten. The only true recovery is restoring from a backup
taken from the same physical device before the data was lost.

> Back up `nvram`, `nvcfg`, and `md1img` before any flashing operation.
> These backups are non-transferable between devices.

---

## Partition Backup via MTKClient

Before any modification, back up critical partitions:

```bash
# Enter preloader/BootROM mode first (Volume Down + USB, or run bypass_utility)

python mtk r boot boot_backup.img
python mtk r vbmeta vbmeta_backup.img
python mtk r lk lk_backup.img
python mtk r preloader preloader_backup.img
python mtk r super super_backup.img

# Device-specific — back up, never flash from external source
python mtk r nvram nvram_backup.img
python mtk r nvcfg nvcfg_backup.img
python mtk r md1img md1img_backup.img
```

Store all backups off-device. These cannot be obtained from any public
firmware source.

---

## Restoration Procedures

### Boot image restore (bootloop / bad patch)

```bash
python mtk w boot boot_backup.img
```

Or via SP Flash Tool: scatter flash with only `boot` selected.

### Full firmware restore (corrupted system)

Use SP Flash Tool with the scatter file from the correct firmware package.

**Before starting:**

Uncheck: nvram

Uncheck: nvcfg

Uncheck: md1img

Flash all remaining partitions. This restores `super`, `lk`, `vbmeta`,
`preloader`, and all other partitions to stock state without touching modem
or calibration data.

### Preloader/BootROM recovery (hard brick)

If Preloader is corrupted and the device does not respond normally:

1. Run `mtkclient` to force BootROM(Brom) mode
2. Flash preloader.img with a compatible preloader file from firmware or dump.
3. Flash `preloader` and `lk` individually

If BootROM is not accessible via USB, hardware-level recovery (JTAG or direct
eMMC access) is required and outside the scope of this document.

---

---

## Common Failure States

| State | Symptoms | Resolution |
|-------|----------|------------|
| Bad boot patch | Bootloop at logo | Restore `boot` via MTKClient |
| Corrupted super | Bootloop post-kernel | Restore `super` via SP Flash Tool |
| vbmeta mismatch | No boot, AVB error | Restore `vbmeta` via MTKClient |
| OTA overwrote root | Boots, no root | Repatch and reflash `boot` |
| Preloader corrupted | No USB detection | mtkclient/SP Flash Tool |
| nvram/nvcfg/md1img overwritten | No IMEI, no signal | Restore from device backup or paid tool |
| MTKClient auth rejected | Connection fails | Run bypass_utility first |
| Incompatible firmware | MTKClient fails | Downgrade via Realme OTA |

---

## Summary

The Realme C55 has no dedicated recovery partition and no useful flashing
interface in its recovery menu. All partition-level recovery is performed
externally through the MediaTek Preloader/BootROM interface using MTKClient or
SP Flash Tool.

SP Flash Tool is faster for bulk flashing operations. MTKClient is more
suitable for individual partition operations and scripted workflows. Both
require bypassing Preloader authentication on this device via bypass_utility.

Firmware packages are available from realmefirmware.com. When flashing stock
firmware, `nvram`, `nvcfg`, and `md1img` must always be excluded — these
contain device-specific IMEI and calibration data that cannot be recovered
from public sources once overwritten.

---
