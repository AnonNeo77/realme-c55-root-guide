# 12 - Kernel Modules and WiFi Adapter Drivers

## Overview

Loading out-of-tree kernel modules on the Realme C55 (RMX3710) is blocked by
kernel module signature enforcement. The kernel is built with
`CONFIG_MODULE_SIG_FORCE=y`, which causes the kernel to reject any module that
is not signed with the private key used during the original kernel build.

This document covers the signature enforcement mechanism, the implications for
driver development, the available paths to load custom modules, and the
workflow for building and loading an out-of-tree WiFi adapter driver.

---

## Verifying Signature Enforcement

Check enforcement state on a running rooted device:

```bash
adb shell su -c "cat /proc/sys/kernel/modules_disabled"
# 0 = loading allowed (but signature still required if FORCE is set)

adb shell su -c "zcat /proc/config.gz | grep MODULE_SIG"
```

Expected output on RMX3710:
CONFIG_MODULE_SIG=y

CONFIG_MODULE_SIG_FORCE=y

CONFIG_MODULE_SIG_ALL=y

CONFIG_MODULE_SIG_SHA256=y

Attempting to load an unsigned module:

```bash
adb shell su -c "insmod /data/local/tmp/test.ko"
# insmod: ERROR: could not insert module: Key was rejected by service
```

`EKEYREJECTED` — the kernel refuses the module outright. There is no runtime
override for `CONFIG_MODULE_SIG_FORCE=y` without modifying the kernel itself.

---

## How Module Signature Enforcement Works

During the kernel build, a keypair is generated:
certs/signing_key.pem   ← private key (signs modules)

certs/signing_key.x509  ← public key  (embedded into kernel image)

Every module built in-tree is signed with the private key by `scripts/sign-file`
at build time. The public key is baked into the kernel binary itself.

At load time, the kernel extracts the signature appended to the `.ko` file and
verifies it against the embedded public key. If verification fails or no
signature is present, with `CONFIG_MODULE_SIG_FORCE=y` the load is
unconditionally rejected.

The private key from the original Realme kernel build is not distributed.
Without it, modules cannot be signed in a way the stock kernel will accept.

---

## Available Paths

| Path | Requires | Tradeoff |
|------|----------|----------|
| Custom kernel (sig disabled) | Kernel source + unlocked bootloader | Loses stock kernel; must maintain |
| Custom kernel (own signing key) | Kernel source + unlocked bootloader | Full module signing control |
| Magisk module with prebuilt `.ko` | Prebuilt matching stock kernel exactly | No flexibility; kernel version locked |
| KernelSU with custom kernel | Kernel source + unlocked bootloader | Combines root + module loading cleanly |

Since the RMX3710 bootloader is already unlocked, flashing a custom kernel is
the practical path. There is no way to load unsigned modules on the stock
kernel without modifying it.

---

## Getting Kernel Source

Realme is obligated to release kernel source under GPLv2. Check:
https://github.com/realme-kernel-opensource

Search for `RMX3710` or `c55`. If not available at the time of writing, the
MediaTek kernel for the MT6769 platform (the SoC used in the RMX3710) may be
used as a base, though device-specific patches will be missing.

Clone the kernel source:

```bash
git clone https://github.com/realme-kernel-opensource/realme_C55-AndroidT-kernel-source.git
cd realme_C55-AndroidT-kernel-source
```

---

## Build Environment Setup

### Toolchain

```bash
# On Arch Linux
sudo pacman -S aarch64-linux-gnu-gcc

# Verify
aarch64-linux-gnu-gcc --version
```

Or use a dedicated Android kernel toolchain (Clang-based):

```bash
git clone https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 \
    --depth=1 --filter=blob:none --sparse
```

### Required packages

```bash
sudo pacman -S bc flex bison openssl libelf pahole --needed
```

### Kernel config for the RMX3710

```bash
# Use the defconfig for the device
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     realme_c55_defconfig

# Or extract config from running device
adb shell su -c "zcat /proc/config.gz" > .config
make ARCH=arm64 olddefconfig
```

---

## Disabling Signature Enforcement in the Kernel

Edit `.config` or use `menuconfig`:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

Navigate to:
Enable loadable module support

→ Module signature verification        [unset this]

→ Require modules to be validly signed [unset this]

Or directly in `.config`:

```bash
scripts/config --disable CONFIG_MODULE_SIG
scripts/config --disable CONFIG_MODULE_SIG_FORCE
scripts/config --disable CONFIG_MODULE_SIG_ALL
make ARCH=arm64 olddefconfig
```

Alternatively, keep signing enabled but use your own keypair (allows you to
sign your own modules while keeping the enforcement mechanism intact):

```bash
scripts/config --enable CONFIG_MODULE_SIG
scripts/config --enable CONFIG_MODULE_SIG_FORCE
scripts/config --set-str CONFIG_MODULE_SIG_KEY "certs/signing_key.pem"
```

The build process generates a new keypair. Keep `certs/signing_key.pem` — you
will need it to sign every module you build.

---

## Building the Custom Kernel

```bash
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     -j$(nproc) \
     Image.gz-dtb
```

The output kernel image is at `arch/arm64/boot/Image.gz-dtb`.

