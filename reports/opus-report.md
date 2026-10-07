# Migration Readiness Report — opus (libopus)

**Project:** opus (Xiph.Org libopus)
**Reference platform:** x86_64 (native, local container)
**Target platform:** riscv64 (cross-compiled with riscv64-linux-gnu-gcc, executed under QEMU user-mode emulation `qemu-riscv64-static`)
**Pipeline:** Amphimixis migration-readiness (analyze → configure → build/verify → profile → optimize → repeat → report)
**Report language / terminology:** reference platform = x86_64; target platform = riscv64.

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Project | opus (Xiph.Org libopus) |
| Resolved clone URL | https://github.com/xiph/opus.git |
| Canonical upstream | https://gitlab.xiph.org/xiph/opus.git |
| Latest commit | `503d81b138d76621aae4b12786e90de48aa8db3a` — "Add generic PFA FFT/MDCT C implementation" (2026-09-11) |
| Total commits | 5713 |
| Latest tag | v1.6.1 (prior: v1.6, v1.5.2, v1.5.1, v1.5, v1.4, v1.3.1) |
| Activity | Actively maintained (last push 2026-09-11) |
| License | BSD-3-Clause |
| Build systems | CMake, autotools (`configure.ac`/`Makefile.am`), Meson, `Makefile.unix` |
| Test count | CMake registers 5 tests; autotools `make check` runs 16 tests |
| External dependencies | None at runtime beyond the C runtime / `libm`. Optional build-time DNN model tarball (`media.xiph.org/opus/models/opus_data-<sha>.tar.gz`) is fetched only for DRED/DeepPLC/OSCE, which are disabled by default. `libssp` is MinGW-only. |
| Distro packages | Debian `opus 1.6.1-1` (testing/unstable) and `1.5.2-2` (stable); Arch extra `opus 1.6.1-1`; Yocto meta-oe `libopus 1.6.1` |
| Forks with target-arch patches | One known: xiph/opus PR #476 "Add initial RISC-V platform recognition" (author `carlosqwqqwq`, fork `carlosqwqqwq/opus`, branch `codex/riscv-opus`, commit `09f25206`), opened 2026-06-10 and CLOSED UNMERGED. It adds only `__riscv` platform recognition (no RVV backend). No other riscv64/arm64 fork found. |

## 2. Platform-Specific Code Analysis

### Architecture macros

| Macro | Location (example) | Meaning / semantics | Portability concern for riscv64 |
|---|---|---|---|
| `__x86_64__` | `celt/arch.h:124` | selects `OPUS_FAST_INT64 1` | none (also matched by `__LP64__`) |
| `__LP64__` | `celt/arch.h:124` | fast 64-bit int assumption | riscv64 defines it, so `OPUS_FAST_INT64=1` already |
| `__i386__` | `celt/x86/x86cpu.c:61`; `cmake/cpu_info_by_asm.c:8` | 32-bit x86 CPUID helper | x86-only, harmless |
| `__SSE__` | `celt/float_cast.h:65`; `celt/x86/x86_arch_macros.h:31` | `float2int` via `_mm_cvt_ss2si` | x86-only; generic `lrintf` fallback exists |
| `__SSE2__` | `dnn/vec.h:38`; `celt/x86/x86_arch_macros.h:37` | SSE2 DNN kernels | x86-only |
| `__SSE4_1__` | `dnn/x86/nnet_sse4_1.c:34`; `celt/x86/x86_arch_macros.h:43` | SSE4.1 DNN/CELT kernels | x86-only |
| `__SSSE3__` | `dnn/vec_avx.h:639` | SSSE3 DNN dot-product path | x86-only |
| `__AVX__` | `dnn/vec_avx.h:54,171,254,389,558` | AVX DNN kernels | x86-only |
| `__AVX2__` | `dnn/nnet_avx2.c:34`; `dnn/vec_avx.h:165,305,629` | AVX2 DNN kernels | x86-only |
| `__FMA__` | `dnn/vec_avx.h:300` | FMA fast-convolution path | x86-only |
| `_M_X64` / `_M_IX86` | `celt/float_cast.h:70,82` | MSVC x64/x86 `float2int` | Windows-only |
| `__aarch64__` | `celt/float_cast.h:101`; `celt/arm/*`; `celt/celt_tx_tables.h:78` | enables AArch64 NEON intrinsics/asm | ARM64 only; C fallback elsewhere |
| `__arm__` / `__ARM_ARCH` / `__ARM_NEON__` / `__ARM_FEATURE_DOTPROD` | `dnn/vec_neon.h:37`; `celt/arm/*` | ARMv7/aarch64 NEON/DOTPROD DNN and CELT kernels | ARM-only; generic C fallback |
| `__thumb__` / `__thumb2__` | `celt/arm/kiss_fft_armv5e.h:35` | Thumb-mode FFT selection | ARM-only |
| **RISC-V (`__riscv*`)** | **none** | **No RISC-V recognition anywhere; no `__riscv_vector`** | **RISC-V relies entirely on the generic C fallback** |

### Platform preprocessor guards

| Guard | Platform | Scope (file:line examples) |
|---|---|---|
| `_WIN32` | Windows | `silk/debug.h:50,81,93,151`; `silk/debug.c:42,48,72,83`; `silk/typedef.h:54`; `dnn/nnet.c:49`; `include/opus_types.h:59`; `include/opus_defines.h:67`; `celt/stack_alloc.h:43,105` |
| `_WIN64` | Win64 | `celt/arch.h:124` |
| `_WINCE` | WinCE | `silk/debug.h:50` |
| `_MSC_VER` | MSVC | `silk/x86/NSQ_del_dec_avx2.c:74`; `silk/typedef.h:70`; `celt/ecintrin.h:51-52`; `celt/arch.h:76`; `celt/float_cast.h`; `include/opus_defines.h:92,104`; `dnn/nnet.h:158` |
| `__linux__` | Linux | `celt/arm/armcpu.c:95` (`/proc/cpuinfo` NEON detection) |
| `__APPLE__` | macOS | `celt/arm/armcpu.c:168`; `celt/arm/celt_arm_asm.h:39,45,116,151`; `include/opus_types.h:93` |
| `__MINGW32__` | MinGW | `include/opus_types.h:67`; `tests/test_opus_encode.c:39`; `tests/test_opus_custom.c:16` |
| `__CYGWIN__` | Cygwin | `include/opus_types.h:61` |
| `__GNUC__` | GCC | many (builtins, `OPUS_GNUC_PREREQ`, visibility) |
| `__clang__` | Clang | `celt/arm/armcpu.h:80`; `dnn/vec_neon.h:37`; `silk/decode_core.c:63`; `celt/x86/x86cpu.h:86` |
| `__ELF__` (implicit) | ELF vs Mach-O | `celt/arm/celt_arm_asm.h` (`.hidden` vs `.private_extern`) |

