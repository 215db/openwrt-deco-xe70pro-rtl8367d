# OpenWrt port: TP-Link Deco XE70 Pro (v3)

This document describes the work-in-progress OpenWrt port for the **TP-Link Deco XE70 Pro (v3)** on the `deco-xe70pro-rtl8367d` branch.

- **Repository:** <https://github.com/215db/openwrt-deco-xe70pro-rtl8367d>
- **Branch:** `deco-xe70pro-rtl8367d`
- **Target/OpenWrt:** `qualcommax` / `ipq50xx`, Linux 6.18
- **Device (OpenWrt):** `tplink_deco-xe70pro` (`tplink,deco-xe70pro`)

> **Status:** WIP / bring-up. Core boot, wired networking, NAND A/B sysupgrade, and tri-band Wi-Fi bring-up are functional in testing with locally supplied, device-specific board data (see “Wi-Fi board data (BDF)”). Wi-Fi board data is **not** included in this repo, and there is no stock web-UI install path. Read the caveats before flashing.

## Hardware

- **SoC/platform:** Qualcomm IPQ5018 (mp03.5-c1)
- **RAM/storage:** 512 MiB RAM, 128 MiB SPI-NAND (128 KiB eraseblocks, 2048-byte pages)
- **Wi-Fi:** IPQ5018 (2.4 GHz) + 2× QCN6122 (5 GHz / 6 GHz) via ath11k
- **Ethernet/switch:** RTL8367SC (managed switch), WAN on EXT1/SDS1
- **Ports:** 2× LAN + 1× WAN (`lan1, lan2` bridged as LAN; `wan` as WAN in default config). No USB.
- **Boot:** U-Boot with A/B NAND rootfs slots and `tp_boot_idx` slot selection (see below)

Identification (GPL/factory): compatible `tplink,deco-xe70pro`, `qcom,ipq5018-mp03.5-c1`, `qcom,ipq5018`; DTS config `config@mp03.5-c1`.

## Current status

### Working (validated in testing)
- Build/boot on `qualcommax/ipq50xx` (Linux 6.18) for this device.
- **RTL8367SC** switch bring-up with **EXT1/SDS1** (WAN) via rtl8365mb driver support (target patches).
- **NAND** install and **A/B sysupgrade** (inactive-slot flash, slot switch, reboot into updated slot).
- **Wired networking** (LAN/WAN via RTL8367SC, GMAC link up).
- **Tri-band Wi-Fi bring-up** (ath11k, with locally packaged OEM-derived BDFs): all three radios/remoteprocs initialize with no firmware/remoteproc crash; 2.4 GHz and 5 GHz functional; 6 GHz radio operational and **6 GHz client association/traffic confirmed** in testing (ch33 / 6115 MHz, HE80) when using the Stage 1 QCN6122 BDF workaround documented below.
- **ART/factory handling:** per-device calibration data and MACs read from device flash (ART/factory partitions) via device integration.

### Known limitations / caveats
- **Wi-Fi board data (BDF/`board-2.bin`) not included:** compatible/working ath11k board data for this device is **not committed** (gitignored/local). The stock/distributed board data available in the build was not usable as-is for this device; local builds require supplying properly sourced `board-2.bin` files (see “Wi-Fi board data (BDF)”). This is the main gap for out-of-the-box Wi-Fi and for upstream/OpenWrt inclusion (provenance/redistribution).
- **No stock web-UI factory install:** TP-Link stock/vendor web-UI factory image format is not addressed. Installation requires **serial console + TFTP/U-Boot** (and `sysupgrade` once running OpenWrt).
- **6 GHz:** client association/traffic has been confirmed in testing with the documented BDFs; broader on-site/regulatory/interoperability validation and confirmation of the optimal BDF remain open.
- **5 GHz / regulatory:** runtime config is transient; 5 GHz ACS may select DFS channels causing vdev start failures (`-22`) on the 5 GHz radio — prefer a fixed non-DFS channel when enabling a 5 GHz AP. Non-fatal ath11k messages like `Failed to set the requested Country regulatory setting` / `failed to process regulatory info -22` (self-managed regulatory, e.g. `US: DFS-FCC`) may appear.
- **A/B fallback:** `sysupgrade` flashes the inactive slot and switches the active slot; there is **no boot-success/rollback counter** implemented. Rely on serial/TFTP recovery and keeping a known-good slot/image.
- **Boot metadata:** `tp_boot_idx` in `0:APPSBLENV` controls slot selection (validated). `0:BOOTCONFIG`/`0:BOOTCONFIG1` primaryboot fields were observed unchanged/not used for OS slot selection; their semantics are not fully characterized and the handler does not modify them.
- **Early port:** expect rough edges; validate on your hardware with recovery available.

