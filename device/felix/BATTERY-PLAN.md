<!-- SPDX-License-Identifier: MIT -->
# Battery plan (felix / gs201)

Status (2026-10-25): stage 6 is not started on the phone. This file is the
offline plan. No kernel branch, no board-file change, no boot. Charging,
Type-C, and the charger policy stay a later stage
([CONTRIBUTING.md](../../CONTRIBUTING.md) §5.4, stage 12).

Goal: both cells report a capacity on the pack's design scale and a
temperature that follows the pack. The pass is the two fuel gauges. The
Android `dualbatt` combiner is not part of this stage.

Path abbreviations:

- `F/` = factory `dtbo` entry 10 for the build named in [NOTES.md](NOTES.md)
  (board id `0x30606`, rev `0x10000`). The gauges live in that overlay.
- `R/` = the stock capture of this phone (live device tree, `getprop`,
  fastboot, vendor module names), kept outside the repo.
- `L/` = the gs201 SoC tree and the mainline gauge drivers
  (`gs201.dtsi`, `max17042_battery.c`, `max1720x_battery.c`).

## 1. Two cells

The Fold is a dual pack. Android names the combined supply `dualbatt`
(`google,fg-psy-name`) and names the gauges `maxfg_base` and
`maxfg_secondary` (`google,dual_batt_gauge`). [V] `F/` fragment `@battery`.

| | Base | Secondary |
| --- | --- | --- |
| Supply name | `maxfg_base` | `maxfg_secondary` |
| Design capacity | 1460 mAh (`google,chg-base-battery-capacity` `0x5b4`) | 3267 mAh (`google,chg-sec-battery-capacity` `0xcc3`) |
| Bus | HSI2C13 `0x10d60000` | HSI2C15 `0x10da0000` |
| Address | `0x36` | `0x36`, non-volatile memory at `0x0b` |
| Stock compatible | `maxim,max77759` | `maxim,max1720x` |
| Interrupt | `gpa9-3`, pull-up, active-low | `gpa7-3`, pull-up, active-low |
| Sense resistor | 5.00 mΩ (`maxim,rsense-default` `0x1f4`) | 3.00 mΩ (`maxim,rsense-default` `0x12c`) |

[V] `F/` `max77759fg@36` (`maxim,dual-battery = "maxfg_base"`) and
`max1720x@36` (`maxim,dual-battery = "maxfg_secondary"`). The pack total in
the same overlay is 4727 mAh (`google,chg-battery-capacity` `0x1277`),
which is 1460 + 3267.

HSI2C13 is the Pixel 7 Pro's gauge bus: the same address, the same pins
(`gpp25-0`, `gpp25-1`), and the same alert pin. The 7 Pro has one cell on
that gauge. The Fold's larger cell is the MAX1720x on HSI2C15. [V] `F/`
and [WIFI-BATTERY-PLAN.md](../cheetah/WIFI-BATTERY-PLAN.md) §4.1.

The captured device tree has both controllers `disabled` and no gauge
children. [V] `R/live.dts`, `usi@10d600c0` and `i2c@10d60000`. The board
overlay is what adds the devices. Fastboot on the day of that capture
reported 100 %, 4384 mV, and −231 mA. [V] `R/` `fastboot-getvar-all.txt`.
That is the bootloader's reading from that day, so it is not the number a
later gauge probe has to match. The capture has no `dumpsys battery`.

The vendor modules on that build are named `max77779-fg-i2c`,
`max77779-charger-i2c`, `max77779_pmic_i2c`, and the rest of the
`max77779*` set. [V] `R/vendor-modules.txt`. The overlay's compatibles
remain `maxim,max77759` and `maxim,max1720x`. The ID registers in B1.1 and
B1.3 are what the chips are. The module filenames do not choose the driver.

## 2. Buses, drivers, and what else is on them

