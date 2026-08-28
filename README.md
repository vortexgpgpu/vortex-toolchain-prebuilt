# vortex-toolchain-prebuilt

Prebuilt binaries (LLVM, POCL, chipStar, libc/libcrt, riscv-gnu-toolchain,
verilator, yosys, sv2v, sta, SLASH) for the [Vortex GPGPU](https://github.com/vortexgpgpu/vortex).
Pulled by `ci/toolchain_install.sh` from the Vortex repo via the
`TOOLCHAIN_REV` tag pin.

## Installation

    $ git clone --depth=1 https://github.com/vortexgpgpu/vortex-toolchain-prebuilt.git

Or use the Vortex repo's installer (recommended):

    $ cd vortex && ./configure && cd build && ./ci/toolchain_install.sh --all

## Releases

### v3.0.1 (refreshed 2026-08-28)

- **mesa-vortex + POCL**: refreshed bundles.
- **SLASH** *(new)*: AMD Alveo V80 platform userspace (VRT runtime,
  `vrtd`, `v80-smi`, `slashkit`), built from the
  [`vortexgpgpu/SLASH`](https://github.com/vortexgpgpu/SLASH) fork,
  branch `vortex_3.x`. Installed by `toolchain_install.sh --slash` into
  `$TOOLDIR/slash` (use as `VRT_HOME`). Note: built on Ubuntu 24.04
  (noble); although published under the `ubuntu/focal` path (vortex's
  `configure` maps focal/jammy/noble to it), the binaries need
  noble-era glibc — on older distros build SLASH from source. The V80
  kernel driver is not in this tarball; it ships as `.deb` packages
  built from the same fork.

### v3.0 (2026-05-15)

- **LLVM**: LLVM 20.1.8 (Vortex fork, RISC-V + X86 + clang + lld). Bundles
  `llvm-spirv` (SPIRV-LLVM-Translator `llvm_release_200`).
- **POCL**: pocl_vortex on top of upstream `release_7_0`. KMU dispatch
  model. ENABLE_ICD=ON (ships `libpocl.so` + `libOpenCL.so` + `pocl.icd`).
  ENABLE_SPIRV=ON.
- **chipStar** *(new)*: HIP-on-Vortex via SPIR-V. `bin/hipcc`,
  `lib/libCHIP.so`, `lib/llvm/libLLVMHipSpvPasses.so`, device-libs.
- **libcrt32/libcrt64**: compiler-rt builtins rebuilt against LLVM 20.
  libcrt64 previously shipped ELF32 `.S` objects (v2 toolchain defaulted
  to riscv32 so `.S` files never got `-march=rv64`) — now correct ELF64.
- **libc32/libc64**: musl v1.2.5 rebuilt against LLVM 20. riscv32
  setjmp/longjmp patched `fld/fsd` → `flw/fsw` for the ilp32f ABI.

#### Known limitations

- **chipStar HIP is rv64-only.** chipStar's `hipcc` emits SPIR-V with
  `OpMemoryModel Physical64`; POCL checks SPIR-V address bits against
  the device's `address_bits` and refuses to compile Physical64 SPIR-V
  on a 32-bit Vortex device (`CL_INVALID_OPERATION: No device in context
  supports SPIR`). HIP works on rv64 (verified: vecadd, sgemm). For HIP
  on rv32 the upstream chipStar would need a `--offload=spirv32` mode,
  or the native HIPVortex toolchain (see
  [`hip_support_proposal.md`](https://github.com/vortexgpgpu/vortex/blob/master/docs/proposals/hip_support_proposal.md)).
  OpenCL is fully supported on both XLENs.

### v2.3

Prior release (LLVM 21 line, POCL pre-7.0, no chipStar). Kept for users
on the v2.x branch.