## Building

Standard OpenWrt build process from this branch:

1. Set up an OpenWrt build environment and feeds as usual.
2. Select target/subtarget/device:
   - Target system: `Qualcomm Atheros IPQ based (qualcommax)`
   - Subtarget: `Qualcomm Atheros IPQ50xx`
   - Target profile: `TP-Link Deco XE70 Pro`
3. Build (example): `make -j$(nproc)`
4. Images (typical): `bin/targets/qualcommax/ipq50xx/openwrt-qualcommax-ipq50xx-tplink_deco-xe70pro-{initramfs-uImage.itb,squashfs-sysupgrade.bin,squashfs-factory.ubi,initramfs-factory.ubi}` (availability depends on config, e.g. initramfs artifacts when initramfs is enabled).

Device packages include `kmod-ath11k-ahb`, `ath11k-firmware-ipq5018`/QCN6122 firmware as configured, and `kmod-dsa-rtl8365mb`.

## Installation / flashing (NAND, A/B)

**Prerequisites:** serial console access (3.3V UART, **115200 8N1**, e.g. `/dev/ttyUSB0`), a TFTP server, and backups/recovery plan. Flashing can brick devices—proceed at your own risk and keep full NAND/critical-partition backups.

### 1) Boot OpenWrt initramfs via U-Boot (RAM, no NAND write)

1. Connect serial and interrupt autoboot: send the magic string **`tpl`** at the prompt `Enter magic string to stop autoboot in 1 seconds`.
2. Configure network/TFTP as appropriate for your setup (example: `ipaddr=192.168.0.1`, `serverip=192.168.0.10`).
3. Load and boot the initramfs image, e.g.:
   ```text
   tftpboot 0x44000000 openwrt-qualcommax-ipq50xx-tplink_deco-xe70pro-initramfs-uImage.itb
   bootm 0x44000000
   ```
4. Once booted (RAM), verify access (serial/SSH as configured) before writing NAND.

### 2) Install to NAND with `sysupgrade`

From the running OpenWrt system (initramfs or NAND), transfer the sysupgrade image to the device and run:

```sh
sysupgrade -n /tmp/openwrt-qualcommax-ipq50xx-tplink_deco-xe70pro-squashfs-sysupgrade.bin
```

Behavior (implemented):
- Selects the **inactive** rootfs slot (or infers from `tp_boot_idx`/cmdline when booted from RAM), flashes it, and sets `tp_boot_idx` to boot the newly written slot, then reboots.
- A/B slots: `tp_boot_idx=0` → `rootfs` (**mtd11**, slot A), `tp_boot_idx=1` → `rootfs_1` (**mtd12**, slot B).
- Stock slot `kernel`/`ubi_rootfs` UBI volumes on the target slot are removed before flashing.

Validated example: sysupgrade from slot B (`tp_boot_idx=1`, `ubi.mtd=rootfs_1`) flashed slot A (mtd11), set `tp_boot_idx=0`, and booted successfully (`ubi.mtd=rootfs`).

## NAND layout (reference) and recovery

128 MiB SPI-NAND, 128 KiB eraseblocks (`0x20000`), UBI LEB 126976. Offsets/sizes observed (`/proc/mtd`):