Both controllers are already in the SoC tree, `status = "disabled"`, with
their pin groups. HSI2C13 is GIC SPI 695. HSI2C15 is GIC SPI 696. The USI
sysreg offsets are `0x1014` and `0x1018` on `sysreg_peric1`. [V] `L/`
`gs201.dtsi` (`hsi2c_13`, `hsi2c_15`) and `gs201-pinctrl.dtsi`
(`hsi2c13_bus` on `gpp25-0/1`, `hsi2c15_bus` on `gpp24-0/1`, function 2,
no pull).

The binding requires `samsung,mode` on the USI node. [V] `L/`
`exynos-usi.yaml`. Stock's overlay does not set it. This stage sets
`USI_MODE_I2C` on each USI it enables, as the 7 Pro board does, because
the children are I2C devices.

Stock's clock on HSI2C13 is 400 kHz in one fragment and 370 kHz in a later
fragment on the same node, so the running overlay is 370 kHz. HSI2C15 is
400 kHz. [V] `F/` fragments `@42`, `@121`, and `@117`.

`gpa9` is an alive bank. An interrupt group there gets
`GS101_PIN_FUNC_EINT` and a pull, and no power-down pin config. `gpa7` is
on the far-alive controller. The inner touch interrupt is `gpa7-0` and the
inner panel reset is `gpa7-1` ([NOTES.md](NOTES.md)). The secondary gauge's
alert is `gpa7-3` on that same bank. Stock's group for it is function
`0xf`, pull-up `0x3`, drive `0x0`, with no power-down properties. [V] `F/`
`alrt-gauge-m-irq`.

Other stock children, and why they stay disabled for this stage:

| Bus | Address | What it is | Stage |
| --- | --- | --- | --- |
| HSI2C13 | `0x66` | `maxim,max77729pmic` | charger MFD, later |
| HSI2C13 | `0x69` | `maxim,max77759chrg` | stage 12. Probe turns charging off on the 7 Pro |
| HSI2C13 | `0x25` | `max77759tcpc` | stage 12. It owns the USB-C port |
| HSI2C13 | `0x57` | `nxp,pca9468` | direct charger, later |
| HSI2C15 | `0x43` | `cirrus,cs40l26a` | haptics, stage 9 |
| HSI2C15 | `0x50` | `at,24c08` | battery EEPROM for `google_bms` |

[V] `F/` fragments `@17`, `@42`, `@46`, `@41`, `@117`, `@122`.

Mainline binds different compatible strings than stock:

- The small cell's driver is `max17042_battery`, compatible
  `maxim,max77759-fg`. [V] `L/` `max17042_dt_match`. It is already
  `CONFIG_BATTERY_MAX17042=m` in `kernel/configs/omamobile.config`.
- `monitored-battery` does not arm POR init for this chip. POR init of the
  characterization table runs only when the node has `maxim,fg-model`.
  [V] `L/` `max17042_apply_battery_properties` (returns for every chip
  other than the MAX17055) and `max77759_fg_parse_model`.
- With no interrupt, probe writes `SALRT_Th` (`0x03`) to `0xff00`. With an
  interrupt, probe sets the alert enable in `CONFIG`. [V] `L/`
  `max17042_probe`.
- STATUS bit 1 is POR (`STATUS_POR_BIT`). When it is set, probe schedules
  the init worker. [V] `L/` `max17042_probe`.
- The secondary driver is `max1720x_battery`. Its only compatible is
  `maxim,max17201`. After it binds, `DEV_NAME` (`0x21`) low nibble 1 is a
  MAX17201 and low nibble 5 is a MAX17205. [V] `L/` `max1720x_of_match`
  and `MAX172XX_DEV_NAME_TYPE_*`. `CONFIG_BATTERY_MAX1720X` is not in the
  omamobile fragment, so the build that first enables this node adds
  `CONFIG_BATTERY_MAX1720X=m`.
- That driver has no register write. Probe reads `nRSense` (`0xCF`) through
  an ancillary device whose `reg-names` entry is `nvmem`, and falls back
  to address `0x0b`. A zero sense value is replaced in software with
  10 mΩ and the supply still registers. [V] `L/` `max1720x_probe_nvmem`,
  `i2c_new_ancillary_device`. Capacity is `reg * (500000 / rsense)` µAh
  with `rsense` in 10⁻⁵ ohm.

