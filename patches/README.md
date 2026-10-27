# Linux kernel patches for the Lantiq xRX330 (GRX390 family)

- **Base:** Linux 6.18 (OpenWrt `target/linux/lantiq/patches-6.18`)
- **Tested:** DrayTek Vigor 2862, xRX330 rev 1.1 (part 0x011, GRX369)
- **Author:** Gregory Dymarek <gregd72002@gmail.com>

## Patches

### [`802-serial-lantiq-support-the-xRX330-interrupt-bit-layout.patch`](802-serial-lantiq-support-the-xRX330-interrupt-bit-layout.patch)

The xRX330 ASC has an extra TX buffer interrupt bit, which moves RX and
error up by one bit. Adds the `lantiq,xrx330-asc` compatible (driver,
binding, early console). The TX buffer interrupt stays disabled.

### [`803-net-lantiq_xrx200-do-not-lose-DMA-interrupts-on-NAPI-re-arm.patch`](803-net-lantiq_xrx200-do-not-lose-DMA-interrupts-on-NAPI-re-arm.patch)

On the xRX330 DMA (rev 8) a TX completion during NAPI could leave the
channel interrupt pending without raising it, stalling TX. Acknowledge,
re-enable, then re-check the next descriptor.

## Status

OpenWrt-local patches, also included in commit 0002 of the board series.
802 and 803 are candidates for upstream Linux. The GPHY firmware needs no
kernel patch: U-Boot boots the GPHYs (commit 0001 of the board series).
PCIe (VRX318 on root complex 0) needs no kernel patch: it is device tree
only (in commit 0003 of the board series).
