<!-- SPDX-License-Identifier: MIT -->
# Pixel Fold (felix) first light

Status (2026-10-13): first light booted from slot B. SSH came up at
`root@172.16.42.1`, both cluster watchdogs were stopped, and DSI and UFS
stayed disabled. The cover kept the bootloader's Google logo because this
image does not take over the framebuffer. The inner panel stayed off.
GrapheneOS is still on slot A. The bootloader is unlocked. Slot B has not
been marked successful.

## Facts

| Fact | Value | Source |
| --- | --- | --- |
| SoC | GS201, MP1.0, package mode 0, SoC rev 2. Same B0 match as cheetah (`soc_id` 0x09855001) | [V] `getprop` ro.boot.hw.soc.pkg_mode, ro.boot.hw.soc.rev, ro.boot.hardware.revision |
| Board id | 0x30606, rev 0x10000, dtbo index 10 | [V] `ro.boot.cdt_hwid`, `ro.boot.dtbo_idx` |
| Bootloader | felix-17.0-15902555, unchanged by the unlock | [V] `fastboot getvar version-bootloader` |
| Cover panel | 1080×2092, 120 Hz, DSI port 1. Active while folded (device state 0) | [V] `dumpsys display` |
| Inner panel | 2208×1840, 120 Hz, DSI port 0. Not described in the board file | [V] `dumpsys display`; Google `gs201-felix-display.dtsi` |
| Slots after the GrapheneOS reinstall | A successful and current. B bootable, not yet successful, not unbootable | [V] `fastboot getvar` |

The framebuffer address is not copied from the 7 Pro. On that phone ABL
injects the splash carve-out, and the 7 Pro tree now also names
`0xfac00000`. Using that address here would `no-map` the wrong RAM if the
Fold's splash lives elsewhere. The first boot reads the injected node from
`/sys/firmware/fdt` with the phone folded.

## What this tree enables

`gs201-felix.dts` (kernel branch `omandroid-felix`) enables the USB
controller and relies on the SoC tree for the cluster watchdogs. Both
panels, both touch controllers, the GPU, and UFS stay disabled. The
initramfs does not write the 7 Pro display trigger register and does not
read the display DMA registers.

`device/felix/device.conf` is what `bin/omamobile` reads. The default
device stays cheetah. A Fold image is `DEVICE=felix`.

## First boot

Phone folded. Flash only slot B. Slot A stays GrapheneOS. Expect SSH at
`root@172.16.42.1`. The inner panel stays at the bootloader's off state
because nothing in the tree has its reset pin or its rails. The cover
can keep showing the bootloader logo; that is not a hung boot.

Stop if the cover panel changes brightness on its own, if the inner panel
lights, or if the phone does not come back to slot A after a bad boot.
Hold volume down and power for Fastboot Mode, then
`fastboot --set-active=a`.