## 3. Which model the small cell has

The MAX77759 gauge has no non-volatile model. Stock ships four tables on
`max77759fg@36` and picks one from the pack ID. The words below are the
`maxim,fg-params` byte string read as big-endian pairs, in the order of
`struct max77759_fg_params`. [V] `F/` and `L/` `max77759_fg_params`.

| Table | `maxim,batt-id-kohm` | DesignCap | TaskPeriod | At 2 mAh/LSB |
| --- | --- | --- | --- | --- |
| `default-fake-batt` | `0xaa` | `0x08d6` | `0x2d00` | 4524 mAh |
| `default-pack-b-0k` | 0 | `0x030f` | `0x2d00` | 1566 mAh |
| `atl-pack-b-1k` | 1 | `0x02e9` | `0x2d00` | 1490 mAh |
| `cos-pack-b-2k` | 2 | `0x02e9` | `0x2d00` | 1490 mAh |

TaskPeriod `0x2d00` is the same value the 7 Pro uses to make one capacity
LSB 2 mAh at 5 mΩ. The 1 kΩ and 2 kΩ tables share DesignCap `0x02e9`
(1490 mAh), next to the 1460 mAh pack property. DesignCap separates the
fake table and the 0 kΩ table from those two. It does not separate 1 kΩ
from 2 kΩ. This phone's ID is not in the capture.

The fake table is not a candidate. No `maxim,fg-model` goes in the board
file until B1.2 has read DesignCap and a later step has told 1 kΩ from
2 kΩ. A node without that property does not load a table on POR.

The MAX1720x keeps its model in non-volatile memory. Its overlay node
carries an aging table (`maxim,fg-aafv`) and no characterization table.
The read-only driver uses the sense resistor it reads from the chip.

## 4. Design

1. Add the gauge nodes disabled. An overlay enables one controller per
   boot. `device.conf` keeps no `DTB_OVERLAYS` entry for this stage, so a
   normal image leaves both buses off.
2. The first boot of a controller has no child nodes. Reads are `i2cget`
   of the known addresses. There is no bus scan: HSI2C15 address `0x43`
   is the haptic amplifier.
3. The first bind of either gauge has no interrupt. `gpa9-3` and `gpa7-3`
   stay unconfigured until a later boot asks for the alert.
4. `max17042_battery` and `max1720x_battery` are blacklisted in
   `omamobile-device-felix` before a boot whose overlay enables a gauge
   node. The module is loaded by hand in one SSH session.
5. The small cell's model stays out of the device tree until its DesignCap
   is known. The 7 Pro's 1 kΩ table is a different pack.
6. Stage 12 owns the charger, the TCPC, and the PMIC. This stage does not
   describe those nodes.

Kernel branch for the device tree, when it is cut: `omamobile-gs201-felix-battery`,
worktree `wt-felix-battery`, output `out-felix-battery`.

## 5. What was built

Nothing yet.

## 6. Risks

- Enabling a USI programs its pinmux and `SW_CONF`. The pins are the ones
  stock uses for these buses. A wrong mode leaves `i2cget` with no answer.
  It does not touch the inner panel.
- `max17042` probe writes `SALRT_Th` (`0x03`) to `0xff00` when the node
  has no interrupt. With no `shunt-resistor-micro-ohms` it also writes
  `CGAIN`, `MiscCFG`, and `LearnCFG`. The model write path stays closed
  while `maxim,fg-model` is absent. POR set means the init worker runs;
  that boot stops before `modprobe`.
- The secondary driver's non-volatile read is a read. A zero `nRSense`
  makes the capacity use 10 mΩ and look about a third of the real value.
  That result fails the capacity pass.
- `gpa7-3` shares a bank with the inner touch interrupt and sits next to
  the inner panel reset. The boots below do not put a pin group on
  `gpa7`. The inner picture changing is a stop on every boot.