| MTD | Name | Offset | Size | Notes |
|---:|---|---:|---:|---|
| mtd0 | `0:SBL1` | `0x00000000` | `0x00080000` | Boot chain (critical) |
| mtd1 | `0:MIBIB` | `0x00080000` | `0x00080000` | Partition table (critical) |
| mtd2 | `0:BOOTCONFIG` | `0x00100000` | `0x00040000` | Boot config |
| mtd3 | `0:BOOTCONFIG1` | `0x00140000` | `0x00040000` | Boot config mirror |
| mtd4 | `0:QSEE` | `0x00180000` | `0x00100000` | TZ/TEE |
| mtd5 | `0:DEVCFG` | `0x00280000` | `0x00040000` | Device config |
| mtd6 | `0:CDT` | `0x002C0000` | `0x00040000` | CDT |
| mtd7 | `0:APPSBLENV` | `0x00300000` | `0x00080000` | U-Boot env (incl. `tp_boot_idx`) |
| mtd8 | `0:APPSBL` | `0x00380000` | `0x00140000` | U-Boot/APPSBL |
| mtd9 | `0:ART` | `0x004C0000` | `0x00100000` | **Radio caldata (critical, device-unique)** |
| mtd10 | `0:TRAINING` | `0x005C0000` | `0x00080000` | Training data |
| mtd11 | `rootfs` | `0x00640000` | `0x02A00000` | OS slot A (UBI) |
| mtd12 | `rootfs_1` | `0x03040000` | `0x02A00000` | OS slot B (UBI) |
| mtd13 | `0:ETHPHYFW` | `0x05A40000` | `0x00080000` | Ethernet PHY FW |
| mtd14 | `factory_data` | `0x05AC0000` | `0x00900000` | **Factory data (critical)** |
| mtd15 | `runtime_data` | `0x063C0000` | `0x01100000` | Runtime data |

**Recovery (U-Boot, TFTP):** restore only what’s needed (prefer explicit offset+size). Example pattern:
```text
tftpboot 0x44000000 <backup-file>
nand erase <offset> <size>
nand write 0x44000000 <offset> <size>
```
- **Never lose** `0:ART` (mtd9) and `factory_data` (mtd14). Avoid touching `0:SBL1/0:MIBIB/0:QSEE/0:APPSBL/0:CDT/0:BOOTCONFIG*` unless necessary.
- To change the active slot manually, set `tp_boot_idx` in `0:APPSBLENV` (`0` → `rootfs`/mtd11, `1` → `rootfs_1`/mtd12) via `fw_setenv` in OpenWrt or U-Boot env handling, consistent with a valid image in that slot.
- TFTP-booting initramfs is the safest inspection/recovery path.

## Wi-Fi details

- **Stack/firmware:** ath11k (IPQ5018 + QCN6122, AHB), firmware family used in testing: `WLAN.HK.2.7.0.1-01744-QCAHKSWPL_SILICONZ-1` (`fw_version 0x270206d0`, build 2022-08-04). Tested on OpenWrt `qualcommax/ipq50xx`, Linux 6.18.
- **Empirical radio mapping (native cal kept per node), as integrated:**
  - `&wifi` / `c000000.wifi` (q6_wcss_pd1, `cd00000.remoteproc`) / `phy0`: **2.4 GHz** — BDF addr `0x4c400000`, ART cal `0:ART@0x1000`
  - `&wifi1` / `b00a040.wifi` (q6_wcss_pd2) / `phy1`: **6 GHz** — BDF addr `0x4d100000`, m3 dump `0x4df00000`, ART cal `0:ART@0x26800`
  - `&wifi2` / `b00b040.wifi` (q6_wcss_pd3) / `phy2`: **5 GHz** — BDF addr `0x4d200000`, m3 dump `0x4e000000`, ART cal `0:ART@0x4C000`
  - MAC assignment (from `factory_data`/defaults) follows this mapping (label+2 / +4 / +3 for 2.4/6/5 GHz respectively). See `target/linux/qualcommax/dts/ipq5018-deco-xe70pro.dts` and `target/linux/qualcommax/ipq50xx/base-files/etc/hotplug.d/firmware/11-ath11k-caldata`.

## Wi-Fi board data (BDF) — detailed findings

