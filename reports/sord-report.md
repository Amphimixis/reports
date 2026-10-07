# Migration Readiness Report — sord

**Reference platform:** x86_64  
**Target platform:** x86_64 (the provided `input.yml` defines only an `x86` platform; both builds run on the local x86_64 machine)  
**Pipeline:** Amphimixis migration readiness analysis (analyzer → configurator → builder → profiler → optimizer → final report)

> **Architecture scope note.** The container also provides a `riscv64-linux-gnu` cross toolchain and `qemu-riscv64`, but the provided configuration declares only `arch: x86` and both builds use `build_machine: 1, run_machine: 1`. The analysis was therefore executed as an x86_64 → x86_64 (recipe-level build comparison) pipeline exactly as configured. No RISC-V build/test was configured or performed.

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Resolved clone URL | https://github.com/drobilla/sord.git |
| Canonical author source | https://gitlab.com/drobilla/sord |
| Local clone | /work/sord-workspace/sord |
| Latest commit | 4b5232bb91e7f2f1b84df384494d073fabbcd312 (2026-07-03) |
| Total commits | 590 |
| Latest tag / release | v0.16.22 (HEAD = v0.16.22-2-g4b5232b; NEWS documents an unreleased 0.16.23) |
| Activity | Actively maintained: 10 commits / 12 months, 57 / 3 years, 0 open issues/PRs, sole maintainer David Robillard |
| License | ISC (COPYING); source files dual `0BSD OR ISC` (REUSE) |
| Build systems | Meson only (`meson.build`, `meson_options.txt`, `c_std=c99`). No CMake / Makefile / autotools |
| Tests | 3 Meson unit executables: `sord` (test/test_sord.c), `sordmm` (test/cpp/sordmm_test.cpp), `headers` (test/headers/test_headers.c) |
| CI | None in repository (no `.github/workflows`, `.gitlab-ci.yml`) |
| Benchmarks | None |
| External dependencies | `serd-0 >= 0.30.10` (required), `zix-0 >= 0.4.0` (required); `libm`; `libpcre2-8` (optional, `sord_validate` only). Build-only: meson/python3, doxygen (optional docs), clang-tidy/reuse/autoship (optional lint) |
| Distro packages | 192 packages (Debian, Arch, Fedora, Alpine, openSUSE, Gentoo, Homebrew, MacPorts, MSYS2, Nix, Termux, Raspbian, …) |
| Forks with target-architecture patches | None found (4 GitHub forks; none ahead of upstream with riscv64/arm64 patches) |

## 2. Platform-Specific Code Analysis

### Architecture macros
**None found.** Zero matches for x86 (`__x86_64__`, `_M_X64`, `__SSE*`, `__AVX*`, `__BMI*`), ARM (`__arm__`, `__aarch64__`, `__ARM_NEON__`), RISC-V (`__riscv*`), endianness (`__BYTE_ORDER__`), or pointer-size (`__LP64__`, `__SIZEOF_POINTER__`) macros in `src/`, `include/`, `test/`, or `meson.build`.

### Vectorization intrinsics in source
**None found.** Zero `_mm_*`, `__m128/256/512`, NEON (`vld1`/`vadd`/`vmul`), `__riscv_v`, `vector_size`, `immintrin.h`, `arm_neon.h`, `xmmintrin.h`. The library is scalar C99; hashing/B-trees are delegated to zix.

### Platform preprocessor guards
| Guard | File:Line | What it guards | Semantics / portability |
|---|---|---|---|
| `_WIN32` | include/sord/sord.h:19 | `SORD_API __declspec(dllexport)` | Windows DLL export ABI only |
| `_WIN32` | include/sord/sord.h:21 | `SORD_API __declspec(dllimport)` | Windows DLL import ABI only |
| `__GNUC__` | include/sord/sord.h:23 | `__attribute__((visibility("default")))` | GCC/Clang symbol visibility; harmless on all targets |
| `__clang__` / `__GNUC__ > 4` | src/sord_internal.h:12 | `SORD_UNREACHABLE()` → `__builtin_unreachable()` | Compiler hint; portable fallback below |
| `_MSC_VER` | src/sord_internal.h:14 | `SORD_UNREACHABLE()` → `__assume(0)` | MSVC-only hint |
| (fallback) | src/sord_internal.h:16 | empty `SORD_UNREACHABLE()` | Neutral default |
| `__GNUC__` | src/sord.c:25, src/sord_validate.c:32, test/test_sord.c:14 | `SORD_LOG_FUNC` → `format(printf,…)` | Diagnostics formatting only |
| `__clang__` | src/sord_validate.c:12,20; include/sord/sordmm.hpp:12,20 | clang warning suppression/pragmas | Diagnostics only |
| `__has_include` | src/sord_config.h:25-26 | feature-detect `<pcre2.h>` (`HAVE_PCRE2`) | Compiler feature detection, portable |
| `__cplusplus` | include/sord/sord.h:30,585 | `extern "C"` linkage | Language interop, portable |
| `host_machine.system()` | meson.build (~60-80, 136-142) | warning flags / `soversion` | Build-system OS check, no ISA impact |

Semantic note: every guard is a compiler-ABI, symbol-visibility, diagnostics, or C++-linkage concern — none gates an algorithm or ISA path.