Feature macros with checked semantics: `OPUS_FAST_INT64` (64-bit int speed class; already 1 on rv64 via `__LP64__`), `OPUS_HAVE_RTCD` (runtime CPU dispatch; x86/ARM only), `OPUS_X86_MAY_HAVE_*`/`OPUS_X86_PRESUME_*`, `OPUS_ARM_*`, `FIXED_POINT` (integer mode), `FLOAT_APPROX` (fast float approximations; autotools enables for x86/arm/ppc/ia64 but NOT riscv), `DISABLE_NEON`, `_M_IX86_FP`.

### Portability verdict

| Aspect | Verdict |
|---|---|
| Platform exceptions / unguarded assumptions | PASS — pure C project (no `.cpp`/`.cc`/`.cxx` sources found); every SIMD/intrinsic path is wrapped in architecture guards and has a generic C fallback. No C++ exception machinery. |
| Alignment safe | PASS (assessment) — alignment is explicitly managed: `celt/stack_alloc.h` defines `ALIGN`/`PUSH` stack-alignment macros, `celt/kiss_fft.h` uses `__attribute__((aligned(...)))`, `silk/SigProc_FIX.h` defines `silk_DWORD_ALIGN`. No unaligned-load assumptions were found in the scalar paths used by riscv64. |
| Embedded usability | Good — supports `FIXED_POINT` integer mode, uses stack allocation/`VAR_ARRAYS`, has no external runtime dependencies, and builds with multiple toolchains. |
| Overall portability | **LOW concern** — Opus is a mature, highly portable codebase; a riscv64 build succeeds out-of-the-box via the generic C path. The gaps are performance (no RVV backend) and explicit riscv platform recognition, not correctness. |

## 3. Build & Test Results

| Platform / Build | Recipe | Build result | Tests run | Passed | Failed | Skipped | Notes |
|---|---|---|---|---|---|---|---|
| x86_64 `1_1_1` (reference baseline) | 1 (RelWithDebInfo, tests) | PASS | 5 | 5 | 0 | 0 (5 sub-tests skipped) | native |
| x86_64 `1_1_2` (reference optimized) | 2 (RelWithDebInfo + `-O3 -march=native -g`) | PASS | 5 | 5 | 0 | 0 (5 sub-tests skipped) | native; net flags were actually `-O2` (see Section 5) |
| riscv64 `1_2_3` (target) | 3 (`-O3 -march=rv64gc -g`, cross, dynamic) | PASS | 5 | 5 | 0 | 0 (5 sub-tests skipped) | run under `qemu-riscv64-static -L /usr/riscv64-linux-gnu` |
| x86_64 `1_1_4` (reference optimized v2) | 4 (Release + FLOAT_APPROX + LTO) | PASS | 5 | 5 | 0 | 0 (5 sub-tests skipped) | native |
| riscv64 `1_2_5` (target optimized v2) | 5 (Release + FLOAT_APPROX + LTO + `-march=rv64gc_zba_zbb` + static) | PASS | 5 | 5 | 0 | 0 (5 sub-tests skipped) | run under `qemu-riscv64-static`; static |

**Build/test failures detail:** No builds failed and no tests failed on either platform. In `test_opus_api`, 5 `malloc()`-failure sub-tests are reported `SKIPPED` with "(Test only supported with GLIBC and without valgrind)" — identical on x86 native and riscv under emulation, so this is a test-harness environment limitation, not a portability regression.

**Target binary verification:** `readelf -h` confirmed the `1_2_3` and `1_2_5` artifacts are ELF64 RISC-V (Machine: RISC-V); `1_2_5` additionally advertises `zba1p0`/`zbb1p0` in `Tag_RISCV_arch` and has no dynamic section (statically linked). x86 artifacts are ELF64 x86-64.

## 4. Performance Comparison

### Experimental conditions

| Condition | Reference x86_64 (`1_1_2`, `1_1_4`) | Target riscv64 (`1_2_3`, `1_2_5`) |
|---|---|---|
| Host CPU | 13th Gen Intel Core i7-13620H (hybrid P/E), 16 logical CPUs | same host; guest emulated RV64GC |
| Cores pinned | `taskset -c 0` (P-core) | `taskset -c 0` |
| Priority | `nice -n -20` **failed (Permission denied)** in container → effective nice 0 | same |
| Governor / frequency | `powersave`; observed scaling 0.40–4.86 GHz, cpuinfo_max 4.70 GHz | same host |
| Warmup runs | 1 per workload | 1 per workload |
| Measurement runs | LEGACY: encode n=4 / decode n=3; CONTROLLED: encode n=3 + decode n=3 (fixed `SEED=12345`) | LEGACY: n=3; CONTROLLED: n=3 (fixed `SEED=12345`) |
| Workload | CMake `test_opus_encode` (primary), `test_opus_decode` | same, under `qemu-riscv64-static -L /usr/riscv64-linux-gnu` |
| Measurement tools | `amixis profile` (real_time), `/usr/bin/time -v`, `perf stat -ddd --repeat 3`, `perf record`/`perf report` | `/usr/bin/time -v`; host `perf stat` around the emulator (NOT guest-representative) |

Methodology note: the Opus test harness seeds its RNG from `time(NULL)^pid` unless `SEED` is set. The first (legacy) run did not fix `SEED`, so its runs executed different workloads and showed 15–20% spread. The controlled fixed-seed (`SEED=12345`) measurements below are the valid pre/post comparison.

### Key metrics (workload `test_opus_encode`)

