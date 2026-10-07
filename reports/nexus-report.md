# nexus — Migration Readiness Report

- **Project:** `nexus` (NeXus — Neutron & X-ray Common Data Format)
- **Resolved upstream repository:** https://github.com/nexusformat/code.git
- **Reference platform:** x86_64 (local container, native)
- **Target platform:** riscv64 (cross-compiled; executed under `qemu-riscv64` user-mode emulation)
- **Workspace:** `/work/nexus-workspace`
- **Config file:** `/work/nexus-workspace/input.yml`

---

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Repository URL | https://github.com/nexusformat/code.git |
| Latest commit | `5b803b3a0014bd9759b3d846da3cd3c1cfafd7d5` ("Add build manual"), dated 2020-01-26 (~2444 days / ~6.7 years ago) |
| Total commits | 2128 (first commit 1997-08-07) |
| Latest tag / release | `v4.4.3` (also `v4.4.3-rc1`) |
| Branches | `master` + remote `4.0`, `4.1`, `4.2`, `4.3`, `4.4`, `test_4.4.4`, `Ticket447_vs2015_stack_variables`, `issue-364` |
| Activity | Dormant upstream / sporadic — no upstream commits since January 2020; distribution maintenance continues independently |
| Build systems | CMake (root `CMakeLists.txt`; `cmake_minimum_required 2.8.7`; `project(NeXus)`; `enable_testing()` at line 37; default `CMAKE_BUILD_TYPE=Debug`). Legacy autotools files also present (`Makefile.am`, `configure.ac`) |
| Test count | 19 CTest targets declared in `test/CMakeLists.txt` (C API `napi_test.c`, C++ `napi_test_cpp.cxx`, Fortran `napif*_test.f`, plus `leak_test1/2/3`, `test_nxunlimited.c`). Autotest framework also present (`testsuite.at`, `atlocal.in`) |
| CI | Absent (no `.github/`, `.travis.yml`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `appveyor.yml`, Jenkins/Drone) |
| Benchmarks | Absent |
| Documentation | Present (`doc/`, build-manual commit) |
| External dependencies | tclap, HDF5, ZLIB, JPEG, SWIG, TCL, LibXml2; optional/conditional: HDF4, MXML, pthread, readline, dl (`HAVE_DLFCN_H`), Fortran toolchain |
| Distro packages | Debian source package `nexus` exists (main; Debian Science Maintainers; version 4.4.3-11) with 6 Debian patches applied. Arch AUR / Yocto availability not confirmed |
| Forks with riscv patches | None found. GitHub search `nexusformat+riscv` = 0 results; `repo:nexusformat/code riscv` = 0 issues. The `test_4.4.4` branch is version work, not architecture work |
| Repository health | Upstream dormant since Jan 2020; no CI; no benchmarks; no upstream RISC-V patches/forks. Long-term maintenance risk is the main non-technical concern |
| Clone path | `/work/nexus-workspace/nexus` |

---

## 2. Platform-Specific Code Analysis

### Architecture macros

No x86, ARM, or RISC-V architecture macros were found (no `__x86_64__`, `__SSE*`, `__AVX*`, `__arm__`, `__aarch64__`, `__ARM_NEON*`, `__riscv*`, etc.) anywhere in `src/`, `include/`, `applications/`, `bindings/`, `test/`, or `third_party/`. The only macros are OS/compiler/feature guards and CMake-generated type-size definitions:

| Macro | File | Line | Guards | Category |
|---|---|---|---|---|
| `_WIN32` | src/napi.c | 50, 66 | Windows socket/lib includes, locking init | OS |
| `_MSC_VER` | src/nxxml.c | 38 | MSVC-specific include workaround | OS/compiler |
| `__VMS` | include/napiconfig.h | 8 | OpenVMS include/typedef handling | OS |
| `__VMS` | include/napi.h.in | 187 | Name-mangling | OS |
| `__FreeBSD__` | src/napi.c | 101 | Recursive-pthread mutex detection | OS |
| `HAVE_LIBPTHREAD` | src/napi.c | 95 | pthread locking backend | OS/feature |
| `HAVE_TLS` | src/napi.c | 292–339 | Thread-local-storage lock keys | OS/feature |
| `WITH_HDF5` | src/napi5.c / src/napi.c | 30 / 391 | Enable HDF5-backed NAPI5 backend | feature |
| `WITH_HDF4` | src/napi.c | 394, 424 | Enable legacy HDF4 backend | feature |
| `WITH_MXML` | src/nxxml.c / src/nxio.c | 26 / 29 | Mini-XML backend selection | feature |
| `HAVE_STDINT_H` / `HAVE_INTTYPES_H` | include/napiconfig.h | 21 / 23 | Fixed-width integer typedefs | compiler/feature |
| `__GNUC__` / `__clang__` | include/napi.h.in | 261 | Compiler attribute (deprecated/visibility) | compiler |
| `SIZEOF_VOIDP` | include/nxconfig.h.in | 30 | Pointer-width-dependent type sizing (CMake `@CMAKE_SIZEOF_VOID_P@`; `CMakeLists.txt:286`) | pointer size |
| `SIZEOF_INT` / `SIZEOF_LONG_INT` / `SIZEOF_LONG_LONG_INT` / `SIZEOF_SIZE_T` | include/nxconfig.h.in | 27–31 | Integer type sizing for the data model | type size |
| `PRINTF_INT64` / `PRINTF_UINT64` | include/nxconfig.h.in | 26 | printf format strings for 64-bit ints | type size |

### Vectorization intrinsics (source)

None. No `_mm_` / `_mm256_` / `_mm512_`, `immintrin.h`, NEON (`vld1` / `arm_neon.h`), RVV (`__riscv_v`), or AltiVec intrinsics exist in any source tree. No hand-written SIMD; the project relies on scalar/portable C and compiler auto-vectorization.

### Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `#ifdef _WIN32` (src/napi.c:50) | Windows | Winsock/locking headers & init only |
| `#ifdef __VMS` (include/napiconfig.h:8, napi.h.in:187) | OpenVMS | Include/typedef/mangling only |
| `#ifdef _MSC_VER` (nxxml.c:38) | MSVC | Small include workaround |
| `#if defined(PTHREAD_MUTEX_RECURSIVE) \|\| defined(__FreeBSD__)` (napi.c:101) | FreeBSD/POSIX | Recursive mutex capability detection |
| `#ifdef USE_FTIME` (napi.c:1980/1993, napi5.c:366) | Windows/legacy | Timestamp implementation (`ftime` vs `gmtime`) |
| `#ifdef NEED_TZSET` (napi.c:1989) | some libc | `tzset()` before `mktime` |
| `#ifdef SWAP_ENDIAN` (SNS_retriever.cpp:3,134) | enabled by default | Byte-swapping of histogram data (endianness) |
| `#ifdef __MOTOROLA_ENDIAN__` (contrib/applications/NXextract/src/membuf.cpp, 16 sites) | big-endian | Optional big-endian byte conversion |
| `#if HAVE_DLFCN_H` (dynamic_retriever.cpp:19) | POSIX | `dlopen` dynamic plugin loading, else stubbed |

