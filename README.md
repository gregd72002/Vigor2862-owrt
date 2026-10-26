OpenWrt support for the DrayTek Vigor 2862
==========================================

Contents
--------

- `0001-uboot-lantiq-add-DrayTek-Vigor-2862.patch` — RAM-boot U-Boot
  (`vigor2862_ram`) chainloaded by DrayBoot from the NAND slot at
  `0x1380000`; xRX330 clocks, EBU, RAM sizing; keeps DrayBoot's GPHY
  firmware in reserved RAM for Linux.
- `0002-lantiq-add-kernel-patches-for-the-xRX330-SoC.patch` — kernel
  patches 802 (serial), 803 (DMA), 804 (GPHY firmware from RAM).
- `0003-lantiq-add-support-for-DrayTek-Vigor-2862.patch` — `xrx330.dtsi`,
  board DTS, image definition, `board.d` and sysupgrade.
- `xrx330_draytek_vigor2862.dts` — the board DTS (copy of the file added
  in 0003).
- `openwrt-lantiq-xrx200-draytek_vigor2862-squashfs-factory.bin` — factory
  image built from this series (Linux 6.18.55), md5
  `53fe31bd5c5ac286af9d4aeab7c97af6`.
- `boot.log` — serial console of a cold boot with this image, from power
  on (boot ROM, DrayBoot, U-Boot, kernel) to the OpenWrt shell.
- `patches/` — the kernel patches 802-804 on their own (see
  [`patches/README.md`](patches/README.md)).

Dependencies
------------

- The DrayTek Vigor 2860 series (OpenWrt PR 26084, 3 commits); the patches
  apply on top of "lantiq: add support for DrayTek Vigor 2860ac".
- The draytek-drayboot image tool (openwrt/firmware-utils PR 85).

Apply with:

```sh
git am 000*.patch
```

Switch (GPHY) firmware
----------------------
The four LAN ports are on the xRX330 internal GPHYs, which run firmware
from RAM. Linux normally loads it as a file (lantiq/xrx300_phy11g_a21.bin
for the xRX330), but OpenWrt ships no xRX330 GPHY firmware package.

DrayBoot already has the firmware in SPI-NOR: on every boot it unpacks
its "VR9 V1.2 GPHY GE" image (version 0x8435) to RAM at 0x200000 and boots
the GPHYs from there. The kernel later overwrites that RAM, so it cannot
be used in place. The approach is therefore:

1. U-Boot (0001) copies the 64 KiB image to the reserved top of RAM
   (0x7fe0000) and writes a magic/length/crc32 trailer in the last 16
   bytes of that 128 KiB region. It does this only when the GPHYs were
   booted from 0x200000 by DrayBoot.
2. The board DTS (0003) reserves 0x7fe0000-0x7ffffff (no-map) and points
   each GPHY node at it with "memory-region".
3. lantiq_gswip (patch 804, 0002) checks the trailer and boots the GPHYs
   from that region, without request_firmware(). If the region is missing
   or invalid it warns and falls back to the firmware file as before.

So no firmware file is shipped or needed, but the switch depends on all
three parts and on DrayBoot having booted the GPHYs (seen on cold boot,
TFTP recovery and reboot). Without a valid region and without a file,
the gswip probe fails and no switch ports, including WAN, come up.

Installation
------------

1. Hold reset while powering on to enter the DrayBoot TFTP recovery.
2. Set the computer to 192.168.1.2/24 and connect it to LAN1.
3. Rename the factory image to v2862x.bin. DrayBoot only accepts images
   whose name starts with the model; the character before .bin selects
   the action: '0' boots the image from memory, any other character
   writes it to NAND.
4. tftp -m binary 192.168.1.1 -c put v2862x.bin
5. DrayBoot writes the image to the primary firmware slot and boots
   OpenWrt. The stock firmware in the backup slot (0x3380000) is kept.

Recovery / revert to stock
--------------------------

Same as the Vigor 2860:

1. Download the DrayOS firmware for the 2862 from DrayTek's website and
   extract the .zip file.
2. Enter the DrayBoot TFTP recovery (hold reset while powering on).
3. Upload the DrayOS .rst file via the same TFTP recovery.
DrayBoot also restores the backup slot (stock firmware) by itself when
the primary slot checksum is bad.

Outstanding
-----------
- PCIe: the xRX330 has three PCIe root complexes; the OpenWrt driver only
  handles the VR9 one. Work in progress, kept out of this series. On RC2
  the link does not train yet (the endpoint is detected but never answers).
- DSL: VDSL2/ADSL2+ uses a VRX318 on PCIe. Needs PCIe first, and there is
  no VRX318 driver in OpenWrt. The DSL LED is defined but stays off.
- Wi-Fi: not supported. The Wi-Fi variants (n/ac) are expected to have
  their radios on PCIe as well, so this also waits for PCIe.

Tested
------
This image: factory flash over DrayBoot TFTP, first boot and cold boot;
WAN (AR8035) DHCP and traffic; LAN link; LAN1-4 = switch ports 1, 4, 2, 3;
LAN LEDs hardware-offloaded.
Earlier builds of the same code: USB 1/2, reset button, watchdog,
sysupgrade, panel LEDs.
