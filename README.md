# Realme C55 (RMX3710) — Root & Firmware Analysis Guide

Documentation for rooting, bootloader analysis, and firmware research on the
Realme C55 (RMX3710). Covers the full process from stock firmware to rooted
device, including the MediaTek-specific toolchain required, AVB internals,
dynamic partition structure, and first-hand recovery from a hard brick.

<img width="1600" height="1600" alt="relame-c55" src="https://github.com/user-attachments/assets/6e52e47b-3005-4d03-8bb3-73a05e3b1e9c" />

This is not a generic Android rooting guide. Everything here is specific to
the RMX3710 hardware, its MediaTek MT6769 SoC, and the firmware constraints
introduced across Android 13, 14, and 15 releases.

---

## Device

| Field      | Value                        |
|------------|------------------------------|
| Model      | Realme C55                   |
| Codename   | RMX3710                      |
| SoC        | MediaTek Helio G88 (MT6769)  |
| Android    | Ships Android 13 / 14 / 15   |
| Bootloader | Not exposed via standard Fastboot |
| Recovery   | No dedicated recovery partition |

---

## Prerequisites

- Unlocked bootloader (covered in [07](Docs/07-Flashing-the-Patched-Image.md))
- Device on Android 13 or Android 14 pre-1800 build (MTKClient compatibility)
- Python 3.x on Linux (Arch Linux used throughout this documentation)
- USB cable with data support
- Battery above 60% before any flashing operation

> If the device is on Android 14 (≥1800 build) or Android 15, downgrading
> to Android 13 is required before MTKClient can connect. This is covered in
> the flashing document.

---

## Documentation

### Device Architecture

| # | Document | Description |
|---|----------|-------------|
| 01 | [Device Overview](Docs/01-device-overview.md) | Hardware specs, partition layout, SoC overview |
| 02 | [Boot Chain](Docs/02-boot-chain.md) | Preloader → LK → kernel boot sequence |
| 03 | [Bootloader](Docs/03-bootloader.md) | LK internals, lock states, available interfaces |
| 04 | [Stock Firmware](Docs/04-stock-firmware.md) | Firmware structure, version identification, sources |

### Rooting

| # | Document | Description |
|---|----------|-------------|
| 05 | [Extracting boot.img](Docs/05-extracting-boot-img.md) | Pulling stock boot image from firmware package |
| 06 | [Magisk and Boot Image](Docs/06-magisk-and-boot-image.md) | Patching boot.img with Magisk |
| 07 | [Flashing the Patched Image](Docs/07-Flashing-the-Patched-Image.md) | Downgrade path, MTKClient unlock, flashing |
| 08 | [Verifying Root Access](Docs/08-verifying-root-access.md) | Magisk verification, su access, SELinux state |

### Internals

| # | Document | Description |
|---|----------|-------------|
| 09 | [Android Verified Boot (AVB)](Docs/09-android-verified-boot-avb.md) | AVB chain, boot states, dm-verity |
| 10 | [Dynamic Partitions and super.img](Docs/10-dynamic-partitions-and-super.md) | liblp format, lpunpack, lpmake, erofs |
| 11 | [OTA Updates After Root](Docs/11-ota-updates-after-root.md) | OTA behavior on rooted device, repatching |
| 12 | [Kernel Modules](Docs/12-kernel-modules.md) | Module signature enforcement, custom kernel, WiFi drivers |

### Recovery

| # | Document | Description |
|---|----------|-------------|
| 13 | [Recovery and Unroot](Docs/13-recovery-and-unroot.md) | MTKClient/SP Flash Tool recovery, partition restore |
| 15 | [Lessons Learned](Docs/15-lessons-learned.md) | Hard brick, BootROM recovery, IMEI loss and restoration |
| 16 | [Tools Used](Docs/16-tools-used.md) | Full tool reference with commands and device-specific notes |

---

## Rooting Overview

Standard Android rooting via Fastboot does not work on the RMX3710. The
bootloader Fastboot interface is not exposed on retail firmware. 

See [07 - Flashing the Patched Image](Docs/07-Flashing-the-Patched-Image.md)
for the full workflow including firmware compatibility details.

---

## Critical Warnings

**nvram / nvcfg / md1img — back these up before touching anything**

These three partitions contain device-specific IMEI, modem calibration, and
RF tuning data written at the factory. They are unique to each physical unit
and cannot be recovered from any firmware package.

```bash
python mtk r nvram nvram_backup.img
python mtk r nvcfg nvcfg_backup.img
python mtk r md1img md1img_backup.img
```

Losing these without a backup means either a dead modem or a paid IMEI
restoration service. See [15 - Lessons Learned](Docs/15-lessons-learned.md)
for the full account of what happens when you skip this step.

**Never flash a scatter file without unchecking nvram, nvcfg, and md1img.**

**Do not run a full partition erase as a recovery step.** A failed erase is
harder to recover from than a bad flash. Always target specific partitions.

---

## Firmware

Official Realme C55 firmware packages (including downgrade OTAs):

**https://realmefirmware.com/realme-c55-firmware/**

Match the firmware build number exactly to the version installed on the device
before extracting any partition image for patching.

---

## Tools

| Tool | Purpose |
|------|---------|
| [MTKClient](https://github.com/bkerler/mtkclient) | Partition R/W, bootloader unlock |
| [bypass_utility](https://github.com/MTK-bypass/bypass_utility) | Preloader crash → BootROM access |
| [SP Flash Tool](https://spflashtool.com) | Bulk firmware flash, BootROM recovery |
| [Magisk](https://github.com/topjohnwu/Magisk) | Systemless root |
| [MTK-SuperBuilder](https://github.com/AnonNeo77/MTK-SuperBuilder) | MediaTek super.img pack/unpack |

Full tool reference with commands: [16 - Tools Used](Docs/16-tools-used.md)

---

## Disclaimer

This documentation is for educational and security research purposes on
hardware I own. Rooting voids the device warranty. Bootloader unlock triggers
a factory reset and permanently places the device in ORANGE AVB state.
Incorrect partition operations can brick the device.

Everything in this repository reflects operations performed on my own
Realme C55 (RMX3710). Your results may differ based on firmware version,
hardware revision, or build variant.

---

## Author

**0xNeo77** — [github.com/AnonNeo77](https://github.com/AnonNeo77)

Security researcher. Offensive security, firmware analysis, Android internals.