### Portability verdict
- **Exceptions handled:** pure C99 library; no C++ exceptions in the core (a thin C++ binding exists).
- **Alignment safe:** no `vector_size`, no manual SIMD, no ISA-specific unaligned access; data structures delegated to zix.
- **Embedded usability:** low — only malloc-based allocation via zix; no OS-specific requirements beyond the Windows ABI guards.
- **No hand-written intrinsics:** confirmed.
- **Overall portability level: LOW concern.**

## 3. Build & Test Results

Because the bundled `amixis` 0.2.0 cannot drive the Meson build system (its high-level registry implements only cmake/make), builds were performed manually with `meson`/`ninja` into the exact build-name directories that `amixis profile` expects. See Section 6.

| Build | Platform | Recipe | Configure | Compile | Tests (pass/fail/skip) | Executable |
|---|---|---|---|---|---|---|
| 1_1_1 | x86_64 (reference = target) | 1: `-Dbuildtype=debugoptimized -Dtests=enabled -Dtools=enabled` | OK | OK | 3 / 0 / 0 | /work/sord-workspace/1_1_1/test/test_sord (102584 B) |
| 1_1_2 | x86_64 (reference = target) | 2: recipe 1 + `-O3 -march=native -g` | OK | OK | 3 / 0 / 0 | /work/sord-workspace/1_1_2/test/test_sord (104888 B) |
| 1_1_3 | x86_64 (reference = target) | 3: recipe 2 + `-Db_lto=true` (Phase-6 optimization) | OK | OK | 3 / 0 / 0 | /work/sord-workspace/1_1_3/test/test_sord (102944 B) |

Tests run: `unit - sord:headers`, `unit - sord:sord`, `unit - sord:sordmm`.

Dependencies built and installed into `/work/sord-workspace/deps`: **zix 0.8.3**, **serd 0.32.11**. Optional `libpcre2-8` was not found, so `sord_validate` is skipped; it does not affect the library or the tests.

**Build/test failures: none.**

## 4. Performance Comparison

### Experimental conditions
| Item | Value |
|---|---|
| CPU | 13th Gen Intel(R) Core(TM) i7-13620H (hybrid; 10 cores / 16 threads; P-cores 0–11, E-cores 12–15) |
| Core pinning | CPU 0 (P-core) via `taskset -c 0` |
| Priority | `nice -n -20` was requested but **not permitted** (container lacks `CAP_SYS_NICE`); runs executed at default priority |
| Frequency governor | `powersave`; `scaling_cur_freq` observed ≈ 0.4–1.6 GHz, not locked |
| Warmup | 2 runs per build |
| Measurement runs | 10 per build (`perf stat --repeat 10`; `/usr/bin/time` ×10) |
| perf | 7.0.14; `/proc/sys/kernel/perf_event_paranoid = -1` |
| Target | **native x86_64, identical to reference — no QEMU/emulation** |

### Key metrics (reference `1_1_1` vs optimized `1_1_2`)
| Metric | 1_1_1 | 1_1_2 | Provenance |
|---|---|---|---|
| Elapsed time, median | 0.013600 s | 0.012965 s | REAL MEASURED (10 runs) |
| Cycles | 28,954,111 | 27,233,754 | REAL MEASURED |
| Instructions | 74,862,289 | 73,334,880 | REAL MEASURED |
| IPC | 2.586 | 2.693 | DERIVED from measured counters |
| L1-dcache miss rate | 0.486 % | 0.469 % | DERIVED from measured counters |
| LLC miss rate | 48.19 % | 57.21 % | DERIVED — **low confidence** (very low event counts) |
| Branch misprediction rate | 1.391 % | 1.396 % | DERIVED from measured counters |
| Frontend Bound | 30.6 % | 26.3 % | REAL MEASURED (TMA) |
| Backend Bound | 7.4 % | 9.4 % | REAL MEASURED (TMA) |
| Retiring | 46.1 % | 46.9 % | REAL MEASURED (TMA) |

### Hotspots — reference `1_1_1` (`-O2`), cycles sample weight
| % cycles | Symbol | Module | Analysis |
|---:|---|---|---|
| 48.45 | `[unknown]` | kernel + deleted mappings | Non-project overhead |
| 6.74 | `sord_iter_end@plt` | test_sord (PLT) | Iterator teardown; PLT call not inlined at -O2 |
| 6.72 | `sord_drop_quad_ref` | libsord | Reference-free hot loop |
| 6.71 | `zix_digest@plt` | libsord (PLT) | Hashing via un-inlined PLT call |
| 6.60 | `sord_free` | libsord | Teardown/alloc churn |
| 6.52 | `zix_btree_get` | libzix | B-tree lookup |
| 6.41 | `sord_quad_compare` | libsord | Quad comparison |
| 6.26 | `sord_add` | libsord | Insert path |
| 5.58 | `zix_hash_plan_insert_prehashed` | libzix | Hash-table insert |

### Hotspots — optimized `1_1_2` (`-O3 -march=native`), cycles sample weight
| % cycles | Symbol | Module | Analysis |
|---:|---|---|---|
| 42.97 | `[unknown]` | kernel + deleted mappings | Non-project overhead |
| 13.65 | `sord_add` | libsord | Hot path concentrated ~2.2× vs baseline (inlining/hoisting) |
| 6.86 | `zix_btree_get` | libzix | B-tree lookup |
| 6.70 | `__snprintf_chk` | libc (deleted) | URI serialization in `generate`/`uri` |
| 6.62 | `sord_quad_compare` | libsord | Quad comparison |
| 6.51 | `test_read.constprop.0` | test_sord | Test harness inlined/const-propagated (absent at -O2) |
| 6.00 | `sord_quad_match` | libsord | Match path now visible |
| 5.62 | `sord_iter_scan_next` | libsord | Iterator scan (inlined at -O3) |
| 5.06 | `sord_iter_next` | libsord | Iterator step |

