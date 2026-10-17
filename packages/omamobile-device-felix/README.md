# omamobile-device-felix

Omarchy Mobile's first userspace package for the Google Pixel Fold (codename felix, Tensor G2 / gs201). It is the storage guard and the USB link, and nothing that drives the radio, the charger, or a camera. It does not mark a boot slot successful.

| Path | Installs to | What it does |
| --- | --- | --- |
| `vendor/udev/rules.d/10-omamobile-storage.rules` | `/usr/lib/udev/rules.d` | Continues the initramfs storage guard. The whole-disk node stays inaccessible, and desktop tools ignore every Android partition except `userdata`. Same rule as the Pixel 7 Pro, because the partition names match |
| `vendor/systemd/network/20-usb-gadget.network` | `/usr/lib/systemd/network` | Keeps the USB link at 172.16.42.1 after the initramfs hands over |
| `vendor/NetworkManager/conf.d/10-omamobile-usb.conf` | `/usr/lib/NetworkManager/conf.d` | Keeps NetworkManager off that link |
| `src/reboot-to.c` | `/usr/bin/reboot-to` | Reboot with a reason the bootloader understands. Same bytes as the Pixel 7 Pro package; the test checks |
| `device.conf` | `/usr/share/omamobile/device.conf` | The descriptor, the same bytes as `device/felix/device.conf` |

`./build [DIR]` compiles `reboot-to` from this directory, `./install DESTDIR [DIR]` stages the files, and `test/all` checks the two copies when it is run from the repository.

Not in this package: the boot-success marker, Wi-Fi, the charger, cameras, and the slot-B image hook. Those stay out until each one has its own runbook.
