# Firmware Layout (Realme C55)

## Introduction

The Realme C55 firmware package contains every component required to boot the device, including bootloaders, Android partition images, modem firmware, and configuration files.

Unlike Google Pixel devices, the Realme C55 is based on the **MediaTek MT6768 (Helio G88)** platform and distributes its firmware as an **OFP package**. Once extracted, the firmware contains a MediaTek scatter file together with all partition images required for flashing.

The scatter file identifies the platform as **MT6768**, project **k69v1_64_k419**, and specifies that the device uses **eMMC** storage.

Understanding this firmware layout is essential before attempting to analyze, modify, root, or recover the device.

---

## Firmware Package Structure

After extracting the official firmware package, the Realme C55 contains files similar to:

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
md1img.img
spmfw.img
scp.img
sspm.img
gz.img

super.img
userdata.img
logo.bin
```

Each image corresponds to a partition defined inside the scatter file.

---

## Scatter File

The Realme C55 firmware includes a MediaTek scatter file named similar to:

```text
MT6768_Android_scatter.txt
```

Unlike a GPT partition table, the scatter file is a flashing configuration used by tools such as **SP Flash Tool**.

It defines:

* Platform information
* Storage type
* Partition names
* Flash regions
* Start addresses
* Partition sizes
* Image filenames

For example, the scatter file defines the **preloader** partition, its associated image (`preloader_k69v1_64_k419.bin`), and specifies that it resides in the **EMMC_BOOT1_BOOT2** boot region rather than the main user storage.

---

## Preloader

The first flashable component in the firmware package is:

```text
preloader_k69v1_64_k419.bin
```

The Boot ROM loads this image during the earliest stage of the boot process.

The Preloader is responsible for:

* Initializing DRAM
* Configuring clocks
* Initializing eMMC
* Loading Little Kernel (LK)

Because it initializes the hardware, the Preloader must match the exact hardware platform and firmware version. Flashing an incompatible Preloader can prevent the normal boot process.

---

## Little Kernel (LK)

The firmware also contains:

```text
lk.img
```

Little Kernel (LK) is the second-stage bootloader.

On the Realme C55, LK initializes additional hardware, displays boot warnings, and loads the Linux kernel from `boot.img`.

It also controls the availability of bootloader Fastboot, which is not exposed by default on retail firmware.

---

## boot.img

The `boot.img` partition contains:

* Linux kernel
* Generic ramdisk
* Boot configuration

Magisk patches this image to inject its initialization code while leaving the Android system partition unchanged.

The scatter file maps `boot.img` to the `boot_a` partition.

---

## vbmeta

The Realme C55 uses multiple AVB metadata partitions:

* vbmeta
* vbmeta_system
* vbmeta_vendor

These partitions contain Android Verified Boot metadata used to verify boot-critical components before Android starts. The scatter file defines these partitions separately.

---

## super.img

Instead of storing `system`, `vendor`, and `product` as separate physical partitions, the Realme C55 uses Android Dynamic Partitions.

These logical partitions are stored inside `super.img`.

The scatter file defines `super` as a dedicated partition, while the logical layout inside the image is described by dynamic partition metadata.

---

## NVRAM and NVDATA

The Realme C55 stores modem calibration and radio configuration in dedicated partitions such as:

* nvram
* nvdata

These partitions contain device-specific information including IMEI and radio calibration.

Corruption or accidental erasure may result in:

* Missing IMEI
* Unknown Baseband
* Loss of cellular connectivity

---

## Summary

The firmware package for the Realme C55 consists of far more than Android itself. It includes multiple bootloaders, firmware components, modem images, security metadata, and partition images that work together to initialize the device.

Understanding the purpose of each component makes it easier to understand later topics such as boot image patching, Android Verified Boot, Magisk, and firmware recovery.

---

## Next Document

The next document explains how `boot.img` is extracted from the official Realme C55 firmware package and prepared for analysis or patching.