Endianness semantics: `__MOTOROLA_ENDIAN__` and `SWAP_ENDIAN` only trigger big-endian paths; riscv64 is little-endian, so they are inactive (not a portability blocker). `SIZEOF_VOIDP` / `SIZEOF_*` are CMake-generated and resolve correctly on riscv64 LP64.

### Portability verdict

| Aspect | Verdict |
|---|---|
| No exceptions | N/A — project core is C; no C++ exception paths exercised in the measured code |
| Alignment safe | YES — no architecture-specific macros; size/type macros are CMake-generated for the target |
| Embedded usability | NOT AVAILABLE — no embedded target was assessed; default HDF5-backed design is heavyweight for embedded use |
| Overall portability risk | LOW — generically portable C/C++ with no architecture-specific code paths, no SIMD intrinsics, and no architecture-gated dependencies. Migration to riscv64 is expected to be a standard LP64 build; risk is concentrated in build-system/toolchain configuration and unmaintained-upstream drift rather than the source itself |

---

## 3. Build & Test Results

| Build name | Role | Platform | Toolchain | Result | Tests run | Passed | Failed |
|---|---|---|---|---|---|---|---|
| `1_1_1` | Seed recipe 1 (HDF5 on) | x86_64 | `/usr/bin/cc` | FAIL (configure) | 0 | 0 | 0 |
| `1_1_2` | Seed recipe 2 (HDF5 on) | x86_64 | `/usr/bin/cc` | FAIL (configure) | 0 | 0 | 0 |
| `1_1_3` | Reference (x86_64 baseline) | x86_64 | system GCC | OK | 2 | 1 | 1 |
| `1_1_4` | Target (riscv64 baseline) | riscv64 (cross + qemu) | `riscv64-linux-gnu-gcc/g++` 15.2.0 | OK | 2 | 1 | 1 |
| `1_1_5` | Reference (x86_64 optimized) | x86_64 | system GCC | OK | 2 | 1 | 1 |
| `1_1_6` | Target (riscv64 optimized) | riscv64 (cross + qemu) | `riscv64-linux-gnu-gcc/g++` 15.2.0 | OK | 2 | 1 | 1 |

Built executables: `test/leak_test1` and `test/test_nxunlimited` (plus `src/libNeXus.so.1.0.0` and `src/libNeXus.a`) for every successful build. Build trees were placed by Amphimixis at `/work/<build_name>/` (i.e. `/work/1_1_3`, `/work/1_1_4`, `/work/1_1_5`, `/work/1_1_6`).

### Build/test failures detail

- `1_1_1` / `1_1_2` fail at CMake configure. The first error is the CMake policy check (CMake 4.2.3): `Compatibility with CMake < 3.5 has been removed from CMake ... Or, add -DCMAKE_POLICY_VERSION_MINIMUM=3.5 to try configuring anyway.` With the policy bypassed, the next blocker is missing HDF5: `Could NOT find HDF5 (missing: HDF5_LIBRARIES HDF5_INCLUDE_DIRS)`. These seed recipes were preserved byte-for-byte from the provided `input.yml` and are expected to fail in this environment.
- `NAPI-C-leak-test-1` (`leak_test1`) fails identically on both platforms with `ERROR: Attempt to create HDF5 file when not linked with HDF5 / NXopen failed!` because the buildable recipes disable HDF5 (`-DENABLE_HDF5=OFF`). The test's `NXopen(..., NXACC_CREATE5, ...)` cannot work without HDF5. Actual measured behaviour under this configuration: 0 loop iterations, exit code 1.
- `NAPI-C-test-nxunlimited` (`test_nxunlimited`) "passes" vacuously: its entire body is guarded by `#ifdef WITH_HDF4 / WITH_MXML / WITH_HDF5`; with all three file formats disabled, `main` compiles to `return 0`. It verifies launch/link only, not file-format behaviour.
- Build warning (both platforms, 2 occurrences each): `src/napi_fortran_helper.c:150:9: warning: 'nxicompress_' is deprecated [-Wdeprecated-declarations]`. Also CMake deprecation warning for `cmake_minimum_required(VERSION 2.8.7)`.
- `-DBUILD_TESTING=ON` is reported by CMake as unused; the project builds tests unconditionally via `enable_testing()` + `add_subdirectory(test)`.
- Emulation: riscv test execution required the manual fallback `QEMU_LD_PREFIX=/usr/riscv64-linux-gnu LD_LIBRARY_PATH=<build>/src` because no binfmt_misc handler is registered. Without it, both riscv tests fail before `main` (`Could not open '/lib/ld-linux-riscv64-lp64d.so.1'`).

---

## 4. Performance Comparison

> **QEMU / EMULATION CAVEAT (read first):** the riscv64 target does not run on real RISC-V hardware. All riscv64 metrics (`cycles`, `instructions`, `branches`, cache/TMA counters) are **host x86_64 PMU counters measured on the `qemu-riscv64` process**, not guest RISC-V retired events. Timing includes QEMU user-mode emulation overhead and does **not** reflect native RISC-V hardware performance. RISC-V results are suitable for correctness/smoke checks only, not for hardware performance conclusions.

### Experimental conditions (baseline Phase 4 run)

| Condition | Value |
|---|---|
| CPU | Intel Core i7-13620H (13th Gen), 16 logical CPUs, max 4.9 GHz |
| CPU pinning | `taskset -c 6` (P-core) — same core for every run on both platforms |
| Priority | `nice -n -20` was DENIED (`setpriority: Permission denied`; no `CAP_SYS_NICE`). All runs at default nice 0 |
| Governor / frequency | `powersave`; observed `scaling_cur_freq` 0.40–1.60 GHz vs max 4.7–4.9 GHz — frequency not fixed |
| Warmup | 2 runs per executable/platform (matched) |
| Measurement runs | 10 per executable/platform (matched) |
| perf | 7.0.14; `perf_event_paranoid = -1` (already unrestricted) |
| Event set | `task-clock,cycles,instructions,branches,branch-misses,L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses`; TMA `perf stat -M TopdownL1` |
| Reference execution | native x86_64 |
| Target execution | `QEMU_LD_PREFIX=/usr/riscv64-linux-gnu LD_LIBRARY_PATH=<build>/src qemu-riscv64 <bin>` |

**Workload reality:** `test_nxunlimited` compiles to a no-op (body compiled out) and `leak_test1` aborts at its first `NXopen` under `-DENABLE_HDF5=OFF` (0 loop iterations). Both measured profiles are therefore dominated by **process startup, dynamic loader and syscalls**, not by NeXus library compute. `libNeXus` accounts for only ~0.35% of x86 samples.

### Key metrics — `leak_test1` (baseline `1_1_3` vs `1_1_4`)

