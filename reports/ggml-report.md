# ggml — Migration Readiness Report

**Project:** ggml (tensor library for machine learning / LLM inference kernels)
**Repository:** https://github.com/ggml-org/ggml.git
**Reference platform:** x86_64 (local container)
**Target platform:** riscv64 (cross-compiled on the local x86_64 host, executed under qemu-user emulation)
**Report date:** 2026-10-05
**Tooling:** Amphimixis (`amixis`) migration-readiness pipeline

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Resolved repository URL | https://github.com/ggml-org/ggml.git (`https://github.com/ggerganov/ggml` redirects to the same repo; `ggml-org` is canonical) |
| Latest commit | `353b63b439f27ab2cc19dac97ab1681ba6d2d084` (2026-09-24, "ggml : bump version to 0.25.3 (#1645)") |
| Total commits | 4731 |
| Latest tag | `v0.25.3` |
| Activity | Actively maintained; 15,443 stars, 1,857 forks, 368 open issues; repo `pushed_at` 2026-10-04 |
| Build systems | CMake (single top-level `CMakeLists.txt`); CTest for tests; GitHub Actions CI (`build-cpu.yml`, `build-self-hosted.yml`, `release.yml`) — no RISC-V CI job |
| Tests | 21 CTest test targets under `tests/` |
| External dependencies (core) | None mandatory beyond `Threads` (pthreads), `libm`, `libdl` — all native on riscv64. Optional/backend-gated: OpenMP, BLAS, Accelerate, CUDA, HIP, Vulkan, Metal, SYCL, OpenCL, OpenVINO, etc. |
| Distro packages | Debian `0.25.3` (sid/forky) + trixie-backports; Arch `extra/ggml 0.25.3-2`; Gentoo `sci-ml/ggml 0.25.3`; Ubuntu, FreeBSD, Homebrew, MacPorts, pkgsrc, vcpkg, Nix, spack. No OpenEmbedded/Yocto recipe found. |
| Forks with target-arch patches | None dedicated. RISC-V support is upstream-native (`src/ggml-cpu/arch/riscv/`, `spacemit/`, `__riscv_xtheadvector`). Relevant open upstream issues/PRs: #1535 (`__riscv_v_intrinsic` gate), #1571 (gate RVV code on `__riscv_v`), #1475 (Zv* hwprobe), #1388 (host x86 flags leak into cross builds); PR #197 (2023) was the original RISC-V enablement. |

## 2. Platform-Specific Code Analysis

### Architecture macros

| Macro | File(s) | What it guards | Category |
|---|---|---|---|
| `__x86_64__` / `_M_AMD64` | `src/ggml-cpu/ggml-cpu.c`, `src/ggml-cpu/arch/x86/cpu-feats.cpp` | x86-64 CPU feature init | x86 |
| `__SSE__` … `__AVX512BF16__` (`__AVX__`, `__AVX2__`, `__AVX512F__`, `__AVX512VNNI__`, `__F16C__`, `__FMA__`, `__BMI2__`) | `src/ggml-cpu/ggml-cpu-impl.h` | `#include <immintrin.h>` and x86 SIMD kernel selection | x86 |
| `_MSC_VER` (synthesizes `__FMA__`, `__F16C__`, `__SSE3__`, `__SSSE3__`) | `src/ggml-cpu/ggml-cpu-impl.h` 46–63 | MSVC macro fill-in (semantic note: not real feature macros on MSVC) | x86/MSVC |
| `__AVX*` / `__AVX512*` | `arch/x86/quants.c`, `repack.cpp`, `iqp.cpp`, `vec.cpp`, `vec.h` | Quantized dot/GEMM kernels | x86 |
| `__ARM_NEON` | `ggml-cpu-impl.h` 78, `simd-mappings.h` 9 | NEON kernels + 32-bit ARM compat shims | ARM |
| `__aarch64__` + `__ARM_FEATURE_DOTPROD` / `__ARM_FEATURE_MATMUL_INT8` | `arch/arm/repack.cpp` | aarch64 int8 kernels | ARM |
| `__ARM_FEATURE_SVE` | `ggml-cpu-impl.h` 74 | SVE kernels / `sys/prctl.h` | ARM |
| `__ARM_FEATURE_FP16_VECTOR_ARITHMETIC` | `llamafile/sgemm.cpp` | FP16 vector paths | ARM |
| `__riscv` | `ggml-cpu.c` 95/528/3918, `arch-fallback.h` 222 | RISC-V arch features, PAUSE hint, thread naming | RISC-V |
| `__riscv_v_intrinsic` | `ggml-cpu-impl.h` 348, `simd-mappings.h` 17, `vec.*`, `ops.cpp`, `simd-gemm.h`, `arch/riscv/*` | `#include <riscv_vector.h>` and all RVV kernel selection | RISC-V |
| `__riscv_zvfh`, `__riscv_zvfhmin`, `__riscv_zvfbfmin`, `__riscv_zvfbfwma` | `vec.h`, `vec.cpp`, `ggml-cpu.c` | Vector FP16/BF16 arithmetic/convert/MMA | RISC-V |
| `__riscv_xtheadvector` | `arch/riscv/quants.c`, `vec.h` | T-Head RVV-0.7.1 vendor variant | RISC-V |
| `__riscv_zihintpause` | `ggml-cpu.c` 530 | PAUSE spin hint | RISC-V |
| `__loongarch64`, `__loongarch_asx`, `__loongarch_sx` | `ggml-cpu-impl.h` 352–359 | LASX/LSX includes | other |
| `__wasm_simd128__` | `ggml-cpu-impl.h` 334 | `wasm_simd128.h` | other |
| `__POWER9_VECTOR__` / `__VXE__` / `__VXE2__` / `__VEC__` / `__s390x__` | `ggml-cpu-impl.h` 65–72, 338, 361 | PowerPC VSX / s390x vector | other |
| `GGML_SIMD` | `simd-gemm.h` 8/112 | Internal per-arch SIMD GEMM gate | internal |

