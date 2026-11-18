# OpenWrt support for TP-Link Deco XE70 Pro (RTL8367SC / EXT1 SDS1 WAN)

This repository contains work-in-progress OpenWrt support for the **TP-Link Deco XE70 Pro (v3, IPQ5018 + QCN6122)** using the **Realtek RTL8367SC** switch (family D, chip ID `0x6642`, version `0x0050`) with the WAN port connected via **EXT1 / SDS1**.

It is published in two forms:

- **`main` branch (this branch):** focused support repo with the relevant patches/files and apply/build notes.
- **`deco-xe70pro-rtl8367d` branch:** full OpenWrt tree/fork with the changes applied as commits (based on OpenWrt commit `9dab3403f5` at the time of creation).

## Hardware / topology

- SoC: Qualcomm IPQ5018
- Wi-Fi: QCN6122
- Switch: RTL8367SC (`RTL8367D` family), `0x6642/0x0050`
- `gmac1` ↔ switch **port 6 / EXT0**: `2500base-x` (CPU uplink)
- Switch **port 7 / EXT1 / SDS1** ↔ external **RTL8221B** PHY: `2500base-x` (WAN)
- LAN ports: `lan1`, `lan2` (DSA user ports on the RTL8367SC)
- Network config: `lan1 lan2` in `br-lan`, `wan` as WAN interface

## What is included

### Kernel patches (qualcommax, Linux 6.18)
- `patches/qualcommax-6.18/0957-net-dsa-realtek-rtl8365mb-add-support-for-RTL8367SC.patch`
  - Adds RTL8367SC to the `rtl8365mb` chip table (family D, EXT0/EXT1 serdes interfaces).
- `patches/qualcommax-6.18/0958-net-dsa-realtek-rtl8365mb-add-RTL8367SC-EXT1-SDS1-support.patch`
  - Adds per-instance SerDes/PCS handling for EXT0/SDS0 (port 6) and EXT1/SDS1 (port 7).
  - Programs EXT1 via SDS index 15 / `SDS1_MISC0` (`0x1d78`) with indirect SDS config, clears `0x1305` GMII_1 and `TOP_CON0.MAC7_SEL_EXT1` for serdes modes (vendor RTL8367D-aligned behavior), and applies post-step `0x612F=0x000C`.
  - Fixes `SDS1_MISC0` PA enable bit positions for SDS1 (`PA33PC_EN_S1=BIT(10)`, `PA12PC_EN_S1=BIT(9)`).
  - Adds verbose `dev_info` debug register dumps around serdes/MAC/force-mode operations (useful for bring-up; can be reduced later).

### Device support
- `files/target/linux/qualcommax/dts/ipq5018-deco-xe70pro.dts`
- `files/target/linux/qualcommax/image/ipq50xx.mk` (adds `tplink_deco-xe70pro` image/device definition)
- `files/target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network` (`lan1 lan2` LAN, `wan` WAN, TP-Link `tp_data/default-mac` handling)

### UNIPHY PCS preconfiguration (IPQ5018)
- `files/target/linux/qualcommax/files/drivers/net/ethernet/stmicro/stmmac/dwmac-ipq5018.c`
- `files/target/linux/qualcommax/files/drivers/net/pcs/pcs-qca-uniphy.c`
- `files/target/linux/qualcommax/files/include/linux/pcs/pcs-qca-uniphy.h`

These correspond to the UNIPHY PCS preconfiguration changes from **openwrt/openwrt#26597** (commit `1a8053e5a1060f094d0ad77407cbeb12134315c4`) and are included because they are needed for reliable SerDes link bring-up on this platform.

## Apply to an OpenWrt tree

The changes were developed and hardware-validated on a qualcommax/ipq50xx Linux 6.18 tree around OpenWrt commit `9dab3403f5` (`generic: rtl8365mb: bring the SerDes up once the conduit is up`).

1. Copy the kernel patches into `target/linux/qualcommax/patches-6.18/`:
   - `0957-net-dsa-realtek-rtl8365mb-add-support-for-RTL8367SC.patch`
   - `0958-net-dsa-realtek-rtl8365mb-add-RTL8367SC-EXT1-SDS1-support.patch`
2. Copy the device/target files from `files/` into the matching paths in your OpenWrt tree (preserving paths):
   - `target/linux/qualcommax/dts/ipq5018-deco-xe70pro.dts`
   - `target/linux/qualcommax/image/ipq50xx.mk`
   - `target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network`
   - `target/linux/qualcommax/files/drivers/net/ethernet/stmicro/stmmac/dwmac-ipq5018.c`
   - `target/linux/qualcommax/files/drivers/net/pcs/pcs-qca-uniphy.c`
   - `target/linux/qualcommax/files/include/linux/pcs/pcs-qca-uniphy.h`
3. Refresh/prepare and build:

```sh
make target/linux/clean
make target/linux/prepare
make menuconfig   # Target System: Qualcomm Atheros IPQ50xx, select TP-Link Deco XE70 Pro
make -j$(nproc)
```

(If `0957`/`0958` sequence numbers conflict with your tree, renumber them while keeping `0958` after `0957` and after the RTL8367S-VB/family-D support patches they depend on.)

## Notes / validation

- EXT1/SDS1 register programming was validated against vendor GPL sources (RTL8367D DAL/register definitions) rather than guessed.
- Hardware-validated behavior observed on Deco XE70 Pro: port 6 (EXT0) 2500base-x CPU link up, port 7 (EXT1/SDS1, idx 15) 2500base-x WAN link up with traffic (`sds1(1d78)=0x0612`, `1305=0xc000`, `1d70=0x0000`, MAC7 force `0x1075/0xffff`).
- `0958` currently includes verbose debug logging; consider downgrading/removing once stable.

## License

Files are provided under their respective upstream/OpenWrt licenses (see file headers/SPDX identifiers where present).
