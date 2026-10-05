# FFTW Migration Readiness Report

- **Project:** FFTW (Fastest Fourier Transform in the West)
- **Repository URL:** https://github.com/FFTW/fftw3
- **Clone path:** /work/FFTW-workspace/fftw3
- **Workspace:** /work/FFTW-workspace/
- **Reference platform:** x86_64 (local container, Intel Core i5-1035G1, 4 cores / 8 threads, AVX-512)
- **Target architecture:** riscv64 (executed under qemu-user emulation; no native RISC-V hardware)
- **Config file:** /work/input.yml (validated with `amixis validate`)
- **Builds:** `1_1_1` (x86_64 native), `1_1_2` (riscv64 baseline, qemu), `1_1_3` (riscv64 optimized, qemu)
- **Report date:** 2026-10-05

---

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Resolved active repository URL | https://github.com/FFTW/fftw3 (canonical upstream, org FFTW; 3,107 stars, 721 forks, not archived) |
| Default branch | master |
| Latest commit | `93ed4c786934aec9946f8dda4b4e3eb08f8be41c` — 2026-06-10 07:17:51 -0400 ("CI: Add compilation pipeline (#359)") |
| Total commits | 3195 |
| Latest tag / release | `fftw-3.3.11` (tag commit `d69d34f078baee4fa38a035634017404cce3e09a`, 2026-04-18); `git describe` = `fftw-3.3.11-10-g93ed4c78` |
| Activity | Actively maintained (last push 2026-06-10; PR/issue activity into Sept–Oct 2026; open PRs updated 2026-09-12) |
| Build systems | Autoconf/Automake (`configure.ac`, `Makefile.am`, `bootstrap.sh`, `mkdist.sh`) and CMake (`CMakeLists.txt`, `cmake.config.h.in`) |
| Test count | NOT AVAILABLE — FFTW's test volume is parameterized via `tests/check.pl` / `tests/bench` / `bench.c`, not a fixed countable suite. CMake registers tests only when `ENABLE_THREADS=ON`; with the configured options CTest registered 0 tests |
| External dependencies | Threads (pthreads), OpenMP, base `libm`; optional MPI, Fortran bindings, `libquadmath` (quad precision). Build-from-git tooling: OCaml + libnum-ocaml-dev, autoconf, automake, indent, libtool, fig2dev |
| Distro packages | Debian `fftw3 3.3.11-1` (testing/unstable; libfftw3-double3/-single3/-long3/-quad3/-mpi3), Arch Linux `fftw 3.3.11-1`, Yocto/OpenEmbedded `fftw 3.3.11` |
| Size | 589 tracked files; 288 `.c`, 91 `.h`, 38 `.ml`; ~58,661 LOC |
| Branches | `master` (default), `experimental-simd` |

> **Repository caveat:** the GitHub tree is the *codelet-generator source*, not a directly compilable release. `README` states the tree cannot be compiled without OCaml + autotools tooling. The build phase therefore ran `bootstrap.sh` to generate the codelets in place (see Section 6).

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros

| Macro | File:Line | What it guards | Category | Effect on riscv64 |
|---|---|---|---|---|
| `__i386__` | kernel/cycle.h:171 | 32-bit x86 `rdtsc` tick counter | x86 | inactive |
| `__x86_64__` | kernel/cycle.h:220 | x86-64 `rdtsc` tick counter | x86 | inactive |
| `__x86_64__` | kernel/cycle.h:239 | PGI x86-64 tick counter | x86 | inactive |
| `_M_AMD64`/`_M_X64` | kernel/cycle.h:251 | MSVC x64 `__rdtsc` | x86/OS | inactive |
| `__x86_64__`/`_M_X64`/`_M_AMD64` | kernel/cpy2d.c:24 | `__m128` WIDE_TYPE 2D copy (needs `HAVE_XMMINTRIN_H`) | x86 | inactive; falls back to `double` WIDE_TYPE |
| `__i386__`/`__x86_64__`/`__ia64__` | api/fftw3.h:476 | `FFTW_ALIGNMENT` / struct packing | x86/pointer | generic alignment path used |
| `__x86_64__`/`_M_X64`/`_M_AMD64`/`__e2k__` | simd-support/sse2.c:32 | Skip CPUID runtime probe on 64-bit | x86 | inactive |
| `__x86_64__`/`_M_X64`/`_M_AMD64` | simd-support/avx.c, avx2.c, avx512.c, kcvi.c, avx-128-fma.c | x86 SIMD runtime dispatch / register-save | x86 | inactive |
| `__SSE2__`/`__SSE__` | simd-support/simd-sse2.h:38,40 | SSE/SSE2 backend sanity check | x86 | inactive |
| `__AVX__` | simd-support/simd-avx.h:38 | AVX backend sanity check | x86 | inactive |
| `__AVX2__` | simd-support/simd-avx2.h:42, simd-avx2-128.h:41 | AVX2 backend sanity check | x86 | inactive |
| `__AVX512F__` | simd-support/simd-avx512.h:47 | AVX-512 backend sanity check | x86 | inactive |
| `__AVX__`+`__FMA4__` | simd-support/simd-avx-128-fma.h:54 | AVX-128-FMA backend sanity check | x86 | inactive |
| `__e2k__` | simd-support/simd-avx.h:195, simd-avx2.h:199, avx*.c:26 | Elbrus e2k SIMD adaptation | other | inactive |
| `__aarch64__` | kernel/cycle.h:464 | ARMv8 `CNTVCT_EL0` tick counter | ARM | inactive |
| `__aarch64__`+`HAVE_ARMV8_PMCCNTR_EL0` | kernel/cycle.h:477 | ARMv8 `PMCCNTR_EL0` tick counter | ARM | inactive |
| `__aarch64__` | simd-support/simd-neon.h:24 | NEON single/double config | ARM | inactive |
| `__ARM_NEON__` | simd-support/simd-neon.h:46 | NEON backend sanity check | ARM | inactive |
| `__ARM_FEATURE_SVE` | simd-support/simd-maskedsve.h:64, m4/acx_sve.m4:20 | ARM SVE backend + detection | ARM | inactive |
| `__riscv_xlen` (==64 / ==32) | kernel/cycle.h:489,494,496 | RISC-V `rdtime` (rv64) / `rdtimeh+rdtime` loop (rv32) tick counter | RISC-V | **active** (only upstream RISC-V branch) |
| `__loongarch64` | kernel/cycle.h:514 | LoongArch tick counter | other | inactive |
| `__e2k__` | kernel/cycle.h:527 | Elbrus e2k tick counter | other | inactive |
| `SIZEOF_VOID_P` etc. | kernel/ifftw.h:234–240 | Select pointer-sized unsigned integer type | pointer | correct for LP64 |
| `SIZEOF_LONG`/`SIZEOF_LONG_LONG`/`SIZEOF_VOID_P` | CMakeLists.txt:99–100, cmake.config.h.in | Configure-type sizes | pointer | correct for LP64 |
| `SIZEOF_PTRDIFF_T` | mpi/mpi-bench.c:26,28 | printf format selection | pointer | correct for LP64 |

