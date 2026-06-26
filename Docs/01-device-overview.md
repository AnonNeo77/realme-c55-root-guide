# Device Overview

## Introduction

The Realme C55 is a MediaTek-based Android smartphone that uses Android's modern boot architecture, including Android Verified Boot (AVB) and Dynamic Partitions. This repository uses the device as a practical platform for documenting firmware analysis, boot image modification, rooting, recovery, and related Android internals.

This document provides an overview of the device hardware and software relevant to the remaining documentation.

---

## Device Specifications

| Component        | Information                  |
| ---------------- | ---------------------------- |
| Device           | Realme C55 (RMX3710)         |
| Manufacturer     | Realme                       |
| SoC              | MediaTek Helio G88 (MT6769H) |
| CPU              | Octa-core ARM64              |
| Architecture     | ARMv8-A (AArch64)            |
| GPU              | ARM Mali-G52 MC2             |
| Platform         | MediaTek                     |
| Operating System | Android 13                   |
| Kernel           | 4.19.191+                    |

---

## Android Software

The Realme C55 ships with Android and Realme UI, receiving firmware updates that include bootloader, kernel, vendor, modem, and security patch changes.

The exact firmware version is important because boot images, kernel modules, and other boot-critical partitions must match the installed firmware.

---

## Storage Layout

The device uses eMMC storage divided into multiple partitions.

Examples include:

* boot
* lk
* vbmeta
* vbmeta_system
* vbmeta_vendor
* super
* vendor_boot
* dtbo
* userdata
* metadata
* persist
* nvram
* nvdata
* seccfg

Many of these partitions are essential during firmware modification and recovery.

---

## Dynamic Partitions

Modern Android devices use Dynamic Partitions instead of fixed system partition layouts.

Rather than having fixed partitions such as `system`, `vendor`, and `product`, these logical partitions are stored inside a larger partition known as `super` that has super.img file .

The layout of these logical partitions is described by partition metadata, allowing manufacturers to resize partitions without changing the physical partition table.

---

## Android Verified Boot (AVB)

The device implements Android Verified Boot (AVB) to verify the integrity of boot-critical partitions during startup.

AVB validates components such as:

* boot
* vendor_boot
* vbmeta

If verification fails, Android may refuse to boot or display verification warnings depending on the bootloader state.

Later documents explain AVB and `vbmeta` in greater detail.

---

## Bootloader

The bootloader controls the device startup process and determines whether unsigned or modified images are permitted to boot.

Unlocking the bootloader allows boot images modified by tools such as Magisk to be flashed for root access.

---

## Why This Device?

The Realme C55 serves as the reference device throughout this repository because it provides practical examples of:

* Boot image extraction
* Firmware analysis
* Android Dynamic Partitions
* Android Verified Boot
* Magisk rooting
* Fastboot operations
* Recovery from common boot issues

The concepts described throughout this repository are also applicable to many other modern Android devices using similar boot architectures.

---

## Next Document

The next document explains the Android boot process used by the Realme C55, starting from the Boot ROM and continuing through the Linux kernel until Android userspace is initialized.