Compatible, redistributable ath11k board data for this exact device is **not** currently shipped in the distributed firmware packages used by this build, and the working BDFs are **not committed** to this repo (kept as local, gitignored `files/` overrides during development) due to provenance/redistribution constraints. This section documents exactly what works locally, where the payloads came from, how the OEM payloads differ from the distributed/working data, and the workaround required for QCN6122.

### What works (locally, validated)

Working `ath11k` `board-2.bin` files (`QCA-ATH11K-BOARD` containers, built with `ath11k-bdencoder`) packaged from OEM Deco XE70 Pro BDF payloads and installed as:
- `files/lib/firmware/ath11k/IPQ5018/hw1.0/board-2.bin`
- `files/lib/firmware/ath11k/QCN6122/hw1.0/board-2.bin`

Per-radio configuration and board match names (as requested by DTS / used successfully):

| Radio (DTS node) | Chip / band | `qcom,calibration-variant` | ART cal | `board-2.bin` match name(s) |
|---|---|---|---|---|
| `&wifi` `c000000.wifi` / `phy0` | IPQ5018 2.4 GHz | `TP-Link-Deco-XE70-Pro` | `0:ART@0x1000` | `bus=ahb,qmi-chip-id=0,qmi-board-id=255` and `bus=ahb,qmi-chip-id=0,qmi-board-id=255,variant=TP-Link-Deco-XE70-Pro` |
| `&wifi1` `b00a040.wifi` / `phy1` | QCN6122 6 GHz | `TP-Link-Deco-XE70-Pro-6G` | `0:ART@0x26800` | `bus=ahb,qmi-chip-id=0,qmi-board-id=255,variant=TP-Link-Deco-XE70-Pro-6G` and fallback `bus=ahb,qmi-chip-id=0,qmi-board-id=255` |
| `&wifi2` `b00b040.wifi` / `phy2` | QCN6122 5 GHz | `TP-Link-Deco-XE70-Pro-5G` | `0:ART@0x4C000` | `bus=ahb,qmi-chip-id=0,qmi-board-id=255,variant=TP-Link-Deco-XE70-Pro-5G` |

With these BDFs, all three radios/remoteprocs come up (no board/caldata load failures, no remoteproc crash). 6 GHz client association/traffic was confirmed in testing (e.g. `phy1-ap0` on ch33 / 6115 MHz, HE80). (Non-fatal ath11k regulatory messages such as `failed to process regulatory info -22` may still appear.)

### OEM payload origins (from stock firmware)

The working BDF payloads are taken from the OEM firmware’s on-device files (raw BDF payloads, **not** `QCA-ATH11K-BOARD` containers) for the `XE75Pro_US_v1` set found in the stock image:

| Source (OEM) | Used for | Size | SHA-256 | Notes |
|---|---|---:|---|---|
| `IPQ5018/bdwlan_XE75Pro_US_v1.b24` | IPQ5018 2.4 GHz | 131072 | `9db26bf9b95eca25caaa9470f278272a3056ccc41b190c8a0b83561b1bd47889` | raw payload, whole-blob `xor16=0xffff`, `byte@0x45c=0x03` |
| `qcn6122/bdwlan_XE75Pro_US_v1.b60` | QCN6122 5 GHz | 131072 | `f54b1eb50ce6458a779f6dcd1e16eb7dbbcb5fa9f0f3a0e8559bbb9f621b39b4` | raw payload, whole-blob `xor16=0xffff`, `byte@0x45c=0x03` (crashes unmodified; see workaround) |
| `qcn6122/bdwlan_XE75Pro_US_v1.bb0` | QCN6122 6 GHz | 131072 | `877a870f6ed222e9169a7c04d571961ff39dbe053ca7a0d34a107c405a93214b` | raw payload, whole-blob `xor16=0xffff`, `byte@0x45c=0x03` (crashes unmodified; see workaround) |

- **IPQ5018:** the OEM `b24` payload is used **unchanged** (byte-identical to the working payload; board MD5 `65ea43d14b9d02551d95b25a332c91ec`) and only wrapped into a `board-2.bin` with the device-correct variant/fallback names above.
- **QCN6122:** direct-copy/wrapped OEM payloads were **not** usable with this firmware as-is (see symptom/workaround below). The two OEM QCN6122 payloads (`b60` vs `bb0`) are substantially different from each other (band-specific data).