- Loading the charger driver turns charging off. Loading the TCPC takes
  the USB port. Neither node exists in this stage.

## 7. Runbook

One SSH session for each step. The phone-ssh helper opens a new login per
command. Slot A stays GrapheneOS. Image commands are `DEVICE=felix
bin/omamobile`. The tool's default device is cheetah, so a bare
`bin/omamobile` packs the 7 Pro. Images written for a boot are slot B's
boot images only, and slot B is not marked successful. Rollback of a bad
boot is the previous slot B images. The inner panel keeps the bootloader
logo. USB back at `172.16.42.1` is part of every pass.

### B0. This plan

Offline. Nothing on the phone. Pass: this file.

### B1.0. Board file, nodes disabled

Offline. On `omamobile-gs201-felix-battery`, add `fuel-gauge@36` under
`hsi2c_13` (`compatible = "maxim,max77759-fg"`, `reg = <0x36>`,
`shunt-resistor-micro-ohms = <5000>`) and `fuel-gauge@36` under
`hsi2c_15` (`compatible = "maxim,max17201"`, `reg = <0x36 0x0b>`,
`reg-names = "gauge", "nvmem"` so the ancillary lookup finds `0x0b`).
Both nodes `disabled`. Both USI nodes stay `disabled`. No
`maxim,fg-model`, no interrupt, no charger, no TCPC, no PMIC, no
`cs40l26`, no EEPROM. `dt-validate` and a DTB build in `out-felix-battery`.
The secondary driver's fragment `CONFIG_BATTERY_MAX1720X=m` lands in the
same kernel branch and is not required to boot B1.1.

Pass: the DTB builds, the new nodes are `disabled`, and the 7 Pro DTB is
byte-identical. `CONFIG_BATTERY_MAX1720X=m` adds a module to the shared
config and does not enable a node on the 7 Pro. Nothing is flashed.

### B1.1. HSI2C13, read only

Risk: low.

Overlay for this image only: `&usi13` gets `samsung,mode = <USI_MODE_I2C>`
and `status = "okay"`; `&hsi2c_13` gets `clock-frequency = <370000>` and
`status = "okay"`. No child is enabled. The committed `device.conf` does
not gain a `DTB_OVERLAYS` line.

Expect: SSH, the cover console, the inner logo unchanged, and
`10d60000.i2c` under `/sys/bus/i2c/devices/`.

Then, in that one session, with `BUS` the number of that controller:

```sh
i2cget -y "$BUS" 0x66 0x00
i2cget -y "$BUS" 0x36 0x00 w
i2cget -y "$BUS" 0x36 0x03 w
i2cget -y "$BUS" 0x36 0x08 w
i2cget -y "$BUS" 0x36 0x09 w
i2cget -y "$BUS" 0x36 0x18 w
i2cget -y "$BUS" 0x36 0x3c w
```

Those are the PMIC id, STATUS, `SALRT_Th`, TEMP, VCELL, DesignCap, and
TaskPeriod. There is no `i2cset` and no `i2cdetect`.

Pass: each read returns. STATUS bit 1 is clear. VCELL is about 3.6–4.4 V
at 78.125 µV/LSB. TEMP is a desk temperature; `0x1600` (22.0 °C) is the
stuck value the 7 Pro showed before its model was understood. DesignCap
is one of `0x02e9`, `0x030f`, or some other value, recorded either way.
TaskPeriod is recorded. The PMIC id byte is recorded; the 7 Pro's `0x3b`
is the comparison, not the pass.

Stop: a read hangs or returns no acknowledgement, STATUS bit 1 is set,
the inner picture changes, or USB does not come back. No module is loaded
on this boot.

### B1.2. Small-cell driver

Risk: low. One threshold write.

A new boot. The gauge node is enabled by the overlay, still with no
interrupt and no `maxim,fg-model`. `max17042_battery` is blacklisted
before the reboot. After SSH, repeat the B1.1 register reads. The booted
node's `shunt-resistor-micro-ohms` bytes are `00 00 13 88` (5000). STATUS
bit 1 is clear. Then:

```sh
modprobe max17042_battery
```

Pass: a `max77759` power supply appears. `voltage_now` is about
3.6–4.4 V. `temp` is a desk temperature. `charge_full_design` is near
1490 mAh when DesignCap was `0x02e9`, or near 1566 mAh when it was
`0x030f`, with TaskPeriod `0x2d00`. `SALRT_Th` reads back `0xff00`.
Record `voltage_now`, `temp`, and `charge_full_design` for B1.3.

Stop: the shunt property is missing or is not 5000, STATUS bit 1 was
set, DesignCap is `0x08d6`, the supply's capacity is on the 4500 mAh
scale, the temperature is stuck, or the inner picture changes. The
blacklist stays until this pass is recorded.

### B1.3. HSI2C15, read only

Risk: low.

A new boot. Overlay enables `&usi15` (`USI_MODE_I2C`) and `&hsi2c_15`
(`clock-frequency = <400000>`). HSI2C13's overlay from B1.2 can stay, with
the small-cell driver still blacklisted or already loaded, as it was left.
No HSI2C15 child is enabled.

Expect: `10da0000.i2c` appears. Read the small cell's `voltage_now`,
`temp`, and `charge_full_design` again and compare them with the B1.2
record.

```sh
i2cget -y "$BUS" 0x36 0x00 w
i2cget -y "$BUS" 0x36 0x21 w
i2cget -y "$BUS" 0x36 0x08 w
i2cget -y "$BUS" 0x36 0x06 w
i2cget -y "$BUS" 0x36 0x18 w
```

STATUS, `DEV_NAME`, TEMP, RepSOC, DesignCap. No read of address `0x43` or
`0x50`.

Pass: STATUS bit 1 is clear. `DEV_NAME` low nibble is 1 or 5. TEMP agrees
with the small cell within a few degrees. The words are recorded.

Stop: no acknowledgement, any other `DEV_NAME`, STATUS bit 1 set, or the
inner picture changes.

### B1.4. Large-cell driver

Risk: low. The driver reads non-volatile `nRSense` and does not write it.

A new boot. `CONFIG_BATTERY_MAX1720X=m` is in the image.
`max1720x_battery` is blacklisted. The overlay enables the HSI2C15 gauge
node with no interrupt. `DEV_NAME` from B1.3 is 1 or 5.

```sh
modprobe max1720x_battery
```

Pass: the `max1720x` supply appears, `voltage_now` is about 3.6–4.4 V,
and `temp` agrees with the other gauge. `charge_full_design` is near
3267 mAh. The kernel log has no "RSense not calibrated" line. The two
design capacities sit near 1460 and 3267 mAh.

Stop: the sense resistor was taken as 10 mΩ, the capacity is far from
3267 mAh, probe fails, or the inner picture changes.

### B1.5. Small-cell model

Risk: medium. This step can write the characterization table. It is not
run until B1.2's DesignCap is in the results below.

If DesignCap is already `0x02e9` or `0x030f` and B1.2's capacity and
temperature passed, the table stock loaded is still in the gauge and this
step is a record, not a write.

A write is its own boot, with one table, and only after the 1 kΩ and 2 kΩ
packs have been told apart. The `0xaa` table is not loaded. The commands
for that boot are added here when B1.2 has the register values. POR set,
or a failed read-back of the table, stops the step with the gauge still
reporting on its previous model.

## 8. Results

None. No boot has run this plan.

## 9. Open questions

- The pack ID on this phone. DesignCap splits the fake table and the 0 kΩ
  table from the other two, and leaves 1 kΩ and 2 kΩ together.
- The byte at HSI2C13 `0x66` register 0. The overlay calls that node
  `maxim,max77729pmic`; the vendor modules are `max77779_pmic_*`.
- `DEV_NAME` on the MAX1720x, and whether `nRSense` reads 300.
- Whether STATUS bit 1 is clear on both gauges. Stock has been running
  them, so the bit should be clear while the gauges have stayed powered.
