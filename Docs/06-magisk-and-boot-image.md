# Magisk and Boot Image Patching

## Introduction

After obtaining the original `boot.img`, the next step is preparing it for root access.

The Realme C55 uses a modern Android boot architecture where root is achieved by modifying the boot image rather than the Android system partition. This approach is known as **systemless root**.

Magisk patches the original `boot.img` by injecting its own initialization code while preserving the Linux kernel and maintaining compatibility with the installed firmware.

---

## Prerequisites

Before patching the boot image, ensure the following requirements are met:

* Bootloader unlocked
* Original `boot.img` extracted from the installed firmware
* Latest Magisk application installed
* Sufficient storage space

Using the stock `boot.img` from the exact firmware version installed on the device is strongly recommended.

---

## Opening Magisk

Launch the Magisk application.

Initially, Magisk reports that it is **not installed**, since the patched boot image has not yet been flashed to the device.


<p align="center">
  <img src="https://github.com/user-attachments/assets/1e7db8a1-4e64-46a8-a05b-ae8b6f783abc" width="300">
</p>

<p align="center">
</p>

---

## Selecting the Boot Image

Press **Install** next to the Magisk section.

Choose:

```text
Select and Patch a File
```

Browse to the previously extracted `boot.img` and select it.

Magisk reads the image and prepares it for modification.

<img width="1080" height="2400" alt="select_boot img" src="https://github.com/user-attachments/assets/0a1e4515-d1dc-4aba-80d4-cd55104fb898" />


---

## Patching Process

After selecting the boot image, Magisk performs several operations automatically.

Typical output includes:

* Detecting device architecture
* Unpacking the boot image
* Checking ramdisk status
* Detecting the stock boot image
* Patching the ramdisk
* Repacking the boot image

Unlike older rooting methods, Magisk does **not** modify the Android system partition.

Instead, it modifies the boot image so its initialization code executes during the early boot process.

<img width="1080" height="2400" alt="patching" src="https://github.com/user-attachments/assets/c26d7f0c-2477-4f46-a85d-15e6fa28bf85" />

---

## Generated Output

When patching completes successfully, Magisk generates a new boot image.

Example:

```text
magisk_patched-30700_xxxxx.img
```

The patched image is typically saved in:

```text
Internal Storage/
└── Download/
    └── magisk_patched-xxxxx.img
```

This image replaces the original boot image during the rooting process.

---

## What Changes Inside the Boot Image?

Magisk does **not** replace the Linux kernel.

Instead, it modifies the boot image by:

* Preserving the original Linux kernel
* Modifying the ramdisk
* Injecting Magisk initialization components
* Repacking the boot image
* Generating a flashable patched image

The Android system partition remains unchanged.

---

## Why Patch the Stock Boot Image?

Every firmware release contains a boot image built specifically for its kernel, vendor components, and Android version.

Using the stock image from the currently installed firmware helps avoid:

* Boot loops
* Android Verified Boot (AVB) verification failures
* Kernel incompatibilities
* Failed OTA updates

For this reason, the original boot image should always be obtained from the same firmware version running on the device.

---

## Summary

At this stage, the Realme C55 has **not yet been rooted**.

The result of this process is a patched boot image that is ready to be flashed back to the device.

The next document explains how to flash the patched boot image using the available bootloader interfaces on the Realme C55 and verify that root access has been installed successfully.


