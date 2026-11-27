# RTL8710BU upstream delta audit for the Linux 6.1 backport

Reference upstream commit:
`torvalds/linux@70664495e3d24803ec47ccaccee9a822220558a0`

Title: `wifi: rtl8xxxu: Support new chip RTL8710BU aka RTL8188GU`

This is an audit record only. It does not apply the upstream patch or change kernel source/configuration.

## Upstream files touched

1. `rtl8xxxu/Kconfig` — documentation of supported chip.
2. `rtl8xxxu/Makefile` — links the new 8710B object.
3. `rtl8xxxu/rtl8xxxu.h` — new chip enum, three Jaguar2 PHY-stat structures, 8710BU efuse layout, private-state additions, mutex, fileops callbacks, declarations.
4. `rtl8xxxu/rtl8xxxu_8188e.c` — CCK RSSI callback signature and fileops wiring.
5. `rtl8xxxu/rtl8xxxu_8188f.c` — exports LC calibration helper and adapts CCK RSSI callback.
6. `rtl8xxxu/rtl8xxxu_8192c.c` — fileops wiring.
7. `rtl8xxxu/rtl8xxxu_8192e.c` — CCK RSSI callback signature and fileops wiring.
8. `rtl8xxxu/rtl8xxxu_8710b.c` — new 1,878-line chip subdriver.
9. `rtl8xxxu/rtl8xxxu_8723a.c` — CCK RSSI callback signature and fileops wiring.
10. `rtl8xxxu/rtl8xxxu_8723b.c` — CCK RSSI callback signature and fileops wiring.
11. `rtl8xxxu/rtl8xxxu_core.c` — firmware declarations, address translation, indirect system-register access, efuse read dispatch, PHY parsing dispatch, USB ID registration, probe/init handling.
12. `rtl8xxxu/rtl8xxxu_regs.h` — 8710B register definitions.

## Compatibility hazards for this 6.1 tree

The target 6.1 source currently has a smaller rtl8xxxu object set and older shared structures. In particular, its Makefile does not currently list 8188e/8188f objects, and the inspected old Kconfig/core/header are from an earlier driver generation. Therefore, copying the upstream files wholesale would not be a safe backport.

The upstream change also modifies callback signatures used by several existing chips. Those edits affect existing driver objects, so they must be adapted consistently and checked for all initializers/call sites. Avoid partial edits that leave a mismatched function pointer or null callback.

The upstream commit's special 8710B behavior includes:
- remapping low register addresses into the 0x8000 address window;
- separate indirect system-register read/write operations protected by a mutex;
- different firmware control register handling;
- a chip-specific efuse format/read path;
- three Jaguar2 PHY status formats;
- special transmit descriptor handling;
- chip-specific RF/PHY initialization and calibration.

These details confirm that the port must be treated as a coordinated driver backport, not a one-line USB alias patch.

## Proposed safe port sequence

A. Establish a minimal, reproducible source comparison for the exact 6.1.177 baseline and upstream commit.
B. Port shared header types and register definitions with compile-time layout checks where practical.
C. Port indirect register access and chip-specific efuse/firmware mechanisms.
D. Port the standalone 8710B subdriver and its fileops.
E. Port RX PHY-stat dispatch and TX quirks.
F. Add USB ID `0bda:b711` only after the probe path and fileops are complete.
G. Add only the new object to the existing Makefile list; preserve all existing object entries.
H. Compile the rtl8xxxu module and full kernel using the exact GKID toolchain/config. Check all existing rtl8xxxu chips for regressions.
I. Keep the driver disabled in the normal defconfig until build and runtime validation pass.

## Firmware and mode-switch

The upstream driver declares two firmware variants:
- `rtlwifi/rtl8710bufw_SMIC.bin`
- `rtlwifi/rtl8710bufw_UMC.bin`

Do not guess the adapter's silicon vendor variant. The adapter may also enumerate first as a virtual CD-ROM and require a device-specific mode switch/eject. Confirm its actual USB descriptors and behavior before packaging or running a mode switch.

## Current work status

- Exact GKID-NH base commit confirmed: `f391d5a36ae325a4f3abdf497335abae87734a05`.
- Isolated branch: `rtl8188gu-backport-6.1-nh`.
- Upstream patch audited across all 12 changed files.
- No source driver, Makefile, Kconfig, defconfig, firmware, or existing driver has been modified.
- No build has been run; this GitHub-only workflow does not provide the target ARM64 kernel build environment.