**Critical semantic trap:** `__riscv_v_intrinsic` means only that the compiler *supports the RVV intrinsic API* — not that the V extension is enabled. Clang defines it for `rv64gc` without V, which makes ggml compile RVV branches that cannot compile (upstream issue #1535; fix in unmerged PR #1571 switching to `__riscv_v`). With the GCC 15.2 cross toolchain used here (default `-march` already includes `v` and GGML forces `rv64gcv_zfh_zvfh_zicbop_zihintpause`), the guard is satisfied correctly and the RVV kernels compile.

### Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `_WIN32` / `_WIN64` | Windows | Threading, atomics, `LoadLibrary`/`dlopen` registry, file I/O |
| `_MSC_VER` | MSVC | `__declspec(align)`, `<intrin.h>`, C11 atomics shim |
| `__MINGW32__` | MinGW | Windows paths in `ggml.c`, `ggml-cpu.c` |
| `__linux__` | Linux | `pthread_setaffinity_np`, `dlopen`, `sys/prctl.h`, `ggml-feats.h` |
| `__APPLE__` | macOS | `sysctl`, Accelerate, `MAP_JIT` |
| `__ANDROID__` | Android | thread/affinity differences |
| `__EMSCRIPTEN__` | WASM | single-thread guard |
| `__FreeBSD__` / `__NetBSD__` / `__OpenBSD__` | BSDs | OS include selection, memory hints |

For riscv64 Linux the `__linux__` path is taken (pthreads, `dlopen`, `m`, no cpuid); Windows/Apple/ARM-specific paths compile out cleanly.

### Portability verdict

| Aspect | Verdict |
|---|---|
| No exceptions | The core library is C11 and does not rely on C++ exceptions; C++ appears only in tests/examples/optional backends. No exception-based portability gate found. |
| Alignment safe | ggml uses its own alignment abstraction (`GGML_MEM_ALIGN`, `ggml_aligned_malloc`); no architecture-specific alignment assumptions were flagged. |
| Embedded usability | Core is dependency-free (pthreads/libm/libdl); OpenMP is optional. With `-static` the target binaries are self-contained but carry ~1.8 MB of runtime each and require RVV (minimum `Zvl128b`). |
| Overall | **LOW–MEDIUM** — low for GCC/rv64gcv (RVV compiled correctly); medium specifically for an unpatched Clang rv64gc build due to the `__riscv_v_intrinsic` gate. RISC-V is upstream-maintained, so this is targeted gating, not structural porting. |

## 3. Build & Test Results

| Platform | Build name | Recipe | Build result | Executables built | Tests run | Tests passed | Notes |
|---|---|---|---|---|---|---|---|
| Reference x86_64 | `1_1_1` | 1 | SUCCESS | 36/36 | 21 (CTest) | 21/21 | `ctest --output-on-failure -j4` 100% passed, 8.56 s; effective `-O2` because `RelWithDebInfo` appends `-O2 -g -DNDEBUG` after recipe `-O3`; shared/PIE |
| Target riscv64 (QEMU) | `1_1_2` | 2 (corrected) | SUCCESS after fixes | 36/36 | 21 (run directly under qemu) | 21/21 | `riscv64-linux-gnu-gcc`; static; `-march=rv64gcv_zfh_zvfh_zicbop_zihintpause -mabi=lp64d`; effective `-O2` |
| Optimized x86_64 | `1_1_3` | 3 | SUCCESS | 36/36 | smoke (`test-opt`, `test-quantize-perf`) | 2/2 | effective `-O3` (Release) |
| Optimized riscv64 (QEMU) | `1_1_4` | 4 | SUCCESS | 36/36 | smoke (`test-opt`, `test-quantize-perf`) | 2/2 | effective `-O3` (Release) |

### Build/test failures detail

- The **provided recipe 2 could not cross-build** as-is. Failures and fixes:
  1. CMake did not enter cross-compile mode (host `x86_64` detected) → ggml added `-march=native`; `riscv64-linux-gnu-gcc: error: '-march=native': ISA string must begin with rv32 or rv64`. Fixed with `-DCMAKE_SYSTEM_NAME=Linux -DCMAKE_SYSTEM_PROCESSOR=riscv64 -DGGML_NATIVE=OFF`.
  2. `tests/CMakeLists.txt` fell into its generic (x86) branch and appended host flags `-mavx -mavx2 -mfma -mf16c -msse3` → `unrecognized command-line option '‑mavx'`. Fixed by a **2-line source patch adding an explicit `riscv64` branch** (`elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "riscv64")`). This is upstream cross-compile gap issue #1388.
  3. `MATH_LIBRARY` NOTFOUND in cross mode → `-DCMAKE_FIND_ROOT_PATH=/usr/riscv64-linux-gnu -DCMAKE_FIND_ROOT_PATH_MODE_LIBRARY=BOTH -DMATH_LIBRARY=/usr/riscv64-linux-gnu/lib/libm.a`.
  4. `-static` conflicted with shared libs (`attempted static link of dynamic object libggml.so`) → `-DBUILD_SHARED_LIBS=OFF -DGGML_STATIC=ON`.
- All 12 example executables that require model/data files return non-zero with no arguments on both platforms (documented, not a portability failure).
- No test failures and no timeouts on either platform.

## 4. Performance Comparison

### Experimental conditions

| Item | Value |
|---|---|
| Host machine | Local x86_64 container (no SSH/remote machines) |
| CPU | Intel Core i5-1035G1, family 6 model 126; 4 physical cores / 8 logical (SMT-2), 1 socket |
| Caches | L1d 192 KiB, L1i 128 KiB, L2 2 MiB, L3 6 MiB |
| ISA (x86 build) | AVX-512F/DQ/CD/BW/VL, AVX2, FMA, VNNI, SHA, GFNI (`-march=native`) |
| Frequency governor | `powersave`; min 400 MHz, max 3600 MHz; observed ~1.3 GHz (~36% of max), drifting 0.6–1.3 GHz under load |
| `perf_event_paranoid` | `-1` (full access). `/proc/sys` is read-only. |
| Core pinning | `taskset -c 0` (applied by amixis to every time/stat/record command, both platforms) |
| Priority | No `nice` applied by amixis; run as root |
| Warmup runs | NONE (amixis performs no warmup) |
| Measurement repeats | Single measurement per step (no averaging / error bars) |
| Perf record events | `cycles,cache-misses,branch-misses` (chosen because `run_machine.arch == x86`) |
| Reference build | `1_1_1` x86_64, `-O3 -march=native -g`, effective `-O2`, shared/PIE |
| Target build | `1_1_2` riscv64, `-O3 -g`, effective `-O2`, `-march=rv64gcv_zfh_zvfh_zicbop_zihintpause -mabi=lp64d`, static |
| Emulation | QEMU user-mode (`qemu-riscv64-static -L /usr/riscv64-linux-gnu`) |
| Executables | 36 configured; **24 profiled on both platforms**; 12 examples skipped (smoke-test failure: require model files/args) |

### Key metrics (real measured)

Times from `/bin/time` (single run, `taskset -c 0`); IPC/miss/topdown from `perf stat -ddd`. `N/A` ratio means the x86 real time rounded to `0.00` (0.01 s timer resolution).

| Executable | real x86 (s) | real riscv (s) | ratio | IPC x86 | IPC riscv† | L1d% x86 | L1d% riscv† | LLC% x86 | LLC% riscv† | br% x86 | br% riscv† | FE% x86 | FE% riscv† | BE% x86 | BE% riscv† | Ret% x86 | Ret% riscv† |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| test-backend-ops | 0.01 | 0.60 | 60.0 | 1.4 | 2.5 | 1.6 | 1.9 | 22.2 | 8.6 | 3.0 | 0.8 | 24.3 | 22.2 | 27.4 | 4.4 | 42.7 | 29.7 |
| test-opt | 1.95 | 3.62 | 1.86 | 0.6 | 1.5 | 2.5 | 1.7 | 20.6 | 17.0 | 0.7 | 0.6 | 15.3 | 13.1 | 61.6 | 16.7 | 11.7 | 23.8 |
| test-quantize-fns | 42.50 | 730.42 | 17.19 | 1.7 | 3.9 | 0.2 | 0.0 | 20.7 | 22.8 | 4.1 | 0.4 | 21.5 | 5.0 | 3.9 | 0.2 | 13.0 | 6.9 |
| test-quantize-perf | 0.20 | 4.21 | 21.05 | 2.1 | 3.8 | 0.1 | 0.0 | 24.2 | 23.2 | 0.8 | 0.2 | 3.3 | 7.1 | 31.5 | 0.2 | 35.4 | 17.0 |
| test-pool | 0.00 | 0.11 | N/A | 1.6 | 2.6 | 5.5 | 0.3 | 20.7 | 25.2 | 2.6 | 1.0 | 11.7 | 21.0 | 40.3 | 2.5 | 45.5 | 47.0 |
| test-arange | 0.00 | 0.14 | N/A | 1.4 | 2.6 | 5.6 | 0.4 | 24.9 | 32.4 | 2.9 | 0.9 | 18.1 | 22.3 | 37.5 | 2.5 | 39.2 | 40.3 |
| test-timestep_embedding | 0.00 | 0.15 | N/A | 1.4 | 2.6 | 5.7 | 0.5 | 22.4 | 25.6 | 3.2 | 1.0 | 18.9 | 25.1 | 37.1 | 2.8 | 39.6 | 40.1 |
| test-pad-reflect-1d | 0.01 | 0.16 | 16.0 | 1.5 | 2.6 | 5.5 | 0.4 | 22.3 | 25.3 | 2.8 | 1.1 | 17.7 | 22.1 | 39.8 | 2.5 | 38.0 | 40.9 |
| test-roll | 0.02 | 0.24 | 12.0 | 1.6 | 2.9 | 5.0 | 0.4 | 18.5 | 39.2 | 1.3 | 0.7 | 19.1 | 19.3 | 35.6 | 4.1 | 37.7 | 52.4 |
| test-conv-transpose | 0.00 | 0.12 | N/A | 1.7 | 2.6 | 5.6 | 0.4 | 25.1 | 25.1 | 2.1 | 0.8 | 17.7 | 21.7 | 35.6 | 2.7 | 42.4 | 46.2 |
| test-conv-transpose-1d | 0.01 | 0.30 | 30.0 | 1.6 | 3.2 | 1.9 | 0.2 | 21.3 | 25.7 | 2.1 | 0.5 | 13.7 | 13.3 | 38.7 | 1.6 | 43.5 | 38.6 |
| test-dup | 0.01 | 0.12 | 12.0 | 1.5 | 2.6 | 5.7 | 0.4 | 21.4 | 40.0 | 3.0 | 1.0 | 15.7 | 22.2 | 37.2 | 2.6 | 43.1 | 45.3 |
| test-rel-pos | 0.00 | 0.13 | N/A | 1.4 | 2.2 | 5.6 | 0.8 | 24.1 | 25.5 | 3.2 | 0.8 | 14.1 | 26.1 | 38.7 | 2.1 | 43.1 | 43.6 |
| test-customop | 0.00 | 0.15 | N/A | 1.4 | 2.5 | 5.6 | 0.4 | 22.6 | 35.9 | 3.1 | 1.2 | 18.5 | 20.6 | 38.7 | 2.3 | 38.0 | 42.3 |
| test-conv1d | 0.00 | 0.14 | N/A | 1.6 | 2.6 | 5.5 | 0.4 | 22.0 | 26.5 | 2.9 | 1.1 | 15.3 | 22.8 | 38.3 | 2.5 | 42.0 | 42.3 |
| test-conv1d-dw-c1 | 0.00 | 0.13 | N/A | 1.3 | 2.6 | 4.9 | 0.4 | 41.9 | 38.7 | 3.3 | 1.1 | 18.4 | 21.0 | 36.0 | 2.2 | 40.0 | 44.0 |
| test-conv1d-dw-c2 | 0.00 | 0.14 | N/A | 1.3 | 2.5 | 5.5 | 0.4 | 26.1 | 29.7 | 3.3 | 1.1 | 17.5 | 22.3 | 36.5 | 2.7 | 39.3 | 41.1 |
| test-conv2d | 0.01 | 0.15 | 15.0 | 1.5 | 2.7 | 4.4 | 0.5 | 19.8 | 28.3 | 3.1 | 0.9 | 16.4 | 23.1 | 37.6 | 2.7 | 41.2 | 42.1 |
| test-conv2d-dw | 0.01 | 0.17 | 17.0 | 1.6 | 2.6 | 6.1 | 0.5 | 31.3 | 36.9 | 2.8 | 1.0 | 15.3 | 24.1 | 39.8 | 2.8 | 40.8 | 42.8 |
| test-cont | 0.01 | 0.14 | 14.0 | 1.4 | 2.5 | 5.9 | 0.6 | 22.4 | 36.2 | 3.1 | 0.9 | 18.5 | 22.8 | 37.1 | 3.3 | 39.6 | 44.5 |
| test-interpolate | 0.00 | 0.15 | N/A | 1.5 | 2.5 | 6.8 | 0.4 | 28.8 | 22.5 | 2.7 | 1.1 | 14.4 | 21.4 | 40.5 | 2.2 | 41.8 | 41.3 |
| mnist-train | 0.00 | 0.04 | N/A | 1.2 | 1.9 | 5.6 | 0.7 | 23.8 | 28.8 | 3.1 | 1.9 | 36.8 | 35.7 | 25.1 | 6.8 | 25.9 | 37.7 |
| simple-ctx | 0.00 | 0.12 | N/A | 1.4 | 2.6 | 5.6 | 0.3 | 23.0 | 39.2 | 3.2 | 1.0 | 14.1 | 21.5 | 38.7 | 2.2 | 43.1 | 47.5 |
| simple-backend | 0.01 | 1.02 | 102.0 | 1.3 | 2.7 | 1.7 | 2.0 | 28.5 | 4.5 | 2.8 | 0.7 | 25.8 | 21.2 | 22.5 | 2.4 | 44.3 | 20.5 |

† **All riscv IPC / L1d / LLC / branch / topdown values are HOST x86 counters for the `qemu-riscv64-static` process, not guest RISC-V micro-architectural metrics.** They must not be interpreted as guest behavior.

### Hotspot analysis

Reference platform (x86_64 / `1_1_1`):

| Executable / event | Top symbols (% of event period) |
|---|---|
| `test-quantize-fns` / cycles | `iq2_compare_func` 46.4%, `[unknown]` 49.0% (`/ (deleted)` DSO pages), `iq2xs_init_impl._omp_fn.{0,1}` 2.2%, `iq3_compare_func` 0.8%, `memcpy@plt` 0.7% |
| `test-quantize-fns` / branch-misses | `[unknown]` 67.4%, `iq2_compare_func` 31.2%, `memcpy@plt` 0.7% |
| `test-quantize-perf` / cycles | `quantize_row_iq4_nl_impl` 45.3%, `make_qkx2_quants` 29.8%, `make_qx_quants` 7.0%, `quantize_row_mxfp4_ref` 3.0%, `quantize_row_nvfp4_ref` 3.0%, `quantize_row_q8_K_ref` 1.5% |
| `test-quantize-perf` / branch-misses | `make_qkx2_quants` 35.7%, `quantize_row_iq4_nl_impl` 32.2%, `make_qx_quants` 4.0% |
| `test-opt` / cycles | `[unknown]` 93%, `ggml_compute_forward_mul` 2.7%, `..._sub` 1.2%, `..._add_non_quantized` 0.7% |
| `test-backend-ops` / cycles | `[unknown]` 53.7%, `tanhf32` 15.7%, `ggml_cpu_init` 15.7%, `std::filesystem` ~15% |

Target platform (riscv64 under QEMU / `1_1_2`): **`[unknown]` = 99.9–100.0% of every event for every executable.** No guest RISC-V function-level hotspots are recoverable (see QEMU caveat).

### Bottleneck summary

- Elapsed-time ratios on the target (up to 102×) are dominated by **QEMU dynamic binary translation**, not by guest instruction cost or frequency difference. Instruction-light programs show the largest ratios, a pure fixed-overhead/DL-translation signature.
- `taskset -c 0` serializes ggml's OpenMP work onto one logical core; OpenMP frames appear in the x86 perf script. An earlier unpinned vs pinned comparison showed ~3.2× wall-time inflation from pinning.
- The `powersave` governor (~1.3 GHz, drifting) inflates absolute times and adds noise.
- Both platforms auto-vectorize (see below); the RISC-V target's counters describe the emulator, so no guest bottleneck can be attributed from them.
- On x86, real hotspots are the reference quantization kernels (`iq2_compare_func`, `quantize_row_iq4_nl_impl`, `make_qkx2_quants`), which are one-time init/model-conversion routines with data-dependent branches; GEMM/inference kernels are not exercised by these tests.

### Vectorization intrinsics

| ISA | Representative intrinsics (occurrences in `src/`) | Key files |
|---|---|---|
| x86 SSE/AVX/AVX-512 | `_mm_` 2440, `_mm256_` 5001, `_mm512_` 2986 | `arch/x86/quants.c`, `arch/x86/repack.cpp`, `vec.cpp`, `vec.h`, `iqp.cpp`, `simd-mappings.h`, `amx/mmq.cpp`, `llamafile/sgemm.cpp` |
| ARM NEON | `vld1q` 438, `vdupq` 319, `vdotq` 338, `vshrq` 199, `vcvt` 218 | `arch/arm/repack.cpp`, `arch/arm/quants.c`, `simd-mappings.h`, `kleidiai/kleidiai.cpp` |
| RISC-V RVV | `__riscv_vle` 556, `__riscv_vse` 343, `__riscv_vget` 304, `__riscv_vsetvl` 173, `__riscv_vfmacc` 75, `__riscv_vredsum` 43 | `arch/riscv/quants.c` (1744), `arch/riscv/repack.cpp` (726), `spacemit/rvv_kernels.cpp` (636), `vec.h`, `vec.cpp`, `simd-gemm.h` |
| WASM SIMD | `wasm_f32x4` 122, `wasm_v128` 94, `wasm_i8x16` 34 | `arch/wasm/quants.c`, `simd-mappings.h` |
| PowerPC VSX / s390x VXE | `vec_*` macro-mapped ops | `simd-mappings.h`, `arch/powerpc/*`, `arch/s390/*` |
| LoongArch LSX/LASX | `lasxintrin.h` / `lsxintrin.h` | `ggml-cpu-impl.h`, `arch/loongarch/quants.c` |

No x86 or ARM intrinsics leak into the RISC-V path — all are behind arch guards. RISC-V has first-class RVV kernels for quants, repack, vec and GEMM.

### QEMU / emulation caveat

The target was executed under **QEMU user-mode** (`qemu-riscv64-static -L /usr/riscv64-linux-gnu`). Consequences: (1) all target wall-clock times include emulation overhead and do not reflect native RISC-V hardware; (2) `perf` profiled the **qemu host process**, so target IPC/cache/branch/topdown counters and hotspot attribution describe the emulator, not the guest (guest symbols are absent); (3) `perf record` callchains resolve to QEMU's translated-code region (`[unknown]` ≈ 100%). No guest micro-architectural claims should be drawn from the target columns.

## 5. Optimization Results

### Vector instructions in binary

| Architecture / binary | Vector insns | ISAs found |
|---|---:|---|
| x86 `libggml-cpu.so` | 13,055 | AVX2 + AVX-512 (zmm 4,555, ymm 8,884) + FMA (`vfmadd*` ≈534) + VNNI (`vpdpbusd` 293, `vpdpwssd` 22) + `vperm/vpermd` 569 + `vpmaddubsw` 885 |
| riscv `libggml-cpu.a` | ~7,601 (tool count) / 3,590 broad | RVV 1.0: `vsetvli/vsetvl` 1,067, `vle8` 533, `vand` 223, `vse32` 184, `vadd` 152, `vle32` 147, `vfmul` 111, `vmul` 91, `vfmacc` 73, `vrgather` 54, `vfsub` 33, `vfwmacc` 1 |
| x86 `libggml-base.so` | AVX2 only (zmm 0, ymm 2,590) + FMA | `vmulps` 415, `vfmadd132ps` 143, `vaddps` 85, `vpaddd` 118, `vfmadd*ss` ~159 |
| riscv `libggml-base.a :: ggml-quants.c.o` | 1,685 | `vsetvli` 603, `vle32` 243, `vle8` 200, `vse32` 176, `vfmul` 172, `vadd` 106, `vand` 91, `vfred` 74, `vfmadd` 53, `vfmacc` 4, `vfadd` 27 |

x86 uses AVX-512 + FMA + VNNI; RISC-V auto-vectorizes to RVV 1.0 but with much weaker FP contraction (`vfmul` 172 vs `vfmacc` 4 in base quants) and no integer dot-product equivalent to VNNI.

### Executable size analysis

`.text/.data/.bss` are identical for stripped and unstripped files (stripping removes only debug sections).

| Binary | x86 raw (B) | x86 stripped (B) | riscv raw (B) | riscv stripped (B) | stripped ratio (riscv/x86) |
|---|---:|---:|---:|---:|---:|
| test-quantize-fns | 208,008 | 18,728 | 10,936,752 | 2,947,248 | 157× |
| test-quantize-perf | 338,072 | 35,112 | 11,053,992 | 2,955,440 | 84× |
| test-opt | 661,288 | 55,520 | 12,829,616 | 3,303,600 | 60× |
| simple-backend | 102,000 | 18,736 | 12,009,232 | 3,250,408 | 174× |
| test-backend-ops | 15,628,856 | 1,211,808 | 29,507,536 | 4,091,248 | 3.4× |

Actual ggml `.text` is **smaller** on riscv (base 0.72×, cpu 0.69×, total 0.70×); the 60–174× stripped-size ratio is due to static linking (~1.8 MB libc/libm/libgcc/libgomp/CRT per binary) plus `-g` debug info, not ISA code bloat.

### Optimization attempts

| Optimization | Platform | Build before → after | Parameter / executable | Before | After | Improvement % | Causal analysis |
|---|---|---|---|---|---|---|---|
| Effective `-O3` (CMAKE_BUILD_TYPE=Release) | x86_64 | 1_1_1 → 1_1_3 | real_time / test-quantize-perf | 0.19 | 0.18 | 94.74 | Small win (~5%): `-O3` unrolling/inlining helps the reference quantize loop. Within run-to-run noise on this pinned, DVFS-limited host. |
| Effective `-O3` | x86_64 | 1_1_1 → 1_1_3 | user_time / test-quantize-perf | 0.19 | 0.17 | 89.47 | Consistent ~10% user-time reduction; same causal cause. |
| Effective `-O3` | x86_64 | 1_1_1 → 1_1_3 | real_time / test-opt | 2.04 | 2.05 | 100.49 | Flat/slightly worse (+0.5%) — noise; test-opt is dominated by elementwise ops already at parity. |
| Effective `-O3` | riscv64 | 1_1_2 → 1_1_4 | real_time / test-quantize-perf | 4.185 | 3.955 | 94.50 | ~5.5% faster under QEMU: fewer guest instructions to translate/execute. |
| Effective `-O3` | riscv64 | 1_1_2 → 1_1_4 | user_time / test-quantize-perf | 4.18 | 3.935 | 94.14 | Consistent with the wall-clock gain. |
| Effective `-O3` | riscv64 | 1_1_2 → 1_1_4 | real_time / test-opt | 3.52 | 3.99 | 113.35 | **Slower** under QEMU: `-O3` inlining enlarges the already-large static binary, increasing TCG translation/cache pressure. This is an emulation artifact, not evidence about native RISC-V. |
| Effective `-O3` | riscv64 | 1_1_2 → 1_1_4 | real_time / test-backend-ops | 0.72 | 0.715 | 99.31 | Negligible (CPU backend is skipped by this test). |

### Recommended optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--:|---|---|:--:|---|
| 1 | Make `-O3` effective (`Release` or override `*_RELWITHDEBINFO`) | Few % to ~10% on vectorizable kernels (measured ~5–10% on quantize-perf) | Low | Confirmed baseline was effectively `-O2`. Apply to both recipes. |
| 2 | Fix the measurement harness (dynamic target link, `OMP_NUM_THREADS`, `performance` governor, warmup + ≥5 repeats, real guest counters) | 0% (enables valid analysis) | Low–Med | Without this, no target optimization can be validated; QEMU-user counters are emulator-side. |
| 3 | LTO on both builds | ~2–8% (hypothesis) | Low–Med | Cross-module inlining base↔cpu; test separately from static linking. |
| 4 | Explicit FP contraction for quant kernels (`-ffp-contract=fast`, then `-funsafe-math-optimizations -fno-math-errno`) | ~5–15% on quantize kernels (hypothesis) | Low, needs test validation | Gate on `test-quantize-fns` passing (reference quantizer outputs must not change). |
| 5 | Restore bitmanip in `ggml-cpu` march (`zbb/zbs/zba/zfa/zvbb`) | ~1–5% (hypothesis) | Low | Explicit `-march` narrows the toolchain default; requires a small CMake `MARCH_STR` patch. |
| 6 | Strip target debug info (`-s` / post-strip) | 0% runtime; −70% artifact size | Low | Distribution/CI only. |
| 7 | Manual RVV for reference quantizers (`best_index_int8` via `vrgather`, vectorized reductions) | ~10–20% on quantize workloads (hypothesis) | High | Extend `arch/riscv/quants.c`; validate numerics. |
| 8 | Toolchain experiment: Clang 19+ vs GCC 15 for RVV | Unknown | Med | Better vector-length-agnostic codegen on some kernels. |
| 9 | Allocator swap (mimalloc/jemalloc/tcmalloc) | ~0% | Low | Evidence (L1d miss ≈0.1–0.2%, one-time allocations) does not support this. |
## Improvement of 1_1_1 compared to 1_1_3

| Measured | Baseline value | Optimized value | Improvement % | Parameter | Baseline build | Optimized build |
|---|---:|---:|---:|---|---|---|
| test-quantize-perf | 0.19 | 0.18 | 94.74 | real_time | 1_1_1 | 1_1_3 |
| test-opt | 2.04 | 2.05 | 100.49 | real_time | 1_1_1 | 1_1_3 |
| test-quantize-perf | 0.19 | 0.17 | 89.47 | user_time | 1_1_1 | 1_1_3 |

## Improvement of 1_1_2 compared to 1_1_4

| Measured | Baseline value | Optimized value | Improvement % | Parameter | Baseline build | Optimized build |
|---|---:|---:|---:|---|---|---|
| test-opt | 3.52 | 3.99 | 113.35 | real_time | 1_1_2 | 1_1_4 |
| test-quantize-perf | 4.185 | 3.955 | 94.5 | real_time | 1_1_2 | 1_1_4 |
| test-quantize-perf | 4.18 | 3.935 | 94.14 | user_time | 1_1_2 | 1_1_4 |
| test-backend-ops | 0.72 | 0.715 | 99.31 | real_time | 1_1_2 | 1_1_4 |
## Cross-table: 1. CT-1__1__1..bin_smnist-train-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_smnist-train
# Cross-tables for 1\_\_1\_\_1..bin\_smnist-train and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_smnist-train

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   48.75 |   98.68 |  +49.93 |
| \[unknown\] (/ |   51.25 |    1.32 |  -49.93 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   32.88 |    0.00 |  -32.88 |
| \[unknown\]    |   67.12 |  100.00 |  +32.88 |


## Cross-table: 2. CT-1__1__1..bin_ssimple-backend-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_ssimple-backend
# Cross-tables for 1\_\_1\_\_1..bin\_ssimple-backend and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_ssimple-backend

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   25.54 |   99.43 |  +73.89 |
| \[unknown\] (/ |   74.46 |    0.57 |  -73.89 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                                                                                                                                                                                            | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                                                                                                                       |   43.66 |   99.97 |  +56.31 |
| \[unknown\] (/                                                                                                                                                                                    |   16.81 |    0.03 |  -16.78 |
| ggml\_cpu\_init                                                                                                                                                                                   |    6.78 |    0.00 |   -6.78 |
| tanhf32 (/                                                                                                                                                                                        |    6.75 |    0.00 |   -6.75 |
| std::filesystem::\_\_cxx11::path::\_List::\_List(std::filesystem::\_\_cxx11::path::\_List const&) (/                                                                                              |    6.65 |    0.00 |   -6.65 |
| void std::\_\_cxx11::basic\_string<char, std::char\_traits<char>, std::allocator<char> >::\_M\_construct<char const\*>(char const\*, char const\*, std::forward\_iterator\_tag) \[clone .isra.0\] |    6.57 |    0.00 |   -6.57 |
| std::filesystem::\_\_cxx11::path::\_M\_append(std::basic\_string\_view<char, std::char\_traits<char> >) (/                                                                                        |    6.46 |    0.00 |   -6.46 |
| std::\_\_cxx11::basic\_string<char, std::char\_traits<char>, std::allocator<char> >::\_M\_mutate(unsigned long, unsigned long, char const\*, unsigned long) (/                                    |    6.32 |    0.00 |   -6.32 |


## Cross-table: 3. CT-1__1__1..bin_ssimple-ctx-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_ssimple-ctx
# Cross-tables for 1\_\_1\_\_1..bin\_ssimple-ctx and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_ssimple-ctx

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   34.94 |    1.03 |  -33.91 |
| \[unknown\]    |   65.06 |   98.97 |  +33.91 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   14.08 |    0.00 |  -14.08 |
| \[unknown\]    |   85.92 |  100.00 |  +14.08 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   36.89 |  100.00 |  +63.11 |
| ggml\_cpu\_init |   33.12 |    0.00 |  -33.12 |
| \[unknown\] (/  |   19.09 |    0.00 |  -19.09 |
| tanhf32 (/      |   10.90 |    0.00 |  -10.90 |


## Cross-table: 4. CT-1__1__1..bin_stest-arange-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-arange
# Cross-tables for 1\_\_1\_\_1..bin\_stest-arange and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-arange

## EVENT: BRANCH-MISSES

| Symbol                   | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------|-------:|-------:|-------:|
| \[unknown\]              |   57.39 |   99.00 |  +41.61 |
| \[unknown\] (/           |   29.28 |    1.00 |  -28.29 |
| \_\_cpu\_indicator\_init |   13.33 |    0.00 |  -13.33 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |   91.68 |  100.00 |   +8.32 |
| strlen (/   |    8.32 |    0.00 |   -8.32 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   61.46 |   99.60 |  +38.14 |
| ggml\_cpu\_init |   20.29 |    0.00 |  -20.29 |
| tanhf@plt       |    9.38 |    0.00 |   -9.38 |
| \[unknown\] (/  |    8.87 |    0.40 |   -8.47 |


## Cross-table: 5. CT-1__1__1..bin_stest-backend-ops-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-backend-ops
# Cross-tables for 1\_\_1\_\_1..bin\_stest-backend-ops and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-backend-ops

## EVENT: BRANCH-MISSES

| Symbol                                                                                                                                                      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------------------------------------------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                                                                                 |   72.24 |  100.00 |  +27.76 |
| std::basic\_ios<char, std::char\_traits<char> >::\_M\_cache\_locale(std::locale const&)                                                                     |   14.29 |    0.00 |  -14.29 |
| std::\_\_cxx11::basic\_string<char, std::char\_traits<char>, std::allocator<char> >::\_M\_mutate(unsigned long, unsigned long, char const\*, unsigned long) |   13.47 |    0.00 |  -13.47 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                                                                                                                          | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                                                     |   53.72 |  100.00 |  +46.28 |
| tanhf32                                                                                                                         |   15.70 |    0.00 |  -15.70 |
| ggml\_cpu\_init                                                                                                                 |   15.66 |    0.00 |  -15.66 |
| std::filesystem::\_\_cxx11::path::\_List::\_Impl\_deleter::operator()(std::filesystem::\_\_cxx11::path::\_List::\_Impl\*) const |    7.64 |    0.00 |   -7.64 |
| std::filesystem::\_\_cxx11::path::\_List::\_List()                                                                              |    7.28 |    0.00 |   -7.28 |


## Cross-table: 6. CT-1__1__1..bin_stest-cont-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-cont
# Cross-tables for 1\_\_1\_\_1..bin\_stest-cont and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-cont

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   52.68 |   97.76 |  +45.08 |
| \[unknown\] (/ |   47.32 |    2.24 |  -45.08 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   66.66 |  100.00 |  +33.34 |
| \[unknown\] (/ |   22.31 |    0.00 |  -22.31 |
| .plt (/        |   11.04 |    0.00 |  -11.04 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   51.20 |   99.56 |  +48.37 |
| ggml\_cpu\_init |   29.55 |    0.00 |  -29.55 |
| \[unknown\] (/  |   19.25 |    0.00 |  -19.25 |
| sigsetmask (/   |    0.00 |    0.44 |   +0.44 |


## Cross-table: 7. CT-1__1__1..bin_stest-conv-transpose-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv-transpose
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv-transpose and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv-transpose

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   73.02 |   98.68 |  +25.66 |
| \[unknown\] (/ |   26.98 |    1.32 |  -25.66 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   41.87 |   99.71 |  +57.84 |
| \[unknown\] (/  |   33.34 |    0.29 |  -33.05 |
| tanhf32 (/      |   16.82 |    0.00 |  -16.82 |
| ggml\_cpu\_init |    7.97 |    0.00 |   -7.97 |


## Cross-table: 8. CT-1__1__1..bin_stest-conv-transpose-1d-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv-transpose-1d
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv-transpose-1d and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv-transpose-1d

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   35.63 |    1.45 |  -34.18 |
| \[unknown\]    |   64.37 |   98.55 |  +34.18 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   14.89 |    4.85 |  -10.04 |
| \[unknown\]    |   85.11 |   95.15 |  +10.04 |

## EVENT: CYCLES

| Symbol              | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------|-------:|-------:|-------:|
| \[unknown\]         |   39.16 |   99.87 |  +60.71 |
| ggml\_vec\_dot\_f32 |   33.49 |    0.00 |  -33.49 |
| \[unknown\] (/      |   15.18 |    0.13 |  -15.05 |
| tanhf32 (/          |   12.17 |    0.00 |  -12.17 |


## Cross-table: 9. CT-1__1__1..bin_stest-conv1d-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv1d
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv1d and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv1d

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   29.44 |  100.00 |  +70.56 |
| \[unknown\] (/ |   54.45 |    0.00 |  -54.45 |
| \_\_sysconf (/ |   16.11 |    0.00 |  -16.11 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   91.56 |  100.00 |   +8.44 |
| \[unknown\] (/ |    8.44 |    0.00 |   -8.44 |

## EVENT: CYCLES

| Symbol              | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------|-------:|-------:|-------:|
| \[unknown\]         |   54.42 |  100.00 |  +45.58 |
| \[unknown\] (/      |   20.97 |    0.00 |  -20.97 |
| ggml\_cpu\_init     |   20.14 |    0.00 |  -20.14 |
| \_\_vdso\_getrandom |    4.47 |    0.00 |   -4.47 |


## Cross-table: 10. CT-1__1__1..bin_stest-conv1d-dw-c1-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv1d-dw-c1
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv1d-dw-c1 and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv1d-dw-c1

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   64.15 |   98.06 |  +33.91 |
| \[unknown\] (/ |   35.85 |    1.94 |  -33.91 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   61.58 |   96.06 |  +34.48 |
| \[unknown\] (/ |   31.87 |    3.94 |  -27.94 |
| ggml\_is\_numa |    6.54 |    0.00 |   -6.54 |

## EVENT: CYCLES

| Symbol              | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------|-------:|-------:|-------:|
| \[unknown\]         |   64.70 |   99.75 |  +35.06 |
| \[unknown\] (/      |   22.23 |    0.25 |  -21.98 |
| ggml\_cpu\_init     |    9.21 |    0.00 |   -9.21 |
| \_\_vdso\_getrandom |    3.86 |    0.00 |   -3.86 |


## Cross-table: 11. CT-1__1__1..bin_stest-conv1d-dw-c2-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv1d-dw-c2
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv1d-dw-c2 and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv1d-dw-c2

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   38.29 |    0.95 |  -37.34 |
| \[unknown\]    |   61.71 |   99.05 |  +37.34 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   89.94 |  100.00 |  +10.06 |
| \[unknown\] (/ |   10.06 |    0.00 |  -10.06 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   44.53 |   99.72 |  +55.19 |
| ggml\_cpu\_init |   29.13 |    0.00 |  -29.13 |
| \[unknown\] (/  |   17.23 |    0.28 |  -16.95 |
| tanhf32 (/      |    9.10 |    0.00 |   -9.10 |


## Cross-table: 12. CT-1__1__1..bin_stest-conv2d-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv2d
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv2d and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv2d

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   49.91 |   99.33 |  +49.42 |
| \[unknown\] (/ |   40.93 |    0.67 |  -40.26 |
| .plt           |    9.17 |    0.00 |   -9.17 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |    8.09 |    0.00 |   -8.09 |
| \[unknown\]    |   91.91 |  100.00 |   +8.09 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   39.92 |   99.77 |  +59.85 |
| \[unknown\] (/  |   30.89 |    0.23 |  -30.66 |
| ggml\_cpu\_init |   19.14 |    0.00 |  -19.14 |
| tanhf32 (/      |   10.05 |    0.00 |  -10.05 |


## Cross-table: 13. CT-1__1__1..bin_stest-conv2d-dw-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-conv2d-dw
# Cross-tables for 1\_\_1\_\_1..bin\_stest-conv2d-dw and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-conv2d-dw

## EVENT: BRANCH-MISSES

| Symbol                                                                                             | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                        |   35.71 |   99.25 |  +63.54 |
| \[unknown\] (/                                                                                     |   52.30 |    0.75 |  -51.55 |
| conv\_2d\_dw\_reference(int, int, float const\*, int, int, float const\*, int, int, int, int, int) |   12.00 |    0.00 |  -12.00 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   78.69 |  100.00 |  +21.31 |
| \[unknown\] (/ |   12.45 |    0.00 |  -12.45 |
| .plt           |    8.86 |    0.00 |   -8.86 |

## EVENT: CYCLES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   49.27 |  100.00 |  +50.73 |
| \[unknown\] (/ |   26.32 |    0.00 |  -26.32 |
| tanhf32 (/     |   19.33 |    0.00 |  -19.33 |
| malloc (/      |    5.08 |    0.00 |   -5.08 |


## Cross-table: 14. CT-1__1__1..bin_stest-customop-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-customop
# Cross-tables for 1\_\_1\_\_1..bin\_stest-customop and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-customop

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   30.76 |    0.78 |  -29.99 |
| \[unknown\]    |   69.24 |   99.22 |  +29.99 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   63.55 |   95.58 |  +32.04 |
| \[unknown\] (/ |   24.87 |    4.42 |  -20.46 |
| .plt           |   11.58 |    0.00 |  -11.58 |

## EVENT: CYCLES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   36.24 |  100.00 |  +63.76 |
| \[unknown\] (/ |   53.91 |    0.00 |  -53.91 |
| tanhf32 (/     |    9.86 |    0.00 |   -9.86 |


## Cross-table: 15. CT-1__1__1..bin_stest-dup-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-dup
# Cross-tables for 1\_\_1\_\_1..bin\_stest-dup and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-dup

## EVENT: BRANCH-MISSES

| Symbol                     | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------|-------:|-------:|-------:|
| \[unknown\]                |   65.97 |  100.00 |  +34.03 |
| \[unknown\] (/             |   17.07 |    0.00 |  -17.07 |
| \_\_cxa\_guard\_acquire (/ |   16.97 |    0.00 |  -16.97 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   87.22 |   91.44 |   +4.22 |
| \[unknown\] (/ |   12.78 |    8.56 |   -4.22 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   42.95 |  100.00 |  +57.05 |
| ggml\_cpu\_init |   21.62 |    0.00 |  -21.62 |
| \[unknown\] (/  |   13.66 |    0.00 |  -13.66 |
| tanhf32 (/      |   11.05 |    0.00 |  -11.05 |
| tanhf@plt       |   10.73 |    0.00 |  -10.73 |


## Cross-table: 16. CT-1__1__1..bin_stest-interpolate-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-interpolate
# Cross-tables for 1\_\_1\_\_1..bin\_stest-interpolate and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-interpolate

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   27.83 |   99.12 |  +71.29 |
| \[unknown\] (/ |   72.17 |    0.88 |  -71.29 |

## EVENT: CACHE-MISSES

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   51.73 |  100.00 |  +48.27 |
| ggml\_cpu\_init |   21.39 |    0.00 |  -21.39 |
| \[unknown\] (/  |   16.59 |    0.00 |  -16.59 |
| tanhf32 (/      |   10.29 |    0.00 |  -10.29 |


## Cross-table: 17. CT-1__1__1..bin_stest-opt-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-opt
# Cross-tables for 1\_\_1\_\_1..bin\_stest-opt and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-opt

## EVENT: BRANCH-MISSES

| Symbol                                                                                               | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                          |   74.86 |   99.75 |  +24.89 |
| ggml\_graph\_compute\_thread.isra.0                                                                  |    9.67 |    0.00 |   -9.67 |
| \[unknown\] (/                                                                                       |    7.79 |    0.25 |   -7.54 |
| ggml\_backend\_load\_best(char const\*, bool, char const\*) \[clone .constprop.0\] \[clone .isra.0\] |    0.86 |    0.00 |   -0.86 |
| ggml\_backend\_sched\_graph\_compute\_async                                                          |    0.65 |    0.00 |   -0.65 |
| omp\_get\_thread\_num@plt                                                                            |    0.45 |    0.00 |   -0.45 |
| ggml\_graph\_compute.\_omp\_fn.0                                                                     |    0.45 |    0.00 |   -0.45 |
| ggml\_compute\_forward\_scale                                                                        |    0.44 |    0.00 |   -0.44 |
| ggml\_is\_contiguous\_0                                                                              |    0.39 |    0.00 |   -0.39 |
| ggml\_compute\_forward\_repeat\_back@plt                                                             |    0.32 |    0.00 |   -0.32 |
| ggml\_compute\_forward\_sum@plt                                                                      |    0.28 |    0.00 |   -0.28 |
| ggml\_compute\_forward\_repeat@plt                                                                   |    0.27 |    0.00 |   -0.27 |
| ggml\_opt\_eval                                                                                      |    0.20 |    0.00 |   -0.20 |
| GOMP\_parallel (/                                                                                    |    0.20 |    0.00 |   -0.20 |
| ggml\_compute\_forward\_opt\_step\_sgd                                                               |    0.19 |    0.00 |   -0.19 |
| ggml\_compute\_forward\_opt\_step\_sgd@plt                                                           |    0.19 |    0.00 |   -0.19 |
| ggml\_barrier                                                                                        |    0.18 |    0.00 |   -0.18 |
| ggml\_compute\_forward\_mul@plt                                                                      |    0.18 |    0.00 |   -0.18 |
| GOMP\_barrier (/                                                                                     |    0.15 |    0.00 |   -0.15 |
| ggml\_is\_contiguous                                                                                 |    0.15 |    0.00 |   -0.15 |

## EVENT: CACHE-MISSES

| Symbol                                                              | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                         |   73.27 |   99.38 |  +26.11 |
| \[unknown\] (/                                                      |   19.11 |    0.62 |  -18.49 |
| \_GLOBAL\_\_sub\_I\_gguf.cpp                                        |    2.75 |    0.00 |   -2.75 |
| ggml\_compute\_forward\_sub                                         |    1.57 |    0.00 |   -1.57 |
| ggml\_threadpool\_new\_impl                                         |    0.60 |    0.00 |   -0.60 |
| ggml\_backend\_tensor\_set                                          |    0.47 |    0.00 |   -0.47 |
| ggml\_compute\_forward\_mul                                         |    0.23 |    0.00 |   -0.23 |
| ggml\_graph\_compute\_thread.isra.0                                 |    0.23 |    0.00 |   -0.23 |
| ggml\_graph\_compute                                                |    0.22 |    0.00 |   -0.22 |
| ggml\_aligned\_malloc                                               |    0.19 |    0.00 |   -0.19 |
| ggml\_compute\_forward\_sqr@plt                                     |    0.16 |    0.00 |   -0.16 |
| ggml\_cpu\_extra\_compute\_forward                                  |    0.16 |    0.00 |   -0.16 |
| ggml\_backend\_cpu\_graph\_compute(ggml\_backend\*, ggml\_cgraph\*) |    0.16 |    0.00 |   -0.16 |
| ggml\_nbytes                                                        |    0.15 |    0.00 |   -0.15 |
| pthread\_mutex\_lock (/                                             |    0.15 |    0.00 |   -0.15 |
| ggml\_hash\_find                                                    |    0.14 |    0.00 |   -0.14 |
| ggml\_backend\_cpu\_get\_extra\_buffer\_types()                     |    0.09 |    0.00 |   -0.09 |
| ggml\_is\_numa@plt                                                  |    0.07 |    0.00 |   -0.07 |
| ggml\_dup\_tensor                                                   |    0.07 |    0.00 |   -0.07 |
| GOMP\_barrier@plt                                                   |    0.05 |    0.00 |   -0.05 |

## EVENT: CYCLES

| Symbol                                      | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                 |   28.23 |  100.00 |  +71.77 |
| \[unknown\] (/                              |   64.73 |    0.00 |  -64.73 |
| ggml\_compute\_forward\_mul                 |    2.70 |    0.00 |   -2.70 |
| ggml\_compute\_forward\_sub                 |    1.20 |    0.00 |   -1.20 |
| ggml\_compute\_forward\_add\_non\_quantized |    0.71 |    0.00 |   -0.71 |
| ggml\_compute\_forward\_sqr                 |    0.46 |    0.00 |   -0.46 |
| ggml\_compute\_forward\_repeat\_back        |    0.31 |    0.00 |   -0.31 |
| ggml\_graph\_compute.\_omp\_fn.0            |    0.30 |    0.00 |   -0.30 |
| ggml\_cpu\_init                             |    0.26 |    0.00 |   -0.26 |
| ggml\_compute\_forward\_scale               |    0.18 |    0.00 |   -0.18 |
| ggml\_graph\_compute\_thread.isra.0         |    0.15 |    0.00 |   -0.15 |
| ggml\_blck\_size                            |    0.06 |    0.00 |   -0.06 |
| ggml\_can\_repeat                           |    0.06 |    0.00 |   -0.06 |
| operator delete(void\*, unsigned long) (/   |    0.05 |    0.00 |   -0.05 |
| ggml\_opt\_eval                             |    0.05 |    0.00 |   -0.05 |
| ggml\_compute\_forward\_repeat              |    0.05 |    0.00 |   -0.05 |
| pthread\_mutex\_unlock (/                   |    0.05 |    0.00 |   -0.05 |
| ggml\_is\_empty@plt                         |    0.05 |    0.00 |   -0.05 |
| ggml\_nrows                                 |    0.05 |    0.00 |   -0.05 |
| ggml\_type\_size                            |    0.05 |    0.00 |   -0.05 |


## Cross-table: 18. CT-1__1__1..bin_stest-pad-reflect-1d-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-pad-reflect-1d
# Cross-tables for 1\_\_1\_\_1..bin\_stest-pad-reflect-1d and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-pad-reflect-1d

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   79.56 |   99.26 |  +19.71 |
| \[unknown\] (/ |   20.44 |    0.74 |  -19.71 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   85.68 |  100.00 |  +14.32 |
| \[unknown\] (/ |   14.32 |    0.00 |  -14.32 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   46.22 |  100.00 |  +53.78 |
| ggml\_cpu\_init |   39.57 |    0.00 |  -39.57 |
| \[unknown\] (/  |   14.21 |    0.00 |  -14.21 |


## Cross-table: 19. CT-1__1__1..bin_stest-pool-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-pool
# Cross-tables for 1\_\_1\_\_1..bin\_stest-pool and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-pool

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   54.24 |   98.76 |  +44.53 |
| wcscpy (/      |   22.58 |    0.00 |  -22.58 |
| \[unknown\] (/ |   23.18 |    1.24 |  -21.95 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   16.55 |    0.00 |  -16.55 |
| \[unknown\]    |   83.45 |  100.00 |  +16.55 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   42.74 |  100.00 |  +57.26 |
| ggml\_cpu\_init |   22.38 |    0.00 |  -22.38 |
| tanhf32 (/      |   21.76 |    0.00 |  -21.76 |
| \[unknown\] (/  |   13.13 |    0.00 |  -13.13 |


## Cross-table: 20. CT-1__1__1..bin_stest-quantize-fns-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-quantize-fns
# Cross-tables for 1\_\_1\_\_1..bin\_stest-quantize-fns and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-quantize-fns

## EVENT: BRANCH-MISSES

| Symbol                                               | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                          |    0.14 |  100.00 |  +99.86 |
| \[unknown\] (/                                       |   67.42 |    0.00 |  -67.42 |
| iq2\_compare\_func                                   |   31.16 |    0.00 |  -31.16 |
| memcpy@plt (/                                        |    0.72 |    0.00 |   -0.72 |
| iq3\_compare\_func                                   |    0.47 |    0.00 |   -0.47 |
| iq2xs\_init\_impl.\_omp\_fn.0                        |    0.04 |    0.00 |   -0.04 |
| iq2xs\_init\_impl.\_omp\_fn.1                        |    0.03 |    0.00 |   -0.03 |
| qsort\_r (/                                          |    0.01 |    0.00 |   -0.01 |
| \_pthread\_cleanup\_pop (/                           |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_iq4\_nl\_impl.constprop.0             |    0.00 |    0.00 |   -0.00 |
| make\_qkx2\_quants.constprop.0                       |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_tq1\_0\_ref                           |    0.00 |    0.00 |   -0.00 |
| generate\_data(float, unsigned long, float\*, float) |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_q3\_K\_ref                            |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_q5\_K\_ref                            |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_q8\_K\_ref                            |    0.00 |    0.00 |   -0.00 |

## EVENT: CACHE-MISSES

| Symbol                                                                                                                                                                                                                                                                                                                          | 1_1_1 % | 1_1_2 % | Delta % |
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                                                                                                                                                                                                                                                     |   77.66 |  100.00 |  +22.34 |
| \[unknown\] (/                                                                                                                                                                                                                                                                                                                  |    9.63 |    0.00 |   -9.63 |
| iq2\_compare\_func                                                                                                                                                                                                                                                                                                              |    6.09 |    0.00 |   -6.09 |
| iq2xs\_init\_impl.\_omp\_fn.0                                                                                                                                                                                                                                                                                                   |    4.64 |    0.00 |   -4.64 |
| iq2xs\_init\_impl.\_omp\_fn.1                                                                                                                                                                                                                                                                                                   |    1.03 |    0.00 |   -1.03 |
| .plt                                                                                                                                                                                                                                                                                                                            |    0.44 |    0.00 |   -0.44 |
| qsort\_r (/                                                                                                                                                                                                                                                                                                                     |    0.13 |    0.00 |   -0.13 |
| ggml\_quantize\_init                                                                                                                                                                                                                                                                                                            |    0.11 |    0.00 |   -0.11 |
| cfree (/                                                                                                                                                                                                                                                                                                                        |    0.09 |    0.00 |   -0.09 |
| malloc (/                                                                                                                                                                                                                                                                                                                       |    0.04 |    0.00 |   -0.04 |
| iq3\_compare\_func                                                                                                                                                                                                                                                                                                              |    0.03 |    0.00 |   -0.03 |
| iq3xs\_init\_impl.\_omp\_fn.1                                                                                                                                                                                                                                                                                                   |    0.03 |    0.00 |   -0.03 |
| \_pthread\_cleanup\_push (/                                                                                                                                                                                                                                                                                                     |    0.03 |    0.00 |   -0.03 |
| std::\_Rb\_tree<gguf\_type, std::pair<gguf\_type const, unsigned long>, std::\_Select1st<std::pair<gguf\_type const, unsigned long> >, std::less<gguf\_type>, std::allocator<std::pair<gguf\_type const, unsigned long> > >::\_M\_erase(std::\_Rb\_tree\_node<std::pair<gguf\_type const, unsigned long> >\*) \[clone .isra.0\] |    0.01 |    0.00 |   -0.01 |
| qsort (/                                                                                                                                                                                                                                                                                                                        |    0.01 |    0.00 |   -0.01 |
| \_pthread\_cleanup\_pop (/                                                                                                                                                                                                                                                                                                      |    0.01 |    0.00 |   -0.01 |
| iq3xs\_init\_impl.\_omp\_fn.0                                                                                                                                                                                                                                                                                                   |    0.00 |    0.00 |   -0.00 |
| qsort@plt                                                                                                                                                                                                                                                                                                                       |    0.00 |    0.00 |   -0.00 |
| memcpy@plt (/                                                                                                                                                                                                                                                                                                                   |    0.00 |    0.00 |   -0.00 |
| malloc@plt                                                                                                                                                                                                                                                                                                                      |    0.00 |    0.00 |   -0.00 |

## EVENT: CYCLES

| Symbol                                   | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------------|-------:|-------:|-------:|
| \[unknown\]                              |    0.82 |  100.00 |  +99.18 |
| \[unknown\] (/                           |   48.96 |    0.00 |  -48.96 |
| iq2\_compare\_func                       |   46.40 |    0.00 |  -46.40 |
| iq2xs\_init\_impl.\_omp\_fn.0            |    1.10 |    0.00 |   -1.10 |
| iq2xs\_init\_impl.\_omp\_fn.1            |    1.06 |    0.00 |   -1.06 |
| iq3\_compare\_func                       |    0.77 |    0.00 |   -0.77 |
| memcpy@plt (/                            |    0.65 |    0.00 |   -0.65 |
| iq3xs\_init\_impl.\_omp\_fn.1            |    0.06 |    0.00 |   -0.06 |
| iq3xs\_init\_impl.\_omp\_fn.0            |    0.05 |    0.00 |   -0.05 |
| qsort\_r (/                              |    0.03 |    0.00 |   -0.03 |
| quantize\_row\_iq4\_nl\_impl.constprop.0 |    0.03 |    0.00 |   -0.03 |
| make\_qkx2\_quants.constprop.0           |    0.02 |    0.00 |   -0.02 |
| cfree (/                                 |    0.01 |    0.00 |   -0.01 |
| \_pthread\_cleanup\_pop (/               |    0.01 |    0.00 |   -0.01 |
| qsort (/                                 |    0.00 |    0.00 |   -0.00 |
| ggml\_cpu\_init                          |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_mxfp4\_ref                |    0.00 |    0.00 |   -0.00 |
| quantize\_row\_nvfp4\_ref                |    0.00 |    0.00 |   -0.00 |
| make\_qx\_quants.constprop.0             |    0.00 |    0.00 |   -0.00 |
| \_pthread\_cleanup\_push (/              |    0.00 |    0.00 |   -0.00 |


## Cross-table: 21. CT-1__1__1..bin_stest-quantize-perf-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-quantize-perf
# Cross-tables for 1\_\_1\_\_1..bin\_stest-quantize-perf and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-quantize-perf

## EVENT: BRANCH-MISSES

| Symbol                                                                                  | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                             |    4.61 |   99.91 |  +95.29 |
| make\_qkx2\_quants.constprop.0                                                          |   35.67 |    0.00 |  -35.67 |
| quantize\_row\_iq4\_nl\_impl.constprop.0                                                |   32.20 |    0.00 |  -32.20 |
| \[unknown\] (/                                                                          |    5.49 |    0.09 |   -5.40 |
| make\_qx\_quants.constprop.0                                                            |    3.99 |    0.00 |   -3.99 |
| quantize\_row\_nvfp4\_ref                                                               |    2.45 |    0.00 |   -2.45 |
| quantize\_row\_q4\_K\_ref                                                               |    2.26 |    0.00 |   -2.26 |
| quantize\_row\_q2\_0\_ref                                                               |    1.60 |    0.00 |   -1.60 |
| quantize\_row\_q5\_K\_ref                                                               |    1.51 |    0.00 |   -1.51 |
| quantize\_row\_mxfp4\_ref                                                               |    1.13 |    0.00 |   -1.13 |
| quantize\_row\_q8\_0\_ref                                                               |    1.09 |    0.00 |   -1.09 |
| quantize\_row\_q5\_0\_ref                                                               |    1.05 |    0.00 |   -1.05 |
| quantize\_row\_q3\_K\_ref                                                               |    1.00 |    0.00 |   -1.00 |
| quantize\_row\_q8\_K\_ref                                                               |    0.99 |    0.00 |   -0.99 |
| quantize\_row\_q4\_1\_ref                                                               |    0.96 |    0.00 |   -0.96 |
| quantize\_row\_q4\_0\_ref                                                               |    0.95 |    0.00 |   -0.95 |
| quantize\_row\_tq2\_0\_ref                                                              |    0.79 |    0.00 |   -0.79 |
| \_\_cxa\_finalize (/                                                                    |    0.63 |    0.00 |   -0.63 |
| quantize\_row\_q1\_0\_ref                                                               |    0.48 |    0.00 |   -0.48 |
| benchmark\_function(unsigned long, unsigned long, long, std::function<float ()> const&) |    0.42 |    0.00 |   -0.42 |

## EVENT: CACHE-MISSES

| Symbol                                   | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------------|-------:|-------:|-------:|
| \[unknown\]                              |   78.98 |  100.00 |  +21.02 |
| \[unknown\] (/                           |   12.02 |    0.00 |  -12.02 |
| ggml\_cpu\_fp32\_to\_fp16                |    8.44 |    0.00 |   -8.44 |
| main                                     |    0.35 |    0.00 |   -0.35 |
| quantize\_row\_iq4\_nl\_impl.constprop.0 |    0.21 |    0.00 |   -0.21 |
| \_\_vdso\_clock\_gettime                 |    0.00 |    0.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                                   | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------------|-------:|-------:|-------:|
| \[unknown\]                              |    2.88 |   99.98 |  +97.10 |
| quantize\_row\_iq4\_nl\_impl.constprop.0 |   45.30 |    0.00 |  -45.30 |
| make\_qkx2\_quants.constprop.0           |   29.77 |    0.00 |  -29.77 |
| make\_qx\_quants.constprop.0             |    6.96 |    0.00 |   -6.96 |
| quantize\_row\_mxfp4\_ref                |    2.98 |    0.00 |   -2.98 |
| quantize\_row\_nvfp4\_ref                |    2.97 |    0.00 |   -2.97 |
| quantize\_row\_q8\_K\_ref                |    1.49 |    0.00 |   -1.49 |
| lroundf32 (/                             |    1.00 |    0.00 |   -1.00 |
| quantize\_row\_q4\_K\_ref                |    0.99 |    0.00 |   -0.99 |
| quantize\_row\_q2\_0\_ref                |    0.99 |    0.00 |   -0.99 |
| ggml\_cpu\_init                          |    0.89 |    0.00 |   -0.89 |
| quantize\_row\_tq2\_0\_ref               |    0.50 |    0.00 |   -0.50 |
| quantize\_row\_q3\_K\_ref                |    0.50 |    0.00 |   -0.50 |
| quantize\_row\_q1\_0\_ref                |    0.49 |    0.00 |   -0.49 |
| quantize\_row\_q8\_0\_ref                |    0.48 |    0.00 |   -0.48 |
| quantize\_row\_q5\_1\_ref                |    0.48 |    0.00 |   -0.48 |
| quantize\_row\_q8\_1                     |    0.48 |    0.00 |   -0.48 |
| quantize\_row\_q4\_0\_ref                |    0.47 |    0.00 |   -0.47 |
| \[unknown\] (/                           |    0.38 |    0.02 |   -0.36 |


## Cross-table: 22. CT-1__1__1..bin_stest-rel-pos-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-rel-pos
# Cross-tables for 1\_\_1\_\_1..bin\_stest-rel-pos and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-rel-pos

## EVENT: BRANCH-MISSES

| Symbol                   | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------|-------:|-------:|-------:|
| \[unknown\] (/           |   54.75 |    0.00 |  -54.75 |
| \[unknown\]              |   45.25 |   98.77 |  +53.52 |
| \_\_tunable\_get\_val (/ |    0.00 |    1.23 |   +1.23 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| .plt (/        |   13.11 |    0.00 |  -13.11 |
| \[unknown\] (/ |    0.00 |    6.63 |   +6.63 |
| \[unknown\]    |   86.89 |   93.37 |   +6.48 |

## EVENT: CYCLES

| Symbol          | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |   31.61 |  100.00 |  +68.39 |
| \[unknown\] (/  |   35.50 |    0.00 |  -35.50 |
| ggml\_cpu\_init |   22.20 |    0.00 |  -22.20 |
| expf@plt        |   10.68 |    0.00 |  -10.68 |


## Cross-table: 23. CT-1__1__1..bin_stest-roll-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-roll
# Cross-tables for 1\_\_1\_\_1..bin\_stest-roll and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-roll

## EVENT: BRANCH-MISSES

| Symbol                                                                      | 1_1_1 % | 1_1_2 % | Delta % |
|:----------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                 |   63.53 |   98.38 |  +34.85 |
| \[unknown\] (/                                                              |   16.89 |    1.62 |  -15.27 |
| roll\_reference(float const\*, std::array<long, 4ul>, std::array<int, 4ul>) |   14.71 |    0.00 |  -14.71 |
| ggml\_compute\_forward\_roll                                                |    4.86 |    0.00 |   -4.86 |

## EVENT: CACHE-MISSES

| Symbol                                                                                                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| check\_equal(std::vector<float, std::allocator<float> > const&, std::vector<float, std::allocator<float> > const&) |    5.34 |    0.00 |   -5.34 |
| getppid (/                                                                                                         |    0.00 |    2.86 |   +2.86 |
| \[unknown\]                                                                                                        |   94.66 |   97.14 |   +2.48 |

## EVENT: CYCLES

| Symbol                                                                                                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------------------------------------------------------------------------------------------------|-------:|-------:|-------:|
| \[unknown\]                                                                                                        |   32.97 |   99.66 |  +66.69 |
| \[unknown\] (/                                                                                                     |   17.89 |    0.18 |  -17.71 |
| ggml\_cpu\_init                                                                                                    |   13.13 |    0.00 |  -13.13 |
| roll\_reference(float const\*, std::array<long, 4ul>, std::array<int, 4ul>)                                        |    9.25 |    0.00 |   -9.25 |
| f32\_range(long)                                                                                                   |    9.23 |    0.00 |   -9.23 |
| check\_equal(std::vector<float, std::allocator<float> > const&, std::vector<float, std::allocator<float> > const&) |    4.63 |    0.00 |   -4.63 |
| ggml\_compute\_forward\_roll                                                                                       |    4.60 |    0.00 |   -4.60 |
| tanhf32 (/                                                                                                         |    4.42 |    0.00 |   -4.42 |
| sinf32 (/                                                                                                          |    3.87 |    0.00 |   -3.87 |
| \_\_tunable\_get\_val@plt (/                                                                                       |    0.00 |    0.16 |   +0.16 |


## Cross-table: 24. CT-1__1__1..bin_stest-timestep__embedding-1__1__2.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_sggml-workspace_s1__1__2_sbin_stest-timestep__embedding
# Cross-tables for 1\_\_1\_\_1..bin\_stest-timestep\_\_embedding and 1\_\_1\_\_2..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_sggml-workspace\_s1\_\_1\_\_2\_sbin\_stest-timestep\_\_embedding

## EVENT: BRANCH-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   59.20 |   98.25 |  +39.05 |
| \[unknown\] (/ |   40.80 |    1.75 |  -39.05 |

## EVENT: CACHE-MISSES

| Symbol         | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   74.64 |   84.13 |   +9.49 |
| \[unknown\] (/ |   25.36 |   15.87 |   -9.49 |

## EVENT: CYCLES

| Symbol                           | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------------|-------:|-------:|-------:|
| \[unknown\]                      |   43.26 |  100.00 |  +56.74 |
| ggml\_cpu\_init                  |   28.62 |    0.00 |  -28.62 |
| ggml\_graph\_compute.\_omp\_fn.0 |   10.16 |    0.00 |  -10.16 |
| tanhf32 (/                       |    9.77 |    0.00 |   -9.77 |
| \[unknown\] (/                   |    8.19 |    0.00 |   -8.19 |


## Recorded build/executable measurements (from ggml.json)

The following values are read directly from the tool-owned `/work/ggml-workspace/ggml.json` (no computation):

### Build 1_1_2

| Build | Executable | Run success | real_time | user_time | kernel_time |
|---|---|---|---|---|---|
| 1_1_2 | test-backend-ops | True | 0.60 | 0.58 | 0.01 |
| 1_1_2 | test-opt | True | 3.62 | 3.26 | 0.36 |
| 1_1_2 | test-quantize-fns | True | 730.42 | 729.67 | 0.34 |
| 1_1_2 | test-quantize-perf | True | 4.21 | 4.18 | 0.01 |
| 1_1_2 | test-pool | True | 0.11 | 0.11 | 0.00 |
| 1_1_2 | test-arange | True | 0.14 | 0.13 | 0.00 |
| 1_1_2 | test-timestep_embedding | True | 0.15 | 0.15 | 0.00 |
| 1_1_2 | test-pad-reflect-1d | True | 0.16 | 0.15 | 0.00 |
| 1_1_2 | test-roll | True | 0.24 | 0.22 | 0.01 |
| 1_1_2 | test-conv-transpose | True | 0.12 | 0.11 | 0.00 |
| 1_1_2 | test-conv-transpose-1d | True | 0.30 | 0.28 | 0.01 |
| 1_1_2 | test-dup | True | 0.12 | 0.11 | 0.01 |
| 1_1_2 | test-rel-pos | True | 0.13 | 0.12 | 0.01 |
| 1_1_2 | test-customop | True | 0.15 | 0.14 | 0.01 |
| 1_1_2 | test-conv1d | True | 0.14 | 0.12 | 0.01 |
| 1_1_2 | test-conv1d-dw-c1 | True | 0.13 | 0.12 | 0.00 |
| 1_1_2 | test-conv1d-dw-c2 | True | 0.14 | 0.13 | 0.01 |
| 1_1_2 | test-conv2d | True | 0.15 | 0.13 | 0.00 |
| 1_1_2 | test-conv2d-dw | True | 0.17 | 0.16 | 0.00 |
| 1_1_2 | test-cont | True | 0.14 | 0.13 | 0.00 |
| 1_1_2 | test-interpolate | True | 0.15 | 0.14 | 0.00 |
| 1_1_2 | yolov3-tiny | False |  |  |  |
| 1_1_2 | gpt-2-ctx | False |  |  |  |
| 1_1_2 | gpt-2-alloc | False |  |  |  |
| 1_1_2 | gpt-2-backend | False |  |  |  |
| 1_1_2 | gpt-2-sched | False |  |  |  |
| 1_1_2 | gpt-2-quantize | False |  |  |  |
| 1_1_2 | gpt-2-batched | False |  |  |  |
| 1_1_2 | gpt-j | False |  |  |  |
| 1_1_2 | gpt-j-quantize | False |  |  |  |
| 1_1_2 | mnist-eval | False |  |  |  |
| 1_1_2 | mnist-train | True | 0.04 | 0.03 | 0.00 |
| 1_1_2 | sam | False |  |  |  |
| 1_1_2 | simple-ctx | True | 0.12 | 0.11 | 0.00 |
| 1_1_2 | simple-backend | True | 1.02 | 1.00 | 0.01 |
| 1_1_2 | magika | False |  |  |  |

### Build 1_1_1

| Build | Executable | Run success | real_time | user_time | kernel_time |
|---|---|---|---|---|---|
| 1_1_1 | test-backend-ops | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-opt | True | 1.95 | 1.50 | 0.44 |
| 1_1_1 | test-quantize-fns | True | 42.50 | 42.44 | 0.02 |
| 1_1_1 | test-quantize-perf | True | 0.20 | 0.19 | 0.00 |
| 1_1_1 | test-pool | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-arange | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-timestep_embedding | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-pad-reflect-1d | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-roll | True | 0.02 | 0.01 | 0.01 |
| 1_1_1 | test-conv-transpose | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-conv-transpose-1d | True | 0.01 | 0.01 | 0.00 |
| 1_1_1 | test-dup | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-rel-pos | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-customop | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-conv1d | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-conv1d-dw-c1 | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-conv1d-dw-c2 | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | test-conv2d | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-conv2d-dw | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-cont | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | test-interpolate | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | yolov3-tiny | False |  |  |  |
| 1_1_1 | gpt-2-ctx | False |  |  |  |
| 1_1_1 | gpt-2-alloc | False |  |  |  |
| 1_1_1 | gpt-2-backend | False |  |  |  |
| 1_1_1 | gpt-2-sched | False |  |  |  |
| 1_1_1 | gpt-2-quantize | False |  |  |  |
| 1_1_1 | gpt-2-batched | False |  |  |  |
| 1_1_1 | gpt-j | False |  |  |  |
| 1_1_1 | gpt-j-quantize | False |  |  |  |
| 1_1_1 | mnist-eval | False |  |  |  |
| 1_1_1 | mnist-train | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | sam | False |  |  |  |
| 1_1_1 | simple-ctx | True | 0.00 | 0.00 | 0.00 |
| 1_1_1 | simple-backend | True | 0.01 | 0.00 | 0.00 |
| 1_1_1 | magika | False |  |  |  |
## 6. Notes About Exploration Process

- **Repository resolution.** `https://github.com/ggerganov/ggml` redirects to `https://github.com/ggml-org/ggml`, which was selected (latest commits/tags, most stars, active upstream).
- **`qemu-riscv64-static` was initially missing.** Only `/usr/bin/qemu-riscv64` existed (a static-PIE binary). Installing `qemu-user-static` failed (no installation candidate / no network). A symlink `/usr/bin/qemu-riscv64-static -> /usr/bin/qemu-riscv` was created and verified end-to-end with static and dynamic RISC-V probes. All target executions use `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu`.
- **No SSH client and no SSH keys** are present. All work was local; there are no remote machines and the platform has no `address` field (local execution).
- **`clang` and `valgrind` are not installed.** GCC 15.2 and `perf` 7.0.14 are available.
- **The provided riscv recipe initially failed** (detailed in Section 3): CMake did not enter cross mode, host `-march=native` leaked, `tests/CMakeLists.txt` injected host x86 flags (upstream issue #1388), `MATH_LIBRARY` was NOTFOUND, and `-static` conflicted with shared libraries. These were fixed by updating recipe 2's `config_flags` and applying a **2-line source patch** to `ggml/tests/CMakeLists.txt` (`elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "riscv64")`); the original file is backed up at `/work/ggml-workspace/tests-CMakeLists.txt.orig`. This patch is required to cross-build the tests; the library itself builds without it. The canonical `input.yml` was updated to the verified recipe and re-validated (`[Config][✓] /work/input.yml is correct!`).
- **MCP build tool cwd quirk:** an early build via the MCP tool placed artifacts in `/work/1_1_1` instead of the workspace; that directory was removed and the canonical build was redone via the CLI with cwd `/work/ggml-workspace`.
- **`perf_event_paranoid` is -1** (full access), but `/proc/sys` is read-only, so amphimixis's attempt to set it logged a harmless "Read-only file system" error during builds; it was not needed.
- **CPU frequency governor is `powersave`** and observed at ~1.3 GHz (drifting), not the 3.6 GHz maximum; the host was otherwise lightly loaded (loadavg ~1.5–1.9 on 8 CPUs). No frequency matching or warmup was performed.
- **QEMU/emulation caveat:** the target runs under QEMU user-mode. All target timings include emulation overhead and `perf` attributes to the emulator host process, so target IPC/cache/branch/topdown counters and hotspots are emulator-side and are **not** guest micro-architectural data. Guest function-level hotspots are NOT AVAILABLE.
- **12 example executables** (`yolov3-tiny`, `gpt-2-ctx`, `gpt-2-alloc`, `gpt-2-backend`, `gpt-2-sched`, `gpt-2-quantize`, `gpt-2-batched`, `gpt-j`, `gpt-j-quantize`, `mnist-eval`, `sam`, `magika`) failed the smoke test on both platforms because they require model/data files or CLI arguments; they were skipped by the profiler and their metrics are NOT AVAILABLE.
- Small tests have x86 real times at or below the 0.01 s `/bin/time` resolution and are only qualitative.
- The provided `input.yml` now lists all **36 built executables explicitly in each build entry** (no auto-detect); the riscv build entries are absolute qemu command lines. It validates successfully. Optimized builds `1_1_3`/`1_1_4` were produced with a separate, validated `/work/ggml-workspace/input.opt.yml` so that the provided `input.yml` structure was preserved.

## 7. Migration Readiness Summary

| Readiness criterion | Status | Evidence |
|---|---|---|
| Builds on reference platform | ✅ YES | x86_64 build `1_1_1`: 36/36 executables built; `ctest` 21/21 passed |
| Tests pass on reference platform | ✅ YES | 21/21 CTest tests passed (8.56 s) |
| Builds on target platform | ✅ YES (with fixes) | riscv64 build `1_1_2`: 36/36 built after recipe-2 correction + 2-line `tests/CMakeLists.txt` patch |
| Tests pass on target platform | ✅ YES (under QEMU) | 21/21 test binaries pass under `qemu-riscv64-static -L /usr/riscv64-linux-gnu` |
| Zero external dependencies | ✅ YES (core) | Core links only pthreads/libm/libdl; all native on riscv64 |
| No hand-written intrinsics | ⚠️ NO (but arch-guarded) | x86/ARM/WASM/PPC/s390/LoongArch intrinsics are guarded; RISC-V has its own in-tree RVV kernels (`arch/riscv/`). No foreign-ISA intrinsics leak into the riscv path. |
| Alignment safe | ✅ YES | `GGML_MEM_ALIGN` / `ggml_aligned_malloc`; no arch-specific alignment assumptions found |
| Exceptions handled | ➖ N/A | Core is C11 and does not use C++ exceptions; C++ only in tests/examples/optional backends |
| Auto-vectorization | ✅ YES (both) | x86 emits AVX-512+FMA+VNNI; riscv emits RVV 1.0; no manual porting required to vectorize |

**Migration Verdict: MINOR CONCERNS**

ggml is upstream-maintained for RISC-V and the core library cross-compiles and passes its 21 tests under QEMU with only configuration changes and a 2-line test-CMake patch. There are no blocking dependencies, no foreign-ISA intrinsics in the riscv path, and auto-vectorization works (RVV 1.0). The concerns are: (a) a provided-config cross-build gap (issue #1388) requiring a `tests/CMakeLists.txt` patch; (b) the target requires the V extension (`Zvl128b` minimum, plus Zfh/Zvfh) — base RV64GC hardware will not run these binaries; (c) target performance could not be validly measured because QEMU user-mode counters are emulator-side, so native-hardware validation is still required; (d) the `__riscv_v_intrinsic` gate is a latent Clang/`rv64gc` hazard (issue #1535, unmerged PR #1571). Optimizing to effective `-O3` gave a measured ~5–10% win on the quantize workload on both platforms but was neutral-to-negative for `test-opt` under QEMU.

### Required Actions

1. Keep `-DCMAKE_SYSTEM_NAME=Linux -DCMAKE_SYSTEM_PROCESSOR=riscv64 -DGGML_NATIVE=OFF -DBUILD_SHARED_LIBS=OFF -DGGML_STATIC=ON` plus the find-root/MATH_LIBRARY settings for any riscv64 cross build; do not let host x86 flags leak (revert the `tests/CMakeLists.txt` patch only after upstream fixes issue #1388).
2. Use a compiler that defines `__riscv_v` with the V extension enabled, or apply upstream PR #1571 / disable RVV, to avoid the `__riscv_v_intrinsic` compile trap on Clang rv64gc.
3. Verify target hardware supports RVV 1.0 with VLEN ≥ 128 (`Zvl128b`) and Zfh/Zvfh; otherwise build with `GGML_RVV=OFF`.
4. Make the optimization level explicit (e.g. `CMAKE_BUILD_TYPE=Release`) so effective flags are `-O3`; strip target artifacts for deployment.
5. Re-run the performance comparison on real RISC-V hardware (or with a corrected harness: `performance` governor, `OMP_NUM_THREADS`/multi-core pinning, warmup + repeats, guest-capable counters) — current target numbers are QEMU-emulation artifacts and must not be used for capacity planning.
6. Long-term: add RISC-V CI and exercise GEMM/inference kernels (`test-backend-ops` skips the CPU backend; the profiled tests do not cover `arch/riscv/quants.c` `ggml_vec_dot_*`).