This needs to be packed into a boot image. Use `mkbootimg`:

```bash
mkbootimg \
    --kernel arch/arm64/boot/Image.gz-dtb \
    --ramdisk ramdisk.img \          # extracted from stock boot.img
    --base 0x40000000 \              # match stock boot.img parameters
    --pagesize 2048 \
    --output custom_boot.img
```

Extract base parameters from the stock boot image:

```bash
unpackbootimg -i stock_boot.img
# Outputs base, pagesize, ramdisk offset etc.
```

Flash via MTKClient:

```bash
python mtk w boot custom_boot.img
```

---

## Building an Out-of-Tree WiFi Adapter Driver

### Example: RTL8812AU (common USB WiFi adapter)

```bash
git clone https://github.com/aircrack-ng/rtl8812au.git
cd rtl8812au
```

### Module Makefile configuration

Set the kernel source path and cross-compilation variables:

```bash
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     KSRC=/path/to/realme_c55_kernel_source \
     -j$(nproc)
```

Output: `8812au.ko`

### General out-of-tree module Makefile structure

For a custom driver `wifidrv`:

```makefile
obj-m += wifidrv.o
wifidrv-objs := src/core.o src/usb.o src/hal.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) \
		ARCH=arm64 \
		CROSS_COMPILE=aarch64-linux-gnu- \
		M=$(PWD) \
		modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

Build against the custom kernel source:

```bash
make KDIR=/path/to/realme_c55_kernel_source
```

---

## Signing the Module (If Keeping Enforcement Enabled)

If the custom kernel was built with your own signing key, sign modules before
loading:

```bash
# From within the kernel source directory
scripts/sign-file sha256 \
    certs/signing_key.pem \
    certs/signing_key.x509 \
    /path/to/8812au.ko
```

The signature is appended directly to the `.ko` file. The file is modified
in-place.

Verify the signature was appended:

```bash
tail -c 28 8812au.ko | strings
# Should show: "~Module signature appended~"
```

---

## Loading the Module

Push the module to the device:

```bash
adb push 8812au.ko /data/local/tmp/
```

Load:

```bash
adb shell su -c "insmod /data/local/tmp/8812au.ko"
```

Verify it loaded:

```bash
adb shell su -c "lsmod | grep 8812au"
adb shell su -c "dmesg | tail -30"
```

Successful load output in dmesg:
[  timestamp  ] 8812au: loading out-of-tree module taints kernel.

[  timestamp  ] usbcore: registered new interface driver rtl8812au

Check the USB device is recognized after plugging the adapter (via USB OTG):

```bash
adb shell su -c "lsusb"
adb shell su -c "ip link show"
```

---

## Persistent Module Loading via Magisk

To load the module automatically on boot, package it as a Magisk module:
magisk_module/

├── META-INF/

│   └── com/google/android/

│       ├── update-binary     (Magisk module installer script)

│       └── updater-script    (contains "#MAGISK")

├── module.prop

└── system/

└── lib/

└── modules/

└── 8812au.ko

`module.prop`:
id=rtl8812au_driver

name=RTL8812AU WiFi Driver

version=v1

versionCode=1

author=0xNeo77

description=RTL8812AU out-of-tree driver for RMX3710

Or use a `post-fs-data.sh` script in the module to run `insmod` at the
correct boot stage:

```bash
# post-fs-data.sh
#!/system/bin/sh
insmod /data/adb/modules/rtl8812au_driver/system/lib/modules/8812au.ko
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Key was rejected by service` | Unsigned module on stock kernel | Use custom kernel |
| `Invalid module format` | Module built against wrong kernel version | Rebuild against exact kernel source |
| `Unknown symbol in module` | Kernel config mismatch | Enable required symbols in kernel config |
| `exec format error` | Wrong architecture (x86 instead of arm64) | Verify `ARCH=arm64` in build |
| Module loads but device not found | USB OTG not enabled or adapter unsupported | Check `dmesg` for USB enumeration errors |
| dmesg shows taint warning | Expected for out-of-tree modules | Informational only, not an error |

---

## Kernel Version Locking

Out-of-tree modules are tightly coupled to the exact kernel version they were
built against. The `vermagic` string embedded in every `.ko` must match the
running kernel:

```bash
modinfo 8812au.ko | grep vermagic
# vermagic: 5.10.x-android13-... SMP preempt mod_unload aarch64

adb shell su -c "uname -r"
# Must match
```

Any kernel update or custom kernel rebuild requires rebuilding all out-of-tree
modules. This is a hard constraint — mismatched `vermagic` is rejected before
signature verification even runs.

---

## Summary

Kernel module signature enforcement (`CONFIG_MODULE_SIG_FORCE=y`) on the
Realme C55 blocks all unsigned module loading at the kernel level with no
runtime override. The only viable path for loading custom drivers is replacing
the stock kernel with a custom build that either disables enforcement or embeds
a known signing key.

With an unlocked bootloader already in place, flashing a custom kernel via
MTKClient is straightforward. Once the custom kernel is running, out-of-tree
WiFi adapter drivers can be built against the kernel source, optionally signed,
and loaded via `insmod` or packaged as Magisk modules for persistence.

---