| Metric | Reference x86_64 (`1_1_2`) | Target riscv64 (`1_2_3`, QEMU) | Label |
|---|---|---|---|
| Elapsed time (controlled mean) | 19.280 s ± 0.026 (n=3) | 287.220 s ± 2.279 (n=3) | x86: MEASURED NATIVE; riscv: EMULATION-INCLUSIVE WALL-CLOCK |
| Elapsed time, decode (controlled mean) | 7.567 s ± 0.006 (n=3) | 94.683 s ± 0.175 (n=3) | x86: MEASURED NATIVE; riscv: EMULATION-INCLUSIVE WALL-CLOCK |
| IPC | 2.97 | NOT AVAILABLE (guest) | x86 MEASURED NATIVE; riscv guest counters unavailable (host QEMU = 4.83, not guest-representative) |
| L1-dcache miss rate | 0.29 % | NOT AVAILABLE (guest) | same caveat |
| LLC miss rate | 10.96 % | NOT AVAILABLE (guest) | same caveat |
| Branch misprediction rate | 1.72 % | NOT AVAILABLE (guest) | same caveat |
| Frontend Bound | 11.5 % | NOT AVAILABLE (guest) | x86 measured (TopdownL1) |
| Backend Bound | 27.9 % | NOT AVAILABLE (guest) | x86 measured |
| Retiring | 53.8 % | NOT AVAILABLE (guest) | x86 measured |
| Instructions (encode) | 265,248,685,076 | NOT AVAILABLE (guest) | x86 measured |
| Executable size, stripped (`test_opus_encode`) | 506,072 B | 407,640 B | MEASURED |

**QEMU / emulation caveat:** all target measurements were produced under QEMU user-mode emulation. Timing includes dynamic-binary-translation overhead and does not reflect native riscv64 hardware. `perf` around `qemu-riscv64-static` measures the host emulator executing translated x86_64 code, not the RISC-V guest, so all target hardware-counter values (IPC, cache, branch, top-down) are reported as NOT AVAILABLE rather than estimated.

### Hotspots

Reference x86_64 `1_1_2` (`perf report`, `cycles`, self %):

| % | Function |
|---:|---|
| 10.39 | `silk_NSQ_del_dec_avx2` |
| 8.35 | `silk_NSQ_del_dec_c` |
| 6.12 | `celt_encode_with_ec` |
| 4.58 | `opus_fft_impl` |
| 3.13 | `deemphasis.isra.0` |
| 2.63 | `quant_partition` |
| 1.84 | `clt_mdct_backward_c` |
| 1.76 | `opus_encode_frame_native.isra.0` |
| 1.67 | `ec_enc_icdf` |
| 1.65 | `silk_NLSF_del_dec_quant` |
| 1.64 | `op_pvq_search_sse2` |
| 1.64 | `silk_warped_autocorrelation_FLP` |
| 1.57 | `silk_PLC` |
| 1.53 | `silk_CNG` |

Reference x86_64 `1_1_4` (optimized, `perf report`, `cycles`, self %):

| % | Function |
|---:|---|
| 10.57 | `silk_NSQ_del_dec_avx2` |
| 9.51 | `quant_partition` |
| 8.21 | `celt_encode_with_ec` |
| 5.81 | `opus_fft_impl` |
| 5.27 | `silk_NSQ_del_dec_c` |
| 4.23 | `deemphasis.isra.0` |
| 3.61 | `compute_theta` |
| 3.00 | `run_prefilter.constprop.0` |
| 2.96 | `celt_decode_with_ec_dred` |
| 2.70 | `opus_encode_native` |
| 2.05 | `opus_decode_frame.lto_priv.0` |
| 1.77 | `silk_find_pred_coefs_FLP` |
| 1.76 | `clt_mdct_backward_c.isra.0` |
| 1.70 | `silk_CNG` |

Target riscv64 hotspots: **NOT AVAILABLE** — under QEMU user-mode the sampled IPs resolve only to the host emulator / `[unknown]`; no guest hot-function ranking can be produced.

### Bottleneck summary

1. The 12.5–15.6× target wall-clock ratio is dominated by QEMU translation, not the RISC-V ISA: the host emulator executes ~6.16×10^12 instructions / ~1.27×10^12 cycles for the encode loop versus ~0.209×10^12 / ~0.071×10^12 natively. Do not interpret this ratio as native riscv64 speed.
2. Native x86 is a retiring-bound DSP workload (Retiring 53.8%, Backend 27.9%, Frontend 11.5%, IPC 2.97, branch-miss 1.72%) with LLC pressure (10.96%). Top cost is the SILK noise-shaping quantizer (`silk_NSQ_del_dec_avx2` + `_c`) plus CELT encode/MDCT.
3. Structural target gap: libopus has no RISC-V SIMD/RVV backend and no riscv runtime dispatch, so all DSP kernels fall back to scalar C. x86 uses 4,435 SSE/AVX2/FMA instructions; riscv has 0 RVV instructions.

### Vectorization intrinsics (source / binary)

| Architecture | Vector instructions in binary | ISAs found |
|---|---|---|
| x86_64 (`1_1_2`) | 4,435 total | SSE, SSE2, SSE4.1, AVX, AVX2, FMA (no AVX-512) |
| x86_64 (`1_1_4`) | 20,768 total | SSE/AVX/AVX2/FMA (increased via `-O3`+LTO inlining/unrolling) |
| riscv64 (`1_2_3`) | 0 RVV | none (rv64gc) |
| riscv64 (`1_2_5`) | 0 RVV; 1,571 Zba + 492 Zbb | Zba/Zbb bit manipulation only |

### Cross-table Comparison

The following cross-tables are the tool-generated artifacts (`amixis compare`) and are reproduced verbatim from `cross-tables/CT-*.md`.

