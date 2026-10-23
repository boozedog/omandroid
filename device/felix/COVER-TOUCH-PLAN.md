<!-- SPDX-License-Identifier: MIT -->
# Pixel Fold cover touch: chip-id gate

The cover controller is an ST SALIXP (chip id `36 40`) on USI6 SPI. linux-omarchy-mobile#32 describes the node and leaves it disabled, and adds `drivers/input/touchscreen/salixp.c`. This plan is the one gated boot of that module. It does not load firmware or report touches. The inner panel, S2MPG13, and the inner reset stay off.

`kernel/configs/omamobile.config` builds `CONFIG_TOUCHSCREEN_SALIXP=m`. `device/felix/overlays/cover-touch.dtso` enables USI6, SPI6, PERIC0, the controller, and the main PMIC. It does not set `regulator-always-on`. `omamobile-device-felix` blacklists `salixp`, so the module does not autoload when the overlay is on. The descriptor leaves `DTB_OVERLAYS` empty. A normal image does not enable the node.

The earlier id reads are in [NOTES.md](NOTES.md). One was an unpublished tree with `regulator-always-on`. One was #32 head `b1b5f1fd6`, built in, with a kernel overlay that has since moved here. Neither is this gate.

## Gate

Risk: the overlay binds the S2MPG12 regulator driver, and the reader then enables LDO25M and LDO26M. Stop if any other rail changes state, if the UFS error handler runs, if USB drops, or if the chip id is not `36 40`. If the phone does not come back on USB, stop. Do not boot slot A: stock rewrites the calibration partitions. Do not mark slot B successful.

Rollback for the boot: slot B images built without `cover-touch`. The blacklist can stay.

Do not use `bin/omamobile flash` or `bin/omamobile push` for this boot. `push` treats a missed SSH answer as a fall back to slot A. Pack the images, then write only `boot_b`, `init_boot_b`, `vendor_boot_b`, `vendor_kernel_boot_b`, and `dtbo_b`, set slot B active, and reboot. Leave `userdata` alone.

### 0. Read-only, before the overlay image

Compare `efs`, `efs_backup`, `modem_userdata`, `persist`, `modem_a`, and `modem_b` with the 2026-10-16 backup. The sums stay off the repository.

**Expect / pass:** every sum matches the backup.

**Stop:** any mismatch. Take a fresh backup only after a slot A boot, and do not continue this gate on stale sums.

**Rollback:** none. Nothing was written.

**Risk:** read-only.

### 1. Blacklist, then the overlay image

From this repository, with `DEVICE=felix` and the kernel tree on the #32 branch. The blacklist has to be on the phone before the overlay boots. Copy `packages/omamobile-device-felix/vendor/modprobe.d/omamobile-cover-touch.conf` to `/etc/modprobe.d/omamobile-cover-touch.conf` on the phone, or install `omamobile-device-felix` 0.1.3, which ships the same file under `/usr/lib/modprobe.d/`. Confirm `blacklist salixp` is what modprobe will see.

Build the kernel so `salixp` is a module (`CONFIG_TOUCHSCREEN_SALIXP=m` in the fragment). For this pack only, set `DTB_OVERLAYS=cover-touch` in `device/felix/device.conf`, run `bin/omamobile images`, then remove that line. Do not commit it. The felix DTB is already built with `-@`.

**Expect / pass:** the pack prints `DTB overlays: cover-touch`. The committed descriptor has no `DTB_OVERLAYS`.

**Stop:** the overlay fails to apply, or the descriptor still sets `DTB_OVERLAYS` after the pack.

**Rollback:** remove the `DTB_OVERLAYS` line and pack again. Remove `/etc/modprobe.d/omamobile-cover-touch.conf` if the package is not installed.

**Risk:** low. No boot yet.

### 2. Boot slot B

Write the five slot B boot images, set slot B active, and reboot. Do not mark the slot successful.

The reader defers until the S2MPG12 regulators exist. `sec_acpm` probes only after the mailbox and the ACPM protocol are loaded, or it skips its channels. On the phone, after it is up and debugfs is mounted:

```sh
modprobe exynos_mailbox
modprobe acpm_protocol
modprobe sec_acpm
modprobe salixp
dmesg | grep salixp
grep -A2 -E 'ldo25m|ldo26m' /sys/kernel/debug/regulator/regulator_summary
```

**Expect / pass:** dmesg has `salixp spi0.0: chip id 36 40`. In `regulator_summary`, `ldo25m` and `ldo26m` are enabled and the touch device is their consumer. No other rail changed. `dmesg` has no `ufshcd_err_handler` line from this boot.

**Stop:** any other chip id, a rail other than those two changing, a UFS error-handler run, or USB dropping. Do not request the attention interrupt. Do not load firmware.

**Rollback:** slot B images packed without `cover-touch`, then reboot slot B. Do not boot slot A.

**Risk:** the main PMIC regulator driver binds, and the reader enables LDO25M (1.8 V) and LDO26M (3.3 V).

### 3. Smoke, then an optional unbind

```sh
tools/phone-smoke
echo spi0.0 > /sys/bus/spi/drivers/salixp/unbind
grep -A2 -E 'ldo25m|ldo26m' /sys/kernel/debug/regulator/regulator_summary
```

**Expect / pass:** `phone-smoke` reports no touch input. This reader does not create one. After unbind, both rails are disabled and have no users.

**Stop:** unbind leaves either rail enabled, or some other rail changes.

**Rollback:** reboot slot B on images without the overlay.

**Risk:** low. Unbind releases the two rails the driver holds.