### Differences vs a distributed QCN6122 `board-2.bin` (OpenWrt/ipq-wifi example)

As a concrete example of a **distributed** (Linux/OpenWrt-packaged) QCN6122 board file, the `ipq-wifi` package for TP-Link Archer AX55 v1 (`ipq-wifi-tplink_archer-ax55-v1`, built from `package/firmware/ipq-wifi`) ships a QCN6122 `board-2.bin` containing:

- name: `bus=ahb,qmi-chip-id=0,qmi-board-id=255,variant=TP-Link-Archer-AX55-v1`
- payload: 131072 bytes, SHA-256 `369e15d6a3b464f53c726c77ddb7604053b8d5c0bc20e2d35c248f6f5e730327`, MD5 `fa307b8aa474286e77d3756e498bf667`, `byte@0x45c=0x00`, trailing 16-bit word `0x0000`, whole-blob `xor16=0xffff`

Comparison (payload-level):

| Compared payloads | `byte@0x45c` | Differing bytes (of 131072) | Notes |
|---|---:|---:|---|
| OEM 5G `b60` vs distributed AX55 v1 QCN6122 | `0x03` vs `0x00` | 32296 | very different payloads (different device/band data) |
| OEM 6G `bb0` vs distributed AX55 v1 QCN6122 | `0x03` vs `0x00` | 34814 | very different payloads |
| OEM 5G `b60` vs OEM 6G `bb0` | `0x03` vs `0x03` | (substantial) | distinct band-specific OEM payloads |
| OEM QCN6122 (`b60`/`bb0`) vs **working (patched)** | `0x03` → `0x00` | **2** | only `0x45c` and the checksum fix byte (see below) |

No matching Deco XE70 Pro QCN6122 (or IPQ5018) board entry was identified in the distributed linux-firmware/ath11k board data examined in this build; the OEM Deco payloads are not interchangeable with the distributed AX55 v1 QCN6122 payload, and the OEM QCN6122 payloads required the targeted fix below to work with this firmware.

Additionally, the `ath11k-firmware-ipq5018-qcn6122` package (firmware `WLAN.HK.2.7.0.1-01744-QCAHKSWPL_SILICONZ-1` in this build) provides only the radio firmware binaries (`q6_fw`/`m3_fw`, notices/flists) for IPQ5018 and QCN6122 and **no** `board-2.bin` (board data); board data must be supplied separately (e.g. via linux-firmware, ipq-wifi, or device-specific files).

### QCN6122 `0x45c` workaround (symptom, change, and result)

**Symptom (OEM QCN6122 payloads `b60`/`bb0` used as-is, wrapped into `board-2.bin`):** firmware/remoteproc crash during QCN6122 bring-up with  
`phyrf_bdf.c:6245 Assertion axis_value == ANTENNACHAIN_AXIS_Z failed` (reported in the remoteproc/Q6 bring-up path, e.g. `cd00000.remoteproc`), causing the affected QCN6122 radio(s) to fail to start.

**Minimal payload change applied (“Stage 1”):** for each QCN6122 OEM payload (`b60` → 5 GHz, `bb0` → 6 GHz), modify exactly:
1. `byte@0x45c`: `0x03` → `0x00` (offset 1116 decimal; the 16-bit LE word at `0x45c` becomes `0x0000`).
2. Restore the BDF integrity checksum so the **whole 131072-byte blob 16-bit LE XOR sums to `0xffff`**: set `byte@0x1fffe` (offset 131070, the last 16-bit word, which lies in a trailing zero region) `0x00` → `0x03` (making the final word `0x0003`). No other bytes changed.