### Bottleneck summary and causal analysis
- **Frontend Bound is the largest non-retiring cost (30.6 % → 26.3 %).** The hot path is a fan-out of branchy, non-loop calls spread across two DSOs (`libsord`, `libzix`); trivial accessors (`sord_iter_end`, `zix_digest`) remain PLT calls because the executable and libraries are separate DSOs. `-O3 -march=native` inlined the small helpers (visible as PLT symbols disappearing) and shrank the dynamic stream.
- **Branch misprediction is unchanged (1.391 % → 1.396 %).** Hash probing and B-tree descent are inherently data-dependent; the ~200 K branch misses are the same work re-attributed by inlining. The speedup is not branch-driven.
- **Memory latency is NOT the limit** (Backend Bound only 7.4 % → 9.4 %; L1 miss rate < 0.5 %; LLC counts tiny/noisy). The workload working set fits in cache, so AVX2/cache tricks under-deliver.
- **Why `-O3 -march=native` gains only ~5 %:** the compiler selected AVX2 (VEX encodings) but found **no packed arithmetic to vectorize** — FMA = 0 and packed FP/int arithmetic ≈ 0. Sord is a text/RDF engine (hashing, B-tree, string/quad comparison); AVX2 only buys wider moves/zeroing, reducing instructions by ~2 % and cycles by ~6 %, while inflating `libsord .text` by ~29 %.
- **Caveat:** the profiler captures only ~15–20 cycles samples per build over ~13 ms runs, so per-symbol hotspot percentages are indicative, not statistically robust.

### Vectorization intrinsics in source
None — no hand-written SIMD anywhere in the project.

### QEMU / emulation caveats
**Not applicable.** The target platform is the same native x86_64 machine as the reference; there is no cross-compilation and no QEMU/emulation overhead in any measured number.

## Cross-table 1 — 1_1_1 (reference) vs 1_1_2 (optimized), from `cross-tables/CT-1__1__1..test_stest__sord-1__1__2..test_stest__sord.md`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_sord and 1\_\_1\_\_2..test\_stest\_\_sord

## EVENT: CPU\_ATOM/BRANCH-MISSES/

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |    0.00 | -100.00 |

