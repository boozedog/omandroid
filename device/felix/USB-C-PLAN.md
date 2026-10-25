<!-- SPDX-License-Identifier: MIT -->
# Pixel Fold USB-C: the data switch

Status, 2026-10-25. Step 2 passed on the running `7.3.0-rc6-omamobile-push1-g523924301fb3` boot. One unplug opened `USBSW_CTRL` to `0x00`. One write of `0x09`, with VBUS still valid, closed it, and a fresh SSH login answered. No driver. No other register. Slot B was not marked successful.

## Goal

A replug should bring USB back without a reboot. The MAX77759 opens its USB2 data switch on unplug. With no Type-C driver, nothing closes it. The Pixel 7 Pro closes it from `omamobile-usb-rescue`: TCPC `0x25` register `USBSW_CTRL` `0x93` to `0x09` while VBUS is valid.

## Facts

- [V] The Fold's TCPC is on the same bus as the 7 Pro, HSI2C13 `10d60000.i2c`. A read on 2026-10-25, cable in, had `USBSW_CTRL` `0x09` and charger `0x69` `CHG_DETAILS_00` `0xb5` `0x60`. Bits 6:5 of that byte were 3, which is VBUS valid. No write.
- [V] Google's driver writes `0x09` on every attach with a data role (`tcpci_max77759.c`). `0x09` is D+/D- connected. `0x00` is open.
- The Fold package does not run that rescue. `device/felix/NOTES.md` says a replug may need a reboot for that reason.

## What was built

Nothing. No overlay, no module, no package change.

## Risks

The only write this step allows is `0x93` on `0x25` to `0x09`, and only after an unplug, and only while VBUS is valid and the switch is not already `0x09`. A write to any other register, or a write while VBUS is not valid, is the stop. USB is the only link. If the write does not bring the link back, the recovery is a power-button reboot, not another write.

## Runbook

Do not mark slot B successful. Do not load a Type-C or charger module.

### 1. Read the switch

Read-only. Cable in.

**Expect / pass:** the bus is `10d60000.i2c`, nothing owns `i2c-N-0025`, `USBSW_CTRL` is `0x09`, and `CHG_DETAILS_00` bits 6:5 are 3.

**Stop:** a driver owns the TCPC, the bus is not HSI2C13, or either read fails. No write.

**Rollback:** none. Nothing was changed.

**Risk:** two register reads.

### 2. One unplug and replug

Passed, 2026-10-25, on the already-running boot. Written before the pull. A phone-side loop runs for 180 seconds. It reads the same two registers. If VBUS is valid and `USBSW_CTRL` is not `0x09`, it writes `0x09` once and stops. It does not arm the watchdog. It does not write any other register.

Unplug, wait 5 seconds, plug back in, inside that window.

**Expect / pass:** a fresh SSH login answers. The log says whether the write happened. `USBSW_CTRL` is `0x09`.

**Stop:** any other register is written, the write happens while VBUS is not valid, USB does not return after the write, or the phone does not come back.

**Rollback:** the loop exits. If USB is dead, hold Power about 30 seconds, then press Power. Do not mark slot B successful.

**Risk:** one cable pull, and at most one write of `0x09`.

## Results

2026-10-25. The first pull was after the loop had expired, and USB did not return. A power-button reboot brought the link back with `USBSW_CTRL` `0x09`. That pull is not this step.

2026-10-25. Second pull, inside the loop, same boot. The switch had opened. The loop wrote `0x09` once. A fresh SSH login answered. The switch read `0x09` again, and `CHG_DETAILS_00` was still `0x60`.

```
felix-usbc-deadman: wrote 0x09 was 0x00 now 0x09 details 0x60
```