| Payload | Original (`b@0x45c`, last word, xor16) | Patched (`b@0x45c`, last word, xor16) | Patched SHA-256 | Differing bytes vs OEM |
|---|---|---|---|---|
| QCN6122 5G (`b60`) | `0x03`, `0x0000`, `0xffff` | `0x00`, `0x0003`, `0xffff` | `6f538fbb0e4eeb9d18e235e1b9a42bfe92f69536200dce99fb1f823270437240` | 2 (`0x45c`, `0x1fffe`) |
| QCN6122 6G (`bb0`) | `0x03`, `0x0000`, `0xffff` | `0x00`, `0x0003`, `0xffff` | `ff85555170ef838eb93736e57885f1618b9ef31909682e061865cf255cefd622` | 2 (`0x45c`, `0x1fffe`) |

(Context: bytes `0x450..0x467` change from `... 03 00 00 00 ...` to `... 00 00 00 00 ...` at `0x45c`; the checksum correction is confined to the trailing zero word at `0x1fffe`.)

**Result:** with the patched QCN6122 payloads (and the unchanged OEM IPQ5018 `b24` payload) packaged with the correct board names/variants, all three radios and remoteprocs initialize reliably and Wi-Fi operates (including 6 GHz client association in testing) with no assertion/remoteproc crash.

> **Note:** this is an empirically derived, minimal workaround validated for bring-up with the firmware above; the “correct” vendor/QCA BDF data for this board may differ. Treat it as a documented finding pending redistributable, authoritative board data.

### Packaging & operational notes

- OEM `bdwlan*` files are raw BDF payloads; ath11k expects a `QCA-ATH11K-BOARD` `board-2.bin` container with matching `bus`/`qmi-*`/`variant` names (e.g. packaged via `ath11k-bdencoder`).
- Because the working data is OEM-derived (and QCN6122 is locally patched), it is kept out of the repo; builds/images must include compatible, properly sourced `board-2.bin` files (e.g. via `files/`) for functional Wi-Fi. Upstreaming requires redistributable/provenant board data (and validation of the QCN6122 fix) — see “Contributing / feedback”.
- Prefer a fixed non-DFS channel for 5 GHz if ACS selects DFS and triggers `-22`. The non-fatal ath11k regulatory `-22` messages noted above were observed even when radios were otherwise functional.

## What’s implemented (high level)

- `target/linux/qualcommax/patches-6.18/0957-0958`: rtl8365mb support for RTL8367SC and EXT1/SDS1 (WAN).
- `target/linux/qualcommax/dts/ipq5018-deco-xe70pro.dts`: device DTS (identity, partitions, switch/WAN, radios/cal/BDF mapping, etc.).
- `target/linux/qualcommax/ipq50xx/base-files/...`: board/network defaults, ART/factory MAC & caldata extraction, factory/tp_data mounts, NAND A/B `sysupgrade` (`tp_boot_idx`).
- `target/linux/qualcommax/image/ipq50xx.mk`: device/image definitions (incl. initramfs-factory artifact).
- `package/boot/uboot-tools/uboot-envtools/files/qualcommax_ipq50xx`: U-Boot env access for `0:APPSBLENV`.

## Building an ipq-wifi board-data package (validation)

For out-of-the-box OpenWrt support, device-specific Wi-Fi board data is typically shipped as an `ipq-wifi-<board>` package (from `package/firmware/ipq-wifi/`) that installs `board-2.bin` files under `/lib/firmware/ath11k/...` for the device’s radios/variants. The following process was **validated locally** (package built successfully) to document how to produce such an APK for `tplink_deco-xe70pro`. The package-definition changes were **not committed** and the board data used for validation was the locally prepared, OEM-derived/patched data (not redistributable) — see caveats below.

### 1. Define the package

In `package/firmware/ipq-wifi/Makefile`:

- Add the board to `ALLWIFIBOARDS` (e.g. alongside other TP-Link Deco entries):
  ```make
  	tplink_deco-x80-5g \
  	tplink_deco-xe70pro \
  	tplink_eap225-wall-v2 \
  ```
- Register the package (matching position/name):
  ```make
  $(eval $(call generate-ipq-wifi-package,tplink_deco-xe70pro,TP-Link Deco XE70 Pro))
  ```