## EVENT: CPU\_ATOM/CACHE-MISSES/

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |    0.00 | -100.00 |

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| \[unknown\] (/                     |   34.07 |   58.13 |  +24.05 |
| rehash                             |   14.18 |    0.00 |  -14.18 |
| zix\_hash\_plan\_insert\_prehashed |    0.00 |   12.05 |  +12.05 |
| sord\_node\_hash\_equal            |    8.03 |    0.00 |   -8.03 |
| sord\_add                          |    7.05 |    0.00 |   -7.05 |
| zix\_btree\_lower\_bound           |    6.27 |    0.00 |   -6.27 |
| generate.isra.0                    |    6.14 |    0.00 |   -6.14 |
| posix\_memalign@plt                |    6.12 |    0.00 |   -6.12 |
| sord\_quad\_compare                |    7.39 |   13.22 |   +5.83 |
| cfree (/                           |    0.00 |    4.95 |   +4.95 |
| \[unknown\]                        |   10.76 |   11.65 |   +0.89 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol      | 1_1_1 % | 1_1_2 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                             | 1_1_1 % | 1_1_2 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| sord\_add                          |    6.26 |   13.65 |   +7.39 |
| sord\_iter\_end@plt                |    6.74 |    0.00 |   -6.74 |
| sord\_drop\_quad\_ref              |    6.72 |    0.00 |   -6.72 |
| zix\_digest@plt                    |    6.71 |    0.00 |   -6.71 |
| \_\_snprintf\_chk (/               |    0.00 |    6.70 |   +6.70 |
| sord\_free                         |    6.60 |    0.00 |   -6.60 |
| test\_read.constprop.0             |    0.00 |    6.51 |   +6.51 |
| sord\_quad\_match                  |    0.00 |    6.00 |   +6.00 |
| \[unknown\]                        |   28.87 |   23.02 |   -5.85 |
| sord\_iter\_scan\_next             |    0.00 |    5.62 |   +5.62 |
| zix\_hash\_plan\_insert\_prehashed |    5.58 |    0.00 |   -5.58 |
| sord\_iter\_next                   |    0.00 |    5.06 |   +5.06 |
| \[unknown\] (/                     |   19.58 |   19.95 |   +0.37 |
| zix\_btree\_get                    |    6.52 |    6.86 |   +0.34 |
| sord\_quad\_compare                |    6.41 |    6.62 |   +0.21 |



## Cross-table 2 — 1_1_1 (baseline) vs 1_1_3 (LTO-optimized), from `cross-tables/CT-1__1__1..test_stest__sord-1__1__3..test_stest__sord.md`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_sord and 1\_\_1\_\_3..test\_stest\_\_sord

## EVENT: CPU\_ATOM/BRANCH-MISSES/

| Symbol      | 1_1_1 % | 1_1_3 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |    0.00 | -100.00 |

## EVENT: CPU\_ATOM/CACHE-MISSES/

| Symbol      | 1_1_1 % | 1_1_3 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |    0.00 | -100.00 |

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                  | 1_1_1 % | 1_1_3 % | Delta % |
|:------------------------|-------:|-------:|-------:|
| sord\_quad\_compare     |   19.64 |    5.81 |  -13.83 |
| zix\_btree\_insert      |    0.00 |   11.06 |  +11.06 |
| sord\_add               |    7.89 |    0.00 |   -7.89 |
| zix\_hash\_record\_at   |    7.53 |    0.00 |   -7.53 |
| sord\_node\_free        |    0.00 |    6.23 |   +6.23 |
| strcmp@plt              |    6.14 |    0.00 |   -6.14 |
| \_\_snprintf\_chk (/    |    0.00 |    5.84 |   +5.84 |
| sord\_node\_hash\_equal |    0.00 |    5.72 |   +5.72 |
| zix\_hash\_find         |    0.00 |    5.54 |   +5.54 |
| \[unknown\]             |   17.67 |   14.71 |   -2.97 |
| rehash                  |   13.78 |   16.08 |   +2.30 |
| \[unknown\] (/          |   27.35 |   29.01 |   +1.66 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol      | 1_1_1 % | 1_1_3 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |  100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                             | 1_1_1 % | 1_1_3 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| \[unknown\]                        |   21.00 |   13.14 |   -7.86 |
| sord\_quad\_compare                |   18.52 |   11.74 |   -6.78 |
| test\_read.constprop.0             |    6.62 |    0.00 |   -6.62 |
| zix\_hash\_insert\_at              |    0.00 |    6.15 |   +6.15 |
| sord\_iter\_next                   |    0.00 |    6.11 |   +6.11 |
| sord\_node\_hash                   |    0.00 |    5.86 |   +5.86 |
| \[unknown\] (/                     |   34.91 |   29.12 |   -5.78 |
| zix\_hash\_record\_at              |    0.00 |    4.98 |   +4.98 |
| zix\_btree\_iter\_increment        |    0.00 |    4.51 |   +4.51 |
| \_\_libc\_early\_init (/           |    1.99 |    0.00 |   -1.99 |
| serd\_strlen                       |    5.06 |    6.10 |   +1.04 |
| zix\_hash\_plan\_insert\_prehashed |    5.82 |    6.15 |   +0.33 |
| sord\_add                          |    6.08 |    6.12 |   +0.04 |



## Cross-table 3 — 1_1_2 (pre-LTO) vs 1_1_3 (post-LTO), from `cross-tables/CT-1__1__2..test_stest__sord-1__1__3..test_stest__sord.md`

# Cross-tables for 1\_\_1\_\_2..test\_stest\_\_sord and 1\_\_1\_\_3..test\_stest\_\_sord

## EVENT: CPU\_ATOM/BRANCH-MISSES/

| Symbol         | 1_1_2 % | 1_1_3 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\] (/ |   83.22 |    0.00 |  -83.22 |
| \[unknown\]    |   16.78 |    0.00 |  -16.78 |

## EVENT: CPU\_ATOM/CACHE-MISSES/

| Symbol      | 1_1_2 % | 1_1_3 % | Delta % |
|:------------|-------:|-------:|-------:|
| \[unknown\] |  100.00 |    0.00 | -100.00 |

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                             | 1_1_2 % | 1_1_3 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| \[unknown\] (/                     |   57.32 |   29.01 |  -28.31 |
| zix\_hash\_plan\_insert\_prehashed |   17.64 |    0.00 |  -17.64 |
| rehash                             |    0.00 |   16.08 |  +16.08 |
| \[unknown\]                        |    3.40 |   14.71 |  +11.30 |
| zix\_btree\_insert                 |    0.00 |   11.06 |  +11.06 |
| sord\_node\_free                   |    0.00 |    6.23 |   +6.23 |
| sord\_quad\_compare                |   11.76 |    5.81 |   -5.94 |
| \_\_snprintf\_chk (/               |    0.00 |    5.84 |   +5.84 |
| sord\_node\_hash\_equal            |    0.00 |    5.72 |   +5.72 |
| zix\_hash\_find                    |    0.00 |    5.54 |   +5.54 |
| sord\_add@plt                      |    5.21 |    0.00 |   -5.21 |
| sord\_find                         |    4.67 |    0.00 |   -4.67 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol         | 1_1_2 % | 1_1_3 % | Delta % |
|:---------------|-------:|-------:|-------:|
| \[unknown\]    |   59.78 |  100.00 |  +40.22 |
| \[unknown\] (/ |   40.22 |    0.00 |  -40.22 |

## EVENT: CYCLES

| Symbol                             | 1_1_2 % | 1_1_3 % | Delta % |
|:-----------------------------------|-------:|-------:|-------:|
| sord\_iter\_next                   |   16.68 |    6.11 |  -10.57 |
| zix\_hash\_insert\_at              |    0.00 |    6.15 |   +6.15 |
| zix\_hash\_plan\_insert\_prehashed |    0.00 |    6.15 |   +6.15 |
| sord\_add                          |    0.00 |    6.12 |   +6.12 |
| serd\_strlen                       |    0.00 |    6.10 |   +6.10 |
| zix\_btree\_get                    |    5.93 |    0.00 |   -5.93 |
| zix\_btree\_insert                 |    5.91 |    0.00 |   -5.91 |
| zix\_digest64@plt                  |    5.88 |    0.00 |   -5.88 |
| sord\_node\_hash                   |    0.00 |    5.86 |   +5.86 |
| sord\_find                         |    5.72 |    0.00 |   -5.72 |
| sord\_iter\_get@plt                |    5.27 |    0.00 |   -5.27 |
| zix\_hash\_find                    |    5.04 |    0.00 |   -5.04 |
| zix\_btree\_iter\_increment        |    0.00 |    4.51 |   +4.51 |
| \[unknown\]                        |    8.82 |   13.14 |   +4.33 |
| \[unknown\] (/                     |   25.99 |   29.12 |   +3.14 |
| sord\_quad\_compare                |   10.07 |   11.74 |   +1.67 |
| zix\_hash\_record\_at              |    4.70 |    4.98 |   +0.28 |



## 5. Optimization Results

### Vector instructions in the binaries
Verified with `objdump -d` (authoritative) and the amphimixis vectorization analyzer:

| Binary | SSE/legacy moves | VEX-encoded total | Packed int arith | Packed FP arith | FMA | AVX-512 |
|:--|:--|:--|:--|:--|:--|:--|
| `1_1_1/test/test_sord` (-O2) | 21 (`movaps` 18, `movups` 1, `movdq[au]` 2) | 0 | 0 | 0 | 0 | 0 |
| `1_1_2/test/test_sord` (-O3 -march=native) | 8 `vmovaps`, 9 `vmovq`, 12 `vmovdqa`, 7 `vmovdqu` | 51 | `vpinsrq` 9, `vpxor` 6 | 0 | 0 | 0 |
| `1_1_1/libsord-0.so` (-O2) | `movups` 86, `movdqu` 62, `movaps` 33, `movdqa` 18 | 0 | `pxor` 15, `punpck` 4 | 0 | 0 | 0 |
| `1_1_2/libsord-0.so` (-O3 -march=native) | `vmovdqu` 120, `vmovdqa` 15, `vmovaps` 8, `vmovq` 15 | 205 | `vpxor` 25, `vpinsrq` 12, `vinserti128` 4 | 0 | 0 | 0 |

`-march=native` selected **AVX2** (host has avx2/fma/bmi2, no AVX-512) and fully VEX-encoded the optimized binaries, but produced **no packed arithmetic** — the workload has no dense numeric loop to vectorize. This is compiler auto-vectorization / instruction selection, not hand-written intrinsics.

### Executable size analysis (before/after strip)
| Artifact | Raw bytes | Stripped file bytes | `.text` (`size -A`) |
|:--|--:|--:|--:|
| 1_1_1 `test_sord` | 102584 | 35024 | 13340 |
| 1_1_2 `test_sord` | 104888 | 35024 | 13859 |
| 1_1_1 `libsord-0.so.0.16.22` | 114232 | 39528 | 14181 |
| 1_1_2 `libsord-0.so.0.16.22` | 134640 | 43624 | 18332 |

Both `test_sord` binaries strip to the identical 35024 B; the 2304 B difference in `1_1_2` is debug info. `libsord` grows by ~29 % in `.text` under `-O3 -march=native` (unrolling + AVX2 encodings) for ~0 arithmetic speedup — a real I-cache-pressure tradeoff.

### Optimization attempts
One optimization was applied in Phase 6: **Link-Time Optimization** (`-Db_lto=true`) on top of the `-O3 -march=native -g` recipe, producing build `1_1_3`.

| Optimization | Before (1_1_2) | After (1_1_3) | Delta | Causal analysis |
|---|---|---|---|---|
| LTO `-Db_lto=true` (Meson) + `-O3 -march=native -g` | recipe-2 binary (104888 B) | recipe-3 binary (102944 B) | per-parameter values in the Improvement section below | GCC `.gnu.lto_*` sections confirm LTO engaged; cross-TU inlining removed the `@plt` accessor/digest call overhead (the `1_1_3` cycles profile contains **zero** `@plt` samples, versus `sord_iter_get@plt`, `zix_digest64@plt`, `sord_add@plt` in `1_1_2`), and the executable shrank by 1944 B. Gains are small because the remaining bottlenecks (data-dependent hash/B-tree branches, `sord_quad_compare`, `snprintf`) are not addressed by LTO. |

### Recommended optimizations (unchanged from optimizer analysis)
| Priority | Optimization | Expected Gain | Effort | Notes |
|:--:|:--|:--:|:--:|:--|
| 1 | LTO `-Db_lto=true` | 2–6 % | Low | Applied in Phase 6; targets Frontend Bound and PLT/accessor overhead |
| 2 | Static library `-Ddefault_library=static` | 3–8 % | Low | Removes the DSO boundary so accessors inline into `test_sord`; test separately from LTO |
| 3 | `-Dbuildtype=release` / `-Db_ndebug=true` | 1–4 % | Low | Removes live asserts from `sord_add`, `sord_iter_scan_next` |
| 4 | PGO (`-Db_pgo=generate` → `use`) | 5–15 % | High | Best lever for 16–17 % Bad Speculation and 26–31 % Frontend Bound; needs a training run |
| 5 | Faster allocator (mimalloc/jemalloc) via `LD_PRELOAD` | 3–10 % | Med | `posix_memalign`/`cfree` visible in samples; allocator not currently installed |
| 6 | Benchmark `uri()` fast integer formatter (remove `snprintf`) | 3–8 % | Low | `__snprintf_chk` is ~6.7 % of cycles; changes the benchmark, not the library |
| 7 | `-fno-plt -fno-semantic-interposition` | 1–3 % | Low | Reduces PLT/GOT overhead for remaining external calls |
| 8 | `-march=x86-64-v3` instead of `native` | 0–2 % (size down) | Low | Removes AVX2 move bloat, better I-cache, more reproducible |
| 9 | Strip debug info for distribution | 0 % runtime | Low | Distribution-size only (35024/43624 B stripped vs 102584/134640 B raw) |
| 10 | Hugepages / cache blocking | ~0 % | Med | Backend Bound is only 7–9 %; working set fits in cache — not worth it |

## Improvement of 1_1_2 compared to 1_1_3

Phase-6 optimization: enable Link-Time Optimization (`-Db_lto=true`) on top of the `-O3 -march=native -g` recipe. Values below are copied verbatim from `improvements.json`.

| Measured | Baseline value | Optimized value | Improvement % |
|---|---|---|---|
| real_time | 0.013297278 | 0.01277938 | 96.11 |
| cycles | 26912678 | 26809237 | 99.62 |
| instructions | 73354122 | 73298936 | 99.92 |
| IPC | 2.7256 | 2.7341 | 100.31 |
| L1_dcache_miss_rate | 0.4626 | 0.4593 | 99.29 |
| branch_misprediction_rate | 1.3862 | 1.3864 | 100.01 |
| Frontend_Bound | 26.7 | 26.8 | 100.37 |
| Backend_Bound | 9.8 | 10.2 | 104.08 |
| Retiring | 46.1 | 46.3 | 100.43 |
| Bad_Speculation | 17.4 | 16.7 | 95.98 |

## 6. Notes About Exploration Process

1. **Tool gap — `amixis build` does not support Meson.** The bundled `amphimixis` 0.2.0 registers only `cmake` and `make` as high-level build systems (`amphimixis/core/build_systems/build_systems.py`); `build_system: meson` is rejected by `amixis validate`, and leaving it unset fails auto-detection ("No build system found"). Therefore `amixis build` could not be used for sord. Builds were performed manually with `meson`/`ninja` into the exact build-name directories (`1_1_1`, `1_1_2`, `1_1_3`) that `amixis profile` consumes. This is documented rather than hidden.
2. **`input.yml` flag mismatch (corrected).** The provided recipes used CMake-style flags (`-DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_TESTING=ON`) on a Meson project. The configurator translated them to valid Meson options (`-Dbuildtype=debugoptimized -Dtests=enabled -Dtools=enabled`). The provided `input.yml` was updated; a separate `profile.yml` / `profile_opt.yml` with `build_system: cmake` (parser-satisfying only) was used for profiling.
3. **`amixis` profiling and compare worked as designed.** `amixis profile` never invokes the build system, and `amixis compare` is standalone over two `.scriptout` files; all three cross-tables were produced by `amixis compare`.
4. **Priority elevation failed.** `nice -n -20` returned `setpriority: Permission denied` even as root (no `CAP_SYS_NICE`); all measurements ran at default priority.
5. **Kernel symbols unresolved.** `perf report` reported `Invalid HEADER_EVENT_DESC` and "Kernel address maps restricted", so a large `[unknown]` bucket (~20–49 % of cycles) is present in the hotspot data. User-space symbols resolved in `.scriptout` and were used.
6. **Small sample counts.** `perf record -F 1000` over ~13 ms runs yields only ~15–20 cycles samples per build, making per-symbol hotspot percentages indicative rather than statistically robust.
7. **LLC miss-rate low confidence.** The 8-event `perf stat --repeat` multiplexes on this hybrid CPU and the absolute LLC counts are tiny (RSD ±29–39 %); the LLC miss-rate row is labelled low confidence in Section 4.
8. **Hybrid CPU event split.** Events split across `cpu_core/*` and `cpu_atom/*`; runs pinned to CPU 0 (P-core) left the `cpu_atom` counters reading `<not counted>`, which appears as asymmetric empty rows in the cross-tables. This is a sampling artifact, not a code property.
9. **Optional dependency absent.** `libpcre2-8` headers are not installed, so `sord_validate` is skipped; it does not affect the library or tests.
10. **QEMU / emulation caveats.** **Not applicable** — the target platform is the same native x86_64 machine as the reference; no cross-compilation or QEMU/emulation was involved, so no emulation overhead is present in any measurement.
11. **No fabrication.** Every metric in this report is either REAL MEASURED data or explicitly labelled DERIVED from measured counters / low confidence; unavailable data is marked NOT AVAILABLE. No percentage or hotspot was estimated.

## 7. Migration Readiness Summary

| Check | Result |
|---|---|
| Builds on reference platform | Yes (1_1_1, 1_1_2, 1_1_3) |
| Tests pass on reference platform | Yes — 3 / 3 pass, 0 fail |
| Builds on target platform | Yes (target = reference = native x86_64) |
| Tests pass on target platform | Yes — 3 / 3 pass, 0 fail |
| Zero external dependencies | No — requires `serd-0` and `zix-0` (both portable and built successfully here); `libpcre2-8` optional |
| No hand-written intrinsics | Yes — zero SIMD intrinsics in source |
| Alignment safe | Yes — no `vector_size`, no manual SIMD, no ISA-specific unaligned access |
| Exceptions handled | Yes — pure C99 core; no C++ exceptions in the library |
| Auto-vectorization | Yes — `-march=native` selects AVX2/VEX codegen, but the workload has no packed arithmetic to vectorize |

### Migration Verdict: **READY**

For the **configured target (x86_64, identical to the reference)** the project builds and passes all tests, has no architecture-specific code, no hand-written SIMD, and no runtime portability hazards. The migration is ready.

### Required Actions
- **None blocking for the configured x86_64 target.**
- Optional hardening identified by the optimizer: apply LTO (`-Db_lto=true`, already validated), consider static linking, `-Dbuildtype=release`, PGO, and a faster allocator — all optional performance measures, not portability requirements.
- **If a future RISC-V (riscv64) migration is desired**, it is not covered by the provided configuration and no riscv build/test was performed. To extend: cross-compile `zix` and `serd` for riscv64 into the sysroot (the container has `riscv64-linux-gnu-gcc`/`g++` and `qemu-riscv64`, but the riscv sysroot currently lacks `serd`/`zix`), then add a `riscv` platform and a cross recipe to `input.yml`, build, and re-run this pipeline. The analyzer rates sord source-level portability as LOW concern, so no source changes are expected to be needed.

---

## Appendix — Recorded project profile data (`sord.json`)

```json
{
  "1_1_3": {
    "test/test_sord": {
      "build_name": "1_1_3",
      "executable": "test/test_sord",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "5",
          "event": "context-switches",
          "event-runtime": "16984792",
          "pcnt-running": "100.00",
          "metric-value": "294.4",
          "metric-unit": "cs/sec  cs_per_second"
        },
        {
          "counter_value": "1",
          "event": "cpu-migrations",
          "event-runtime": "16984792",
          "pcnt-running": "100.00",
          "metric-value": "58.9",
          "metric-unit": "migrations/sec  migrations_per_second"
        },
        {
          "counter_value": "291",
          "event": "page-faults",
          "event-runtime": "16984792",
          "pcnt-running": "100.00",
          "metric-value": "17133.0",
          "metric-unit": "faults/sec  page_faults_per_second"
        },
        {
          "counter_value": "16.98",
          "unit": "msec",
          "event": "task-clock",
          "event-runtime": "16984792",
          "pcnt-running": "100.00",
          "metric-value": "0.9",
          "metric-unit": "CPUs  CPUs_utilized"
        },
        {
          "counter_value": "106408",
          "event": "cpu_core/L1-dcache-load-misses/",
          "event-runtime": "4642054",
          "pcnt-running": "27.00",
          "metric-unit": "%  l1d_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_core/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "100.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "27422",
          "event": "cpu_core/LLC-loads/",
          "event-runtime": "4049463",
          "pcnt-running": "23.00",
          "metric-value": "13.2",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "217046",
          "event": "cpu_core/branch-misses/",
          "event-runtime": "6050768",
          "pcnt-running": "35.00",
          "metric-value": "1.7",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "13252337",
          "event": "cpu_core/branches/",
          "event-runtime": "7345520",
          "pcnt-running": "43.00",
          "metric-value": "780.2",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "29156446",
          "event": "cpu_core/cpu-cycles/",
          "event-runtime": "8343292",
          "pcnt-running": "49.00",
          "metric-value": "1.7",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "71852759",
          "event": "cpu_core/instructions/",
          "event-runtime": "9343279",
          "pcnt-running": "55.00",
          "metric-value": "2.5",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "22410489",
          "event": "cpu_core/dTLB-loads/",
          "event-runtime": "6291970",
          "pcnt-running": "37.00",
          "metric-value": "0.0",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/LLC-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branch-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branches/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/cpu-cycles/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/instructions/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/dTLB-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "unit": "8292562",
          "event": "48.00",
          "metric-unit": "TopdownL1 (cpu_core)"
        },
        {
          "metric-value": "16.2",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-value": "27.5",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "8292562",
          "event": "48.00",
          "metric-value": "10.9",
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "metric-value": "45.3",
          "metric-unit": "%  tma_retiring"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "TopdownL1 (cpu_atom)"
        },
        {
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-unit": "%  tma_retiring"
        }
      ],
      "perf_record_name": "1__1__3..test_stest__sord.perfdata",
      "perf_script_name": null,
      "perf_archive_name": "1__1__3..test_stest__sord.tar.bz2"
    }
  },
  "1_1_2": {
    "test/test_sord": {
      "build_name": "1_1_2",
      "executable": "test/test_sord",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "5",
          "event": "context-switches",
          "event-runtime": "18100622",
          "pcnt-running": "100.00",
          "metric-value": "276.2",
          "metric-unit": "cs/sec  cs_per_second"
        },
        {
          "counter_value": "1",
          "event": "cpu-migrations",
          "event-runtime": "18100622",
          "pcnt-running": "100.00",
          "metric-value": "55.2",
          "metric-unit": "migrations/sec  migrations_per_second"
        },
        {
          "counter_value": "292",
          "event": "page-faults",
          "event-runtime": "18100622",
          "pcnt-running": "100.00",
          "metric-value": "16132.0",
          "metric-unit": "faults/sec  page_faults_per_second"
        },
        {
          "counter_value": "18.10",
          "unit": "msec",
          "event": "task-clock",
          "event-runtime": "18100622",
          "pcnt-running": "100.00",
          "metric-value": "0.9",
          "metric-unit": "CPUs  CPUs_utilized"
        },
        {
          "counter_value": "113848",
          "event": "cpu_core/L1-dcache-load-misses/",
          "event-runtime": "4957421",
          "pcnt-running": "27.00",
          "metric-unit": "%  l1d_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_core/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "100.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "24460",
          "event": "cpu_core/LLC-loads/",
          "event-runtime": "4067033",
          "pcnt-running": "22.00",
          "metric-value": "18.2",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "223014",
          "event": "cpu_core/branch-misses/",
          "event-runtime": "6064132",
          "pcnt-running": "33.00",
          "metric-value": "1.7",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "13695976",
          "event": "cpu_core/branches/",
          "event-runtime": "8064659",
          "pcnt-running": "44.00",
          "metric-value": "756.7",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "28582642",
          "event": "cpu_core/cpu-cycles/",
          "event-runtime": "9141173",
          "pcnt-running": "50.00",
          "metric-value": "1.6",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "72418832",
          "event": "cpu_core/instructions/",
          "event-runtime": "10140564",
          "pcnt-running": "56.00",
          "metric-value": "2.5",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "22229569",
          "event": "cpu_core/dTLB-loads/",
          "event-runtime": "7079069",
          "pcnt-running": "39.00",
          "metric-value": "0.0",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/LLC-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branch-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branches/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/cpu-cycles/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/instructions/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/dTLB-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "unit": "9076906",
          "event": "50.00",
          "metric-unit": "TopdownL1 (cpu_core)"
        },
        {
          "metric-value": "17.4",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-value": "26.3",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "9076906",
          "event": "50.00",
          "metric-value": "9.4",
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "metric-value": "46.9",
          "metric-unit": "%  tma_retiring"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "TopdownL1 (cpu_atom)"
        },
        {
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-unit": "%  tma_retiring"
        }
      ],
      "perf_record_name": "1__1__2..test_stest__sord.perfdata",
      "perf_script_name": null,
      "perf_archive_name": "1__1__2..test_stest__sord.tar.bz2"
    }
  },
  "1_1_1": {
    "test/test_sord": {
      "build_name": "1_1_1",
      "executable": "test/test_sord",
      "executable_run_success": true,
      "real_time": "0.02",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "5",
          "event": "context-switches",
          "event-runtime": "18464214",
          "pcnt-running": "100.00",
          "metric-value": "270.8",
          "metric-unit": "cs/sec  cs_per_second"
        },
        {
          "counter_value": "1",
          "event": "cpu-migrations",
          "event-runtime": "18464214",
          "pcnt-running": "100.00",
          "metric-value": "54.2",
          "metric-unit": "migrations/sec  migrations_per_second"
        },
        {
          "counter_value": "293",
          "event": "page-faults",
          "event-runtime": "18464214",
          "pcnt-running": "100.00",
          "metric-value": "15868.5",
          "metric-unit": "faults/sec  page_faults_per_second"
        },
        {
          "counter_value": "18.46",
          "unit": "msec",
          "event": "task-clock",
          "event-runtime": "18464214",
          "pcnt-running": "100.00",
          "metric-value": "0.8",
          "metric-unit": "CPUs  CPUs_utilized"
        },
        {
          "counter_value": "97400",
          "event": "cpu_core/L1-dcache-load-misses/",
          "event-runtime": "4474789",
          "pcnt-running": "24.00",
          "metric-unit": "%  l1d_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_core/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "100.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "33800",
          "event": "cpu_core/LLC-loads/",
          "event-runtime": "4041251",
          "pcnt-running": "21.00",
          "metric-value": "23.0",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "219119",
          "event": "cpu_core/branch-misses/",
          "event-runtime": "6039659",
          "pcnt-running": "32.00",
          "metric-value": "1.7",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "13806218",
          "event": "cpu_core/branches/",
          "event-runtime": "8036623",
          "pcnt-running": "43.00",
          "metric-value": "747.7",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "29667039",
          "event": "cpu_core/cpu-cycles/",
          "event-runtime": "10010455",
          "pcnt-running": "54.00",
          "metric-value": "1.6",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "73107015",
          "event": "cpu_core/instructions/",
          "event-runtime": "11012254",
          "pcnt-running": "59.00",
          "metric-value": "2.5",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "22338480",
          "event": "cpu_core/dTLB-loads/",
          "event-runtime": "7949766",
          "pcnt-running": "43.00",
          "metric-value": "0.0",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/L1-icache-load-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  l1i_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/LLC-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branch-misses/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/branches/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/cpu-cycles/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/instructions/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_atom/dTLB-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
          "metric-unit": "%  dtlb_miss_rate"
        },
        {
          "unit": "9949487",
          "event": "53.00",
          "metric-unit": "TopdownL1 (cpu_core)"
        },
        {
          "metric-value": "15.9",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-value": "30.6",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "9949487",
          "event": "53.00",
          "metric-value": "7.4",
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "metric-value": "46.1",
          "metric-unit": "%  tma_retiring"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "TopdownL1 (cpu_atom)"
        },
        {
          "metric-unit": "%  tma_backend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-unit": "%  tma_retiring"
        }
      ],
      "perf_record_name": "1__1__1..test_stest__sord.perfdata",
      "perf_script_name": null,
      "perf_archive_name": "1__1__1..test_stest__sord.tar.bz2"
    }
  }
}
```
