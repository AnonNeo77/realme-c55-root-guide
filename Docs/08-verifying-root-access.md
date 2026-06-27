# 08 - Verifying Root Access

## Overview

After the device boots with the patched `boot.img`, root access needs to be
confirmed at multiple levels — Magisk initialization, `su` binary availability,
and the integrity of the environment itself.

This document covers verification at the application layer, shell layer, and
Magisk internals.

<p align="center">
  <img src="https://github.com/user-attachments/assets/cc162a0a-8d1b-458e-b294-023c77d25415"
       width="350">
</p>

---

## Magisk App Verification

Open the Magisk application. The main screen should report:

<p align="center">
  <img src="https://github.com/user-attachments/assets/fb2e1511-7dae-44da-a3b2-5930301331ae"
       width="300">
</p>




| Field             | Expected Value                        |
|-------------------|---------------------------------------|
| Magisk            | Installed (version number)            |
| App               | Matches installed APK version         |
| Ramdisk           | Yes                                   |
| Zygisk            | On / Off (depending on your config)   |

If Magisk shows **"Install"** rather than a version number, the patched boot
image did not initialize correctly. Re-check the flashing step.

If **Ramdisk** shows `No`, the device does not support ramdisk-based Magisk
patching — this should not occur on the RMX3710 but indicates a patching error
if it does.

---

## Shell-Level Verification

### ADB root check

```bash
adb shell
whoami
# Expected: shell

su
whoami
# Expected: root
```

If `su` returns `Permission denied` or is not found, Magisk did not initialize
the `su` binary correctly. Check Magisk logs.

### Confirming su binary path

```bash
adb shell su -c "which su"
# Expected: /system/bin/su or Magisk's bind-mounted path
```

Magisk mounts `su` systemlessly — the binary is injected without modifying
`/system`. You can confirm the mount origin:

```bash
adb shell su -c "cat /proc/mounts | grep magisk"
```

This should show Magisk's tmpfs overlays active in the mount table.

---

## Magisk Environment Internals

### Magisk tmpfs mount

```bash
adb shell su -c "mount | grep magisk"
```

Magisk operates entirely from a tmpfs mount, leaving the system partition
untouched. If this mount is absent, Magisk is not running correctly.

### MagiskSU daemon

```bash
adb shell su -c "ps -A | grep magiskd"
```

`magiskd` is the root daemon that handles `su` requests from applications.
It should be running as root with a short uptime after boot.

### Magisk version via CLI

```bash
adb shell su -c "magisk -v"
# Returns: version string e.g. 27.0:MAGISK
adb shell su -c "magisk -V"
# Returns: version code e.g. 27000
```

---

## SELinux State

Magisk operates under enforcing SELinux by default through policy patching.
Verify the current SELinux context:

```bash
adb shell getenforce
# Expected: Enforcing
```

If the result is `Permissive`, SELinux has been globally disabled — this is not
how Magisk operates and suggests a misconfiguration or a different root method
was applied. Magisk patches SELinux policy rather than disabling enforcement.

---

## SafetyNet / Play Integrity (Optional)

If you need the device to pass Play Integrity checks (for banking apps, etc.),
run the **YASNAC** or **Play Integrity API Checker** app after installation.

Basic Integrity should pass with Magisk's default DenyList enabled. Strong
Integrity requires additional modules (e.g., `shamiko`, `tricky-store`) and is
outside the scope of this document.

---

## Installed Modules

If any Magisk modules were installed:

```bash
adb shell su -c "magisk --list"
```

Lists all installed modules. Confirm expected modules are active and none are
in a disabled or failed state.

Modules that fail to load are automatically disabled on next boot to prevent
boot loops.

---

## Failure States

| Observation | Cause | Action |
|---|---|---|
| Magisk app shows "Install" | Boot image not initialized | Re-flash patched image |
| `su` not found in shell | Magisk daemon not running | Check Magisk logs in app |
| `magiskd` not in process list | Early init failure | Check ramdisk patch |
| SELinux is Permissive | Policy patch failed or wrong method | Repatch boot image |
| Device reboots on `su` | Module conflict or daemon crash | Boot to safe mode, disable modules |

### Safe mode boot

If a module is causing instability, hold **Volume Down** during boot. Magisk
detects this and disables all modules for that boot cycle only. No data is lost
and modules remain installed.

---

## Summary

A correctly rooted Realme C55 (RMX3710) will show:

- Magisk reporting as installed with a valid version number
- `su` accessible from ADB shell
- `magiskd` running as root in the process table
- Magisk tmpfs mounts visible in `/proc/mounts`
- SELinux in **Enforcing** state (policy-patched, not disabled)
- System partition unmodified

---

## Next

`09-android-verified-boot.md` — How AVB works on the Realme C55, what changes
after bootloader unlock, and what the persistent warning screen means at the
bootloader level.