| Metric | x86_64 (native) | riscv64 (QEMU) |
|---|---:|---:|
| Elapsed (ms) | 0.755 (0.545) | 29.615 (1.309) |
| task-clock (ms) | 0.550 (0.161) | 29.121 (1.196) |
| User time (ms) | 0.173 | 26.354 |
| Sys time (ms) | 0.609 | 3.473 |
| Instructions retired | 983,255 | 146,979,830 |
| Cycles | 1,103,653 | 63,017,111 |
| IPC | 0.930 | 2.334 |
| L1-dcache miss rate | 2.225% | 0.456% |
| LLC miss rate | 19.36% | 36.90% |
| Branch misprediction rate | 3.693% | 1.916% |
| Frontend Bound | 46.53% | 46.28% |
| Backend Bound | 23.80% | 10.16% |
| Bad Speculation | 10.34% | 12.32% |
| Retiring | 19.33% | 31.25% |

### Key metrics — `test_nxunlimited` (baseline `1_1_3` vs `1_1_4`)

| Metric | x86_64 (native) | riscv64 (QEMU) |
|---|---:|---:|
| Elapsed (ms) | 0.864 (0.074) | 24.580 (1.705) |
| task-clock (ms) | 0.417 (0.022) | 24.188 (1.489) |
| User time (ms) | 0.264 | 22.376 |
| Sys time (ms) | 0.640 | 2.482 |
| Instructions retired | 887,161 | 121,197,531 |
| Cycles | 881,386 | 51,916,369 |
| IPC | 1.009 | 2.335 |
| L1-dcache miss rate | 2.223% | 0.465% |
| LLC miss rate | 17.09% | 40.19% |
| Branch misprediction rate | 3.922% | 1.906% |
| Frontend Bound | 49.69% | 47.63% |
| Backend Bound | 20.10% | 10.76% |
| Bad Speculation | 10.33% | 11.89% |
| Retiring | 19.87% | 29.72% |

### Hotspot tables

**Reference platform (x86_64) — DSO distribution**

| DSO | `leak_test1` weight % | `test_nxunlimited` weight % | Analysis |
|---|---:|---:|---|
| `[kernel]` | 59.6% | 70.4% | syscalls/page-faults during exec + dynamic linking and exec/exit |
| `ld-linux-x86-64.so.2` | 35.1% | 27.1% | dynamic loader / tunables / relocation — dominates startup |
| `libc.so.6` | 4.7% | 2.1% | `memcpy`, `mempcpy`, `_IO_vfprintf`, printf-family init |
| `libNeXus.so.1.0.0` | 0.35% | 0.0% | only project-library samples (`nxiopen_` path before the HDF5 failure) |
| other (`seq`/`dash`) | 0.3% | 0.1% | loop harness |

Function-level x86 names are best-effort nearest-symbol (`__tunable_get_val` ~20%, `_dl_rtld_di_serinfo` ~5.5%, `_dl_fatal_printf` ~4.8%, `_dl_mcount` ~2.9%, plus project symbols `nxiopen_`, `main`). Treat as approximate.

**Target platform (riscv64, under QEMU)** — `perf` profiles the host `qemu-riscv64` process; native RISC-V guest events are not observable.

| DSO | `leak_test1` % | `test_nxunlimited` % | Analysis |
|---|---:|---:|---|
| `qemu-riscv64` | 87.6% | 88.1% | QEMU TCG translate/execute — the emulator itself |
| `[kernel]` | 9.4% | 9.4% | QEMU's host syscalls/page-faults |
| `[unknown-mapping]` | 3.0% | 2.4% | transient/deleted mappings |
| ld/libc/dash | <0.1% | <0.1% | harness |

RISC-V function-level names are NOT AVAILABLE: `/usr/bin/qemu-riscv64` is stripped and has no debuginfo; guest RISC-V code is invisible to host perf.

### Bottleneck summary

1. **QEMU user-mode emulation dominates the riscv measurement** (87–88% of samples in `qemu-riscv64`). The 136–150× host-instruction blow-up is the fingerprint of TCG translating each guest instruction. This is an emulation artifact, not native RISC-V performance.
2. **x86 time is startup/loader-bound** (59–70% kernel + 27–35% `ld.so`), not compute-bound. IPC ~0.9–1.0 and 46–50% Frontend Bound reflect fetch/relocation/syscall-heavy startup.
3. **No NeXus library work is measured on either platform** — `leak_test1` aborts after the first failed `NXopen`, `test_nxunlimited` is a no-op. Any conclusion about the library's migration performance from these executables would be invalid.

### Causal conclusions

1. The 29–39× elapsed / 53–58× CPU-time increase on riscv is caused by QEMU dynamic binary translation, not by the RISC-V ISA. Host IPC is higher on riscv (2.33 vs 0.93–1.01) because QEMU's TCG loop is denser/more cache-resident than x86's loader+syscall startup path — it is the emulator's IPC, not the program's.
2. Higher riscv LLC miss rate (1.9–2.35×) and lower L1-d miss rate (0.21×) are explained by the emulator's working set: the translation-block cache / TCG structures add LLC pressure while the hot TCG inner loop has strong L1 temporal locality. x86 LLC figures are noisy (SD 13 pp for `leak_test1`).
3. Lower branch-misprediction on riscv (0.49–0.52×) reflects QEMU's interpreter dispatch mix and cannot be extrapolated to native RISC-V branch prediction.
4. Top-down matches this: both platforms are Frontend-Bound (~46–50%), but riscv-QEMU shows lower Backend Bound (10–11% vs 20–24%) and higher Retiring (30–36% vs 19–22%), consistent with the emulator's denser TCG loop versus x86 syscall/relocation stalls.
5. Smaller riscv binaries are a compressed-instruction / linker-segment-layout effect, not a speed indicator (see Section 5).

### Vectorization intrinsics (source-level and binary-level)

No hand-written intrinsics. Auto-vectorization only:

| Architecture | Binary | Vector insns (baseline) | ISAs found |
|---|---|---:|---|
| x86 | `src/libNeXus.so.1.0.0` | 51 (tool) / 116 VEX-like (objdump) | SSE + AVX2 (`vmovaps`, `vxorps`, `vxorpd`, `vpermd`, `vpcmp`) |
| x86 | `test/leak_test1`, `test/test_nxunlimited` | 0 | none |
| riscv | `src/libNeXus.so.1.0.0` | 129 (objdump fallback) | RVV 1.0 (`vsetvli`, `vle8ff.v`, `vle8.v`, `vse8.v`) |
| riscv | `test/test_nxunlimited` | 8 | RVV 1.0 |
| riscv | `test/leak_test1` | 0 | none |

`amphimixis-analyze-vectorization` succeeded for x86 but failed on the RISC-V ELF; counts were obtained with the `riscv64-linux-gnu-objdump` fallback.

### Cross-tables

Each sub-block below is copied verbatim from the corresponding tool-owned `cross-tables/CT-*.md` file. Column headers are rewritten to the file basenames; row cells are unchanged.

#### Cross-table — x86_leak_test1 vs riscv_leak_test1 — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_leak_test1 vs riscv_leak_test1 — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_leak_test1 vs riscv_leak_test1 — EVENT: CYCLES