The `ipq-wifi` package installs `board-<devicename>.*` files from its build/source directory (`$(PKG_BUILD_DIR)`) based on filename suffix (see `ipq-wifi-install-one`): `.ipq5018` → `ath11k/IPQ5018/hw1.0/board-2.bin`, `.qcn6122` → `ath11k/QCN6122/hw1.0/board-2.bin`.

### 2. Provide the board data files

For a proper/upstreamable package, `board-tplink_deco-xe70pro.ipq5018` and `board-tplink_deco-xe70pro.qcn6122` (ath11k `QCA-ATH11K-BOARD` `board-2.bin` containers with the correct match names/variants) must be added to the ipq-wifi source fetched by the package (the `qca-wireless` repo / `PKG_SOURCE_*` tarball) and the package version/bump updated accordingly.

For local validation in this tree (where the source is extracted to `build_dir/target-aarch64_cortex-a53_musl/ipq-wifi-<version>/`), copy/prepare the board files into that build directory with the expected names:

```sh
# from the OpenWrt build root (example: using locally prepared working BDFs)
cp files/lib/firmware/ath11k/IPQ5018/hw1.0/board-2.bin \
  build_dir/target-aarch64_cortex-a53_musl/ipq-wifi-2026.10.25~49ce14c0/board-tplink_deco-xe70pro.ipq5018
cp files/lib/firmware/ath11k/QCN6122/hw1.0/board-2.bin \
  build_dir/target-aarch64_cortex-a53_musl/ipq-wifi-2026.10.25~49ce14c0/board-tplink_deco-xe70pro.qcn6122
```

(The exact `ipq-wifi-<version>` directory name follows `PKG_SOURCE_DATE/PKG_SOURCE_VERSION` in `package/firmware/ipq-wifi/Makefile`.)

### 3. Enable and build the package

Enable the new package (e.g. append to `.config`) and build:

```sh
echo 'CONFIG_PACKAGE_ipq-wifi-tplink_deco-xe70pro=m' >> .config
make package/firmware/ipq-wifi/clean
make package/firmware/ipq-wifi/prepare
# (re-)place board-tplink_deco-xe70pro.{ipq5018,qcn6122} into PKG_BUILD_DIR after prepare
make package/firmware/ipq-wifi/compile -j1 V=sc
```

Notes: `clean` removes the extracted package build dir (so board files placed there must be restored after `prepare`), while `compile` installs the matching `board-*.{ipq5018,qcn6122}` files into the package’s `lib/firmware/ath11k/...` paths and produces the APK.

### 4. Result (validated)

A successful build produced:

- `bin/targets/qualcommax/ipq50xx/packages/ipq-wifi-tplink_deco-xe70pro-2026.10.25~49ce14c0-r1.apk`
- Package payload:
  - `/lib/firmware/ath11k/IPQ5018/hw1.0/board-2.bin` (from `board-tplink_deco-xe70pro.ipq5018`)
  - `/lib/firmware/ath11k/QCN6122/hw1.0/board-2.bin` (from `board-tplink_deco-xe70pro.qcn6122`)

(Validation used the locally prepared, working board-2.bin files documented in “Wi-Fi board data (BDF) — detailed findings”: OEM-derived payloads packaged with the correct Deco XE70 Pro variant names, including the QCN6122 `0x45c`/checksum workaround.)

### Caveats

- **Redistribution/provenance:** the working board data for this device is OEM-derived (and QCN6122 is locally patched) and is not cleared for redistribution. An ipq-wifi package for upstream/OpenWrt must contain properly sourced, redistributable (and, for QCN6122, validated) board data. Do not ship unprovenanced blobs.
- **Upstreaming:** add the board files to the ipq-wifi firmware source repository referenced by `package/firmware/ipq-wifi/Makefile`, bump `PKG_SOURCE_*`/hash accordingly, add the board/package entries, and have the device image select `ipq-wifi-tplink_deco-xe70pro`. Until such data is available, provide board data locally (e.g. via `files/`) as documented above.

## Contributing / feedback

Testing, corrections (especially BDF/board data provenance for this model, 6 GHz behavior, and bootloader nuances), and improvements are welcome via issues/PRs on the repository. Please include device revision, serial logs, and exact images/commits used when reporting results.