### Cross-table: CT-1__1__1..test__opus__encode-1__1__2..test__opus__encode.md
# Cross-tables for 1\_\_1\_\_1..test\_\_opus\_\_encode and 1\_\_1\_\_2..test\_\_opus\_\_encode

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                        | 1_1_1 % | 1_1_2 % | Delta % |
|:------------------------------|-------:|-------:|-------:|
| silk\_NSQ\_del\_dec\_c        |    7.72 |   16.45 |   +8.73 |
| decode\_pulses                |    8.60 |    3.65 |   -4.96 |
| ec\_enc\_icdf                 |    2.54 |    5.98 |   +3.44 |
| tonality\_analysis.isra.0     |    4.55 |    1.45 |   -3.09 |
| ec\_dec\_normalize            |    2.34 |    4.34 |   +2.00 |
| quant\_partition              |    4.47 |    2.51 |   -1.97 |
| encode\_pulses                |    3.31 |    1.52 |   -1.80 |
| celt\_encode\_with\_ec        |    4.14 |    2.42 |   -1.72 |
| ec\_dec\_icdf                 |    1.77 |    3.48 |   +1.71 |
| spreading\_decision           |    2.76 |    1.11 |   -1.65 |
| silk\_noise\_shape\_quantizer |    0.86 |    2.45 |   +1.59 |
| silk\_encode\_pulses          |    0.74 |    2.11 |   +1.36 |
| silk\_NLSF\_del\_dec\_quant   |    2.54 |    3.88 |   +1.34 |
| silk\_decode\_core            |    0.73 |    1.78 |   +1.05 |
| silk\_encode\_signs           |    0.98 |    1.88 |   +0.90 |
| ec\_enc\_carry\_out.part.0    |    1.46 |    2.28 |   +0.82 |
| alg\_quant                    |    1.16 |    0.41 |   -0.75 |
| celt\_inner\_prod\_sse        |    1.83 |    1.12 |   -0.72 |
| quant\_band                   |    1.32 |    0.66 |   -0.66 |
| opus\_fft\_impl               |    1.04 |    0.48 |   -0.55 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| celt\_encode\_with\_ec             |    8.29 |    1.35 |   -6.95 |
| \[unknown\]                        |   17.55 |   23.03 |   +5.47 |
| generate\_music                    |    5.23 |   10.41 |   +5.18 |
| tonality\_get\_info                |    3.93 |    0.07 |   -3.86 |
| opus\_encode                       |   18.21 |   15.27 |   -2.94 |
| opus\_encode\_frame\_native.isra.0 |    3.30 |    0.99 |   -2.31 |
| silk\_NSQ\_wrapper\_FLP            |    0.02 |    2.25 |   +2.24 |
| silk\_noise\_shape\_analysis\_FLP  |    0.02 |    1.65 |   +1.63 |
| celt\_float2int16\_c               |   11.54 |   12.95 |   +1.40 |
| celt\_decode\_with\_ec\_dred       |    2.68 |    1.28 |   -1.40 |
| silk\_NSQ\_sse4\_1                 |    0.05 |    1.32 |   +1.26 |
| opus\_encode\_native               |    6.08 |    7.26 |   +1.18 |
| silk\_NSQ\_del\_dec\_avx2          |    0.45 |    1.38 |   +0.93 |
| decode\_pulses                     |    0.92 |    0.05 |   -0.87 |
| quant\_fine\_energy                |    0.86 |    0.03 |   -0.83 |
| \[unknown\] (/                     |    5.00 |    4.20 |   -0.80 |
| silk\_encode\_frame\_FLP           |    0.84 |    0.08 |   -0.77 |
| quant\_partition                   |    0.93 |    0.26 |   -0.67 |
| silk\_NSQ\_del\_dec\_c             |    0.10 |    0.68 |   +0.58 |
| opus\_fft\_impl                    |    0.38 |    0.91 |   +0.53 |

## EVENT: CYCLES

| Symbol                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| silk\_NSQ\_del\_dec\_c             |    3.72 |    8.35 |   +4.63 |
| celt\_encode\_with\_ec             |    9.30 |    6.12 |   -3.19 |
| opus\_fft\_impl                    |    7.31 |    4.58 |   -2.72 |
| silk\_NSQ\_del\_dec\_avx2          |    7.89 |   10.39 |   +2.50 |
| ec\_enc\_icdf                      |    0.54 |    1.67 |   +1.13 |
| clt\_mdct\_backward\_c             |    2.68 |    1.84 |   -0.85 |
| decode\_pulses                     |    1.65 |    0.87 |   -0.78 |
| clt\_mdct\_forward\_c              |    1.67 |    0.89 |   -0.78 |
| silk\_decode\_core                 |    0.59 |    1.33 |   +0.74 |
| deemphasis.isra.0                  |    3.86 |    3.13 |   -0.73 |
| silk\_NLSF\_del\_dec\_quant        |    0.93 |    1.65 |   +0.71 |
| silk\_noise\_shape\_quantizer      |    0.32 |    0.98 |   +0.66 |
| tonality\_analysis.isra.0          |    1.11 |    0.53 |   -0.58 |
| op\_pvq\_search\_sse2              |    2.22 |    1.64 |   -0.58 |
| ec\_dec\_icdf                      |    0.48 |    1.03 |   +0.55 |
| \[unknown\] (/                     |    2.61 |    2.12 |   -0.49 |
| quant\_partition                   |    3.11 |    2.63 |   -0.48 |
| \[unknown\]                        |    0.07 |    0.54 |   +0.47 |
| pitch\_downsample                  |    1.12 |    0.65 |   -0.47 |
| opus\_encode\_frame\_native.isra.0 |    1.32 |    1.76 |   +0.44 |

### Cross-table: CT-1__1__2..test__opus__encode-1__1__4..test__opus__encode.md
# Cross-tables for 1\_\_1\_\_2..test\_\_opus\_\_encode and 1\_\_1\_\_4..test\_\_opus\_\_encode

## EVENT: CYCLES

| Symbol                             | 1_1_2 % | 1_1_4 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| quant\_partition                   |    2.63 |    9.51 |   +6.88 |
| silk\_NSQ\_del\_dec\_c             |    8.35 |    5.27 |   -3.08 |
| run\_prefilter.constprop.0         |    0.00 |    3.00 |   +3.00 |
| compute\_theta                     |    1.21 |    3.61 |   +2.40 |
| celt\_encode\_with\_ec             |    6.12 |    8.21 |   +2.09 |
| opus\_decode\_frame.lto\_priv.0    |    0.00 |    2.05 |   +2.05 |
| clt\_mdct\_backward\_c             |    1.84 |    0.00 |   -1.84 |
| opus\_encode\_native               |    0.91 |    2.70 |   +1.78 |
| clt\_mdct\_backward\_c.isra.0      |    0.00 |    1.76 |   +1.76 |
| silk\_find\_pred\_coefs\_FLP       |    0.01 |    1.77 |   +1.76 |
| celt\_decode\_with\_ec\_dred       |    1.21 |    2.96 |   +1.76 |
| silk\_NLSF\_del\_dec\_quant        |    1.65 |    0.00 |   -1.65 |
| op\_pvq\_search\_sse2              |    1.64 |    0.00 |   -1.64 |
| silk\_warped\_autocorrelation\_FLP |    1.64 |    0.00 |   -1.64 |
| ec\_enc\_icdf                      |    1.67 |    0.03 |   -1.64 |
| compute\_mdcts.isra.0              |    0.00 |    1.57 |   +1.57 |
| silk\_PLC                          |    1.57 |    0.00 |   -1.57 |
| fuzz\_encoder\_settings            |    0.00 |    1.43 |   +1.43 |
| silk\_decode\_core                 |    1.33 |    0.00 |   -1.33 |
| generate\_music                    |    1.31 |    0.00 |   -1.31 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol                            | 1_1_2 % | 1_1_4 % | Delta % |
|:----------------------------------|-------:|-------:|-------:|
| opus\_encode                      |   15.27 |    0.00 |  -15.27 |
| opus\_encode.constprop.0          |    0.00 |   13.64 |  +13.64 |
| celt\_float2int16\_c              |   12.95 |    0.00 |  -12.95 |
| fuzz\_encoder\_settings           |    0.02 |   12.66 |  +12.64 |
| opus\_decode.constprop.0          |    0.00 |   11.74 |  +11.74 |
| generate\_music                   |   10.41 |    0.00 |  -10.41 |
| \[unknown\]                       |   23.03 |   17.89 |   -5.14 |
| opus\_decode\_frame.lto\_priv.0   |    0.00 |    2.59 |   +2.59 |
| opus\_encoder\_ctl                |    0.14 |    2.68 |   +2.54 |
| celt\_encode\_with\_ec            |    1.35 |    3.37 |   +2.02 |
| silk\_NSQ\_wrapper\_FLP           |    2.25 |    0.25 |   -2.00 |
| silk\_decode\_core.isra.0         |    0.00 |    1.69 |   +1.69 |
| silk\_burg\_modified\_FLP         |    0.15 |    1.79 |   +1.64 |
| run\_test1.isra.0                 |    0.00 |    1.54 |   +1.54 |
| silk\_NSQ\_sse4\_1                |    1.32 |    0.03 |   -1.29 |
| silk\_noise\_shape\_analysis\_FLP |    1.65 |    0.44 |   -1.21 |
| silk\_NSQ\_del\_dec\_avx2         |    1.38 |    0.17 |   -1.21 |
| opus\_encode\_native              |    7.26 |    8.16 |   +0.90 |
| quant\_partition                  |    0.26 |    1.12 |   +0.86 |
| run\_test1                        |    0.85 |    0.00 |   -0.85 |

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                        | 1_1_2 % | 1_1_4 % | Delta % |
|:------------------------------|-------:|-------:|-------:|
| quant\_partition              |    2.51 |   24.06 |  +21.56 |
| silk\_NSQ\_del\_dec\_c        |   16.45 |    9.67 |   -6.78 |
| silk\_find\_pred\_coefs\_FLP  |    0.02 |    6.06 |   +6.04 |
| ec\_enc\_icdf                 |    5.98 |    0.05 |   -5.92 |
| silk\_decode\_pulses          |    0.46 |    4.89 |   +4.43 |
| ec\_dec\_normalize            |    4.34 |    0.00 |   -4.34 |
| ec\_enc\_icdf.constprop.3     |    0.00 |    4.24 |   +4.24 |
| compute\_theta                |    1.58 |    5.75 |   +4.18 |
| silk\_NLSF\_del\_dec\_quant   |    3.88 |    0.00 |   -3.88 |
| decode\_pulses                |    3.65 |    0.00 |   -3.65 |
| ec\_dec\_icdf                 |    3.48 |    0.00 |   -3.48 |
| celt\_decode\_with\_ec\_dred  |    0.71 |    3.42 |   +2.71 |
| isqrt32                       |    2.60 |    0.00 |   -2.60 |
| silk\_noise\_shape\_quantizer |    2.45 |    0.00 |   -2.45 |
| celt\_encode\_with\_ec        |    2.42 |    4.83 |   +2.41 |
| ec\_enc\_carry\_out.part.0    |    2.28 |    0.00 |   -2.28 |
| opus\_encode\_native          |    0.18 |    2.23 |   +2.05 |
| run\_prefilter.constprop.0    |    0.00 |    1.95 |   +1.95 |
| silk\_encode\_signs           |    1.88 |    0.00 |   -1.88 |
| find\_best\_pitch             |    1.86 |    0.00 |   -1.86 |

### Cross-table: CT-1__1__2..test__opus__encode-1__2__3..test__opus__encode.md
# Cross-tables for 1\_\_1\_\_2..test\_\_opus\_\_encode and 1\_\_2\_\_3..test\_\_opus\_\_encode

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                        | 1_1_2 % | 1_2_3 % | Delta % |
|:------------------------------|-------:|-------:|-------:|
| \[unknown\]                   |    0.23 |  100.00 |  +99.77 |
| silk\_NSQ\_del\_dec\_c        |   16.45 |    0.00 |  -16.45 |
| ec\_enc\_icdf                 |    5.98 |    0.00 |   -5.98 |
| ec\_dec\_normalize            |    4.34 |    0.00 |   -4.34 |
| silk\_NSQ\_del\_dec\_avx2     |    4.08 |    0.00 |   -4.08 |
| silk\_NLSF\_del\_dec\_quant   |    3.88 |    0.00 |   -3.88 |
| decode\_pulses                |    3.65 |    0.00 |   -3.65 |
| ec\_dec\_icdf                 |    3.48 |    0.00 |   -3.48 |
| isqrt32                       |    2.60 |    0.00 |   -2.60 |
| quant\_partition              |    2.51 |    0.00 |   -2.51 |
| silk\_noise\_shape\_quantizer |    2.45 |    0.00 |   -2.45 |
| \[unknown\] (/                |    2.43 |    0.00 |   -2.43 |
| celt\_encode\_with\_ec        |    2.42 |    0.00 |   -2.42 |
| ec\_enc\_carry\_out.part.0    |    2.28 |    0.00 |   -2.28 |
| silk\_encode\_pulses          |    2.11 |    0.00 |   -2.11 |
| silk\_encode\_signs           |    1.88 |    0.00 |   -1.88 |
| find\_best\_pitch             |    1.86 |    0.00 |   -1.86 |
| silk\_decode\_core            |    1.78 |    0.00 |   -1.78 |
| compute\_theta                |    1.58 |    0.00 |   -1.58 |
| encode\_pulses                |    1.52 |    0.00 |   -1.52 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol                             | 1_1_2 % | 1_2_3 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| \[unknown\]                        |   23.03 |  100.00 |  +76.97 |
| opus\_encode                       |   15.27 |    0.00 |  -15.27 |
| celt\_float2int16\_c               |   12.95 |    0.00 |  -12.95 |
| generate\_music                    |   10.41 |    0.00 |  -10.41 |
| opus\_encode\_native               |    7.26 |    0.00 |   -7.26 |
| \[unknown\] (/                     |    4.20 |    0.00 |   -4.20 |
| silk\_NSQ\_wrapper\_FLP            |    2.25 |    0.00 |   -2.25 |
| silk\_noise\_shape\_analysis\_FLP  |    1.65 |    0.00 |   -1.65 |
| silk\_NSQ\_del\_dec\_avx2          |    1.38 |    0.00 |   -1.38 |
| celt\_encode\_with\_ec             |    1.35 |    0.00 |   -1.35 |
| silk\_NSQ\_sse4\_1                 |    1.32 |    0.00 |   -1.32 |
| celt\_decode\_with\_ec\_dred       |    1.28 |    0.00 |   -1.28 |
| opus\_encode\_frame\_native.isra.0 |    0.99 |    0.00 |   -0.99 |
| opus\_fft\_impl                    |    0.91 |    0.00 |   -0.91 |
| run\_test1                         |    0.85 |    0.00 |   -0.85 |
| silk\_NSQ\_del\_dec\_c             |    0.68 |    0.00 |   -0.68 |
| downmix\_and\_resample             |    0.50 |    0.00 |   -0.50 |
| ec\_dec\_icdf                      |    0.48 |    0.00 |   -0.48 |
| silk\_resampler\_down2             |    0.47 |    0.00 |   -0.47 |
| opus\_custom\_encoder\_ctl         |    0.43 |    0.00 |   -0.43 |

## EVENT: CYCLES

| Symbol                             | 1_1_2 % | 1_2_3 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| \[unknown\]                        |    0.54 |  100.00 |  +99.46 |
| silk\_NSQ\_del\_dec\_avx2          |   10.39 |    0.00 |  -10.39 |
| silk\_NSQ\_del\_dec\_c             |    8.35 |    0.00 |   -8.35 |
| celt\_encode\_with\_ec             |    6.12 |    0.00 |   -6.12 |
| opus\_fft\_impl                    |    4.58 |    0.00 |   -4.58 |
| deemphasis.isra.0                  |    3.13 |    0.00 |   -3.13 |
| quant\_partition                   |    2.63 |    0.00 |   -2.63 |
| \[unknown\] (/                     |    2.12 |    0.00 |   -2.12 |
| clt\_mdct\_backward\_c             |    1.84 |    0.00 |   -1.84 |
| opus\_encode\_frame\_native.isra.0 |    1.76 |    0.00 |   -1.76 |
| ec\_enc\_icdf                      |    1.67 |    0.00 |   -1.67 |
| silk\_NLSF\_del\_dec\_quant        |    1.65 |    0.00 |   -1.65 |
| op\_pvq\_search\_sse2              |    1.64 |    0.00 |   -1.64 |
| silk\_warped\_autocorrelation\_FLP |    1.64 |    0.00 |   -1.64 |
| silk\_PLC                          |    1.57 |    0.00 |   -1.57 |
| silk\_CNG                          |    1.53 |    0.00 |   -1.53 |
| clt\_compute\_allocation           |    1.51 |    0.00 |   -1.51 |
| silk\_resampler\_private\_IIR\_FIR |    1.47 |    0.00 |   -1.47 |
| silk\_decode\_core                 |    1.33 |    0.00 |   -1.33 |
| generate\_music                    |    1.31 |    0.00 |   -1.31 |

### Cross-table: CT-1__2__3..test__opus__encode-1__2__5..test__opus__encode.md
# Cross-tables for 1\_\_2\_\_3..test\_\_opus\_\_encode and 1\_\_2\_\_5..test\_\_opus\_\_encode

## EVENT: CYCLES

| Symbol         | 1_2_3 % | 1_2_5 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |  100.00 |   22.47 |  -77.53 |
| \[unknown\] (/ |    0.00 |   77.53 |  +77.53 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol         | 1_2_3 % | 1_2_5 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |    0.00 |   39.21 |  +39.21 |
| \[unknown\]    |  100.00 |   60.79 |  -39.21 |

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol         | 1_2_3 % | 1_2_5 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |  100.00 |   65.68 |  -34.32 |
| \[unknown\] (/ |    0.00 |   34.32 |  +34.32 |



## 5. Optimization Results

### Executable size analysis (`test_opus_encode`)

| Build | Unstripped (B) | Stripped (B) | Notes |
|---|---:|---:|---|
| x86 `1_1_1` | 2,087,912 | 506,072 | debug info ≈ 75% of file |
| x86 `1_1_2` | 2,083,704 | 506,072 | debug info ≈ 75% of file |
| riscv `1_2_3` | 1,875,496 | 407,640 | smaller `.text` due to absent SIMD kernels |
| x86 `1_1_4` | — | 743,640 | larger from `-O3`+LTO code expansion |
| riscv `1_2_5` | — | 982,928 | static libc included |

### Optimization attempts (before → after)

| Optimization | Before | After | Delta | Causal analysis |
|---|---|---|---|---|
| Effective `-O2` → true `-O3` (`RelWithDebInfo`→`Release`) | `-O3 … -O2 -g -DNDEBUG` (net `-O2`) | `-O3 -DNDEBUG` | dynamic instructions −11.2% (encode) / −15.1% (decode) on x86 | `RelWithDebInfo` appended `-O2` after the recipe `-O3`, so GCC used `-O2`; `Release` appends `-O3` last |
| `OPUS_FLOAT_APPROX=ON` | OFF (double-precision `libm` `log`/`exp`/reciprocal calls) | ON (inline polynomial/table approximations) | fewer instructions; IPC 2.97→2.78; branch-miss 1.72%→2.24% | removes call/PLT overhead but adds longer FP dependency chains/conditionals |
| LTO (`-flto -fuse-linker-plugin`) | off | on | cross-TU inlining (`.lto_priv`/`.constprop`/`.isra` symbols); vector instruction count 4,435→20,768 on x86 | enables cross-translation-unit optimization, valuable for the scalar riscv build |
| riscv `-march=rv64gc_zba_zbb` | rv64gc | rv64gc + Zba/Zbb | 2,063 bitmanip instructions (1,571 Zba + 492 Zbb) | accelerates address/bit arithmetic in entropy/SILK code |
| riscv `-static` | dynamic (loader/PLT emulation) | static | part of the target wall-clock gain is a QEMU/loader artifact | removes host-side dynamic-loader emulation under qemu-user |
| `SEED` fix in measurement | RNG seeded from `time(NULL)^pid` | fixed `SEED=12345`, interleaved A/B | honest x86 −5.3%/−8.3% and riscv −8.4%/−10.2% (vs inflated legacy −16%/−17%) | baseline and optimized previously ran different random workloads |

### Pre/post controlled wall-clock

| Platform / workload | Baseline mean ± sd (s) | Optimized mean ± sd (s) | Optimized / Baseline |
|---|---:|---:|---:|
| x86 encode (`1_1_2`→`1_1_4`) | 19.280 ± 0.026 | 18.250 ± 0.020 | 0.9466 |
| x86 decode (`1_1_2`→`1_1_4`) | 7.567 ± 0.006 | 6.940 ± 0.000 | 0.9172 |
| riscv encode (`1_2_3`→`1_2_5`) | 287.220 ± 2.279 | 263.103 ± 0.605 | 0.9160 |
| riscv decode (`1_2_3`→`1_2_5`) | 94.683 ± 0.175 | 85.040 ± 0.105 | 0.8982 |

### Recommended optimizations (not yet applied)

| Priority | Optimization | Expected Gain (estimate) | Effort | Notes |
|---:|---|---|:--:|---|
| 1 | Keep `-O3` effective (`Release` + `-g` or override `CMAKE_C_FLAGS_RELWITHDEBINFO`) | ~1–5% | Low | Applied in this pipeline; prevents silent `-O2` |
| 2 | `OPUS_FLOAT_APPROX=ON` (+ optional `OPUS_FAST_MATH=ON`) | ~1–5% | Low | Applied; slight numeric change |
| 3 | LTO (`-flto -fuse-linker-plugin`, cross `gcc-ar`/`gcc-ranlib`) | ~1–5% | Low–Med | Applied |
| 4 | riscv `-march=rv64gc_zba_zbb` | ~0–3% | Low | Applied |
| 5 | riscv `-static` | target-only startup effect | Low | Applied; measurable only as QEMU artifact |
| 6 | Strip debug info | ~75% size reduction, ~0% runtime | Low | Debug info dominates binary size |
| 7 | Manual RVV backend (port x86/SILK kernels to `__riscv_v*` + riscv RTCD) | potentially 1.3–2× | High (weeks) | Only source of fundamental speedup; `-march=rv64gcv` alone does not auto-vectorize in GCC 13 |
| 8 | Port PR #476 explicit riscv recognition | ~0% directly | Low | Groundwork for future RTCD/RVV; `OPUS_FAST_INT64` already correct via `__LP64__` |

## Improvement of 1_1_2 compared to 1_1_4

| Measured | Baseline value | Optimized value | Improvement % |
|---|---|---|---|
| test_opus_encode | 19.28 | 18.25 | 94.66 |
| test_opus_decode | 7.567 | 6.94 | 91.71 |

## Improvement of 1_2_3 compared to 1_2_5

| Measured | Baseline value | Optimized value | Improvement % |
|---|---|---|---|
| test_opus_encode | 287.22 | 263.103 | 91.6 |
| test_opus_decode | 94.683 | 85.04 | 89.82 |

### Recorded profile data (`opus.json`)

```json
{
  "1_1_4": {
    "test_opus_api": {
      "build_name": "1_1_4",
      "executable": "test_opus_api",
      "executable_run_success": true,
      "real_time": "1.69",
      "user_time": "1.69",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__4..test__opus__api.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_decode": {
      "build_name": "1_1_4",
      "executable": "test_opus_decode",
      "executable_run_success": true,
      "real_time": "7.23",
      "user_time": "7.23",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__4..test__opus__decode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_encode": {
      "build_name": "1_1_4",
      "executable": "test_opus_encode",
      "executable_run_success": true,
      "real_time": "16.58",
      "user_time": "16.56",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__4..test__opus__encode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_1": {
    "test_opus_api": {
      "build_name": "1_1_1",
      "executable": "test_opus_api",
      "executable_run_success": true,
      "real_time": "2.06",
      "user_time": "2.06",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__1..test__opus__api.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_decode": {
      "build_name": "1_1_1",
      "executable": "test_opus_decode",
      "executable_run_success": true,
      "real_time": "8.11",
      "user_time": "8.10",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__1..test__opus__decode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_encode": {
      "build_name": "1_1_1",
      "executable": "test_opus_encode",
      "executable_run_success": true,
      "real_time": "18.06",
      "user_time": "18.04",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__1..test__opus__encode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_2": {
    "test_opus_api": {
      "build_name": "1_1_2",
      "executable": "test_opus_api",
      "executable_run_success": true,
      "real_time": "2.07",
      "user_time": "2.05",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__2..test__opus__api.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_decode": {
      "build_name": "1_1_2",
      "executable": "test_opus_decode",
      "executable_run_success": true,
      "real_time": "7.75",
      "user_time": "7.74",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__2..test__opus__decode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test_opus_encode": {
      "build_name": "1_1_2",
      "executable": "test_opus_encode",
      "executable_run_success": true,
      "real_time": "18.30",
      "user_time": "18.28",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "Error: switch `x' requires a value"
        },
        {
          "counter_value": "Usage: perf stat [<options>] [<command>]"
        },
        {
          "counter_value": "-x, --field-separator <separator>"
        },
        {
          "counter_value": "print counts with custom separator"
        }
      ],
      "perf_record_name": "1__1__2..test__opus__encode.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  }
}
```

## 6. Notes About Exploration Process

- **Provided config vs target platform.** The provided `input.yml` defined only an x86 platform plus two flag recipes, with both builds on x86. Because the container has a full riscv64 cross-toolchain and `qemu-riscv64-static`, the configurator extended it with a riscv64 platform (id 2) and a cross recipe to enable a real x86_64→riscv64 migration comparison. The canonical config is `/work/opus-workspace/input.yml`; an x86-only derivative `/work/opus-workspace/input_x86.yml` was needed to work around a tooling limitation (below). Both `amixis validate` cleanly.
- **Amphimixis v0.2.0 rejects the mixed-arch config at parse time.** `parse_config` rejects a local run-machine whose arch differs from the host (`Invalid local machine arch: riscv, your machine is x86_64`), and it validates every build before applying `--build-name`. Consequently `amixis build`/`amixis profile` could not operate on the riscv builds at all, and even the x86 builds had to be run through `input_x86.yml`. Workaround: x86 via `input_x86.yml`; riscv built/run manually with the cross toolchain + `qemu-riscv64-static`. The installed tool has no emulator field, so qemu-user is represented by prefixing executables in the config and invoked explicitly.
- **`amixis` built-in `perf stat` is broken on this host** (shell-quoting bug: it emits `perf stat -ddd -x| taskset …`), so the `perf_stat` entries recorded in `opus.json` contain usage/error text and are invalid. All hardware counters were re-collected manually with `perf stat -ddd --repeat 3` and are reported only in structured form (no raw dumps).
- **QEMU/emulation caveats.** All riscv64 target execution used QEMU user-mode emulation (`qemu-riscv64-static -L /usr/riscv64-linux-gnu`; `binfmt_misc` is not registered so qemu is invoked explicitly). Target timings include emulation overhead and are not native-hardware performance; target hardware counters and guest hotspot profiles are NOT AVAILABLE because `perf` samples the host emulator, not the guest.
- **RNG seed confound.** The Opus tests seed their RNG from `time(NULL)^pid` unless `SEED` is set, so the initial baseline runs executed different workloads. The pre/post optimization comparison was redone with fixed `SEED=12345`, interleaved A/B, n=3; only the controlled values are used for improvement percentages.
- **`opus_demo` / `opus_compare` are not built** (`OPUS_BUILD_PROGRAMS=OFF`); test executables (`test_opus_encode` in particular) were used as profiling workloads.
- **Other tooling issues:** `amphimixis-analyze-vectorization` exits 1 on riscv ELF (host `objdump` cannot disassemble RISC-V) → `riscv64-linux-gnu-objdump` fallback; host `strip` cannot process riscv ELF → `riscv64-linux-gnu-strip`; `perf archive` is not a command in perf 6.8.12; `nice -n -20` is denied in the container (no `CAP_SYS_NICE`).
- **riscv improvement interpretation.** Part of the riscv64 `1_2_3`→`1_2_5` gain is attributable to static linking / QEMU loader behavior rather than pure native compute, and cannot be separated because guest hardware counters are unavailable under QEMU.
- **No fabricated data:** unmeasured target hardware metrics and hotspots are marked NOT AVAILABLE; all host-QEMU counters are labelled not guest-representative; all target timings are labelled emulation-inclusive.

## 7. Migration Readiness Summary

| Check | Result |
|---|---|
| Builds on reference platform (x86_64) | PASS — `1_1_1`, `1_1_2`, `1_1_4` built successfully |
| Tests pass on reference platform | PASS — 5/5 CTest tests on every x86 build |
| Builds on target platform (riscv64) | PASS — `1_2_3` and `1_2_5` cross-built successfully (RISC-V ELF confirmed) |
| Tests pass on target platform | PASS — 5/5 CTest tests under `qemu-riscv64-static` |
| Zero external dependencies | PASS — no runtime dependencies beyond libc/libm; optional DNN model disabled |
| No hand-written intrinsics that block portability | PASS — x86/ARM intrinsics exist but are fully arch-guarded with generic C fallbacks; riscv uses the C path |
| Alignment safe | PASS (assessment) — explicit stack/FFT alignment macros and attributes; no unaligned assumptions in scalar paths |
| Exceptions handled | N/A — pure C codebase (no C++ exception machinery) |
| Auto-vectorization on target | NOT PRESENT — 0 RVV instructions; GCC 13 does not auto-vectorize opus for rv64gcv; no RISC-V SIMD backend upstream |

**Migration Verdict: READY (with performance caveats)**

libopus cross-compiles and passes its test suite on riscv64 out of the box via the generic C path. The principal caveats are performance-related: there is no RISC-V RVV/SIMD backend and no riscv runtime dispatch, so target DSP kernels run scalar, and the only measured target timings are emulation-inclusive. The applied optimization bundle (effective `-O3`, `FLOAT_APPROX`, LTO, Zba/Zbb, static) yields a real but modest improvement.

**Required Actions**
1. Do not treat the QEMU wall-clock ratios as native riscv64 performance; validate on real RISC-V hardware before making performance decisions.
2. If target performance matters, implement a RISC-V RVV backend (port `celt/x86/*`, `silk/x86/*`, `silk/float/x86/*` to `__riscv_v*` plus a riscv runtime-dispatch table) — the only path to a fundamental speedup.
3. Adopt the low-risk build-flag bundle permanently (`Release`/true `-O3`, `OPUS_FLOAT_APPROX=ON`, LTO, `-march=rv64gc_zba_zbb`); consider porting PR #476 for explicit riscv platform recognition as groundwork.
4. Fix the measurement methodology for any future benchmarking: pin the RNG with `SEED`, use interleaved A/B runs, and enable `CAP_SYS_NICE` or otherwise control frequency.
5. If a native riscv64 machine or SSH target becomes available, re-run profiling there to obtain real guest hardware counters and confirm the portability verdict.
