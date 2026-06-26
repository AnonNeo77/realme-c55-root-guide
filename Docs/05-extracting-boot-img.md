# Extracting boot.img from Realme C55 Firmware

## Introduction

Before a device can be rooted using Magisk, the original `boot.img` must be extracted from the official firmware package.

On the Realme C55, the boot image is not downloaded separately. Instead, it is distributed as part of the complete firmware package provided by Realme.

This document explains where `boot.img` comes from, how it is extracted, and why using the correct version is important.

---

## Why boot.img?

Unlike traditional Android modifications that altered the system partition, modern rooting methods modify the boot image.

The boot image contains:

* Linux kernel
* Generic ramdisk
* Boot configuration

Magisk patches this image to inject its initialization code while leaving the system partition unchanged.

---

## Obtaining the Firmware

The first step is downloading the official firmware for the exact Realme C55 software version installed on the device.

Using firmware from a different Android version, security patch level, or regional build may result in:

* Boot loops
* AVB verification failures
* Kernel incompatibilities
* Failed OTA updates

Always verify that the firmware matches the software currently installed on the device.

---

## Extracting the OFP Package

Realme distributes firmware as an **OFP** package.

After extraction, the package typically contains:

```text
preloader_k69v1_64_k419.bin
MT6768_Android_scatter.txt

boot.img
vbmeta.img
vbmeta_system.img
vbmeta_vendor.img

dtbo.img
lk.img
tee.img

super.img
userdata.img
...
```

At this stage, `boot.img` is available for analysis or patching.

---

## Other Sources of `boot.img`

Although the official firmware package is the recommended source, the original `boot.img` can also be obtained directly from the device, depending on the available bootloader interfaces and tools.

Common sources include:

* **Official Realme firmware (OFP)** after extracting the firmware package.
* **MTKClient**, by dumping the `boot` partition from supported MediaTek devices.
* **Fastboot**, on devices where the standard bootloader Fastboot interface is available.
* **Firmware backups** created before modifying the device.

Regardless of the source, the extracted `boot.img` should match the exact firmware version currently installed on the device. Using a boot image from a different software version or regional build may result in boot failures, Android Verified Boot (AVB) verification errors, or kernel compatibility issues.

---

## Verifying the Image

Before modifying the image, confirm that:

* It matches the installed firmware version.
* It originates from official firmware.
* It has not been modified previously.

Using the stock boot image reduces the risk of compatibility issues.

---

## Understanding boot.img

The extracted image is a complete Android boot image.

Internally it contains:

* Linux kernel
* Ramdisk
* Boot header
* Boot configuration

Later documents examine these components in greater detail.

---

## Common Mistakes

Using an incorrect boot image is one of the most common causes of rooting failures.

Examples include:

* Extracting `boot.img` from another firmware version.
* Using a boot image from another device variant.
* Flashing a previously modified image.
* Mixing Android versions.

---

## Summary

Extracting the correct `boot.img` is one of the most important steps in the rooting process.

Every later modification—including Magisk patching—depends on having the original boot image from the exact firmware version installed on the device.

---

## Next Document

The next document examines the internal structure of `boot.img`, including the boot header, Linux kernel, ramdisk, and how Magisk modifies the image during the rooting process.
