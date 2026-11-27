# RTL8188GU safer integration route: isolated external module candidate

## Finding

The upstream RTL8710BU support is a 12-file coordinated change against a newer in-tree rtl8xxxu implementation. The GKID 6.1.177 tree is older and lacks the 8188e/8188f subdrivers assumed by that upstream patch. Porting the complete change into the in-tree driver would require shared-core changes and creates avoidable regression exposure for the user's existing Realtek devices.

A lower-risk candidate is the standalone backport repository `lwfinger/rtl8xxxu`, whose README states support for Linux 5.6.x through 6.8.x and lists RTL8188GU. Its Makefile builds a separate module named `rtl8xxxu_git.ko`, with the 8710b subdriver included. This is a candidate to test against the exact GKID 6.1.177 build tree, not yet a proven compatible binary.

## Evidence inspected

Repository: https://github.com/lwfinger/rtl8xxxu
Branch inspected: `main`
Files inspected:
- `README.md`
- `Makefile`
- `rtl8xxxu.h`
- `rtl8xxxu_core.c`
- `rtl8xxxu_8710b.c`

The README claims compatibility for Linux 5.6.x–6.8.x and documents RTL8188GU support. The Makefile builds `rtl8xxxu_git.ko` separately from the kernel's in-tree `rtl8xxxu.ko`, and includes `rtl8xxxu_8710b.o`.

## Important install-script hazard

The repository's `make install` target:
- copies the external module into `/lib/modules/...`,
- writes `blacklist rtl8xxxu` into `/etc/modprobe.d/blacklist-rtl8xxxu.conf`,
- runs `depmod`.

**Do not run `make install` or `make uninstall` blindly.** The blacklist could prevent the existing in-tree driver from loading. For this project, use only the external-module build target initially, then manually stage a test copy in a separate location without blacklisting or replacing any in-tree module.

## Required validation before accepting this route

1. Pin the exact external repository commit SHA (do not build from moving `main`).
2. Review its full diff/history and license/firmware provenance.
3. Build `rtl8xxxu_git.ko` against the exact GKID 6.1.177 prepared kernel build output, exact config, generated headers, and toolchain.
4. Check unresolved symbols with `modpost`; inspect vermagic and module ABI compatibility.
5. Confirm the target kernel exports the required mac80211/USB symbols and has compatible cfg80211/mac80211 configuration.
6. Check the module's USB ID table for 0bda:b711 and ensure it binds only that adapter during the first test. Do not blacklist existing drivers.
7. Test load/unload and bind/unbind without writing to boot/vendor partitions. Verify the existing Wi-Fi adapter and all current Realtek drivers remain available.
8. Only after successful runtime tests, decide whether to package this module in a separate test ZIP.

## Rollback

Because this approach is an external module, rollback should be limited to unloading the test module and deleting only the separately staged test copy. Do not change the in-tree rtl8xxxu module, builder configs, boot image, or original GKID ZIP.

## Status

- No kernel driver source or builder config has been changed.
- No external module has been compiled yet.
- No module has been installed or loaded.
- No blacklist has been created.
- No ZIP has been generated.
- This route is a candidate pending an exact-toolchain build and device test.
