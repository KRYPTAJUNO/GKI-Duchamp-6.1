# RTL8188GU external-module build procedure (GKID 6.1.177)

Status: preparation only. No module has been compiled or loaded yet.

## Pinned driver source

- Repository: `https://github.com/lwfinger/rtl8xxxu`
- Commit: `b6411d673759740b5cf94f150937383dc2efeda6`
- Module output name: `rtl8xxxu_git.ko`
- Driver source advertises USB ID `0bda:b711` and firmware names:
  - `rtlwifi/rtl8710bufw_SMIC.bin`
  - `rtlwifi/rtl8710bufw_UMC.bin`

Do not track the moving `main` branch for a reproducible build. Recheck this pin and its source files before changing it.

## Exact GKID build-tree requirements

The GKID builder uses the Linux source checkout in `$WORKDIR/ksrc`, the generated kernel output tree in `$WORKDIR/out`, and the Neutron Clang toolchain in `$WORKDIR/neutron-clang/bin`. Its build configuration includes `LLVM=1`, `LLVM_IAS=1`, `ARCH=arm64`, `CROSS_COMPILE=aarch64-linux-gnu-`, and the same `O=$WORKDIR/out` output directory.

The external module must be built against the *same* prepared source tree, output tree, final `.config`, generated headers, `Module.symvers`, compiler/toolchain, and build flags as the exact GKID kernel image. A generic Android/Ubuntu header package or the host's running kernel headers are not substitutes.

After the GKID kernel build has completed successfully, the intended out-of-tree build shape is:

```bash
export PATH="$WORKDIR/neutron-clang/bin:$PATH"
make -C "$WORKDIR/ksrc" \
  O="$WORKDIR/out" \
  M="$DRIVER_SOURCE_DIR" \
  ARCH=arm64 LLVM=1 LLVM_IAS=1 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  modules
```

This is a build recipe, not a claim that it has been run. Confirm the actual builder environment and generated output before executing it. Preserve the kernel build tree; do not run a clean target against the kernel output.

## Required checks before any device test

1. Confirm the prepared tree belongs to source commit `f391d5a36ae325a4f3abdf497335abae87734a05` and the exact GKID 6.1.177 configuration used for the target image.
2. Confirm `$WORKDIR/out/.config`, `include/config/auto.conf`, generated headers, and `Module.symvers` exist and correspond to the final kernel build.
3. Build only the external module. Do not change `configs/nethunter.config`, the kernel's in-tree RTL drivers, or the original flashable ZIP as part of this test.
4. Record compiler output and inspect `modinfo rtl8xxxu_git.ko` for vermagic, architecture, dependencies, aliases, and firmware declarations.
5. Run modpost with no unresolved-symbol errors. Check symbol version compatibility against the exact kernel's `Module.symvers`; matching vermagic alone is not sufficient.
6. Check the final module's USB aliases for `0bda:b711` and check for collisions with any in-tree driver's aliases before loading it.
7. Stage the module in a separate test directory. Do not install it into system module directories during initial validation.

## Critical install-script hazard

The pinned repository's Makefile `install` target writes:

```
blacklist rtl8xxxu
```

to `/etc/modprobe.d/blacklist-rtl8xxxu.conf`, in addition to installing the module. Its uninstall target removes that file. Therefore **never run `make install`, `make uninstall`, or `make install_fw` blindly** in the NetHunter/device environment. Do not blacklist the in-tree driver and do not replace any existing module.

Firmware must be staged separately and only after its provenance, licensing, required variant, and packaging destination are verified. Do not assume both firmware variants are interchangeable or copy firmware into the live system during the compile-only phase.

## Device isolation and rollback

Before any load test, capture the current USB descriptors and loaded driver/module state. Verify whether the adapter enumerates directly as Wi-Fi or first as a virtual CD-ROM; only use USB modeswitch/eject after the descriptors confirm that mode. During the first test, bind only the target adapter to the test module and verify existing Wi-Fi and other kernel functions remain intact. Keep the original kernel ZIP and current bootable image untouched, and prepare a verified recovery path before any flash test.

## Current gate

This branch currently contains documentation only. The pinned external source has not been compiled against GKID 6.1.177, its firmware has not been staged, and no module has been loaded or flashed. Do not enable the in-tree `CONFIG_RTL8XXXU` option or publish a flashable ZIP on the basis of this external-module candidate alone.
