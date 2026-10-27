# OpenWrt support for the DrayTek Vigor 2862

## Contents

| File | Description |
| --- | --- |
| [`0001-uboot-lantiq-add-DrayTek-Vigor-2862.patch`](0001-uboot-lantiq-add-DrayTek-Vigor-2862.patch) | RAM-boot U-Boot (`vigor2862_ram`) chainloaded by DrayBoot from the NAND slot at `0x1380000`; xRX330 clocks, EBU, RAM sizing; boots the GPHYs from a copy of DrayBoot's GPHY firmware above Linux's memory. |
| [`0002-lantiq-add-kernel-patches-for-the-xRX330-SoC.patch`](0002-lantiq-add-kernel-patches-for-the-xRX330-SoC.patch) | Kernel patches 802 (serial) and 803 (DMA). |
| [`0003-lantiq-add-support-for-DrayTek-Vigor-2862.patch`](0003-lantiq-add-support-for-DrayTek-Vigor-2862.patch) | `xrx330.dtsi` (including PCIe root complex 0 and its PHY), board DTS, image definition, board.d and sysupgrade. |
| [`xrx330_draytek_vigor2862.dts`](xrx330_draytek_vigor2862.dts) | The board DTS (copy of the file after 0003). |
| [`openwrt-lantiq-xrx200-draytek_vigor2862-squashfs-factory.bin`](openwrt-lantiq-xrx200-draytek_vigor2862-squashfs-factory.bin) | Factory image built from this series (Linux 6.18.55), md5 `a649c98675aa2ced239327176ba256d8`. |
| [`boot.log`](boot.log) | Serial console of a cold boot with this image, from power on (boot ROM, DrayBoot, U-Boot, kernel) to the OpenWrt shell. |
| [`patches/`](patches/) | The kernel patches 802 and 803 on their own (see [`patches/README.md`](patches/README.md)). |

## Dependencies

- The DrayTek Vigor 2860 series (OpenWrt PR 26084, 3 commits); the patches
  apply on top of "lantiq: add support for DrayTek Vigor 2860ac".
- The `draytek-drayboot` image tool (openwrt/firmware-utils PR 85).

Apply with:

```sh
git am 000*.patch
```

## Switch (GPHY) firmware

The four LAN ports are on the xRX330 internal GPHYs, which run firmware
from RAM. Linux normally loads it as a file (`lantiq/xrx300_phy11g_a21.bin`
for the xRX330), but OpenWrt ships no xRX330 GPHY firmware package.

DrayBoot already has the firmware in SPI-NOR: on every boot (cold boot,
reboot, TFTP recovery) it unpacks its "VR9 V1.2 GPHY GE" image (version
`0x8435`) to RAM at `0x200000` and boots the GPHYs from there. The GPHYs keep
running from that RAM, which the kernel later overwrites. So:

1. U-Boot (0001) copies the 64 KiB image to `0x7fe0000`, in the top 1 MiB
   of RAM that is outside the memory given to Linux, and restarts the
   GPHYs from the copy (RCU firmware address registers + GPHY resets).
2. The board DTS (0003) deletes the `gphy-fw` node, so `lantiq_gswip` neither
   resets the GPHYs nor requests a firmware file.

No firmware file and no kernel change are needed. The GPHYs stay
recoverable while Linux runs: unloading/reloading `lantiq_gswip` leaves
them alone, and a hardware GPHY reset restarts them from the same copy.
If DrayBoot did not boot the GPHYs, U-Boot prints `GPHY: no DrayBoot
firmware` and leaves them as they are.

## Installation

1. Hold reset while powering on to enter the DrayBoot TFTP recovery.
2. Set the computer to `192.168.1.2/24` and connect it to LAN1.
3. Rename the factory image to `v2862x.bin`. DrayBoot only accepts images
   whose name starts with the model; the character before `.bin` selects
   the action: `0` boots the image from memory, any other character
   writes it to NAND.
4. Upload it:

   ```sh
   tftp -m binary 192.168.1.1 -c put v2862x.bin
   ```

5. DrayBoot writes the image to the primary firmware slot and boots
   OpenWrt. The stock firmware in the backup slot (`0x3380000`) is kept.

## Recovery / revert to stock

Same as the Vigor 2860 (tested on the 2862):

1. Download the DrayOS firmware for the 2862 from DrayTek's website and
   extract the `.zip` file.
2. Enter the DrayBoot TFTP recovery (hold reset while powering on).
3. Upload the DrayOS `.rst` file via the same TFTP recovery.

DrayBoot also restores the backup slot (stock firmware) by itself when
the primary slot checksum is bad.

## PCIe

The VRX318 DSL modem sits on PCIe root complex 0 of the xRX330, the
VR9-compatible one the existing `ifxmips_pcie` driver handles, so it is
device tree only (in 0003). Its reset (PERST#) is STP output 11, active low: while
it is low the link stays in Polling.Compliance, once released the link
trains. The VRX318 enumerates as `1bef:0020` (BAR0 8 MiB at `0x1c000000`),
link 2.5 GT/s x1. Root complexes 1 and 2 are not described: the driver
does not support them and nothing is connected there on this board (the
stock firmware also fails to link them).

## Outstanding

- **DSL:** VDSL2/ADSL2+ uses the VRX318, which now enumerates on PCIe, but
  there is no VRX318 driver in OpenWrt. The DSL LED is defined but stays
  off.
- **Wi-Fi:** the tested unit has no Wi-Fi card (the stock firmware finds
  only the VRX318 on PCIe). Wi-Fi variants (n/ac) are untested.

## Tested

- **This image:** factory flash over DrayBoot TFTP, first boot and cold boot;
  WAN (AR8035) DHCP and traffic; LAN link; VRX318 enumerates on PCIe
  (2.5 GT/s x1, no AER errors).
- **Test build of the same series with devmem:** TFTP flash, reboot and cold
  boot (GPHY firmware address registers `0x07fe0000`); rmmod/modprobe
  `lantiq_gswip`; GPHY hardware reset from Linux (link back after ~4 s);
  all eight LAN LEDs hardware-offloaded.
- **Revert to stock:** DrayOS `.rst` upload over DrayBoot TFTP recovery.
- **Earlier builds:** LAN1-4 = switch ports 1, 4, 2, 3; LAN4 LEDs with and
  without a cable; USB 1/2, reset button, watchdog, sysupgrade, panel LEDs.
