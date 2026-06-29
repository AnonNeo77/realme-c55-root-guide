# Bootloader & Fastboot

## Introduction

The bootloader is responsible for preparing the Android operating system for execution. It initializes the hardware, verifies the integrity of boot-critical components, and loads the Linux kernel.

Unlike many Android devices, the Realme C55 has several device-specific bootloader behaviors that affect rooting and firmware modification.

This document explains the bootloader, Fastboot mode, official unlocking methods, and the Realme-specific implementation.

---

## What is a Bootloader?

A bootloader is low-level software executed immediately after the device's early boot stages.

Its responsibilities include:

* Initializing hardware required for boot
* Verifying boot-critical partitions
* Loading the Linux kernel
* Passing boot parameters
* Providing firmware update interfaces
* Displaying boot warnings

Without a functioning bootloader, Android cannot start.

---

## Locked vs Unlocked Bootloader

### Locked Bootloader

A locked bootloader only allows images verified by the manufacturer to boot.

Characteristics include:

* Verified boot enabled
* Only officially signed images are accepted
* Boot image modification is blocked
* System integrity is enforced

This is the default state for retail devices.

---

### Unlocked Bootloader

Unlocking the bootloader allows modified boot images and other custom software to be flashed.

After unlocking:

* Custom boot images can be flashed
* Root solutions such as Magisk become possible
* Android displays a bootloader warning during startup
* User data is typically erased as part of the unlocking process

Unlocking the bootloader does **not** root the device by itself. It simply removes one restriction that prevents modified images from booting.

---

## Fastboot Mode

Fastboot is a bootloader interface used to communicate with the device over USB.

Common operations include:

* Flashing partitions
* Booting temporary images
* Unlocking the bootloader (supported devices)
* Reading bootloader variables
* Erasing partitions

Fastboot is commonly accessed from a computer using the Android Platform Tools.

---

## Fastboot on the Realme C55

Unlike many Android devices, the Realme C55 does not expose the standard Fastboot interface on retail firmware by default.

<p align="left">
  <img src="https://github.com/user-attachments/assets/8365ad01-9b60-4ede-b88b-451d040a9a47"
       width="350">
</p>

Historically, Realme provided a **Deep Testing** application for selected devices. This application initiated the official bootloader unlocking process and enabled Fastboot mode on supported models.

However, Deep Testing was not released for every Realme device or firmware version. As a result, official bootloader unlocking support varies depending on the device model and software version.

Because of these differences, community research has explored the Realme bootloader implementation and its interaction with the Little Kernel (LK) bootloader to better understand Fastboot behavior on unsupported devices.

This repository focuses on documenting the architecture and behavior of the bootloader rather than providing instructions for modifying it.

---

## Fastbootd on the Realme C55

The Realme C55 provides access to Fastbootd through the recovery environment.

Fastbootd supports many partition management operations, including:

Flashing logical partitions
Flashing boot
Flashing vendor_boot
Flashing vbmeta
Flashing dtbo
Erasing supported partitions
Rebooting between boot modes

However, Fastbootd cannot unlock the bootloader, because bootloader unlocking is a function of the bootloader itself rather than Android userspace.

---

## Little Kernel (LK)

Many MediaTek devices use **Little Kernel (LK)** as the second-stage bootloader.

LK is responsible for tasks such as:

* Hardware initialization
* Display initialization
* Boot mode selection
* Fastboot support (when enabled)
* Loading the Linux kernel
* Passing boot arguments

Understanding LK is useful when studying the Android boot process because it forms the bridge between the early bootloader stages and the Linux kernel.

---

## Orange State

After unlocking the bootloader, many Realme devices display an **Orange State** warning during startup.

<img width="150" height="262" alt="orange-state" src="https://github.com/user-attachments/assets/f5fd4df4-937d-45b9-a014-b2786aad5820" />

This warning indicates that the bootloader is unlocked and the integrity of the software can no longer be guaranteed by the manufacturer.

The warning is expected behavior and does not necessarily indicate a problem with the device.

---

## Risks of Unlocking

Unlocking the bootloader may result in:

* Complete user data wipe
* Reduced device security
* Boot verification warnings
* Failed OTA updates in some cases
* Warranty implications depending on manufacturer policies

Before modifying firmware, it is recommended to create backups of important data.

---

## Summary

Understanding the bootloader helps explain why:

* Rooting requires an unlocked bootloader.
* Fastboot is important for firmware modification.
* Bootloader warnings appear after unlocking.
* Android Verified Boot protects the boot chain.
* Manufacturer implementations differ across devices.

These concepts are essential before modifying `boot.img`, `vbmeta`, or other boot-critical partitions.

---

## Next Document

The next document explains Android firmware packages, partition images, and how files such as `boot.img`, `vbmeta`, and `super.img` are obtained from official firmware.
