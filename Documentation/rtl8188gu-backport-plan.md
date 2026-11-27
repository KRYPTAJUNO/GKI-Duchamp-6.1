# RTL8188GU / RTL8710BU backport plan for GKID Linux 6.1.177

## Scope and safety

Target kernel: GKID-Duchamp-6.1, Linux 6.1.177, branch `GKID-NH` baseline.
Target USB ID: Realtek `0bda:b711` (RTL8710BU, marketed as RTL8188GU).

This work must be additive and backward-compatible. Do not remove, disable, replace, or silently alter existing Realtek drivers or existing USB IDs. Keep the original kernel ZIP and the original upstream repository untouched. Do not enable the new driver in the shipped defconfig until the complete backport builds and is reviewed.

## Working branch

This integration branch is based on the exact source commit used by the GKID NetHunter build path:

- Repository: `KRYPTAJUNO/GKI-Duchamp-6.1`
- Branch: `rtl8188gu-backport-6.1-nh`
- Base commit: `f391d5a36ae325a4f3abdf497335abae87734a05`
- Upstream branch: `ahmed-alnassif/GKI-Duchamp-6.1:GKID-NH`

The user's other working branch, `rtl8188gu-backport-6.1`, was created from `GKID-6.1`; it is retained untouched. The `dev` branch in the builder repository is also out of scope.

## Baseline audit

The target tree's existing `drivers/net/wireless/realtek/rtl8xxxu/` is an older implementation. Its Makefile currently links only:

- `rtl8xxxu_core.o`
- `rtl8xxxu_8192e.o`
- `rtl8xxxu_8723b.o`
- `rtl8xxxu_8723a.o`
- `rtl8xxxu_8192c.o`

The current Kconfig description does not list RTL8710BU/RTL8188GU. The core's USB probe table does not yet include product ID `0xb711`. The current `rtl8xxxu_priv` / `rtl8xxxu_fileops` interfaces also predate the RTL8710BU support.

The official Linux support was introduced by commit
`70664495e3d24803ec47ccaccee9a822220558a0` ("wifi: rtl8xxxu: Support new chip RTL8710BU aka RTL8188GU"). It is not a USB-ID-only change. The implementation adds a dedicated `rtl8xxxu_8710b.c` subdriver and coordinated changes to shared headers, register access, firmware loading, efuse handling, PHY-stat parsing, probe/disconnect lifecycle, and existing chip fileops.

RTL8710BU has a distinct register map: normal low register addresses are accessed through the 0x8000 range, and some system registers require indirect access. It also has distinct Jaguar2 PHY status formats. A superficial USB ID addition would therefore be unsafe and nonfunctional.

## Required backport work

1. Port the dedicated RTL8710BU subdriver and all required register definitions.
2. Port the minimum shared-core changes required for RTL8710BU:
   - chip enum and private state;
   - indirect system-register access and synchronization;
   - chip-specific register read/write routing;
   - firmware download/control-register handling;
   - efuse read/parse path;
   - Jaguar2 PHY-stat parsing dispatch;
   - TX descriptor checksum quirk;
   - USB probe ID and fileops registration;
   - safe init/cleanup of new synchronization state.
3. Update the rtl8xxxu Makefile to include the new object without removing any existing object.
4. Update Kconfig help text and dependencies only as required by the 6.1 tree.
5. Add firmware declarations and package the correct firmware files only after identifying the adapter's silicon/efuse vendor variant. The upstream firmware names are:
   - `rtlwifi/rtl8710bufw_SMIC.bin`
   - `rtlwifi/rtl8710bufw_UMC.bin`
   Do not guess which one the user's adapter needs; keep both available if licensing and packaging permit.
6. Enable `CONFIG_RTL8XXXU` only in a dedicated test config after the code compiles. Do not disturb existing Realtek options such as `CONFIG_R8188EU` and `CONFIG_RTW88`.
7. Build against the exact GKID 6.1.177 toolchain/config, inspect warnings and symbol dependencies, and compare resulting config and module list against baseline.
8. Only after build and runtime tests should the builder fork's integration branch be pointed at this source branch and a separate test ZIP be produced.

## Firmware and adapter-mode notes

The official linux-firmware history added the two RTL8710BU firmware variants. The device may initially enumerate in virtual CD-ROM mode and require a USB modeswitch/eject step before the Wi-Fi interface appears. This behavior must be diagnosed from the actual adapter; do not apply a modeswitch blindly.

## Current status

- [x] Confirmed the target kernel is Linux 6.1.177.
- [x] Confirmed the existing rtl8xxxu integration points and older fileops/core layout.
- [x] Created an isolated branch from the exact GKID-NH source commit.
- [x] Identified the official upstream RTL8710BU support commit and firmware names.
- [ ] Backport source implementation.
- [ ] Review all shared-core changes for 6.1 compatibility and regressions.
- [ ] Compile and validate config/module output.
- [ ] Runtime test with USB ID 0bda:b711.
- [ ] Update only the builder integration branch and create a separate test ZIP.

No kernel source files or configuration files have been modified by this audit document.
