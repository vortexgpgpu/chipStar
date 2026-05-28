# chipStar HIPCC Patches

Patches applied at chipStar CMake configure time to the
`HIPCC` submodule (`CHIP-SPV/HIPCC`). Same shape as
[../llvm-patches/](../llvm-patches): the patches sit in tree and are
applied to a clean upstream submodule before chipStar's build.

```
HIPCC-patches/
├── 0001-hipcc-accept-offload-spirv32-offload-pointer-width-3.patch
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
