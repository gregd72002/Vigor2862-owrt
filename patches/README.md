Linux kernel patches for the Lantiq xRX330 (GRX390 family)
==========================================================

- **Base:** Linux 6.18 (OpenWrt `target/linux/lantiq/patches-6.18`)
- **Tested:** DrayTek Vigor 2862, xRX330 rev 1.1 (part 0x011, GRX369)
- **Author:** Gregory Dymarek <gregd72002@gmail.com>

Patches
-------

- `802-serial-lantiq-support-the-xRX330-interrupt-bit-layout.patch` — the
  xRX330 ASC has an extra TX buffer interrupt bit, which moves RX and error
  up by one bit. Adds the `lantiq,xrx330-asc` compatible (driver, binding,
  early console). The TX buffer interrupt stays disabled.
- `803-net-lantiq_xrx200-do-not-lose-DMA-interrupts-on-NAPI-re-arm.patch` —
  on the xRX330 DMA (rev 8) a TX completion during NAPI could leave the
  channel interrupt pending without raising it, stalling TX. Acknowledge,
  re-enable, then re-check the next descriptor.
- `804-net-dsa-lantiq_gswip-boot-GPHYs-from-firmware-left-in-RAM.patch` —
  optional `memory-region` on the gphy-fw children: a valid image there
  (magic/length/crc32 trailer, written by the bootloader) is used to boot
  the GPHY; otherwise the firmware file is requested as before.

Status
------
OpenWrt-local patches, also included in commit 0002 of the board series.
802 and 803 are candidates for upstream Linux. 804 would need a DT binding
update (memory-region on lantiq,gphy-fw children) before upstreaming.