| Symbol | x86_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_loader_leak_test1_loop vs riscv_leak_test1_loop — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_loader_leak_test1_loop % | riscv_leak_test1_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/             |      0.66 |     95.83 |  +95.17 |
| \[unknown\]                |     95.23 |      4.15 |  -91.08 |
| strlen                     |      0.32 |      0.00 |   -0.32 |
| \_dl\_allocate\_tls\_init  |      0.32 |      0.00 |   -0.32 |
| wcscmp                     |      0.32 |      0.00 |   -0.32 |
| nxiopen\_                  |      0.32 |      0.00 |   -0.32 |
| strchrnul                  |      0.17 |      0.00 |   -0.17 |
| \_\_do\_global\_dtors\_aux |      0.17 |      0.00 |   -0.17 |
| \_\_stpncpy                |      0.17 |      0.00 |   -0.17 |
| \_\_strncasecmp\_l         |      0.17 |      0.00 |   -0.17 |
| strrchr                    |      0.17 |      0.00 |   -0.17 |
| memrchr                    |      0.17 |      0.00 |   -0.17 |
| main                       |      0.16 |      0.00 |   -0.16 |
| wcsnlen                    |      0.16 |      0.00 |   -0.16 |
| wmemcmp                    |      0.15 |      0.00 |   -0.15 |

