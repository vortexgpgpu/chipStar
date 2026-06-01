# chipStar HIPCC Patches

Patches applied at chipStar CMake configure time to the
`HIPCC` submodule (`CHIP-SPV/HIPCC`). Same shape as
[../llvm-patches/](../llvm-patches): the patches sit in tree and are
applied to a clean upstream submodule before chipStar's build.

```
HIPCC-patches/
├── 0001-hipcc-accept-offload-spirv32-offload-pointer-width-3.patch
├── 0002-hipcc-relocatable-install-paths.patch
└── README.md
```

## Application

`CMakeLists.txt`'s `apply_submodule_patches(HIPCC ...)` step is
idempotent: it uses `git apply --reverse --check` to detect an
already-applied patch and skips it; otherwise it forward-applies.
A clean `git submodule update --init` followed by `cmake ..` is the
expected flow.

To re-apply manually:

```bash
cd HIPCC
git apply ../HIPCC-patches/*.patch
```

## Patches

- **0001 — `--offload=spirv32` + `--offload-pointer-width={32,64}`.**
  Lets chipStar's `hipcc` emit `Physical32` SPIR-V for 32-bit OpenCL
  device targets (e.g. rv32 Vortex). The duplicate-arg filter learns
  `--offload=spirv32`; a new `--offload-pointer-width=N` flag is
  consumed by hipcc and substituted into HIPCXXFLAGS/HIPCFLAGS/
  HIPLDFLAGS via a `regex_replace` over the `--offload=spirvNN`
  triple (preserving any `vN.N-unknown-chipstar` OS suffix used on
  LLVM 23+). Required by
  [docs/proposals/chipstar_opencl_32bit_proposal.md](https://github.com/vortexgpgpu/vortex/blob/tinebp-patch-2/docs/proposals/chipstar_opencl_32bit_proposal.md)
  §5.4.

- **0002 — relocatable install paths.** Makes a prebuilt chipStar work
  from any extraction path instead of only the build-time
  `CMAKE_INSTALL_PREFIX`. `.hipInfo` records `HIP_PATH` and embeds that
  absolute prefix inside `HIP_OFFLOAD_{COMPILE,LINK,RDC}_OPTIONS`
  (`--hip-path=<prefix>`, `-include <prefix>/include/hip/spirv_fixups.h`,
  `-L<prefix>/lib`, `-Wl,-rpath,<prefix>/lib`). `getHipPath()` already
  self-locates (`/proc/self/exe` → parent, or `HIP_PATH` env), but those
  embedded flags were emitted verbatim — so a tarball unpacked anywhere
  but the original prefix (CI `/home/runner/...`, a non-default
  `TOOLDIR`, another user's home) emitted stale paths and failed. In
  `HipBinSpirv::detectPlatform()`, after `.hipInfo` is parsed, the baked
  prefix (`hipInfo_.hipPath`) is rewritten to the self-located/env prefix
  across the offload flag strings. In-place installs are a no-op
  (real == baked). The external LLVM/clang path (`HIP_CLANG_PATH`, a
  sibling toolchain component, not under the chipStar prefix) is not
  self-locatable and is supplied by the consumer via the standard
  `HIP_CLANG_PATH` env var (Vortex sets it from `$(LLVM_PATH)` in
  `tests/hip/common.mk`).
