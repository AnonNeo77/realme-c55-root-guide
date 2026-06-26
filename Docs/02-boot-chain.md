# Android Boot Chain

## Introduction

Before modifying firmware or rooting an Android device, it is important to understand how the device boots.

The Android boot process is a sequence of bootloaders and software components that initialize the hardware, verify system integrity, load the Linux kernel, and finally start Android userspace.

Although implementation details vary between manufacturers, the overall boot chain is similar across modern Android devices.

---

## Boot Chain Overview

```
Power Button
      │
      ▼
Boot ROM (BROM)
      │
      ▼
Preloader
      │
      ▼
Little Kernel (LK)
      │
      ▼
Android Verified Boot (AVB)
      │
      ▼
boot.img
      │
      ▼
Linux Kernel
      │
      ▼
init
      │
      ▼
Android Framework
      │
      ▼
Home Screen
```

Each stage prepares the system for the next stage while maintaining the device's security model.

---

## 1. Boot ROM (BROM)

The Boot ROM is the first code executed after the device receives power.

It is permanently stored inside the MediaTek SoC and cannot be modified through software updates.

Responsibilities include:

* Initial hardware initialization
* Loading the Preloader from storage
* Providing recovery functionality for firmware flashing
* Verifying the next boot stage (device dependent)

If the Preloader cannot be loaded successfully, execution remains in Boot ROM mode.

---

## 2. Preloader

The Preloader is the first software bootloader stored on the device's storage such as boot1 and boot2 .

Responsibilities include:

* Initializing DRAM
* Configuring clocks
* Initializing storage controllers
* Loading the Little Kernel (LK)

Without a working Preloader, the device cannot continue the normal boot process.

---

## 3. Little Kernel (LK)

Little Kernel (LK) is the second-stage bootloader used on many MediaTek devices.

Its responsibilities include:

* Initializing additional hardware
* Displaying boot warnings
* Starting Fastboot (when available)
* Loading the Linux kernel from `boot.img`
* Passing boot parameters to the kernel

On modern Realme devices, Fastboot is not exposed on retail firmware by default. Bootloader behavior varies between firmware versions and device models.

---

## 4. Android Verified Boot (AVB)

Before the Linux kernel is started, Android Verified Boot (AVB) verifies the integrity of boot-critical partitions.

AVB uses `vbmeta` metadata to validate partitions such as:

* boot
* vendor_boot
* dtbo
* vbmeta chained partitions

If verification fails, the device may refuse to boot or display a boot verification warning depending on the bootloader state.

---

## 5. boot.img

The `boot.img` partition contains the components required to start Android.

Typical contents include:

* Linux kernel
* Generic ramdisk
* Boot configuration

Magisk modifies this image ramdisk to provide root access while preserving the Android system partition.

---

## 6. Linux Kernel

After verification succeeds, the Linux kernel begins execution.

The kernel is responsible for:

* Memory management
* Process scheduling
* Hardware drivers
* Filesystems
* Security mechanisms
* Loading the initial userspace process

Once initialization is complete, the kernel starts `init`.

---

## 7. init

`init` is the first userspace process (PID 1).

Responsibilities include:

* Mounting partitions
* Starting Android services
* Reading initialization scripts
* Applying system properties
* Launching framework services

Every Android process ultimately originates from this stage.

---

## 8. Android Userspace

After `init` completes the early boot sequence, Android starts framework services, system applications, and finally presents the lock screen or launcher.

At this point, the device has completed the boot process.

---

## Why Understanding the Boot Chain Matters

Understanding the Android boot chain helps explain why:

* Patching `boot.img` enables root access.
* AVB prevents unauthorized modifications.
* `vbmeta` is required for verified boot.
* A corrupted bootloader or boot image can prevent Android from starting.
* Firmware components must match the installed software version.

These concepts form the foundation for the remaining documents in this repository.

---

## Next Document

The next document covers the Android bootloader, Fastboot mode, and Realme's bootloader implementation, including official unlocking methods and device-specific behavior.