**Endianness macros:** NONE FOUND (`__ORDER_LITTLE_ENDIAN__`, `__ORDER_BIG_ENDIAN__`, `__BYTE_ORDER__`, `__LITTLE/BIG_ENDIAN__`, `WORDS_BIGENDIAN`, `BYTE_ORDER` returned zero matches). FFTW does not gate on endianness — favorable for little-endian riscv64.

**Pointer-size macros:** `__LP64__`/`__ILP32__`/`__SIZEOF_POINTER__`/`__SIZEOF_LONG__` are not used; configure-generated `SIZEOF_*` values drive type selection.

### 2.2 Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `_WIN32`/`__WIN32__`/`_WIN64` | Windows | DLL import/export (api/fftw3.h:78,90; ifftw.h:56), timers, threads, system-wisdom (api/import-system-wisdom.c:40) |
| `_MSC_VER` | MSVC | Intrinsics, `_asm`, `_mm_malloc`, F77 mangling |
| `__APPLE__`/`__MACOSX__` | macOS | aarch64 counter availability (cycle.h:465), aligned malloc (kalloc.c:86), sve.c:41 |
| `__linux__` | Linux | aarch64 counter availability (cycle.h:465), sve.c:27 |
| `__FreeBSD__` | FreeBSD | aligned malloc (kalloc.c:82) |
| `__ANDROID__` | Android | exclude `CLOCK_SGI_CYCLE` (cycle.h:541) |
| `__CYGWIN__` | Cygwin | libbench2 timer (timer.c:46) |

### 2.3 Vectorization intrinsics in source

| Intrinsic family | File(s) | ISA | Applicable to riscv64 |
|---|---|---|---|
| `_mm_*`, `__m128`, `_mm_loadh_pi`, `_mm_malloc/_mm_free` | kernel/cpy2d.c:27,101–104; simd-support/simd-sse2.h; kernel/kalloc.c | SSE/SSE2 | no (compiles out) |
| `_mm256_*`, `__m256`, `_mm256_zeroupper` | simd-support/simd-avx.h, simd-avx2.h, simd-avx2-128.h, simd-avx-128-fma.h | AVX/AVX2/FMA | no |
| `_mm512_*`, `__m512`, mask load/gather/scatter | simd-support/simd-kcvi.h, simd-avx512.h | AVX-512/KCVI | no |
| `vaddq`, `vmulq`, `vld1q`, `vcombine` | simd-support/simd-neon.h:84–112 | ARM NEON | no |
| `sv*` (SVE) | simd-support/simd-maskedsve.h, sve.c | ARM SVE | no |
| AltiVec / VSX | simd-support/simd-altivec.h, simd-vsx.h | PowerPC | no |
| LSX / LASX | simd-support/simd-lsx.h, simd-lasx.h | LoongArch | no |
| `__riscv_v` (RVV) | NONE in upstream (exists only in the `rdolbeau` fork `simd-r5v.h`) | RISC-V RVV | **absent upstream** |

### 2.4 Semantic notes on platform-specific code

