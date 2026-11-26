# vortex-toolchain-prebuilt

Prebuilt binaries (LLVM, POCL, chipStar, libc/libcrt, riscv-gnu-toolchain,
verilator, yosys, sv2v, sta, CIRCT, Foldsim, Beacon, the Khronos Vulkan loader, and optionally SLASH) for the [Vortex GPGPU](https://github.com/vortexgpgpu/vortex).
Pulled by `ci/toolchain_install.sh` from the Vortex repo via the
`TOOLCHAIN_REV` tag pin.

## Installation

    $ git clone --depth=1 https://github.com/vortexgpgpu/vortex-toolchain-prebuilt.git

Or use the Vortex repo's installer (recommended):

    $ cd vortex && ./configure && cd build && ./ci/toolchain_install.sh --all

## Releases

### v3.0.4 (2026-11-26)

v3.0.3 (`d3d3390`) plus a fixed llvm-vortex, the bundles built with it
(libc, libcrt, `rhel/9` pocl), fixed Foldsim and Beacon, and a new Vulkan
loader bundle. Every other bundle is byte-identical to v3.0.3: CIRCT
(`orcaslab/circt` `90c5fd313`), verilator, yosys, sta, riscv-gnu, chipStar,
mesa-vortex, the `ubuntu/focal` pocl, sv2v, gem5 and SLASH.

- **llvm-vortex** (`ubuntu/focal`, `rhel/9`): `vortex_3.x` @
  `a0df07abbec2` ("A join is never duplicated": `VX_JOIN` is not
  duplicable), built as before (`ubuntu:22.04` GCC 11.4; EL9 system GCC 11),
  with `llvm-spirv`. v3.0.3's `a81a5957f406` duplicated `vx_join` blocks,
  which emptied the IPDOM stack at run time ("IPDOM stack is empty!") and
  crashed the register allocator. It also crashed on musl's `wordexp.c`,
  `lookup_serv.c` and `resolvconf.c` (invalid `vx_split`/`vx_join` dominance
  after instruction selection) and never finished `getgr_a.c`. With
  `a0df07abbec2` all four compile in about a second and pass
  `-verify-machineinstrs`. Same installed file list as v3.0.3 (the focal
  bundle drops a stale `include/llvm/IR/IntrinsicsRISCVXVortex.td`, deleted
  upstream).
- **libc32/libc64**: musl v1.2.5 rebuilt with `a0df07abbec2`. The previous
  libc was compiled on 2026-05-17 by llvm-vortex `87f0227cb`, whose
  divergent-branch lowering turned a `>=` select into `>` (fixed by
  llvm-vortex `69af3fd0084d`): musl's `rint`, and `nearbyint` through it,
  returned `-0.0` for `+0.0` on RV64. Same 245 files and 1346 objects as
  before. riscv32 `setjmp`/`longjmp` use `fsw`/`flw` under `ilp32f` (as in
  v3.0). No per-file workaround is needed any more.
- **libcrt32/libcrt64**: compiler-rt builtins and crtbegin/crtend from the
  `a0df07abbec2` sources, built with that compiler (the old ones were also
  `87f0227cb`). Same 155 (rv64) and 156 (rv32) archive members as before.
- **pocl** (`rhel/9`): `vortex_3.x` @ `471f88096` (the source of both
  v3.0.3 bundles) built natively on EL9 (system GCC 11) against the `rhel/9`
  `a0df07abbec2`. v3.0.3's `rhel/9` slot held a focal binary linked against
  `a81a5957f406`, which imports the old `createLoadStoreVectorizerPass()` and
  does not load with the new LLVM. Same ICD layout as the focal bundle
  (`lib/libpocl.so*`, `etc/OpenCL/vendors/pocl.icd`, rv32 and rv64 kernel
  bitcode) plus `include/CL`, since EL9 ships no OpenCL headers. libpocl
  resolves all of its LLVM imports and loads under `LD_BIND_NOW`. The
  `ubuntu/focal` pocl, chipStar and mesa-vortex resolve all of theirs
  against the new LLVM and are unchanged.
- **Beacon** (`ubuntu/focal`, `rhel/9`): `orcaslab/beacon` `main` @
  `134d03b`, three fixes to the depth mapping. Module bodies that two
  instances share are now mapped in context (`beacon-inline-shared`, at most
  25 % more ops). `(N-1) - i` index reversals are no longer timed as carry
  chains (`beacon-lower-complement`). `a[i][w] <= 1` updates no longer keep a
  read mux under the write (`beacon-inject-in-place`). The area mapping is
  unchanged. `check-beacon` 26/26 on both OSes.
- **Foldsim** (`ubuntu/focal`, `rhel/9`): `orcaslab/foldsim` `main` @
  `5f4df13` = `7b990bb` (lane-scan crash fix: the v3.0.3 `foldsim-opt`
  segfaulted in `Scan::matchPatterns` on every design with a rotated chain,
  because GCC treats a for-condition variable read in the loop step as dead)
  with its `third_party/beacon` moved to `134d03b`, so Foldsim's built-in
  depth model matches the Beacon above. CIRCT `90c5fd313`, built as in
  v3.0.3. `check-foldsim` with Verilator 5.050: 289 passed, 1 unsupported,
  0 failed on both OSes.
- **vulkan-loader** *(new)*: Khronos Vulkan-Loader `v1.4.369`
  (`lib/libvulkan.so*`), `ubuntu/focal` built in `ubuntu:22.04`, `rhel/9` in
  `almalinux:9` (glibc <= 2.34, only libc/libm linked), with the XCB, Xlib,
  Xrandr and Wayland WSI and the distribution search paths
  (`SYSCONFDIR=/etc`). Its own test suite: 702/702 on both. Loaders before
  `v1.4.333` (EL9 ships 1.4.328.1) close the ICD twice when an allocation
  fails in `vkCreateInstance` (upstream `dc509c2de`); CTS
  `api.device_init.create_instance_device_intentional_alloc_fail` crashed
  with the EL9 loader and passes with this one, and `api.device_init`
  gives 241 Pass, 11 NotSupported, 0 Crash. Installed by Vortex
  `ci/toolchain_install.sh --vulkan-loader` (needs the Vortex change that
  moves `TOOLCHAIN_REV` to v3.0.4).

Checked on SimX with the new compiler, libc, libcrt and pocl:
- `rhel/9` (staged tree; installed tree byte-identical): rint, nearbyint,
  rintf and nearbyintf on +-0, +-0.5, +-1.5, +-2.5, NaN and +-inf, one
  thread and one input per lane: all pass on RV64 and RV32 (with the v3.0.3
  libc64, RV64 returned `-0.0` for `+0.0`). OpenCL `tests/opencl` run-simx
  (33 tests): 33/33 on x64 and x32. Vulkan rtquery, tex_fetch, blend and tess
  on x64 and x32: all pass, no device-compiler failure, no IPDOM-stack error.
- `ubuntu/focal`, installed in a clean `ubuntu:22.04`: rint all pass on RV64
  and RV32; OpenCL run-simx 33/33 on x64 and x32. `clang`, `foldsim-opt`,
  `beacon-opt` and `libvulkan.so.1` need at most glibc 2.34.

### v3.0.3 (2026-11-25, CIRCT/Beacon/Foldsim refreshed and rhel/9 added the same day)

**Refresh.** CIRCT, Beacon and Foldsim are rebuilt on `orcaslab/circt`
`main` @ `90c5fd313`. That commit fixes the memory use of CIRCT's cut-based
mapping (Synth `CutRewriter`), which `circt-opt`, Beacon's `beacon-opt` and
Foldsim use. The enumerator now frees the cuts nothing reads any more.
Before, it kept every candidate cut of every node until the module was done.
Results are byte-identical. `beacon run` max RSS: core 3.57 -> 0.73 GB,
gpu32_full 30.3 -> 7.27 GB, gpu_rtx5080 > 64 GB (OOM-killed) -> 26.4 GB.
The Beacon (`4060e3af4`) and Foldsim (`50551085`) sources are unchanged.
`circt-verilog` now prints `CIRCT 90c5fd313`. Vortex's Beacon gate keys its
baselines on that line, so baselines recorded with `ab733d13d` read STALE
until they are re-recorded.

- `ubuntu/focal`: `circt`, `beacon` and `foldsim` rebuilt in `ubuntu:22.04`
  (GCC 11.4), as before. `circt` is still one part.
- `rhel/9` *(new for these three)*: built natively on RHEL 9.8 with
  gcc-toolset-13 (libstdc++ static, glibc <= 2.34). EL9 hosts no longer need
  to patch the focal `circt-verilog`, which binds `hypot@GLIBC_2.35`.

**Original release:**

v3.0.2 plus CIRCT, Foldsim, Beacon (new) and SLASH from their
[orcaslab](https://github.com/orcaslab) `main` branches, the organization the
forks moved to. Every other bundle is unchanged from v3.0.2. All four are built
in an `ubuntu:22.04` container (GCC 11.4, glibc 2.35, Python 3.10), the CI
runners' release, and sit in the `ubuntu/focal` folders that Vortex's
`configure` selects on 20.04, 22.04 and 24.04. The v3.0.2 tag is unchanged.

- **CIRCT**: `circt-verilog`, `circt-opt` and `firtool` from `orcaslab/circt`
  `main` @ `ab733d13d`, built as Vortex `ci/circt_install.sh` builds it
  (Release, slang frontend, libstdc++ static). Only glibc is linked
  dynamically. One 50 MB part.
- **Foldsim**: `orcaslab/foldsim` `main` @ `50551085`, built on that CIRCT.
  Ships `bin/` (the `foldsim` driver, `foldsim-opt`, `foldsim-lift`) and the
  FPGA shell (`share/foldsim`). The Beacon passes left it for Beacon. This
  build includes the fixes for GCC and Python 3.10: a crash of `foldsim-opt`
  on folds with pages in HBM (a read of freed stack), compiler-dependent op
  order in the `fame5` pass, and `--manifest` reading TOML with `tomli` on
  Python 3.10 (`apt install python3-tomli`). `check-foldsim` passes.
- **Beacon** *(new)*: `orcaslab/beacon` `main` @ `4060e3af4`, built on that
  CIRCT: the `beacon` driver, `beacon-opt`, the gate summary and path tools,
  and the calibrations (`bin/`). It replaces Vortex's in-tree `ci/beacon.py`.
  Install it with `toolchain_install.sh --beacon`. `check-beacon` passes.
- **SLASH 1.0.0** *(optional, manual install)*: the Alveo V80 userspace from
  `orcaslab/slash` `main` @ `29eb84fb`, packed from its Debian packages
  without the Vivado static shell. The code is the v3.0.2 bundle's, from the
  orcaslab history; `29eb84fb` lets its Debian packaging build on 22.04
  (`dh-dkms | dkms (<< 3.0)`). Built on 22.04 now, so the Python package moves
  to `lib/python3.10/dist-packages`. Needs `libjsoncpp25`, `libxml2`, `libzmq5`,
  `libinih1`, `python3-jinja2`. Not used by CI or `toolchain_install.sh`.

Checked from the extracted tarballs on a clean `ubuntu:22.04`: the CIRCT and
Beacon binaries load with glibc alone and run; `v80-smi`, `vrtd` and `slashkit`
run with the libraries above. Vortex's Beacon gates (14 Xilinx and 14 ASIC
builds) run with these CIRCT and Beacon bundles and pass against baselines
re-recorded with them.

### v3.0.2 (2026-11-08, pocl refreshed 2026-11-18)

v3.0.1 plus a new LLVM, the pocl rebuilt on it, and two new tools. The LLVM
changes code generation, so instruction and cycle counts move against
baselines recorded with v3.0.1. The v3.0.1 tag stays on its 2026-11-02 refresh.

- **LLVM** (`ubuntu/focal`): llvm-vortex rebuilt at `vortex_3.x` @
  `f44a6bb2af4c`. Adds the XVortex wide loads and stores (`vx_flw.d/.q`,
  `vx_fsw.d/.q`) behind the new `+xvortexldstwide` target feature, and 64-bit
  atomics on RV32 register pairs behind `+xvortexamod`. Also register-group
  and divergence codegen fixes. A `__UNIFORM__` local is promoted to a
  register instead of being reloaded from the stack. With `+xvortex` alone no
  new instruction is emitted. `llvm::createLoadStoreVectorizerPass` now takes
  an optional function filter. The `rhel/9` bundle is still `a81a5957f406`.
- **pocl** (`ubuntu/focal`): the same source (`vortex_3.x` @ `5e06f2f44`) and
  configuration, rebuilt against that LLVM. libpocl now resolves every LLVM
  symbol it imports. The previous build referenced the old
  `createLoadStoreVectorizerPass()` and failed to load under `LD_BIND_NOW`.
- **pocl refresh (2026-11-18)**: rebuilt at `vortex_3.x` @ `471f88096` with
  the same configuration and LLVM. `barrier()` and `mem_fence()` now emit the
  fence their memory scope needs. `atomic_cmpxchg` is correct under contention
  within a warp. `atomic_xchg` on `float` and the `atom_*` names of the
  `cl_khr_*_int32_*_atomics` extensions are added. An image's texture
  descriptor is rewritten only when it changes.
- **CIRCT** *(new)*: `circt-verilog`, `circt-opt` and `firtool` from
  `vortexgpgpu/circt` `vortex_3.x` @ `892a65a01`. These are the SystemVerilog
  frontend of the Vortex synthesis flows. They lower processes that drive
  several interface ports, fold zero-width comparisons, keep RAM initial
  contents and keep port attributes. libstdc++ is linked statically.
- **Foldsim** *(new)*: `vortexgpgpu/foldsim` @ `d8350bc`, built on that CIRCT.
  Ships the `foldsim` driver, `foldsim-opt` (Beacon `foldsim-estimate-timing`
  and `foldsim-estimate-area`), `foldsim-lift` and the FPGA shell
  (`share/foldsim`). Install it with `toolchain_install.sh --foldsim`.
- **SLASH 1.0.0** *(optional, manual install)*: the Alveo V80 userspace
  (`slashkit`, VRT, `vrtd`, `v80-smi`, headers and CMake configs) from
  `vortexgpgpu/SLASH` `vortex_3.x` @ `720754bb`. `vrtd` no longer writes a
  design the card already holds, as XRT does for an xclbin: `slashkit`
  stamps a UUID into each vbin, `vrtd` remembers the UUID it wrote,
  `VRT_FORCE_PROGRAM=1` writes anyway, `v80-smi forget-design` reports a
  design loaded over JTAG, and a design another client uses cannot be
  replaced. The daemon's request format and the client libraries' ABI
  changed while their SONAMEs did not: the daemon and the libraries must
  come from the same build, and a program built against an earlier SLASH
  must be rebuilt. Built on Ubuntu 24.04 (glibc 2.38, Python 3.12); not used
  by CI or `toolchain_install.sh`. See Vortex `docs/xilinx_slash_setup.md`.

### v3.0.1 (refreshed 2026-11-02)

- **RHEL 9 (`rhel/9`)**: new per-OS slot for EL9 hosts (RHEL, Rocky, AlmaLinux,
  CentOS Stream 9). The `ubuntu/focal` builds need glibc 2.35 / `GLIBCXX_3.4.30`
  and do not load there. Built natively on RHEL 9.8 (glibc 2.34, GCC 11):
  llvm-vortex (`vortex_3.x` @ `a81a5957f406`, + `llvm-spirv`), verilator 5.050,
  yosys 0.69 and sta (OpenSTA `2f6ccdae`). The pocl, chipstar and mesa-vortex
  focal binaries are copied unchanged, because they already fit EL9 and link the
  `rhel/9` LLVM ABI. The Vortex installer reuses the `ubuntu/focal` riscv-gnu and
  sv2v bundles for `rhel/9`. No gem5 bundle.

### v3.0.1 (refreshed 2026-10-04)

- **LLVM**: llvm-vortex rebuilt at `vortex_3.x` @ `cd5be82eb447`: divergence
  lowering in large kernels, SimplifyCFG/LowerSwitch order, register-group
  copies, late vector-op legalization.
- **mesa-vortex**: rebuilt at `vortex_3.x` @ `a5650eba435` against Vortex
  master `6f372b0ae` (Vulkan ray-tracing pipelines on the RTU).
- **POCL**: rebuilt at `vortex_3.x` @ `5e06f2f44` against Vortex master
  `6f372b0ae` (both XLEN kernel libraries): by-value struct kernel
  arguments passed inline, single-precision `exp` in float arithmetic.
- **FireSim and SLASH removed.** Both are platform stacks an administrator
  installs once per machine — like XRT, which was never packaged here — and CI
  never consumed either. The matching `toolchain_install.sh --firesim` /
  `--slash` flags are gone from the Vortex repo; install instructions live in
  `docs/building_toolchain.md` (FireSim) and `docs/xilinx_slash_setup.md`
  (SLASH).

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
