# 13 - Lessons Learned: From Rooting to Hard Brick and Back

## Overview

This document is a first-person account of the full rooting process on my
Realme C55 (RMX3710) — what I did, what went wrong, how the device hard
bricked during a partition erase operation, how I recovered it via BootROM,
and how I dealt with IMEI loss afterward.

This exists because most guides describe the happy path. This is the other one.

---

## Starting Point

- Device: Realme C55 (RMX3710)
- Firmware at start: Android 14 (build ≥1800 — MTKClient already patched)
- Bootloader: Locked
- Goal: Root access for firmware analysis and security research

The first problem was immediate — MTKClient could not connect. Preloader
authentication on Android 14 (≥1800 build) rejected every connection attempt.

---

## Stage 1: Downgrading to Android 13

Since MTKClient was blocked on Android 14, the only path was downgrading to
Android 13 where the Preloader interface was still accessible.

Downgrade OTA packages were obtained from the Realme community. The path
required two steps:
Android 14 (≥1800) → Android 14 (compatible build) → Android 13

Each step wiped the device. After reaching Android 13, MTKClient detected the
device successfully.

---

## Stage 2: Bootloader Unlock

```bash
python mtk da seccfg unlock
```

Device rebooted into ORANGE AVB state. Android performed an automatic factory
reset. Bootloader was now unlocked.

---

## Stage 3: Patching and Flashing Boot Image

Extracted `boot.img` from the stock Android 13 firmware, patched with Magisk,
flashed via MTKClient:

```bash
python mtk w boot magisk_patched.img
```

Device booted. Magisk initialized. Root confirmed.

---

## Stage 4: Where It Went Wrong

Flashing the custom ROM left the device in bootloop .

**First recovery attempt:** erase the `super` partition and reflash it from
the firmware package. That partially worked. The device powered on — but shut
itself off automatically every time, stuck in a power-on/power-off loop.

At that point I panicked. The logic seemed sound: erase every partition and
reflash the complete firmware package to get back to a guaranteed clean state.

That decision made things significantly worse.

**First:** The firmware package does not contain device-specific partition
images. `nvram` and `nvcfg` store modem calibration data and the IMEI written
to the specific unit at manufacture. These are not included in the firmware
package because they cannot be generalized across units. Erasing them and
reflashing firmware leaves those partitions either empty or populated with
generic factory defaults.

**Second:** Erasing all partitions via MTKClient is not equivalent to
reflashing clean firmware. Partitions such as `nvram` and `nvcfg` must exist
with device-specific data intact. Once erased, the firmware package has no
replacement images for them.

The device booted after the reflash. But the IMEI was gone — the baseband
showed as unknown, and cellular connectivity was lost entirely.

The actual sequence of mistakes:

- Custom ROM flash → broke boot, device unresponsive
- Erase `super` →  persistent bootloop
- Panic erase all partitions → `nvram` and `nvcfg` destroyed
- Firmware reflash → device boots, IMEI lost, baseband dead

The bootloop after erasing `super` was almost certainly recoverable without
touching `nvram` or `nvcfg` at all. The erase-everything decision turned a
software problem into permanent calibration/IMEI data loss.
---

## Stage 5: Diagnosing the Brick

The key question: was BootROM still accessible?

BootROM is embedded in the MediaTek SoC itself and cannot be erased or
overwritten by any software operation. If the SoC was physically intact,
BootROM would still respond over USB regardless of what happened to the
eMMC contents.

Tested BootROM detection:

```bash
# Power off, connect USB — no button held
python mtk printgpt
```

mtkclient detects my device. 

---

## Stage 6: BootROM Recovery via bypass_utility and SP Flash Tool

With BootROM accessible, SP Flash Tool was used for recovery. Getting SP Flash
Tool to connect reliably required bypassing Preloader authentication first via
bypass_utility, then using an alternative DA file:

```bash
git clone https://github.com/MTK-bypass/bypass_utility.git
cd bypass_utility
pip install -r requirements.txt --break-system-packages
python bypass_utility.py
```

After the bypass dropped the device into BootROM mode, SP Flash Tool connected
using `MTK_all-in-one DA`. 

the lk partition should be erased for the tool to work!

Loaded the scatter file from the correct Android 13 firmware package.

**Before flashing — unchecked these partitions:**

nvram     ← do not flash from firmware package

nvcfg     ← do not flash from firmware package

md1img    ← do not flash from firmware package

Flashed everything else: `preloader`, `lk`, `boot`, `super`, `vbmeta`, and
remaining partitions.

SP Flash Tool completed. Device rebooted. Bootloader warning screen appeared.
Android booted. No longer bricked.

---

## Stage 7: The IMEI Problem

After Android came up, SIM was inserted: no signal. IMEI showed as `null` in
Settings → About Phone and `*#06#`.

The `nvram`, `nvcfg`, and `md1img` that had been on the device before the
erase were gone. The firmware package versions of those partitions contain
generic defaults — not the RF calibration and IMEI data written to this
specific unit at the factory.

Because they had been erased and not restored from a device-specific backup
(I did not have one), the modem had no valid identity or calibration data.

---

## Stage 8: IMEI Recovery via UnlockTool

Options at this point:

- **Restore from device-specific backup** — did not exist. This would have
  been the correct path.
