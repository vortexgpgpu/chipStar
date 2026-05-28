# chipStar ROCm-Device-Libs Patches

Patches applied at chipStar CMake configure time to the
`bitcode/ROCm-Device-Libs` submodule (`CHIP-SPV/ROCm-Device-Libs`).
Same shape as [../llvm-patches/](../llvm-patches) and
[../HIPCC-patches/](../HIPCC-patches).

```
ROCm-Device-Libs-patches/
├── 0001-ROCm-Device-Libs-enable-multi-width-OCML-builds-for-.patch
└── README.md
```

## Application

`CMakeLists.txt`'s `apply_submodule_patches(bitcode/ROCm-Device-Libs ...)`
step is idempotent (see [../HIPCC-patches/README.md](../HIPCC-patches/README.md)
for the mechanism).

To re-apply manually:

```bash
cd bitcode/ROCm-Device-Libs
git apply ../../ROCm-Device-Libs-patches/*.patch
```

## Patches

- **0001 — multi-width OCML builds.** Lets chipStar invoke
  `add_subdirectory(ROCm-Device-Libs)` once per target SPIR-V
  pointer width (`CHIP_TARGET_POINTER_WIDTHS="32;64"`) without
  CMake target-name collisions. `opencl_bc_lib` macro and the
  `irif` target suffix their target names by an optional
  `${AMD_DEVICE_LIBS_TARGET_SUFFIX}` parent-scope variable
  (defaults to empty for the standalone-build case so the AMD
  upstream flow is byte-identical). `irif/CMakeLists.txt`'s
  `TARGET_TRIPLE` regex is widened from `spirv64.*` to
  `spirv(32|64).*` so the irif build works on spirv32 too.
  `CMakeLists.txt` guards `add_subdirectory(utils/prepare-builtins)`
  (host-side tool, single instance) and `add_subdirectory(test/
  constant_folding)` (add_test names collide on re-entry) against
  multi-scope invocation. Required by
  [docs/proposals/chipstar_opencl_32bit_proposal.md](https://github.com/vortexgpgpu/vortex/blob/tinebp-patch-2/docs/proposals/chipstar_opencl_32bit_proposal.md)
  §5.2.