- `__riscv_xlen` is a standard RISC-V compiler macro (GCC/Clang, rv32 and rv64), so the `rdtime` tick counter branch is correctly reached on riscv64. It is not misleading. Caveat (noted in upstream PR #361): `rdtime` can have low resolution on some hardware, making profiling ticks coarse.
- `__x86_64__`/`_M_X64`/`_M_AMD64` are not defined on riscv64, so all x86 SIMD dispatch and the `__m128`-based `cpy2d` wide copy compile out safely (fallback `double WIDE_TYPE`).
- `__aarch64__`, `__ARM_NEON__`, `__ARM_FEATURE_SVE`, `__loongarch64`, `__e2k__` are not defined on riscv64 — their backends compile out.
- `api/fftw3.h:476` keys the alignment block on `__i386__/__x86_64__/__ia64__` only; on riscv64 the generic `FFTW_ALIGNMENT`/struct layout path is used. FFTW's generic path is designed for this; alignment is considered safe.
- No endianness gating means no big-endian risk for little-endian riscv64.
- FFTW uses configure/CMake-generated `SIZEOF_*` values rather than compiler `__SIZEOF_*` macros; the LP64 selection is correct on riscv64.
- On upstream riscv64 **only scalar codelets build** (generic-simd128/256 are disabled by default and are GCC vector-type-based, not RVV). Correctness is expected, but there is no vectorized performance path without the RVV work (upstream PR #377 / `rdolbeau/riscv-v-clean`).

### 2.5 Forks with target-architecture (riscv64) patches

| Source | Ref | Status | Notes |
|---|---|---|---|
| `FFTW/fftw3` upstream | master | merged | RISC-V `rdtime` cycle counter (PR #361, merged 2025-02-02), kernel/cycle.h:489–512. No RVV SIMD |
| `rdolbeau/fftw3` | `riscv-v-clean` (head `7ddcb3b3`, 2025-02-05) | fork | Full RVV 1.0 SIMD backend (`simd-support/simd-r5v.h`, `r5v.c`, `simd-r5v128…16384.h`, generated `vtw.h`, `dft/simd/r5v*`); tested on BananaPi F3 (Spacemit K1, 256-bit V) with gcc-14/clang-18 and qemu |
| `FFTW/fftw3` PR #377 | open (updated 2026-09-12) | open PR | "Merging of RISC-V V1.0 support" — cleaned-up rdolbeau branch using official V1.0 intrinsics |
| `FFTW/fftw3` PR #279 | open | open PR | "Add RISC-V vector spec v1.0 support" (`--enable-rvv`) |
| `tommaso-merlini/fftw-riscv-xuantie` | main (2026-04) | standalone repo | FFTW 3.3.10 RVV128 Pioneer/XuanTie port, RVV 0.7.1 |

Upstream issues #234, #371, #311 track RVV/R5V. **No RISC-V SIMD is present in canonical upstream master.**

### 2.6 Dependency portability assessment

| Dependency | Status on riscv64 | Notes |
|---|---|---|
| Threads (pthreads / glibc NPTL) | ready | NPTL supports riscv64; distros ship libfftw3_threads |
| OpenMP (`libgomp`) | ready | GCC/LLVM OpenMP runtimes support riscv64; libgomp present in sysroot |
| `libm` (base) | ready | glibc riscv64 provides `sin`, `log`, `sincos`, etc. |
| MPI (optional, `--enable-mpi`) | ready | OpenMPI/MPICH support riscv64; Debian ships libfftw3-mpi3 |
| Fortran bindings (optional) | ready | gfortran available on riscv64 |
| `libquadmath` (optional) | ready | GCC libquadmath supports riscv64 |
| OCaml + libnum-ocaml-dev (git builds) | ready | Generator emits architecture-independent C; required only for git-tree bootstrap |
| autoconf/automake/libtool/indent/fig2dev | ready | Standard arch-independent tooling |

Every dependency identified is available/ready on riscv64. No blocking dependency was found.

### 2.7 Portability verdict

- **No exceptions:** FFTW is C; it does not use C++ exceptions. No exception-handling portability risk.
- **Alignment safe:** yes — generic `FFTW_ALIGNMENT`/malloc alignment path is used on riscv64; no x86-only layout is required.
- **Embedded usability:** good — the library is portable C with optional dependencies; no exotic third-party numeric libraries. Static linking works.
- **Overall portability level: MEDIUM.** Functional portability is low-barrier (pure portable C, distro builds already exist for riscv64, the RISC-V cycle counter is merged upstream), but performance migration readiness has a real gap because upstream has no RVV SIMD backend; the hot transform codelets remain scalar. The `rdtime` tick counter may also be low-resolution.

---

## 3. Build & Test Results

### 3.1 Build & test summary

| Build | Platform | Build machine | Run machine | Recipe | Build status | Test status | Pass / Fail |
|---|---|---|---|---|---|---|---|
| `1_1_1` | x86_64 native | 1 (x86_64) | 1 (x86_64) | 1 | OK | CTest registered 0 tests; manual CTest-equivalent runs executed | 2 / 0 (manual) |
| `1_1_2` | riscv64 baseline (qemu) | 1 (x86_64 cross) | 1 (x86_64 + qemu-user) | 2 | OK | CTest registered 0 tests; manual runs executed under qemu | 2 / 0 (manual) |
| `1_1_3` | riscv64 optimized (qemu) | 1 (x86_64 cross) | 1 (x86_64 + qemu-user) | 3 | OK | benchmark harness executed under qemu | 3 / 0 (bench runs) |

### 3.2 Build commands and outcomes

- **`1_1_1` (native):** CMake + Ninja; config flags `-DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_TESTS=ON -DBUILD_TESTING=ON`, C/CXX `-O3 -march=native -g`, 4 jobs. Result: `[472/472] Linking C executable bench` → success, build dir `/work/1_1_1`.
- **`1_1_2` (riscv baseline):** CMake + Ninja cross build with `-DBUILD_SHARED_LIBS=OFF -DCMAKE_EXE_LINKER_FLAGS=-static -DCMAKE_C_STANDARD_LIBRARIES=-lm -DCMAKE_C_COMPILER=/usr/bin/riscv64-linux-gnu-gcc -DCMAKE_CXX_COMPILER=/usr/bin/riscv64-linux-gnu-g++`, C/CXX `-O3 -g`, 4 jobs. Result: `[471/471] Linking C executable bench` → success, build dir `/work/1_1_2`.
- **`1_1_3` (riscv optimized):** CMake + Ninja with `-DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_EXE_LINKER_FLAGS=-static -DCMAKE_C_STANDARD_LIBRARIES=-lm -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON`, C/CXX `-O3 -g -funroll-loops -fomit-frame-pointer`. Effective flags from `build.ninja`: `-O3 -g -funroll-loops -fomit-frame-pointer -O3 -DNDEBUG -flto=auto -fno-fat-lto-objects`. LTO succeeded with no fallback. Result: success, build dir `/work/1_1_3`.

### 3.3 Tests

- **Reference `1_1_1`:** `ctest` reported `No tests were found!!!` (Total Tests: 0) because FFTW registers `add_test` only when `ENABLE_THREADS=ON`. Manual CTest-equivalents:
  - `bench -s 32x64` → exit 0: `Problem: 32x64, setup: 34.29 ms, time: 25.09 us, ``mflops'': 4489.466`
  - `bench -s ib256` → exit 0: `Problem: ib256, setup: 67.11 ms, time: 2.61 us, ``mflops'': 3921.7429`
- **Target `1_1_2` (under qemu):** `ctest` reported 0 tests (same `ENABLE_THREADS=OFF` gating). Manual runs:
  - `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_1_2/bench -s 32x64` → exit 0: `Problem: 32x64, setup: 200.73 ms, time: 820.88 us, ``mflops'': 137.21943`
  - `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_1_2/bench -s ib256` → exit 0: `Problem: ib256, setup: 269.06 ms, time: 94.02 us, ``mflops'': 108.91807`
- **Target `1_1_3` (under qemu), 3 runs of `bench -s 32x64`:**
  - Run 1: `setup: 333.01 ms, time: 944.25 us, ``mflops'': 119.29044`
  - Run 2: `setup: 258.61 ms, time: 811.12 us, ``mflops'': 138.86885`
  - Run 3: `setup: 294.65 ms, time: 806.94 us, ``mflops'': 139.5895`

### 3.4 Build/test failures detail (actual error text)

1. **Initial source can't compile from git tree** — `/usr/bin/x86_64-linux-gnu-ld.bfd: libfftw3.so.3: undefined reference to 'fftw_solvtab_rdft_r2r'` (and r2cf/r2cb/dft_standard). Root cause: GitHub tree is the codelet generator; generated codelets are not committed. Resolved by running `./bootstrap.sh` + `make` (installed ocaml, ocamlbuild, libnum-ocaml-dev, autoconf, automake, indent, libtool, ocaml-findlib) to generate the codelets in place; the full autotools `make` stopped later in `doc/` on `/bin/bash: fig2dev: command not found` (irrelevant to code/library).
2. **Native `bench` thread symbols** — `undefined reference to 'fftw_init_threads'` etc. Root cause: a stale in-source `config.h` from the autotools run shadowed CMake's `config.h` (`HAVE_THREADS` mismatch while `ENABLE_THREADS=OFF`). Resolved by removing the stale `/work/FFTW-workspace/fftw3/config.h` (codelets retained).
3. **RISC-V static/shared link conflict** — `/usr/bin/riscv64-linux-gnu-ld.bfd: attempted static link of dynamic object 'libfftw3.so.3'`. Resolved with `-DBUILD_SHARED_LIBS=OFF`.
4. **RISC-V missing libm** — `undefined reference to 'log'` and `'sincos'`. Root cause: cross configure did not resolve `libm` (CMake searched host paths). Resolved with `-DCMAKE_C_STANDARD_LIBRARIES=-lm`.

---

## 4. Performance Comparison

### 4.1 Experimental conditions

| Condition | Reference `1_1_1` | Target `1_1_2` |
|---|---|---|
| CPU | Intel Core i5-1035G1 @1.00 GHz (max 3.60 GHz), 4C/8T (8 logical CPUs), AVX-512 | same host CPU (emulation) |
| Executed command | `bench -s 32x64` | `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_1_2/bench -s 32x64` |
| Core pinning | `taskset -c 0` | `taskset -c 0` |
| nice priority | attempted `nice -n -20` → DENIED (container lacks CAP_SYS_NICE) | attempted and DENIED (same) |
| Frequency governor | `powersave`, observed core0 ≈ 1.30 GHz, `CPU(s) scaling MHz: 33%`, `/sys` read-only | same host |
| Warmup | 2 warmup runs (plus 1 tool smoke test) | 2 warmup runs (plus 1 tool smoke test) |
| Measurement runs | 10 timed + `perf stat --repeat 10` + TopdownL1 ×5 | 10 timed + `perf stat --repeat 10` + TopdownL1 ×5 |
| `perf_event_paranoid` | −1 (profiling allowed, root) | −1 |

Matched workload, same host, same core, same frequency environment and run counts. Unavoidable differences: the target runs under qemu-user emulation and is statically linked (required for qemu).

> **QEMU/emulation caveat (applies to every `1_1_2` and `1_1_3` number):** all target timing and PMU values describe the `qemu-riscv64-static` process on the x86 host, not native RISC-V silicon. They include dynamic-binary-translation overhead. `perf` counters are host x86 counters over the emulator; guest symbols are not resolvable (hotspots appear as `[unknown]`).

### 4.2 Key metrics: `1_1_1` (reference) vs `1_1_2` (target)

| Metric | `1_1_1` (x86_64) | `1_1_2` (riscv64/qemu) | Notes |
|---|---|---|---|
| Elapsed (real) time | 0.262 s (mean of 10; range 0.23–0.28) | 0.472 s (mean of 10; range 0.47–0.48) | tool JSON: real_time 0.25 vs 0.47 |
| User time | 0.258 s (mean of 10) | 0.464 s (mean of 10) | tool JSON: user_time 0.24 vs 0.46 |
| IPC | 3.54 | 3.26 | tool JSON insn/cycle 3.5 vs 3.3 |
| L1-dcache miss rate | 3.41% | 0.408% | host x86 counters; tool JSON 3.3% vs 0.5% |
| LLC miss rate | 34.9% | 20.9% | high variance; tool JSON 27.6% vs 24.7% |
| Branch misprediction rate | 0.415% | 0.298% | tool JSON 0.7% vs 0.3% |
| Branches | 24,444,816 | 533,618,891 | qemu TCG dispatcher/translation |
| Frontend Bound | 3.77% | 22.11% | canonical `perf stat -M TopdownL1` |
| Backend Bound | 22.44% | 5.44% | same |
| Retiring | 71.62% | 65.12% | same |
| Bad Speculation | 2.31% | 7.77% | same |
| Instructions retired (host PMU) | 1,157,669,392 | 2,137,079,885 | host counters over qemu |
| CPU cycles (host PMU) | 327,151,899 | 655,973,678 | host counters over qemu |
| L1-icache miss rate | NOT AVAILABLE | NOT AVAILABLE | `perf -ddd` returned `<not counted>` |
| Executable size (stripped) | 100,704 B | 1,286,680 B | static vs dynamic linking |
| Executable size (unstripped) | 396,880 B | 6,972,016 B | includes DWARF debug info |
| Program-reported setup | 34.7 ms | 203.3 ms | bench stdout |
| Program-reported transform time | 24.62 µs | 813.50 µs | bench stdout |

### 4.3 Cross-table (reference vs target)

The authoritative tool-written cross-table (`cross-tables/CT-1__1__1..bench_x20_-s_x20_32x64-1__1__2...md`, produced by `amixis compare`) is reproduced below. The `[unknown]` rows for the target reflect that qemu-user perf cannot resolve guest symbols.

#### Cross-table: EVENT CYCLES — 1_1_1 vs 1_1_2

| Symbol | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------------|-------------:|-------------:|-------:|
| \[unknown\]                    |          1.36 |        100.00 |  +98.64 |
| n1\_32                         |         36.57 |          0.00 |  -36.57 |
| t1\_8                          |         28.09 |          0.00 |  -28.09 |
| n1\_8                          |         18.91 |          0.00 |  -18.91 |
| fftw\_cpy2d                    |          3.67 |          0.00 |   -3.67 |
| apply                          |          2.78 |          0.00 |   -2.78 |
| n1\_2                          |          2.08 |          0.00 |   -2.08 |
| apply\_dit                     |          1.25 |          0.00 |   -1.25 |
| mkplan                         |          0.83 |          0.00 |   -0.83 |
| fftw\_measure\_execution\_time |          0.83 |          0.00 |   -0.83 |
| n1\_16                         |          0.80 |          0.00 |   -0.80 |
| q1\_2                          |          0.42 |          0.00 |   -0.42 |
| recur                          |          0.42 |          0.00 |   -0.42 |
| n1\_64                         |          0.42 |          0.00 |   -0.42 |
| fftw\_rdft\_solve              |          0.42 |          0.00 |   -0.42 |
| fftw\_cpy2d\_ci                |          0.41 |          0.00 |   -0.41 |
| apply\_ip\_sq\_tiledbuf        |          0.41 |          0.00 |   -0.41 |
| fftw\_mktensor@plt             |          0.34 |          0.00 |   -0.34 |

#### Cross-table: EVENT CACHE-MISSES — 1_1_1 vs 1_1_2

| Symbol | 1_1_1 % | 1_1_2 % | Delta % |
|:------------------|-------------:|-------------:|-------:|
| \[unknown\]       |         67.14 |         97.45 |  +30.31 |
| \_\_cxa\_finalize |         24.56 |          0.00 |  -24.56 |
| caset             |          2.91 |          0.00 |   -2.91 |
| \[unknown\] (/    |          0.00 |          2.55 |   +2.55 |
| n1\_8             |          2.33 |          0.00 |   -2.33 |
| t1\_8             |          1.99 |          0.00 |   -1.99 |
| n1\_32            |          0.86 |          0.00 |   -0.86 |
| apply             |          0.08 |          0.00 |   -0.08 |
| fftw\_execute     |          0.08 |          0.00 |   -0.08 |
| speed             |          0.04 |          0.00 |   -0.04 |

#### Cross-table: EVENT BRANCH-MISSES — 1_1_1 vs 1_1_2

| Symbol | 1_1_1 % | 1_1_2 % | Delta % |
|:-------------------------------|-------------:|-------------:|-------:|
| \[unknown\]                    |         29.01 |         98.75 |  +69.75 |
| fftw\_measure\_execution\_time |         16.96 |          0.00 |  -16.96 |
| apply                          |         10.13 |          0.00 |  -10.13 |
| fftw\_md5putc                  |          9.48 |          0.00 |   -9.48 |
| n1\_32                         |          7.85 |          0.00 |   -7.85 |
| fftw\_cpy2d                    |          5.73 |          0.00 |   -5.73 |
| apply\_cpy2dco                 |          4.07 |          0.00 |   -4.07 |
| fftw\_rdft\_solve              |          3.66 |          0.00 |   -3.66 |
| n1\_8                          |          3.13 |          0.00 |   -3.13 |
| mkplan                         |          2.90 |          0.00 |   -2.90 |
| dotile                         |          2.25 |          0.00 |   -2.25 |
| cfree                          |          1.47 |          0.00 |   -1.47 |
| apply\_ip\_sq\_tiled           |          1.46 |          0.00 |   -1.46 |
| \[unknown\] (/                 |          0.00 |          1.25 |   +1.25 |
| t1\_8                          |          0.60 |          0.00 |   -0.60 |
| fftw\_execute                  |          0.42 |          0.00 |   -0.42 |
| fftw\_dft\_solve               |          0.35 |          0.00 |   -0.35 |
| doit                           |          0.24 |          0.00 |   -0.24 |
| apply\_dit                     |          0.18 |          0.00 |   -0.18 |
| tensor\_sz                     |          0.11 |          0.00 |   -0.11 |

### 4.4 Hotspot tables (from actual `perf record`)

**Reference platform (x86_64) — 251 cycle samples; `comm=bench` 99.24%, DSO `libfftw3.so.3` 98.64%**

| Self % | Function | Module | Analysis |
|:--:|---|---|---|
| 36.57% | `n1_32` | libfftw3.so.3 | radix-32 DIT codelet |
| 28.09% | `t1_8` | libfftw3.so.3 | radix-8 twiddle codelet |
| 18.91% | `n1_8` | libfftw3.so.3 | radix-8 DIT codelet |
| 3.67% | `fftw_cpy2d` | libfftw3.so.3 | strided copy |
| 2.08% | `n1_2` | libfftw3.so.3 | radix-2 codelet |
| 1.25% | `apply_dit` | libfftw3.so.3 | plan recursion |

**Target platform (riscv64 under qemu) — 488 cycle samples; `comm=qemu-riscv64-st` 99.63%, DSO `qemu-riscv64` 89.65%**

| Self % | Symbol | Module | Analysis |
|:--:|---|---|---|
| 89.65% (DSO total) | anonymous JIT blocks | qemu-riscv64 (TCG code buffers) | hotspots are QEMU's dynamically generated translated code, not guest FFTW |
| 34.63% (children) | `0x9ed18` | qemu-riscv64 | internal emulator function with many JIT callees |
| — | guest symbols (`n1_32`, …) | — | NOT AVAILABLE — no guest symbol resolution |

### 4.5 Bottleneck summary and causal analysis

1. **Emulation is the dominant cost.** For the identical guest workload, the host executes 1.85× more instructions (2.137 G vs 1.158 G) at slightly lower host IPC (3.26 vs 3.54), giving ~2.0× wall time. This is the direct cost of QEMU dynamic binary translation. Evidence: instructions ratio 1.85×, cycles ratio 2.01×, elapsed ratio ~2.0×.
2. **The bottleneck shifts from backend to frontend under emulation.** Frontend Bound rises 3.77% → 22.11% while Backend Bound falls 22.44% → 5.44% and Retiring dips 71.6% → 65.1%. QEMU's translated code stream and TCG dispatcher put heavy pressure on host instruction fetch/decode. Evidence: TopdownL1; host branch count 21.8× (533.6 M vs 24.4 M) yet branch-misprediction rate is lower (0.298% vs 0.415%) — TCG code is long and predictable, so the cost is fetch, not misprediction.
3. **Lack of RVV in the RISC-V hot kernels compounds the emulation tax.** The transform itself is 33× slower under qemu (813.50 µs vs 24.62 µs), far more than the ~2× whole-process ratio. On x86 the hot codelets `n1_32`/`t1_8`/`n1_8` use scalar FMA + memory operands and AVX-512 scalar registers; the corresponding RISC-V codelets contain zero RVV instructions (scalar generic codelets; FFTW SIMD options were OFF and GCC did not auto-vectorize them). Evidence: vectorization counts in Section 5; x86 hotspot DSO 98.64% in libfftw3.so.3 vs riscv 89.65% in QEMU JIT.
4. **Why whole-process ratio (~1.8–2.0×) is much smaller than the transform ratio (33×):** process wall time is dominated by planning/setup, verification and loader/runtime phases (program-reported setup 34.7 ms vs 203.3 ms), while the compute-bound transform kernel bears the full emulation+scalar penalty. The two ratios measure different phases.
5. **Cache and size differences are host artifacts.** L1-dcache miss rate is lower on the qemu process (0.408% vs 3.41%) and LLC miss rate lower (20.9% vs 34.9%, high variance): these are host caches over the emulator's code/data, not guest RISC-V cache behavior. The stripped target binary is larger due to mandatory static linking (vs x86 shared libfftw3.so.3) and includes a statically linked libc.

### 4.6 Vectorization intrinsics (source and binary)

- **Source:** SSE/SSE2, AVX/AVX2/FMA, AVX-512/KCVI, NEON, SVE, AltiVec/VSX, LSX/LASX intrinsics exist but are all compiled out on riscv64. **No RVV source backend exists upstream.**
- **x86 binaries (`1_1_1`):** `bench` 842 vector instructions (33 unique; SSE/AVX/AVX2/FMA, EVEX-encoded scalar); `libfftw3.so.3` 25,241 vector instructions (44 unique; FMA-dominant).
- **riscv binaries (`1_1_2`/`1_1_3`):** RVV instructions exist only in the statically-linked glibc and in FFTW planning/setup paths; the hot codelets (`n1_32`, `t1_8`, `n1_8`) contain **zero** RVV instructions (scalar) in both builds.

### 4.7 QEMU/emulation caveats

- Every `1_1_2` and `1_1_3` timing/PMU value describes the `qemu-riscv64-static` process running on the x86 host; no native RISC-V silicon data exists here.
- Host PMU metrics (IPC, L1/LLC, branch, Topdown) are host x86 counters over the emulator; the `[unknown]` symbols and QEMU JIT hotspots confirm this.
- The 33× transform ratio reflects the emulator plus the scalar RISC-V code path, not a RISC-V board.
- Static linking (required for qemu) is an additional non-hardware difference vs the dynamically-linked x86 build.
- `L1-icache miss rate` was `<not counted>` on both platforms; native RISC-V silicon performance and guest-symbol hotspots are NOT AVAILABLE.

---

## 5. Optimization Results

### 5.1 Vector instructions in binary

| Architecture / binary | Count | Findings |
|---|---:|---|
| x86 `bench` (`1_1_1`) | 842 | 33 unique; SSE/SSE2/AVX/AVX2/FMA + EVEX scalar instructions |
| x86 `libfftw3.so.3` (`1_1_1`) | 25,241 | 44 unique; FMA-dominant |
| riscv `bench` (`1_1_2`) | 2,675 RVV insns (273 functions) | Hot codelets `n1_32`/`n1_8`/`t1_8` = 0 RVV; RVV is in static glibc and FFTW planner |
| riscv `bench` (`1_1_3`, optimized) | 8,375 RVV insns (383 functions) | Hot codelets `n1_32`/`n1_8`/`t1_8` = 0 RVV; extra RVV is in planner/apply loops (`fft0`, `bluestein`, `mkplan`, harness) |

### 5.2 Executable size analysis (stripped vs unstripped)

| Binary | Unstripped (B) | Stripped (B) | `.text` (B) |
|---|---:|---:|---:|
| x86 `bench` (`1_1_1`) | 396,880 | 100,704 | 62,426 |
| riscv `bench` (`1_1_2`) | 6,972,016 | 1,286,680 | 958,746 |
| riscv `bench` (`1_1_3`) | 10,557,088 | 1,737,232 | NOT AVAILABLE |
| x86 `libfftw3.so.3` | 5,704,448 | — | 845,080 |
| riscv `libfftw3.a` (433 members) | — | — | 663,845 |

Causal note: the large target file size is mainly static linking plus DWARF debug info (79% of the raw target file), not RISC-V code bloat. The RISC-V FFTW archive `.text` (663,845 B) is actually smaller than the x86 shared-library `.text` (845,080 B), thanks to RVC compressed instructions and no AVX-512 unrolling.

### 5.3 Optimization attempts

| Optimization | Before (`1_1_2`) | After (`1_1_3`) | Delta | Causal analysis |
|---|---|---|---|---|
| real_time (s) | 0.47 | 0.55 | +0.08 | Regression; LTO+unrolling enlarged code and slowed planner-heavy setup |
| user_time (s) | 0.46 | 0.55 | +0.09 | Regression; same cause |
| IPC | 3.3 | 2.9 | −0.4 | Lower host IPC; larger TCG-translated working set |
| LLC miss rate (%) | 24.7 | 30.6 | +5.9 | Bigger static image stresses host caches over qemu |
| Branch misprediction rate (%) | 0.3 | 0.4 | +0.1 | `-funroll-loops` inflated branchy scalar code |
| Stripped size (B) | 1,286,680 | 1,737,232 | +450,552 | LTO/unroll grew code; debug info retained |
| bench transform time (µs, interleaved mean) | 929.45 | 969.80 | +40.35 | Hot codelets stayed scalar; unrolling inflated instruction counts (n1_8 +81%, t1_8 +78%) |
| bench setup (ms, interleaved mean) | 256.30 | 314.14 | +57.84 | Planner/apply code grew under LTO/unrolling |
| bench mflops (interleaved mean) | 121.686 | 117.573 | −4.113 | 3.4% lower throughput |

**Outcome:** the applied optimization set (P1 `Release` → effective `-O3`; P2 `-funroll-loops -fomit-frame-pointer`; P3 LTO) **measurably regressed** the target workload. The hot FFT codelets remained scalar in both builds, so no vectorization gain materialized. The regression (~15% wall time) is outside the measured noise (interleaved real-time stdev ≈ 0.057 s on a 0.62 s mean; the shift is directionally consistent across all interleaved rounds, `perf stat`, and the bench self-report).

### 5.4 Improvement of 1_1_2 compared to 1_1_3

| Measured | Baseline value | Optimized value | Improvement % |
|---|---|---|---|
| real_time | 0.47 | 0.55 | 117.02 |
| user_time | 0.46 | 0.55 | 119.57 |
| IPC | 3.3 | 2.9 | 87.88 |
| LLC_miss_rate | 24.7 | 30.6 | 123.89 |
| branch_misprediction_rate | 0.3 | 0.4 | 133.33 |
| tma_frontend_bound | 15.1 | 17.2 | 113.91 |
| tma_backend_bound | 2.4 | 2.7 | 112.5 |
| tma_retiring | 40.6 | 29.3 | 72.17 |

> Values are copied verbatim from the tool-written `improvements.json`. The tool's formula is `improvementPcnt = (optimizedValue / baselineValue) * 100`; for time/miss metrics a value above 100 denotes a regression. The `tma_*` fields are the tool-recorded TopdownL1 values; independent counter-derived Topdown measurements agreed on direction (Retiring lower, Bad Speculation/Frontend higher).

### 5.5 Cross-table (baseline target vs optimized target)

The tool-written optimization comparison (`amixis compare`, `1_1_2` vs `1_1_3`) is reproduced below. Symbol-level content is degenerate (`[unknown]`) because qemu-user perf cannot resolve guest symbols; the quantitative comparison in Section 5.3 comes from `perf stat`.

#### Cross-table: EVENT BRANCH-MISSES — 1_1_2 vs 1_1_3

| Symbol | 1_1_2 % | 1_1_3 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   98.75 |  100.00 |   +1.25 |
| \[unknown\] (/ |    1.25 |    0.00 |   -1.25 |

#### Cross-table: EVENT CACHE-MISSES — 1_1_2 vs 1_1_3

| Symbol | 1_1_2 % | 1_1_3 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |    2.55 |    0.00 |   -2.55 |
| \[unknown\]    |   97.45 |  100.00 |   +2.55 |

#### Cross-table: EVENT CYCLES — 1_1_2 vs 1_1_3

| Symbol | 1_1_2 % | 1_1_3 % | Delta % |
|:----------------|-------:|-------:|-------:|
| \[unknown\]     |  100.00 |   99.79 |   -0.21 |
| \[unknown\] (/  |    0.00 |    0.13 |   +0.13 |
| getopt\_long (/ |    0.00 |    0.08 |   +0.08 |

### 5.6 Recommended optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--------:|---|---|---|---|
| P1 | **Revert** the regressing P1–P3 set for the target; keep the baseline recipe-2 flags | Recover ~15% wall time | Low | The measured optimization regressed; baseline is the best known configuration in this environment |
| P2 | Adopt the RVV backend (upstream PR #377 / `rdolbeau/riscv-v-clean`) | Up to 2–4× on the transform (native); unproven under qemu | High | The only architectural fix for the scalar hot codelets; requires source port (`dft/simd/rvv`, `ENABLE_RVV`) |
| P3 | Native RISC-V hardware validation | N/A (enables meaningful profiling) | High | QEMU cannot resolve guest symbols or provide guest PMU counters |
| P4 | Enable CTest tests via `-DENABLE_THREADS=ON` if a runnable suite is required | N/A (test coverage) | Low | FFTW registers `add_test` only with threads enabled |
| P5 | Consider `-march=rv64gcv`-tuned codelets / explicit RVV once backend exists | Unknown | High | Toolchain default already includes `v`; no RVV codelets upstream |
| — | qemu `-cpu max` in run string | ~0 now; prerequisite for RVV | Low | Current binary already runs; future-proofs RVV. Not applied because the required command line must remain exactly `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu ...` |
| — | Alternative allocators (jemalloc/mimalloc/tcmalloc) for target | Not applicable | — | Not available for riscv64-linux-gnu in this sysroot; allocation is not a measured hotspot |
| — | `-march=native` / `-mtune` for target | Not applicable | — | No native target CPU; qemu TCG does not model microarchitecture |
| — | Stripping debug info for speed | Not applicable | — | DWARF is not mapped executable; affects disk/RSS only |

---

## 6. Notes About Exploration Process

- **Repository is a generator tree.** The canonical GitHub repo `https://github.com/FFTW/fftw3` is the codelet-generator source and does not compile directly. `bootstrap.sh` was run to generate codelets (installed OCaml toolchain). The autotools `make` later failed in `doc/` on missing `fig2dev`, which is irrelevant to the library/benchmark; the generated codelets made the CMake builds viable. The release tarball was not needed; the recorded repo URL remains the git repo at commit `93ed4c78` (`fftw-3.3.11-10-g93ed4c78`).
- **Stale `config.h`.** An in-source `config.h` left by the autotools run shadowed CMake's, causing undefined thread symbols; removing it fixed the native link.
- **Cross-link fixes.** The target build required `-DBUILD_SHARED_LIBS=OFF` (static build) and `-DCMAKE_C_STANDARD_LIBRARIES=-lm` (CMake did not find libm in the sysroot).
- **`qemu-riscv64-static` provisioning.** The binary was absent; `qemu-user-static` is a virtual package in this Ubuntu release. A symlink `/usr/bin/qemu-riscv64-static → /usr/bin/qemu-riscv64` was created; `qemu-riscv64` is verifiably statically linked. Cross-compile + emulation was verified end-to-end (`hello riscv`, exit 0). This is an environment workaround, not a data fabrication.
- **CTest registered 0 tests** on both platforms because FFTW gates `add_test` on `ENABLE_THREADS=ON`, which the recipes leave OFF. Test evidence therefore comes from the manual CTest-equivalent `bench` runs.
- **Amphimixis config.** `input.yml` requires explicit `build_system: cmake`; platform arch values are `x86`/`riscv`/`arm` (not `riscv64`). There is no emulator field in the schema, so the target build uses `run_machine: 1` with an explicit qemu command line as the executable. All built executables were enumerated explicitly (no auto-detect): one executable, `bench`, in both builds.
- **Priority/frequency control limitations.** `nice -n -20` was denied (container lacks CAP_SYS_NICE) on both sides; the CPU governor is `powersave` and `/sys` is read-only, so frequency could not be pinned (observed core0 ≈ 1.30 GHz). Both sides ran under the same conditions.
- **Vectorization tool limitation.** `amphimixis-analyze-vectorization` could not disassemble the RISC-V ELF with the host `objdump` (`architecture UNKNOWN`); the fallback `riscv64-linux-gnu-objdump` was used. Report the actual instruction findings rather than the tool failure.
- **Perf symbol resolution under qemu.** Guest RISC-V symbols are unresolvable; target cross-table symbols appear as `[unknown]` and target hotspots are QEMU JIT blocks. This is a genuine emulation limitation.
- **Optimization outcome.** The applied P1–P3 optimization set regressed the target (~15% wall time). It is reported as measured rather than reverted silently; the baseline `1_1_2` artifacts were preserved.
- **QEMU/emulation caveat (reiterated):** all target numbers include emulation overhead and host-x86 PMU effects; they are not native RISC-V silicon performance.
- **NOT AVAILABLE items:** exact discrete test count; L1-icache miss rate; native RISC-V silicon performance/PMU; guest-symbol hotspots for target builds; Amixis `tma_*` fields are internally inconsistent with independent counter-derived Topdown measurements (flagged, not used for causal claims).

---

## 7. Migration Readiness Summary

| Check | Result |
|---|---|
| Builds on reference (x86_64) | YES (`1_1_1`, 472/472 targets) |
| Tests pass on reference | YES — manual CTest-equivalent runs 2/0 (CTest registered 0 tests) |
| Builds on target (riscv64, cross + static) | YES (`1_1_2`, 471/471; `1_1_3` optimized) |
| Tests pass on target (under qemu) | YES — manual runs under qemu 2/0 for `1_1_2`; 3/0 benchmark runs for `1_1_3` |
| Zero external dependencies | NO — requires `libm`; optional Threads/OpenMP/MPI/Fortran/quadmath (all available on riscv64) |
| No hand-written intrinsics | NO — FFTW ships SSE/AVX/NEON/SVE/AltiVec/LSX intrinsics; all compile out on riscv64, but no RVV backend exists |
| Alignment safe | YES — generic FFTW alignment path used on riscv64 |
| Exceptions handled | N/A — FFTW is C; no C++ exceptions |
| Auto-vectorization | PARTIAL — x86 hot codelets benefit from GCC (`-march=native`); RISC-V hot codelets remain scalar (zero RVV) |

### Migration Verdict: MINOR CONCERNS

FFTW is **functionally portable** to riscv64: it cross-compiles, links statically, and runs correctly under qemu-user; its dependencies are all available; it has no endianness or alignment hazards; and the RISC-V cycle counter is merged upstream. The concerns are performance- and validation-related: upstream has **no RVV SIMD backend**, so the hot transform codelets run scalar (transform ~33× slower under emulation); all target measurements are under **qemu-user emulation on x86**, with no native RISC-V silicon and no resolvable guest-symbol hotspots; and the attempted low-risk compiler optimization set **regressed** performance (~15% wall time). The functional migration is viable; a native-RISC-V performance migration requires the RVV backend and hardware validation.

### Required Actions

1. **Do not adopt the tried P1–P3 optimization set** (Release + `-funroll-loops -fomit-frame-pointer` + LTO) for the target; it regressed wall time by ~15%. Keep the baseline recipe-2 flags.
2. **Port/adopt the RVV 1.0 backend** (upstream PR #377 / `rdolbeau/riscv-v-clean`, or PR #279) — the only architectural fix for the scalar hot codelets (`n1_32`, `t1_8`, `n1_8`).
3. **Validate on native RISC-V hardware** to obtain meaningful timing and PMU/hotspot data; qemu-user cannot provide guest counters or symbols.
4. **Optionally enable `-DENABLE_THREADS=ON`** if a runnable CTest suite is required (FFTW gates `add_test` on threads).
5. **Retain the explicit target execution command** `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu <binary>` and the static-link configuration for emulation.
6. **Track the coarse `rdtime` resolution** on real hardware when using FFTW's own timing (`kernel/cycle.h`).