- **UnlockTool** — a paid commercial platform used by phone repair technicians
  that can write IMEI data back to MediaTek modem partitions via the MediaTek
  diagnostic interface.

UnlockTool is available via online rental services — paid by the hour or day,
no full subscription required for a one-time operation.

**Requirements UnlockTool enforced:**

- Unlocked bootloader — required for the MediaTek IMEI write operation.
  Would not proceed on a locked device.
- Active rental session with operation credits.

**What UnlockTool actually does:**

It writes a supplied IMEI value into the modem partition via the MediaTek
diagnostic interface. It does not restore original factory calibration — it
programs a replacement IMEI into `md1img` / `nvram`.

After the operation, IMEI was visible in Settings and `*#06#`. Cellular
connectivity returned after a reboot.

**Caveats:**

- The restored IMEI is a written value, not a restored factory calibration.
  RF performance may differ from factory state in ways that are not obviously
  visible.
- This costs money. The exact amount depends on the rental platform and
  session duration.
- An unlocked bootloader is a hard requirement. Without it, this tool cannot
  perform the write.
- If you have backups of the original `nvram`, `nvcfg`, and `md1img`, restoring
  via MTKClient is free, immediate, and produces a genuinely original result.
  UnlockTool exists for when you do not have those backups.

---

## Final State

| Component     | State After Recovery                          |
|---------------|-----------------------------------------------|
| Bootloader    | Unlocked (ORANGE AVB state)                   |
| Android       | Booting normally on Android 13                |
| Root          | Magisk installed and functioning              |
| IMEI          | Restored via UnlockTool (not original backup) |
| Cellular      | Signal and data functional                    |
| `nvram`       | Generic replacement — no original backup      |

---

## What Actually Caused the Brick

The hard brick was caused by a failed full partition erase in MTKClient. The
assumption driving that decision — that the firmware package contained
everything needed for a complete restore — was incorrect. Firmware packages
are not complete device images. They contain partition data that is valid for
most partitions but not for device-specific partitions, and they assume the
bootloader chain is intact enough to receive a flash.

A partial or failed erase that touches bootloader partitions (`preloader`, `lk`)
before they can be rewritten leaves the device unable to reach even Preloader
mode. BootROM is the only remaining recovery interface at that point.

The situation before the erase — device not powering on — was almost certainly
simpler than a corrupted firmware. A drained battery, a failed boot from a bad
patch, or a recoverable bootloop would all produce the same symptom. Erasing
everything was the wrong response to a device that would not power on.

---

## Actual Lessons Learned

### 1. "Device won't power on" is not a diagnosis

It is a symptom. The correct response is methodical elimination: try charging,
try different boot modes (Volume Up, Volume Down, Volume Down + USB), try
MTKClient detection before assuming the firmware needs to be erased. Erasing
is irreversible. Diagnosis is not.

### 2. Firmware packages are not complete device images

`nvram`, `nvcfg`, and `md1img` in firmware packages contain generic factory
defaults. They are not your device's calibration data. Flashing or erasing
these without a device-specific backup destroys information that has no
public source.

### 3. Back up device-specific partitions before the first MTKClient operation

```bash
python mtk r nvram nvram_backup.img
python mtk r nvcfg nvcfg_backup.img
python mtk r md1img md1img_backup.img
python mtk r persist persist_backup.img
```

These four cannot be obtained from anywhere after the fact. Everything else
can be sourced from firmware packages. These cannot.

### 4. Never run a full erase as a recovery step

A full erase is more destructive than a bad flash. A bad flash leaves
something to recover from. A failed full erase can remove the bootloader
stages needed for any subsequent recovery operation.

If something needs to be reflashed, flash only the specific partitions that
need replacing. Use SP Flash Tool's scatter file with surgical partition
selection rather than erasing the entire eMMC.

### 5. BootROM is the floor, not the ceiling

MediaTek's BootROM cannot be erased. As long as the SoC is physically intact,
there is always a recovery path. A hard brick on MediaTek is recoverable more
often than it appears. The detection can be physically unreliable — try
multiple cables, ports, and connection timings before concluding there is no
BootROM access.

### 6. UnlockTool requires an unlocked bootloader

IMEI restoration via software on MediaTek requires bootloader unlock as a
prerequisite. This is relevant if IMEI loss happens on a device that was never
unlocked — the unlock would need to happen first, which has its own firmware
compatibility requirements on the RMX3710.

### 7. Document the state before every operation

Build number, IMEI (from `*#06#`), partition backup checksums, MTKClient
version. Takes two minutes. The absence of this information during recovery
made every step slower and introduced uncertainty about whether the correct
firmware version was being used.

---

## Summary

The hard brick came from a wrong assumption — that erasing everything and
reflashing was a safe recovery move for a device that wouldn't power on. It
wasn't. The erase failed partway through and removed the bootloader. BootROM
kept the situation recoverable. The IMEI loss came from not having
device-specific partition backups before the erase.

The recovery required bypass_utility, SP Flash Tool with an alternative DA,
careful partition selection during reflash, and a paid UnlockTool session for
IMEI restoration.

Everything that made this expensive — in time, money, and stress — came down
to two missing steps: proper diagnosis before taking destructive action, and
backing up device-specific partitions before touching anything.
