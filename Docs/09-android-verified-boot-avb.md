# 09 - Android Verified Boot (AVB)

## Overview

Android Verified Boot (AVB) is a chain-of-trust mechanism that verifies the
integrity of each stage of the boot process before execution. On the Realme C55
(RMX3710), AVB is active by default and enforced by the bootloader.

Understanding AVB explains why certain behaviors occur after rooting — the
warning screen on boot, the color-coded boot states, and why Magisk patches the
boot image the way it does.

---

## How AVB Works

AVB establishes a chain of trust starting from the bootloader:

ROM (immutable) → Bootloader → boot.img → system/vendor partitions → Android

Each stage verifies the cryptographic signature of the next before handing off
execution. If any stage fails verification, the bootloader either halts or
warns the user depending on its lock state.

### Verification chain on RMX3710

Preloader

└── LK (Little Kernel bootloader)

└── boot.img (kernel + ramdisk)

└── vbmeta → system, vendor, product partitions

`vbmeta` is a dedicated partition containing hash trees and signature metadata
for all verified partitions. The bootloader validates `vbmeta` first, then uses
it to verify everything downstream.

---

## Boot States

AVB defines four boot states based on the bootloader lock state and
verification result:

| State   | Lock State | Verification | Behavior                              |
|---------|------------|--------------|---------------------------------------|
| GREEN   | Locked     | Passes       | Normal boot, no warning               |
| YELLOW  | Locked     | Custom key   | Warning shown, boots after delay      |
| ORANGE  | Unlocked   | Any          | Warning shown, boots after delay      |
| RED     | Locked     | Fails        | Boot halted or limited                |

After unlocking the bootloader on the RMX3710, the device operates in
**ORANGE** state permanently. This is expected and cannot be removed without
re-locking the bootloader.

---

## The Bootloader Warning Screen

Every boot after unlocking will display a warning screen before Android loads.
On the RMX3710 this appears as:

<img width="150" height="262" alt="orange-state" src="https://github.com/user-attachments/assets/32119f13-7cf7-4899-8f70-0b81b6e9ad69" />

Your device is unlocked and can't be trusted.

This is enforced at the bootloader level (LK). It is not a Realme software
overlay — it is displayed before Android starts. There is no supported method
to suppress this screen while keeping the bootloader unlocked.

The device continues booting automatically after a short delay.

---

## What Magisk Patches (and What It Doesn't)

Magisk's patching strategy is specifically designed to work within AVB
constraints on unlocked devices:

### What Magisk modifies

- The `ramdisk` embedded in `boot.img`
- Specifically, `init` is redirected to Magisk's init binary for early
  userspace setup
- Magisk's `su`, `magiskd`, and policy patches are injected at this stage

### What Magisk does not modify

- The Linux kernel itself
- `system`, `vendor`, or `product` partitions
- `vbmeta`
- Any partition outside of `boot`

Because the bootloader is unlocked (ORANGE state), AVB does not halt on a
modified `boot.img`. The signature check still fails, but the bootloader
proceeds with a warning rather than blocking boot.

On a **locked** bootloader, flashing a modified `boot.img` would result in
RED state and a boot failure — which is why bootloader unlock is a prerequisite.

<img width="1080" height="2400" alt="AVB-protection (1)" src="https://github.com/user-attachments/assets/820ff3f2-d0fd-47dc-9174-59793b9373e1" />

---

## vbmeta and Partition Verification

The `vbmeta` partition controls which partitions are verified and how:

```bash
adb shell su -c "avbctl get-verity"
# Shows current dm-verity state on system/vendor
```

On a rooted device with an unlocked bootloader, `dm-verity` is typically
disabled automatically by the bootloader for `system` and `vendor`. This allows
system-level tools to inspect these partitions without triggering verification
failures.

You can inspect the vbmeta structure from a dumped image:

```bash
avbtool info_image --image vbmeta.img
```

This outputs the algorithm, flags, and which partitions are chained under that
vbmeta descriptor.

---

## AVB and OTA Updates

With an unlocked bootloader:

- OTA updates may be blocked or fail signature verification at the installer
  level
- If an OTA installs successfully, it will overwrite the patched `boot.img`
  with a stock signed image
- After any OTA, the boot image must be repatched and reflashed to restore root

This is expected behavior. Always back up the stock `boot.img` from the new
firmware before repatching.

---

## dm-verity

`dm-verity` is the kernel-level component that enforces block-level integrity
on mounted partitions at runtime (primarily `system` and `vendor`).

On the RMX3710 with an unlocked bootloader, `dm-verity` is disabled for
affected partitions. You can confirm:

```bash
adb shell su -c "cat /proc/mounts | grep dm-"
# dm- entries indicate active dm-verity mapped partitions
```

```bash
adb shell su -c "dmesg | grep verity"
# Kernel log entries showing verity initialization state
```

If `dm-verity` were enforced on a modified system partition, the kernel would
detect a hash mismatch and force a reboot into recovery.

---

## Key Takeaways for the RMX3710

- AVB is enforced by the LK bootloader, not by Android
- Bootloader unlock moves the device to ORANGE state permanently
- The boot warning is a bootloader-level display and cannot be bypassed
  while unlocked
- Magisk survives AVB on an unlocked device because ORANGE state permits
  booting unverified images with a warning
- OTA updates overwrite the patched boot image — repatch after every update
- `dm-verity` is disabled on unlock; system and vendor partitions are not
  actively verified at runtime

---

## Summary

AVB on the Realme C55 operates as a standard MediaTek implementation of the
AOSP AVB2.0 specification. The chain of trust runs from Preloader through LK
to `vbmeta` and downstream partitions. Bootloader unlock breaks the GREEN state
permanently, placing the device in ORANGE — which allows modified boot images
to load with a warning rather than failing hard.

Magisk's systemless design is a direct response to how AVB works: by limiting
modifications to the ramdisk within `boot.img` and operating entirely from
tmpfs at runtime, it avoids touching any partition that AVB actively verifies
post-unlock.

---

## Next

`10-dynamic-partitions.md` — How the Realme C55 implements dynamic partitions
(super.img), what this means for partition layout, and how it relates to
firmware unpacking and analysis.
