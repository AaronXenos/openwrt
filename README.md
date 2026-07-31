![OpenWrt logo](include/logo.png)

# OpenWrt 25.12.5 with EmbedFire LubanCat 2 support

This tree is OpenWrt v25.12.5 (`openwrt-25.12` branch, kernel 6.12, U-Boot
2025.10) plus support for the **EmbedFire LubanCat 2**, a Rockchip RK3568
single board computer, used here as a router.

For the upstream OpenWrt README — what OpenWrt is, how to build it, where to
get help — see [README_openwrt.md](README_openwrt.md).

## Target hardware

Everything here was developed and tested on a **LubanCat 2 revision V3**
(board number EBF410044V3, 2024-03-23), 4 GB RAM.

* SoC: Rockchip RK3568, quad-core Cortex-A55
* Ethernet: 2x 1000M, both RGMII, both driven by a **RTL8211F** PHY
* Storage: onboard eMMC, TF card slot, M.2 M-key slot (PCIe 3.0 x2, NVMe)
* LED: one green user LED (silkscreen `USR`); the red `PWR` LED is wired to the
  power rail and cannot be controlled from software
* Buttons: `ON/OFF` (RK809 power key), `MASKROM`, `REC`

**PHY variant warning.** The hardware datasheet lists the Ethernet PHY as
**JL2101-N040C**, but this board reports `RTL8211F Gigabit Ethernet` for both
ports — different production batches carry different PHYs. No JLSemi driver
support was added, and the RGMII delays were chosen for the RTL8211F. Check
`dmesg | grep PHY` before flashing a board from another batch; see the port
document, section 4.3, for the JL2101 delay values.

Because the mainline device tree describes the pre-V3 board, three patches
(`141`, `142`, `143`) carry V3/V2-specific fixes. Each one says in its header
when to drop it. Patches `141` and `142` must be applied together — V3 moved
the user LED because its old pin became the RTC interrupt.

## What is supported

* **Both Ethernet ports**, split into `wan` (eth0) and `lan` (eth1), with the
  RGMII delays of the V2/V3 board revision and per-port SMP IRQ affinity.
  Do not install `luci-app-irqbalance`; it undoes the manual affinity.
* **ON/OFF button** through `acpid`, which is part of the default package
  selection for this device. The RK809 power key is a standard input device,
  not a `gpio-keys` node, so OpenWrt's `gpio-button-hotplug` cannot see it and
  `/etc/rc.button/` does not apply. The stock `/etc/acpi/events/default` rule
  runs `/sbin/poweroff` — if you would rather have a reboot, change that rule.
* **Green user LED** bound to the standard OpenWrt states (`led-boot`,
  `led-failsafe`, `led-running`, `led-upgrade`).
* **Persistent MAC addresses**, derived from the SoC's 128-bit OTP chip ID.
  The usual MMC CID approach is not stable on this board, which has both eMMC
  and a removable TF card; the OTP ID belongs to the SoC and survives swapping
  any storage device. If the OTP is unreadable, generation falls back to the
  soldered eMMC (`fe310000.mmc`). This needs a backported upstream patch
  enabling the RK356x OTP controller.
* **M.2 NVMe** (`kmod-nvme`) and the onboard SATA port (`kmod-ata-ahci-dwc`).
* **MT7921AU USB Wi-Fi** with current firmware. The blobs bundled with the mt76
  package date from 2023-11-09; a new `mt7961-firmware` package installs them
  from linux-firmware instead, and `kmod-mt7921-firmware` now depends on it.

Not ported: camera, MIPI-DSI panel and touch, IR receiver, RK809 audio codec,
PWM backlight and PWM fan, and the `REC` button. The `MASKROM` button is a
BootROM-level signal and cannot be exposed to Linux at all.

## Test results

Verified on the board:

| Item | Result |
|---|---|
| Ethernet throughput | 930+ Mbps, iperf3 TCP, no errors (RTL8211F, gigabit line rate) |
| MT7921AU | Driver and updated firmware load and work correctly |
| M.2 NVMe | Reading a file larger than 10 GB completes without errors |

Not verified:

| Item | Why |
|---|---|
| NVMe large-file **write** | Only reads were tested. If write errors or link drops ever show up, the vendor PCIe3 PHY firmware patch can be added back — see the port document, section 9.2 |
| External RTC | The battery holder is empty, so power-loss timekeeping could not be tested. The chip itself is detected and `rtc0` is pinned to it; fitting a battery is expected to be enough, with no software change |

## Building

```bash
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig      # Target: Rockchip -> RK33xx/RK35xx -> EmbedFire LubanCat 2
make -j$(nproc)
```

Images land in `bin/targets/rockchip/armv8/`; the one to flash is
`openwrt-*-embedfire_lubancat-2-squashfs-sysupgrade.img.gz`. Write it to a TF
card, or to the eMMC with `rkdeveloptool` / `upgrade_tool`. The matching
bootloader is built as `lubancat-2-rk3568-u-boot-rockchip.bin`.

Note that `make defconfig` requires `gawk`, and that a package Makefile change
needs an explicit `clean` of that package before it takes effect.