#### Cross-table — x86_loader_leak_test1_loop vs riscv_leak_test1_loop — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_loader_leak_test1_loop % | riscv_leak_test1_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/        |     24.13 |     28.78 |   +4.65 |
| \_dl\_debug\_state (/ |      3.86 |      0.00 |   -3.86 |
| \[unknown\]           |     72.01 |     71.22 |   -0.79 |

#### Cross-table — x86_loader_leak_test1_loop vs riscv_leak_test1_loop — EVENT: CYCLES

| Symbol | x86_loader_leak_test1_loop % | riscv_leak_test1_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/        |      0.31 |     89.63 |  +89.32 |
| \[unknown\]           |     97.20 |     10.34 |  -86.85 |
| wcscat                |      0.30 |      0.00 |   -0.30 |
| memrchr               |      0.15 |      0.00 |   -0.15 |
| wcslen                |      0.15 |      0.00 |   -0.15 |
| wcscmp                |      0.15 |      0.00 |   -0.15 |
| memcmp                |      0.15 |      0.00 |   -0.15 |
| memcpy                |      0.15 |      0.00 |   -0.15 |
| strpbrk               |      0.15 |      0.00 |   -0.15 |
| \_\_tunable\_get\_val |      0.15 |      0.00 |   -0.15 |
| \_\_call\_tls\_dtors  |      0.14 |      0.00 |   -0.14 |
| wmemchr               |      0.14 |      0.00 |   -0.14 |
| \_IO\_fwrite          |      0.14 |      0.00 |   -0.14 |
| \_\_strncasecmp\_l    |      0.14 |      0.00 |   -0.14 |
| \_IO\_do\_write       |      0.14 |      0.00 |   -0.14 |

#### Cross-table — x86_loader_leak_test1 vs riscv_leak_test1 — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_loader_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_loader_leak_test1 vs riscv_leak_test1 — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_loader_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_loader_leak_test1 vs riscv_leak_test1 — EVENT: CYCLES

| Symbol | x86_loader_leak_test1 % | riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/ |     11.65 |      0.00 |  -11.65 |
| \[unknown\]    |     88.35 |    100.00 |  +11.65 |

#### Cross-table — x86_loader_test_nxunlimited_loop vs riscv_test_nxunlimited_loop — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_loader_test_nxunlimited_loop % | riscv_test_nxunlimited_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/         |      0.76 |     95.85 |  +95.09 |
| \[unknown\]            |     95.74 |      4.11 |  -91.63 |
| pthread\_mutex\_unlock |      0.36 |      0.00 |   -0.36 |
| \_\_memmove\_chk       |      0.31 |      0.00 |   -0.31 |
| \_\_cxa\_finalize      |      0.18 |      0.00 |   -0.18 |
| \_\_tunable\_get\_val  |      0.17 |      0.00 |   -0.17 |
| \_\_stpcpy             |      0.17 |      0.00 |   -0.17 |
| strrchr                |      0.17 |      0.00 |   -0.17 |
| \_\_vdso\_getrandom    |      0.17 |      0.00 |   -0.17 |
| memcmp                 |      0.17 |      0.00 |   -0.17 |
| register\_tm\_clones   |      0.17 |      0.00 |   -0.17 |
| strncpy                |      0.17 |      0.00 |   -0.17 |
| wcslen                 |      0.17 |      0.00 |   -0.17 |
| strnlen                |      0.17 |      0.00 |   -0.17 |
| wcpncpy                |      0.16 |      0.00 |   -0.16 |

#### Cross-table — x86_loader_test_nxunlimited_loop vs riscv_test_nxunlimited_loop — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_loader_test_nxunlimited_loop % | riscv_test_nxunlimited_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]              |     61.72 |     73.82 |  +12.09 |
| \_\_tunable\_get\_val (/ |      4.62 |      0.00 |   -4.62 |
| execve (/                |      4.27 |      0.00 |   -4.27 |
| \[unknown\] (/           |     29.39 |     25.59 |   -3.80 |
| wcpncpy (/               |      0.00 |      0.60 |   +0.60 |

#### Cross-table — x86_loader_test_nxunlimited_loop vs riscv_test_nxunlimited_loop — EVENT: CYCLES

| Symbol | x86_loader_test_nxunlimited_loop % | riscv_test_nxunlimited_loop % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] (/        |      0.34 |     87.60 |  +87.26 |
| \[unknown\]           |     97.29 |     12.38 |  -84.91 |
| \_\_vdso\_getrandom   |      0.44 |      0.00 |   -0.44 |
| register\_tm\_clones  |      0.31 |      0.00 |   -0.31 |
| strncmp               |      0.31 |      0.00 |   -0.31 |
| \_\_ctype\_init       |      0.28 |      0.00 |   -0.28 |
| \_\_libc\_early\_init |      0.16 |      0.00 |   -0.16 |
| wcslen                |      0.16 |      0.00 |   -0.16 |
| \_\_stpcpy            |      0.14 |      0.00 |   -0.14 |
| strnlen               |      0.14 |      0.00 |   -0.14 |
| strcat                |      0.14 |      0.00 |   -0.14 |
| wcscmp                |      0.14 |      0.00 |   -0.14 |
| strlen                |      0.14 |      0.00 |   -0.14 |
| \_IO\_file\_finish (/ |      0.00 |      0.02 |   +0.02 |

#### Cross-table — x86_loader_test_nxunlimited vs riscv_test_nxunlimited — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_loader_test_nxunlimited % | riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |      3.62 |  -96.38 |
| \[unknown\] (/ |      0.00 |     96.38 |  +96.38 |

#### Cross-table — x86_loader_test_nxunlimited vs riscv_test_nxunlimited — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_loader_test_nxunlimited % | riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |     69.64 |  -30.36 |
| \[unknown\] (/ |      0.00 |     30.36 |  +30.36 |

#### Cross-table — x86_loader_test_nxunlimited vs riscv_test_nxunlimited — EVENT: CYCLES

| Symbol | x86_loader_test_nxunlimited % | riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |      6.39 |  -93.61 |
| \[unknown\] (/ |      0.00 |     93.61 |  +93.61 |

#### Cross-table — x86_leak_test1 vs opt_x86_leak_test1 — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_leak_test1 % | opt_x86_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_leak_test1 vs opt_x86_leak_test1 — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_leak_test1 % | opt_x86_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_leak_test1 vs opt_x86_leak_test1 — EVENT: CYCLES

| Symbol | x86_leak_test1 % | opt_x86_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_test_nxunlimited vs opt_x86_test_nxunlimited — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | x86_test_nxunlimited % | opt_x86_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_test_nxunlimited vs opt_x86_test_nxunlimited — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | x86_test_nxunlimited % | opt_x86_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\] |    100.00 |    100.00 |   +0.00 |

#### Cross-table — x86_test_nxunlimited vs opt_x86_test_nxunlimited — EVENT: CYCLES

| Symbol | x86_test_nxunlimited % | opt_x86_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |     62.20 |    100.00 |  +37.80 |
| \[unknown\] (/ |     37.80 |      0.00 |  -37.80 |

#### Cross-table — riscv_leak_test1 vs opt_riscv_leak_test1 — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | riscv_leak_test1 % | opt_riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |      7.46 |  -92.54 |
| \[unknown\] (/ |      0.00 |     92.54 |  +92.54 |

#### Cross-table — riscv_leak_test1 vs opt_riscv_leak_test1 — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | riscv_leak_test1 % | opt_riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |     60.27 |  -39.73 |
| \[unknown\] (/ |      0.00 |     39.73 |  +39.73 |

#### Cross-table — riscv_leak_test1 vs opt_riscv_leak_test1 — EVENT: CYCLES

| Symbol | riscv_leak_test1 % | opt_riscv_leak_test1 % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |    100.00 |     17.10 |  -82.90 |
| \[unknown\] (/ |      0.00 |     82.90 |  +82.90 |

#### Cross-table — riscv_test_nxunlimited vs opt_riscv_test_nxunlimited — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol | riscv_test_nxunlimited % | opt_riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |      3.62 |      8.95 |   +5.33 |
| \[unknown\] (/ |     96.38 |     91.05 |   -5.33 |

#### Cross-table — riscv_test_nxunlimited vs opt_riscv_test_nxunlimited — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol | riscv_test_nxunlimited % | opt_riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |     69.64 |     41.62 |  -28.02 |
| \[unknown\] (/ |     30.36 |     58.38 |  +28.02 |

#### Cross-table — riscv_test_nxunlimited vs opt_riscv_test_nxunlimited — EVENT: CYCLES

| Symbol | riscv_test_nxunlimited % | opt_riscv_test_nxunlimited % | Delta % |
|:--|--:|--:|--:|
| \[unknown\]    |      6.39 |     12.00 |   +5.61 |
| \[unknown\] (/ |     93.61 |     88.00 |   -5.61 |

---

## 5. Optimization Results

### Optimizations applied (Phase 6)

An optimized build pair was added to the config and built with the same build system and tests as the baseline:

- **x86_64 optimized (`1_1_5`, recipe 5):** `-O3 -march=x86-64-v3 -flto=auto -fno-semantic-interposition -fno-plt -g0`
- **riscv64 optimized (`1_1_6`, recipe 6):** `-O3 -march=rv64gc -flto=auto -fno-semantic-interposition -fno-plt -g0`
- Both keep CMake `RelWithDebInfo` with `-DENABLE_HDF5=OFF -DCMAKE_POLICY_VERSION_MINIMUM=3.5`. Note: `RelWithDebInfo` appends `-O2 -g -DNDEBUG` after the recipe flags, so the **effective optimization is `-O2` and debug info remains enabled** on both baseline and optimized builds (last `-O`/`-g` wins). `-march`, `-flto`, `-fno-semantic-interposition`, `-fno-plt` did take effect.
- Rationale from profiling: target the measured dynamic-loader/startup bottleneck (PLT/symbol-resolution work), make the RISC-V ISA explicit (the toolchain default silently included V/B), and reduce code size (LTO).

### Vector instructions in binary (baseline → optimized)

| Architecture | Binary | Vector insns (baseline) | Vector insns (optimized) | ISAs found |
|---|---|---:|---:|---|
| x86 | `libNeXus.so.1.0.0` | 51 | 49 | AVX2 retained (`vpermd`, `vmovaps`, `vpcmp`, `vpaddd`) |
| x86 | `test_nxunlimited` | 0 | 0 | none |
| x86 | `test/leak_test1` | 0 | 0 | none |
| riscv | `libNeXus.so.1.0.0` | 178 RVV | 0 | `-march=rv64gc` removed all RVV |
| riscv | `test_nxunlimited` | 10 RVV | 0 | none |
| riscv | `test/leak_test1` | 0 | 0 | none |

The riscv baseline silently targeted `rv64...v1p0` (GCC 15.2.0 default `-march` includes V/B), so it required the V extension; the optimized `-march=rv64gc` build is baseline-safe but **lost all RVV auto-vectorization**.

### Executable / library sizes (bytes)

| Build | Artifact | Unstripped | `.text` | Stripped |
|---|---|---:|---:|---:|
| 1_1_3 (x86 baseline) | `test/leak_test1` | 22,464 | 553 | 14,464 |
| 1_1_5 (x86 optimized) | `test/leak_test1` | 21,488 | 553 | 14,328 |
| 1_1_3 (x86 baseline) | `test/test_nxunlimited` | 21,048 | 563 | 14,464 |
| 1_1_5 (x86 optimized) | `test/test_nxunlimited` | 18,672 | 249 | 14,328 |
| 1_1_3 (x86 baseline) | `src/libNeXus.so.1.0.0` | 169,824 | 18,571 | 51,928 |
| 1_1_5 (x86 optimized) | `src/libNeXus.so.1.0.0` | 193,216 | 21,819 | 51,120 |
| 1_1_4 (riscv baseline) | `test/leak_test1` | 15,848 | 634 | 6,544 |
| 1_1_6 (riscv optimized) | `test/leak_test1` | 16,752 | 714 | 6,544 |
| 1_1_4 (riscv baseline) | `test/test_nxunlimited` | 13,928 | 524 | 6,544 |
| 1_1_6 (riscv optimized) | `test/test_nxunlimited` | 13,272 | 246 | 6,544 |
| 1_1_4 (riscv baseline) | `src/libNeXus.so.1.0.0` | 160,952 | 13,150 | 39,816 |
| 1_1_6 (riscv optimized) | `src/libNeXus.so.1.0.0` | 193,080 | 15,640 | 39,040 |

The raw stripped-executable difference (x86 14,464 vs riscv 6,544 B) is largely an ELF segment/page-padding artifact, not code size: real `.text` is 553 vs 634 (tests) and 18,571 vs 13,150 (baseline library).

### Optimization attempts (Before / After / Delta / Causal Analysis)

Measured on the same host/core with the same 2-warmup / 10-run methodology. All riscv values are host x86 counters measured on `qemu-riscv64` (emulation-bound).

| Optimization | Platform | Metric | Before | After | Delta | Causal Analysis |
|---|---|---|---|---|---|---|
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | real_time (`leak_test1`, ms) | 0.7547 | 0.4810 | −36.27% | Apparent gain is dominated by reduced dynamic-linking/PLT work at startup; x86 elapsed timing is itself unreliable (sub-microsecond artifacts), so task-clock is the robust proxy |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | task-clock (`leak_test1`, ms) | 0.5500 | 0.5120 | −6.91% | Median essentially flat (0.45→0.46 ms); the mean shift is outlier-driven |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | sys time (`leak_test1`, ms) | 0.6089 | 0.3284 | −46.07% | Directly targets the loader/PLT startup hotspot; strongest evidence of a real effect |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | cycles (`leak_test1`) | 1,103,653.4 | 952,313.5 | −13.71% | Fewer startup cycles with much lower variance; consistent with reduced relocation/PLT work |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | instructions (`leak_test1`) | 983,254.9 | 964,125.9 | −1.95% | Small instruction reduction; compute in NeXus is never reached |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | IPC (`leak_test1`) | 0.9299 | 1.0139 | +9.04% | Higher IPC from the changed startup mix, not from NeXus compute |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | real_time (`test_nxunlimited`, ms) | 0.8641 | 0.6554 | −24.15% | Elapsed artifact; robust counters (task-clock/cycles) show no real gain |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | task-clock (`test_nxunlimited`, ms) | 0.4170 | 0.4250 | +1.92% | Within noise — flags do not uniformly help a no-op workload |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | cycles (`test_nxunlimited`) | 881,386.4 | 902,670.9 | +2.41% | Within noise; slightly negative |
| ISA pin + LTO + `-fno-plt` + `-fno-semantic-interposition` | x86_64 | instructions (`test_nxunlimited`) | 887,160.8 | 897,358.8 | +1.15% | Within noise |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | real_time (`leak_test1`, ms) | 29.6150 | 27.9794 | −5.52% | Emulation-bound; cannot be attributed to better RISC-V code — this build removed all RVV vectorization |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | task-clock (`leak_test1`, ms) | 29.1210 | 27.6190 | −5.16% | Host counters on the qemu process; reflects simpler guest code for the translator plus temporal drift |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | cycles (`leak_test1`, host) | 63,017,111.0 | 59,744,901.5 | −5.19% | QEMU TCG host cycles, not guest cycles |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | instructions (`leak_test1`, host) | 146,979,830.3 | 140,825,913.1 | −4.19% | Host x86 instructions executed by QEMU; emulation artifact |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | real_time (`test_nxunlimited`, ms) | 24.5796 | 24.2627 | −1.29% | Flat within emulation noise |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | task-clock (`test_nxunlimited`, ms) | 24.1880 | 23.8050 | −1.58% | Flat within emulation noise |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | cycles (`test_nxunlimited`, host) | 51,916,368.7 | 52,152,835.9 | +0.46% | Flat within emulation noise |
| ISA pin (rv64gc) + LTO + `-fno-plt` + `-fno-semantic-interposition` | riscv64 (QEMU) | instructions (`test_nxunlimited`, host) | 121,197,530.9 | 121,540,222.3 | +0.28% | Flat within emulation noise |
| riscv ISA change (`rv64...v1p0` default → `rv64gc`) | riscv64 | RVV vector instructions in `libNeXus` | 178 | 0 | −100% | Correctness/portability fix — removes the implicit V-extension requirement — but a likely real-hardware performance regression because auto-vectorization is lost |

### Improvement of 1_1_3 compared to 1_1_5

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|:--|--:|--:|--:|:--|
| leak_test1 | 0.7547 | 0.481 | 63.73 | real_time |
| test_nxunlimited | 0.8641 | 0.6554 | 75.85 | real_time |

### Improvement of 1_1_4 compared to 1_1_6

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|:--|--:|--:|--:|:--|
| leak_test1 | 29.615 | 27.9794 | 94.48 | real_time |
| test_nxunlimited | 24.5796 | 24.2627 | 98.71 | real_time |

### Recommended optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--:|---|---|:--:|---|
| 1 | Fix the benchmarks: enable HDF5, un-gate `test_nxunlimited` body and the `leak_test1` loop | Enables all real measurement (no perf claim) | Medium | Without this every number profiles the loader. Prerequisite for any NeXus optimization. |
| 2 | Explicit riscv `-march`/`-mcpu` matching deployment (e.g. `-march=rv64gc` baseline; `-mcpu` for V-capable cores) | Correctness (avoids SIGILL on non-V cores); native perf gain not measurable under QEMU | Low | Toolchain default silently emits V + `zicond`/`zicbom`/`zvbb`; must be explicit. |
| 3 | Remove dynamic-loader cost: `-static` / `-static-libgcc -static-libstdc++`, `-fno-semantic-interposition`, `-fno-plt`, `-Wl,-O1,--as-needed` | Startup reduction; direction verifiable only on native x86 | Low–Med | Directly targets the 27–35% `ld-linux` + kernel-startup hotspot; not timeable for riscv under QEMU. |
| 4 | LTO + IPO (`-flto -fuse-linker-plugin`) tested separately from static linking | Code-size reduction + cross-TU inlining | Medium | Test LTO alone first, then combine. |
| 5 | Pin reproducible x86 ISA (`-march=x86-64-v3`/`v2`) instead of `-march=native` | Reproducibility | Low | Removes host dependence of the reference binary. |
| 6 | Strip debug info in release/packaging (`strip`, `-g1`/`-g0`) | Package/disk size only (debug is 69–75% of the library); no runtime gain | Low | Debug sections are not loaded; do not expect timing improvements. Keep a separate `.debug` file. |
| 7 | PGO (`-fprofile-generate` / `-fprofile-use`) once a real workload exists | Hot-path layout; unknown here | High | Requires fixed benchmark + native riscv. |
| 8 | Allocator swap (mimalloc/jemalloc/tcmalloc) once compute exists | Unknown; irrelevant to startup-only runs | Low | Must be cross-compiled for riscv; cannot validate under QEMU. |
| 9 | QEMU hygiene: `qemu-riscv64 -cpu max`; prefer native hardware | Measurement validity, not speed | Low | perf measures the qemu host process. |
| 10 | Vectorization introspection (`-fopt-info-vec-optimized`) | Confirms which loops vectorize; tune target ISA | Low | Source has no intrinsics — purely auto-vec tuning. |

Step-by-step application instructions (from the optimizer):

1. **Step 0 — Fix workloads (blocking).** Enable HDF5 in recipes (`-DENABLE_HDF5=ON`, install HDF5 dev libs for both hosts) and make `test_nxunlimited.c` / `leak_test1.c` execute real iterations; rebuild and re-profile.
2. **Step 1 — Explicit riscv ISA.** In the riscv recipe add `-march=rv64gc` (baseline-safe) or `-mcpu=<chip>` for V-capable hardware; verify with `riscv64-linux-gnu-readelf -A <lib> | grep Tag_RISCV_arch` and `riscv64-linux-gnu-objdump -d <lib> | grep -cE '\bv(setvli|le|se|add|mul|fmadd)'`.
3. **Step 2 — Pin x86 ISA** to `-march=x86-64-v3`.
4. **Step 3 — Reduce dynamic-link cost** with `-static-libgcc -static-libstdc++`, full `-static`, and/or `-fno-plt -fno-semantic-interposition -Wl,-O1,--as-needed`, each measured in isolation on native x86.
5. **Step 4 — LTO** (`-flto -fuse-linker-plugin`) measured alone, then combined with Step 3.
6. **Step 5 — Confirm vectorization** with `-fopt-info-vec-optimized=vec.log` for both ISAs.
7. **Step 6 — Size hygiene:** ship stripped binaries or `-g1`, keep debug info separately.
8. **Step 7 — Allocators** after Step 0 (cross-compile for riscv; not evaluable under QEMU).
9. **Step 8 — Emulation validity:** run `qemu-riscv64 -cpu max`, state emulation-bound caveats, prefer native RISC-V hardware for timing decisions.

---

## 6. Notes About Exploration Process

**Errors, issues and special circumstances:**

1. **Ambiguous project name.** No URL was provided. "nexus" is highly ambiguous. The active repository was resolved to `nexusformat/code` because the Debian source package is literally named `nexus` (main, Debian Science, 4.4.3-11), upstream tag `v4.4.3` matches, and it is a C/C++ CMake project with tests/docs. Rejected candidates: `cnr-isti-vclab/nexus` (mesh lib), `graphql-nexus/nexus` (TS), `sonatype/nexus-public` (Java), `nexus-xyz/nexus-zkvm` (Rust), `X-Gen-Lab/nexus` (embedded).
2. **Dormant upstream.** No upstream commits since 2020-01-26; no CI; no benchmarks; no upstream RISC-V patches or forks. Distribution maintenance continues independently (6 Debian patches).
3. **Missing HDF5 dev libraries** for both x86_64 and riscv64 (`find / -name hdf5.h` and `libhdf5*` empty; `dpkg -l | grep hdf5` empty). The project does `find_package(HDF5 REQUIRED)` when `ENABLE_HDF5=ON` (default), so the provided seed recipes cannot configure. Workaround: `-DENABLE_HDF5=OFF` minimal build. This reduces test/feature coverage and means the measurements do not exercise the real HDF5-backed data path. zlib/libxml2/JPEG/TCL/SWIG dev packages are also absent (optional).
4. **CMake 4.2.3 rejects `cmake_minimum_required(VERSION 2.8.7)`**; `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` was required. As a result, seed builds `1_1_1`/`1_1_2` fail first on the CMake policy check (not HDF5 as initially predicted); with the policy bypassed they then fail on missing HDF5.
5. **`BUILD_TESTING` ignored** by the old project; tests build unconditionally via `enable_testing()`.
6. **Test reality:** `leak_test1` fails with `Attempt to create HDF5 file when not linked with HDF5` and actually runs 0 iterations; `test_nxunlimited` is compiled to a no-op (all format code `#ifdef`-ed out) and "passes" vacuously. Test pass counts overstate real coverage.
7. **No emulator support in Amphimixis; no binfmt handler.** `/proc/sys/fs/binfmt_misc` has no registered riscv handler and this Amphimixis build has no built-in emulator support, so `amixis run`/`amixis profile` fail for riscv with `Could not open '/lib/ld-linux-riscv64-lp64d.so.1'`. Manual fallback `QEMU_LD_PREFIX=/usr/riscv64-linux-gnu LD_LIBRARY_PATH=<build>/src qemu-riscv64 <bin>` (or `-L /usr/riscv64-linux-gnu`) was required and worked.
8. **`nice -n -20` denied** (no `CAP_SYS_NICE`); all measurements ran at default nice. CPU governor is `powersave` and frequency was not fixed; this adds noise. x86 `seconds time elapsed` is unreliable for these sub-millisecond runs (some values were sub-microsecond while task-clock was ~0.45 ms), so task-clock/cycles were used as robust proxies.
9. **perf symbolization unresolved.** Short 0.5–30 ms workloads and restricted kernel symbols (`kptr_restrict=1`) caused most samples to resolve to `[unknown]` (kernel/loader/qemu addresses). RISC-V guest events and function-level hotspot names are NOT AVAILABLE; `qemu-riscv64` is stripped with no debuginfo. Direct perf cross-tables therefore show `[unknown]`-dominated symbols; the loop harness cross-tables carry the usable symbol signal.
10. **`amphimixis-analyze-vectorization` fails on RISC-V ELF** (exit 1); the `riscv64-linux-gnu-objdump` fallback was used for RVV counts.
11. **Flag-order caveat:** CMake `RelWithDebInfo` appends `-O2 -g -DNDEBUG` after recipe flags, so effective optimization is `-O2` (not the nominal `-O3`) and debug info remains enabled (not `-g0`). This applies equally to baseline and optimized builds, so the relative comparison remains valid; the nominal intent was not achieved.
12. **Build-tree location:** Amphimixis placed build trees at `/work/<build_name>/` (not under the workspace directory). The profiler used these exact paths.
13. **Web verification limitation:** per-architecture Debian build status of HDF5, libjpeg-turbo and libxml2 could not be directly verified (`packages.debian.org`, `buildd.debian.org`, `sources.debian.org` returned an anti-bot page). Dependency portability was therefore assessed from Debian main membership and portable C/C++ design, not directly confirmed build logs. Arch AUR / Yocto availability of `nexus` itself is also unconfirmed; the optional Fortran toolchain is unverified.
14. **QEMU/emulation caveat (repeated for emphasis):** all riscv64 timings and counters include QEMU user-mode emulation overhead and do NOT reflect native RISC-V hardware performance. RISC-V results are correctness/portability evidence, not performance evidence.

---

## 7. Migration Readiness Summary

| Check | Status |
|---|---|
| Builds on reference (x86_64) | YES — `1_1_3` (baseline) and `1_1_5` (optimized) build successfully |
| Tests pass on reference | PARTIAL — 1 of 2; `leak_test1` fails by design because HDF5 is disabled in the available build configuration (expected) |
| Builds on target (riscv64) | YES — `1_1_4` (baseline) and `1_1_6` (optimized) cross-build successfully (genuine RISC-V ELF) |
| Tests pass on target | PARTIAL — 1 of 2; same expected HDF5-related `leak_test1` failure; the passing test is vacuous |
| Zero external dependencies | NO — HDF5 is required by default (plus ZLIB, JPEG, SWIG, TCL, LibXml2 and optional backends); HDF5/zlib/libxml2 dev packages are absent in this environment |
| No hand-written intrinsics | YES — no SIMD intrinsics in source; only compiler auto-vectorization |
| Alignment safe | YES — no architecture-specific macros; `SIZEOF_*`/`PRINTF_*` generated by CMake for the target; riscv64 is LP64 little-endian and the big-endian guards are inactive |
| Exceptions handled | N/A — core is C; no C++ exception paths in the measured code |
| Auto-vectorization | YES — x86 AVX2 (baseline and optimized); riscv baseline RVV 1.0, optimized scalar `rv64gc` |

### Migration Verdict: **MINOR CONCERNS**

The source code itself is highly portable (LOW risk): it contains no architecture-specific macros, no hand-written SIMD, and no architecture-gated dependencies, and it cross-compiles and links cleanly for riscv64. The concerns are environmental and methodological rather than architectural:

- The build here required disabling HDF5 (its dev libraries are absent), so the migration was validated only on a reduced/minimal configuration, and the project's tests do not exercise the real data path.
- The provided seed recipes (`1_1_1`, `1_1_2`) are not buildable as-is (CMake 4.2.3 policy + missing HDF5); buildable recipes were added.
- The upstream project is dormant since 2020 with no CI/benchmarks and no RISC-V patches; maintenance relies on distribution patches.
- The target toolchain's default `-march` silently requires the RISC-V V extension; without an explicit `-march=rv64gc` (or a target-specific `-mcpu`) the binaries would SIGILL on baseline rv64gc hardware. QEMU masks this.
- riscv64 performance could not be assessed on real hardware; all riscv measurements are QEMU emulation-bound.

### Required Actions

1. **Install HDF5 development libraries for both x86_64 and riscv64** and rebuild with `-DENABLE_HDF5=ON`; add `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` (or raise `cmake_minimum_required`) to the recipes. Re-run the pipeline on the full configuration.
2. **Provide real benchmarks / a real data-path test**: un-gate `test_nxunlimited`'s body and make `leak_test1` actually iterate, so performance is measured on NeXus compute rather than process startup.
3. **Pin the RISC-V ISA explicitly** (`-march=rv64gc` for baseline-safe deployments, or a specific `-mcpu` for known V-capable hardware) and verify with `readelf -A` / disassembly before shipping.
4. **Profile on native RISC-V hardware** (or an accelerated emulator) before drawing any performance conclusions; treat all current riscv timings as emulation-bound.
5. **Register a binfmt_misc RISC-V handler or upgrade Amphimixis** to support user-mode emulation automatically, so target test/profile runs work without manual `QEMU_LD_PREFIX` injection.
6. **Plan for upstream drift**: given the dormant upstream, track the Debian package patches (currently 6) or maintain a fork for RISC-V enablement and fixes.
7. **Adopt release hygiene**: strip debug info for shipped artifacts (debug is 69–75% of the library) and separate `.debug` files; pin x86 `-march` to a fixed level for reproducibility instead of `-march=native`.
8. **Re-evaluate auto-vectorization**: with `-march=rv64gc` all RVV was removed; if the deployment target has vector hardware, use `-mcpu=<chip>` so RVV is scheduled appropriately rather than disabling it.

---

## Appendix A: Recorded Profile Data (`nexus.json`)

The following is the tool-owned `/work/nexus.json`, presented as recorded (read-only; not modified):

```json
{
  "1_1_4": {
    "test/leak_test1": {
      "build_name": "1_1_4",
      "executable": "test/leak_test1",
      "executable_run_success": false,
      "real_time": null,
      "user_time": null,
      "kernel_time": null,
      "perf_stat": null,
      "perf_record_name": null,
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test/test_nxunlimited": {
      "build_name": "1_1_4",
      "executable": "test/test_nxunlimited",
      "executable_run_success": false,
      "real_time": null,
      "user_time": null,
      "kernel_time": null,
      "perf_stat": null,
      "perf_record_name": null,
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_3": {
    "test/leak_test1": {
      "build_name": "1_1_3",
      "executable": "test/leak_test1",
      "executable_run_success": false,
      "real_time": null,
      "user_time": null,
      "kernel_time": null,
      "perf_stat": null,
      "perf_record_name": null,
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test/test_nxunlimited": {
      "build_name": "1_1_3",
      "executable": "test/test_nxunlimited",
      "executable_run_success": true,
      "real_time": "0.00",
      "user_time": "0.00",
      "kernel_time": "0.00",
      "perf_stat": [
        {
          "counter_value": "4",
          "event": "context-switches",
          "event-runtime": "1945399",
          "pcnt-running": "100.00",
          "metric-value": "2056.1",
          "metric-unit": "cs/sec  cs_per_second"
        },
        {
          "counter_value": "1",
          "event": "cpu-migrations",
          "event-runtime": "1945399",
          "pcnt-running": "100.00",
          "metric-value": "514.0",
          "metric-unit": "migrations/sec  migrations_per_second"
        },
        {
          "counter_value": "198",
          "event": "page-faults",
          "event-runtime": "1945399",
          "pcnt-running": "100.00",
          "metric-value": "101778.6",
          "metric-unit": "faults/sec  page_faults_per_second"
        },
        {
          "counter_value": "1.95",
          "unit": "msec",
          "event": "task-clock",
          "event-runtime": "1945399",
          "pcnt-running": "100.00",
          "metric-value": "0.5",
          "metric-unit": "CPUs  CPUs_utilized"
        },
        {
          "counter_value": "16353",
          "event": "cpu_core/L1-dcache-load-misses/",
          "event-runtime": "275861",
          "pcnt-running": "14.00",
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
          "counter_value": "7886",
          "event": "cpu_core/LLC-loads/",
          "event-runtime": "1669538",
          "pcnt-running": "85.00",
          "metric-value": "37.7",
          "metric-unit": "%  llc_miss_rate"
        },
        {
          "counter_value": "23560",
          "event": "cpu_core/branch-misses/",
          "event-runtime": "1669538",
          "pcnt-running": "85.00",
          "metric-value": "3.5",
          "metric-unit": "%  branch_miss_rate"
        },
        {
          "counter_value": "669200",
          "event": "cpu_core/branches/",
          "event-runtime": "1669538",
          "pcnt-running": "85.00",
          "metric-value": "344.0",
          "metric-unit": "M/sec  branch_frequency"
        },
        {
          "counter_value": "4030987",
          "event": "cpu_core/cpu-cycles/",
          "event-runtime": "1669538",
          "pcnt-running": "85.00",
          "metric-value": "2.1",
          "metric-unit": "GHz  cycles_frequency"
        },
        {
          "counter_value": "3763908",
          "event": "cpu_core/instructions/",
          "event-runtime": "1669538",
          "pcnt-running": "85.00",
          "metric-value": "0.9",
          "metric-unit": "instructions  insn_per_cycle"
        },
        {
          "counter_value": "<not counted>",
          "event": "cpu_core/dTLB-loads/",
          "event-runtime": "0",
          "pcnt-running": "0.00",
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
          "unit": "0",
          "event": "0.00",
          "metric-unit": "TopdownL1 (cpu_core)"
        },
        {
          "metric-value": "0.0",
          "metric-unit": "%  tma_bad_speculation"
        },
        {
          "metric-unit": "%  tma_frontend_bound"
        },
        {
          "unit": "0",
          "event": "0.00",
          "metric-unit": "%  tma_backend_bound"
        },
        {
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
      "perf_record_name": "1__1__3..test_stest__nxunlimited.perfdata",
      "perf_script_name": null,
      "perf_archive_name": "1__1__3..test_stest__nxunlimited.tar.bz2"
    }
  }
}
```
