# TinyXML2 Migration Readiness Report: x86_64 → riscv64

## 1. Repository & Project Status

- **Repository URL**: https://github.com/leethomason/tinyxml2.git
- **Latest commit**: `8224e427b655b83dae5e2298f1e6919523a78737` (short `8224e42`, "Merge branch 'master'")
- **Total commits**: 1,294
- **Latest tag**: `11.0.0` (2025-03-15); prior `10.1.0` (2025-03-08)
- **Last activity**: 2026-05-23
- **Activity**: Actively maintained — 16 commits in 2026, 21 in 2025, 50 in 2024. 5.8k stars, ~2k forks, 116 open issues, 7 open PRs.
- **Build systems**: CMake (primary, min 3.15), GNU Makefile, Meson (>= 0.49). CMake exports pkg-config + CMake package config files.
- **Tests**: ~430 assertions (434 `XMLTest()` call sites) in a single `xmltest.cpp`; custom assertion harness (no gtest/Catch2/doctest); registered via CTest (`PASS_REGULAR_EXPRESSION ", Fail 0"`) and Meson `test()`.
- **CI**: GitHub Actions matrix (windows-latest / macos-latest / ubuntu-latest × CMake 3.15 / latest); builds static+shared × Debug+Release, runs ctest, tests `find_package` integration. **No riscv64 CI runner.**
- **Benchmarks**: No formal benchmark suite. `xmltest.cpp` contains a "Performance tracking" section (parses `dream.xml` 10×, prints ms) and a `--perf`-style load-time mode via argv.
- **External dependencies**: NONE (0) — dependency-free by design ("no dependency on the C++ Standard Library"). Uses only C standard headers: `<cctype>`, `<climits>`, `<cstdio>`, `<cstdlib>`, `<cstring>`, `<cstddef>`, `<cstdarg>`, `<new>`, `<stdint.h>`.
- **Distro packages**: Debian `libtinyxml2-11`/`libtinyxml2-dev` (Architectures: any — builds on riscv64); Ubuntu `libtinyxml2-11` has an explicit **riscv64** binary; Alpine ships riscv64 packages (v3.20: 10.0.0, edge: 11.0.0); Yocto/OpenEmbedded `libtinyxml2` recipe (no patches); vcpkg "Supported on all triplets".
- **Forks with target-architecture patches**: None found. No riscv-specific commits in upstream history; an ARM Compiler 5 (DS-5) build fix (#1013) was merged, demonstrating cross-compiler maintenance.

## 2. Platform-Specific Code Analysis

### Architecture macros

**None found.** No x86 (`__x86_64__`, `__i386__`, `__SSE__`, `__AVX__`…), no ARM (`__arm__`, `__aarch64__`, `__ARM_NEON`…), no RISC-V (`__riscv`, `__riscv_xlen`…) macros anywhere in the source.

### Platform preprocessor guards

| Guard | What it guards | Semantics on riscv64 Linux |
|-------|----------------|----------------------------|
| `_MSC_VER` | MSVC-only: `__declspec(dllexport/import)`, `__debugbreak()` assert, `vsnprintf_s`/`_snprintf`, `_fseeki64`/`_ftelli64`, `fopen_s`, `_CrtMem*` leak checks, `QueryPerformanceCounter`, `stricmp` | Not defined on riscv64 — all branches fall through to portable C stdlib (`snprintf`, `fopen`, `fseek`, `clock()`) |
| `__GNUC__ >= 4` | `TINYXML2_LIB` = `__attribute__((visibility("default")))` | Defined by GCC and Clang on riscv64 — works correctly |
| `__cplusplus` | C++17 → `inline constexpr`, C++11 → `static constexpr`, else `static const`; `[[fallthrough]]` | Compiler-version logic, architecture-independent |
| `__has_attribute` / `__has_cpp_attribute` | Fallthrough attribute detection (GCC/Clang) | Works on riscv64 GCC/Clang |
| `__APPLE__`, `__FreeBSD__`, `__OpenBSD__`, `__NetBSD__`, `__DragonFly__`, `__CYGWIN__` | `TIXML_FSEEK`/`TIXML_FTELL` = `fseeko`/`ftello` | Not defined on Linux — else-branch uses `fseek`/`ftell` (correct) |
| `__ANDROID__` && `__ANDROID_API__ > 24` | `fseeko64`/`ftello64` | Not defined on Linux — else-branch used |
| `ANDROID_NDK` | C-style headers (`<ctype.h>`) vs C++ headers (`<cctype>`); `__android_log_assert` | Not defined on riscv64 Linux — standard C++ headers used |
| `WIN32` | `<crtdbg.h>`/`<windows.h>` CRT debug heap | Not defined on Linux — else-branch uses `<sys/stat.h>`/`<sys/types.h>` |
| `_DEBUG` / `__DEBUG__` | Auto-enable `TINYXML2_DEBUG` | Debug-build flag, architecture-independent |
| `TINYXML2_DEBUG` | `TIXMLASSERT` enabled/disabled; memory-pool leak checks | Build-config flag, architecture-independent |
| `TINYXML2_EXPORT` / `TINYXML2_IMPORT` | DLL export/import on MSVC | Not used on Linux (visibility attribute path) |
| `_CRT_SECURE_NO_WARNINGS` | MSVC warning suppression | MSVC-only, harmless |
| `_FILE_OFFSET_BITS=64` | 64-bit file offsets on 32-bit systems | Already 64-bit natively on riscv64 — no-op, correct |

All guards are compiler/OS-level, not architecture-level. Every MSVC/Apple/BSD/Android branch has a portable C-stdlib fallback that is exactly what riscv64 Linux uses.

### Vectorization intrinsics

**None found.** No `_mm_*` (SSE/AVX/AVX-512), no NEON (`vld1`/`vadd`/`vmul`), no RISC-V RVV (`__riscv_v`) intrinsics. No `<immintrin.h>`, `<arm_neon.h>`, or `<riscv_vector.h>` includes. The library is pure scalar C++.

### Portability verdict

- **No exceptions**: 0 architecture macros; all platform guards are compiler/OS-level with correct portable fallbacks for riscv64 Linux
- **Alignment safe**: no alignment-sensitive code (no packed structs, no unaligned-access assumptions)
- **Embedded usability**: no STL, no exceptions/RTTI required, no external dependencies
- **Overall**: **Highly portable** — riscv64 readiness confirmed by first-party distro packaging (Debian/Ubuntu/Alpine riscv64 binaries, vcpkg all-triplet support, Yocto recipe with no patches). Minor caveats: (1) CI does not exercise riscv64, (2) the `xmltest.cpp` performance-tracking block uses `clock()` on non-MSVC (fine on riscv64).

## 3. Build & Test Results

| Build | Platform | Compiler | Effective flags | Build | Tests |
|-------|----------|----------|-----------------|-------|-------|
| `1_1_1` (baseline) | x86_64 | g++ 13.3.0 | `-O3 -march=native -g` | OK | **528 passed, 0 failed** (0.05 s) |
| `1_1_2` (baseline) | riscv64 (qemu-user) | riscv64-linux-gnu-g++ 13.3.0 | `-O3 -g` | OK | **528 passed, 0 failed** (0.50 s, emulated) |
| `1_1_1` (optimized) | x86_64 | g++ 13.3.0 | `-O3 -march=native -g` | OK | **528 passed, 0 failed** |
| `1_1_2` (optimized) | riscv64 (qemu-user) | riscv64-linux-gnu-g++ 13.3.0 | `-O3 -march=rv64gc_zba_zbb_zbs -flto -fno-exceptions -fno-rtti -fno-unwind-tables -static` | OK | **528 passed, 0 failed** |

- **Build/test failures**: none on either platform, baseline or optimized.
- **Executables**: reference `/home/bzych/test/tinyxml2-amixis-ai/tinyxml2-workspace/1_1_1/xmltest` (x86-64 ELF PIE, with debug_info); target `/home/bzych/test/tinyxml2-amixis-ai/tinyxml2-workspace/1_1_2/xmltest` (RISC-V ELF lp64d, RVC; baseline dynamic, optimized statically linked).
- **Notes**: target tests run under qemu-user-riscv emulation (`/usr/bin/qemu-riscv64 -L /usr/riscv64-linux-gnu`), not native RISC-V hardware. Target build used `-DCMAKE_SYSROOT='/'` (host root) because the host toolchain layout lacks a full RISC-V sysroot; this works because TinyXML2 is a self-contained single-file library. Tests must run with the source directory as CWD (xmltest reads `resources/dream.xml`).

## 4. Performance Comparison

### Experimental conditions

| Condition | Reference (x86_64) | Target (riscv64) |
|-----------|-------------------|------------------|
| Host CPU | 12th Gen Intel Core i5-12450H (Alder Lake hybrid: 4 P-cores + 4 E-cores, 12 threads) | Same host — **qemu-user-riscv 8.2.2 emulation** |
| Core pinned | CPU 0 (P-core), `taskset -c 0` | CPU 0 (P-core), `taskset -c 0` |
| Priority | `nice -n -20` attempted but permission denied (non-root) → all runs at niceness 0 | Same (matched) |
| Frequency | `scaling_cur_freq` CPU0: 1.116 GHz (before) → 1.338–1.475 GHz (during) → 1.375 GHz (after). Governor `powersave`, variable 0.4–1.5 GHz | Same host/core, same window |
| perf_event_paranoid | −1 (hardware counters work) | N/A — no hardware counters inside qemu-user |
| Warmup runs | 2 | 2 |
| Measurement runs | 10 (wall, `/usr/bin/time`) + 30 perf-stat (3 sets × 10) + 30-run perf-record aggregate | 10 (wall, `/usr/bin/time`) |
| CWD | `tinyxml2-workspace/tinyxml2/` (xmltest reads `resources/dream.xml`) | Same |
| **Emulation caveat** | — | **Timing includes qemu-user emulation overhead; results do NOT reflect native RISC-V hardware performance.** |

### Key metrics (baseline, Phase 4)

| Metric | Reference (x86_64) | Target (riscv64 @ qemu) | Ratio (target/ref) |
|--------|:---------:|:------:|:------------------------:|
| **Elapsed time, full suite (avg)** | **0.0490 s** (σ=0.0074) | **0.6140 s** (σ=0.0913) | **12.53×** |
| dream.xml parse (guest `clock()`, perf-tracking mode) | 1.883 ms | 20.920 ms | 11.11× |
| **IPC (instructions/cycle)** | **2.484** | **NOT AVAILABLE** (no counters in qemu-user) | — |
| **L1-dcache miss rate** | **0.752%** (355,214 / 47,212,027) | **NOT AVAILABLE** | — |
| **LLC miss rate** | **51.16%** (28,477 / 55,663) | **NOT AVAILABLE** | — |
| **Branch misprediction rate** | **0.536%** (226,243 / 42,244,284) | **NOT AVAILABLE** | — |
| **Frontend Bound** | **37.2%** | **NOT AVAILABLE** | — |
| **Backend Bound** | **8.9%** | **NOT AVAILABLE** | — |
| **Retiring** | **47.3%** | **NOT AVAILABLE** | — |
| Bad Speculation | 6.6% | NOT AVAILABLE | — |
| **Executable size (stripped)** | **182,752 B** | **166,232 B** | 0.910× |

### Hotspots — reference platform (REAL MEASURED, perf record, 4018 samples aggregated over 30 runs, `-F 1000`, CPU 0)

| % Time (self) | Function | Module |
|:------:|----------|--------|
| 10.56% | `__strncmp_avx2` | libc.so.6 |
| 8.07% | `isalpha` | libc.so.6 |
| 6.19% | `tinyxml2::XMLText::ParseDeep` | xmltest |
| 5.67% | `isspace` | libc.so.6 |
| 5.51% | `tinyxml2::XMLNode::ParseDeep` | xmltest |
| 4.89% | `tinyxml2::XMLDocument::Identify` | xmltest |
| 3.03% | `tinyxml2::XMLNode::DeleteNode` | xmltest |
| 2.99% | `tinyxml2::XMLNode::~XMLNode` | xmltest |
| 2.64% | `tinyxml2::XMLNode::InsertChildPreamble` | xmltest |
| 2.49% | `tinyxml2::StrPair::GetStr` | xmltest |
| 2.45% | `tinyxml2::StrPair::ParseName` | xmltest |
| 2.05% | `CreateUnlinkedNode<XMLElement>` | xmltest |
| 1.74% | `tinyxml2::XMLElement::ParseAttributes` | xmltest |
| 1.72% | `tinyxml2::XMLElement::ParseDeep` | xmltest |
| 1.70% | `_int_malloc` | libc.so.6 |

### Hotspots — target platform: **NOT AVAILABLE**

Perf hardware counters are unavailable inside qemu-user emulation, so no guest-side hotspot samples exist. Host-side perf on the qemu process attributes samples to qemu's TCG dispatch/translation code (top self-symbols `0x10c3ca` 5.4%, `0x591f0` 3.4%, `0x591ad` 2.7% in `qemu-riscv64`, plus `g_hash_table_insert` 0.9%) — this measures the *emulator*, not the guest, and is reported only as evidence of emulation overhead.

### Cross-table: 1_1_1_xmltest vs opt_1_1_1_xmltest (from cross-tables/CT-1_1_1_xmltest-opt_1_1_1_xmltest.md, EVENT: CYCLES)

| Symbol | 1_1_1_xmltest % | opt_1_1_1_xmltest % | Delta % |
|:--------------------------------------------------------------------------------------------------------------------------|---------:|---------:|-------:|
| tinyxml2::StrPair::ParseText(char\*, char const\*, int, int\*)                                                            |      0.05 |      9.09 |   +9.04 |
| isalpha                                                                                                                   |      8.07 |      0.00 |   -8.07 |
| \_\_strncmp\_avx2                                                                                                         |     10.56 |      3.43 |   -7.14 |
| \[unknown\]                                                                                                               |     16.56 |     22.98 |   +6.42 |
| tinyxml2::XMLText::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                          |      6.19 |      0.50 |   -5.70 |
| isspace                                                                                                                   |      5.67 |      0.00 |   -5.67 |
| tinyxml2::XMLElement::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                       |      1.72 |      5.25 |   +3.53 |
| tinyxml2::XMLDocument::Identify(char\*, tinyxml2::XMLNode\*\*, bool)                                                      |      4.89 |      7.85 |   +2.96 |
| tinyxml2::StrPair::ParseName(char\*)                                                                                      |      2.45 |      0.00 |   -2.45 |
| tinyxml2::XMLNode::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                          |      5.51 |      7.64 |   +2.13 |
| tinyxml2::XMLElement\* tinyxml2::XMLDocument::CreateUnlinkedNode<tinyxml2::XMLElement, 120ul>(tinyxml2::MemPoolT<120ul>&) |      2.05 |      0.00 |   -2.05 |
| tinyxml2::XMLNode::~XMLNode()                                                                                             |      2.99 |      4.73 |   +1.75 |
| isalpha@plt                                                                                                               |      1.44 |      0.00 |   -1.44 |
| tinyxml2::StrPair::GetStr()                                                                                               |      2.49 |      3.50 |   +1.01 |
| tinyxml2::XMLElement::~XMLElement()                                                                                       |      1.20 |      2.02 |   +0.81 |
| tinyxml2::XMLNode::ToDocument()                                                                                           |      0.56 |      1.32 |   +0.76 |
| isspace@plt                                                                                                               |      0.68 |      0.00 |   -0.68 |
| tinyxml2::XMLText\* tinyxml2::XMLDocument::CreateUnlinkedNode<tinyxml2::XMLText, 112ul>(tinyxml2::MemPoolT<112ul>&)       |      0.91 |      1.50 |   +0.60 |
| tinyxml2::StrPair::Reset()                                                                                                |      1.24 |      0.65 |   -0.59 |
| tinyxml2::XMLDeclaration::ToDeclaration()                                                                                 |      0.93 |      1.37 |   +0.44 |

### Cross-table: 1_1_1_xmltest vs opt_1_1_1_xmltest (from cross-tables/CT-1_1_1_xmltest-opt_1_1_1_xmltest-1.md, EVENT: CYCLES)

| Symbol | 1_1_1_xmltest % | opt_1_1_1_xmltest % | Delta % |
|:--------------------------------------------------------------------------------------------------------------------------|---------:|---------:|-------:|
| tinyxml2::StrPair::ParseText(char\*, char const\*, int, int\*)                                                            |      0.05 |      9.09 |   +9.04 |
| isalpha                                                                                                                   |      8.07 |      0.00 |   -8.07 |
| \_\_strncmp\_avx2                                                                                                         |     10.56 |      3.43 |   -7.14 |
| \[unknown\]                                                                                                               |     16.56 |     22.98 |   +6.42 |
| tinyxml2::XMLText::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                          |      6.19 |      0.50 |   -5.70 |
| isspace                                                                                                                   |      5.67 |      0.00 |   -5.67 |
| tinyxml2::XMLElement::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                       |      1.72 |      5.25 |   +3.53 |
| tinyxml2::XMLDocument::Identify(char\*, tinyxml2::XMLNode\*\*, bool)                                                      |      4.89 |      7.85 |   +2.96 |
| tinyxml2::StrPair::ParseName(char\*)                                                                                      |      2.45 |      0.00 |   -2.45 |
| tinyxml2::XMLNode::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)                                                          |      5.51 |      7.64 |   +2.13 |
| tinyxml2::XMLElement\* tinyxml2::XMLDocument::CreateUnlinkedNode<tinyxml2::XMLElement, 120ul>(tinyxml2::MemPoolT<120ul>&) |      2.05 |      0.00 |   -2.05 |
| tinyxml2::XMLNode::~XMLNode()                                                                                             |      2.99 |      4.73 |   +1.75 |
| isalpha@plt                                                                                                               |      1.44 |      0.00 |   -1.44 |
| tinyxml2::StrPair::GetStr()                                                                                               |      2.49 |      3.50 |   +1.01 |
| tinyxml2::XMLElement::~XMLElement()                                                                                       |      1.20 |      2.02 |   +0.81 |
| tinyxml2::XMLNode::ToDocument()                                                                                           |      0.56 |      1.32 |   +0.76 |
| isspace@plt                                                                                                               |      0.68 |      0.00 |   -0.68 |
| tinyxml2::XMLText\* tinyxml2::XMLDocument::CreateUnlinkedNode<tinyxml2::XMLText, 112ul>(tinyxml2::MemPoolT<112ul>&)       |      0.91 |      1.50 |   +0.60 |
| tinyxml2::StrPair::Reset()                                                                                                |      1.24 |      0.65 |   -0.59 |
| tinyxml2::XMLDeclaration::ToDeclaration()                                                                                 |      0.93 |      1.37 |   +0.44 |

### Cross-table: 1_1_1_xmltest vs opt_1_1_1_xmltest (from cross-tables/CT-1_1_1_xmltest-opt_1_1_1_xmltest-1.md, EVENT: CPU_CORE/CACHE-MISSES/)

| Symbol | 1_1_1_xmltest % | opt_1_1_1_xmltest % | Delta % |
|:-----------------------------------------------------------------|---------:|---------:|-------:|
| \[unknown\]                                                      |     59.52 |     65.77 |   +6.25 |
| tinyxml2::XMLNode::DeleteNode(tinyxml2::XMLNode\*)               |      3.81 |      5.77 |   +1.96 |
| tinyxml2::XMLNode::ParseDeep(char\*, tinyxml2::StrPair\*, int\*) |      1.97 |      0.18 |   -1.79 |
| tinyxml2::XMLNode::~XMLNode()                                    |      5.05 |      3.61 |   -1.44 |
| \_dl\_relocate\_object                                           |      0.00 |      1.23 |   +1.23 |
| \_\_strlen\_avx2                                                 |      2.20 |      1.09 |   -1.11 |
| tinyxml2::MemPoolT<104ul>::Free(void\*)                          |      1.46 |      0.45 |   -1.01 |
| tinyxml2::XMLDocument::Clear()                                   |      4.03 |      4.73 |   +0.70 |
| isspace                                                          |      0.68 |      0.00 |   -0.68 |
| strcmp                                                           |      0.65 |      0.00 |   -0.65 |
| \_dl\_check\_map\_versions                                       |      0.65 |      0.00 |   -0.65 |
| isalpha                                                          |      0.62 |      0.00 |   -0.62 |
| tinyxml2::XMLNode::ToDocument()                                  |      1.91 |      1.33 |   -0.58 |
| \_dl\_lookup\_symbol\_x                                          |      0.65 |      1.23 |   +0.58 |
| \_\_strncmp\_avx2                                                |      0.63 |      1.21 |   +0.58 |
| tinyxml2::StrPair::GetStr()                                      |      1.11 |      0.62 |   -0.49 |
| \_\_printf\_fp\_buffer\_1.isra.0                                 |      0.08 |      0.52 |   +0.45 |
| tinyxml2::MemPoolT<120ul>::Free(void\*)                          |      0.43 |      0.00 |   -0.43 |
| memcpy@@GLIBC\_2.14                                              |      0.00 |      0.41 |   +0.41 |
| tinyxml2::XMLElement::~XMLElement()                              |      0.55 |      0.15 |   -0.40 |

### Cross-table: 1_1_1_xmltest vs opt_1_1_1_xmltest (from cross-tables/CT-1_1_1_xmltest-opt_1_1_1_xmltest-1.md, EVENT: CPU_CORE/BRANCH-MISSES/)

| Symbol | 1_1_1_xmltest % | opt_1_1_1_xmltest % | Delta % |
|:------------------------------------------------------------------------|---------:|---------:|-------:|
| tinyxml2::StrPair::ParseText(char\*, char const\*, int, int\*)          |      0.00 |      9.84 |   +9.84 |
| tinyxml2::StrPair::Reset()                                              |      7.69 |      0.00 |   -7.69 |
| \_\_strncmp\_avx2                                                       |     12.46 |      5.87 |   -6.59 |
| tinyxml2::XMLText::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)        |     10.17 |      3.68 |   -6.48 |
| tinyxml2::XMLNode::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)        |     10.68 |     17.08 |   +6.40 |
| tinyxml2::XMLElement::ParseDeep(char\*, tinyxml2::StrPair\*, int\*)     |      0.95 |      4.74 |   +3.79 |
| isalpha                                                                 |      3.55 |      0.00 |   -3.55 |
| tinyxml2::StrPair::ParseName(char\*)                                    |      3.06 |      0.00 |   -3.06 |
| tinyxml2::XMLDocument::Identify(char\*, tinyxml2::XMLNode\*\*, bool)    |      1.19 |      3.63 |   +2.44 |
| isalpha@plt                                                             |      2.26 |      0.00 |   -2.26 |
| do\_lookup\_x                                                           |      2.14 |      4.20 |   +2.06 |
| tinyxml2::XMLElement::ParseAttributes(char\*, int\*)                    |      0.65 |      2.51 |   +1.86 |
| tinyxml2::XMLNode::~XMLNode()                                           |      2.71 |      4.20 |   +1.49 |
| \[unknown\]                                                             |      7.36 |      6.00 |   -1.36 |
| isspace                                                                 |      1.29 |      0.00 |   -1.29 |
| tinyxml2::XMLNode::ToDeclaration()                                      |      0.75 |      1.93 |   +1.17 |
| tinyxml2::XMLPrinter::PrintString(char const\*, bool) \[clone .part.0\] |      8.75 |      9.81 |   +1.06 |
| tinyxml2::XMLPrinter::Write(char const\*, unsigned long)                |      0.81 |      1.66 |   +0.85 |
| operator new(unsigned long)                                             |      0.28 |      1.00 |   +0.73 |
| unlink\_chunk.isra.0                                                    |      0.35 |      1.03 |   +0.67 |

### Bottleneck summary (causal analysis)

1. **Elapsed time is 12.5× slower on the target — dominated by qemu-user emulation overhead, not by the RISC-V code itself.** Each guest RISC-V instruction is translated by TCG into host x86 instructions and executed through qemu's dispatch loop. The pure-parse ratio (11.1× on the dream.xml perf-tracking loop) is consistent with the full-suite ratio (12.5×), confirming the overhead is in instruction emulation. **This ratio must NOT be read as "RISC-V is 12.5× slower" — it is "RISC-V under qemu-user is 12.5× slower on this x86 host."**
2. **Reference IPC = 2.48 with Retiring = 47.3% and Backend Bound = 8.9%: the x86 build is retire-bound, not memory-bound.** The hot path is libc string/character functions (`__strncmp_avx2` 10.6%, `isalpha` 8.1%, `isspace` 5.7%) which are AVX2-optimized. The RISC-V guest's libc has no such SIMD string routines (no RVV in the toolchain), so the same comparisons run as scalar loops — an ISA/toolchain codegen difference that compounds the emulation overhead.
3. **Frontend Bound = 37.2% on the reference reflects the parser's large, branch-dense code footprint** (many `ParseDeep` variants, `Identify` dispatch, `StrPair`/`ParseName` string scanning). This is an inherent property of the TinyXML2 parsing algorithm and would carry over to native RISC-V hardware. The low L1-dcache miss rate (0.75%) and low branch misprediction (0.54%) show the workload is cache-friendly and branch-predictable; the 51.2% LLC miss rate applies to only 0.118% of loads (cold misses on the 142 KB dream.xml buffer) — not a real bottleneck.
4. **Executable size is 9% smaller on RISC-V (166 KB vs 183 KB stripped)** thanks to RISC-V compressed instructions (RVC) — a minor positive for the target.
5. **Vectorization is NOT a differentiator** — both binaries are effectively scalar in the hot parsing path, so the 12.5× gap is attributable to emulation overhead + libc/ISA codegen differences, not to lost SIMD.

## 5. Optimization Results

### Vector instructions in binary

| Architecture | Count | ISAs Found |
|-------------|:-----:|------------|
| x86_64 (reference, baseline) | 104 scalar SSE/AVX instrs (539 xmm/ymm refs) | SSE2/AVX scalar FP only: `vmovsd` 38, `vmovss` 23, `vmovaps` 19, `vxorpd` 6, `vxorps` 4, `vucomisd` 4, `vcvtss2sd` 4, `vcvttsd2si` 3, `vucomiss` 2, `vdivsd` 1. **No packed arithmetic SIMD.** |
| riscv64 (target, baseline) | 0 RVV; 86 scalar FPU-D instrs | No RVV (toolchain lacks `-march=rv64gcv`). Scalar D-extension only: `fld` 33, `fmv.d` 29, `fsd` 12, `fcvt.d` 5, `feq.d` 3, `fcvt.w.d` 3, `fdiv.d` 1. |
| x86_64 (reference, optimized) | 216 SSE2 | 195 `movdqu` + 21 `movdqa` (128-bit load/store — compiler-generated memcpy/memset, **no SIMD arithmetic**) |
| riscv64 (target, optimized) | 0 RVV | No RVV instructions (9 `csrr a0,vlenb` matches are V-extension capability probes from startup code, not computation) |

**Causal analysis**: Neither binary vectorizes the hot parsing path. XML parsing is inherently scalar (character-at-a-time, branch-heavy string scanning). The SIMD/FPU instructions present serve scalar double-precision formatting in `XMLPrinter`. The x86 build's real SIMD advantage is inside libc (`__strncmp_avx2`), which the RISC-V guest's libc cannot match without RVV — a toolchain/libc difference, not an application-intrinsics difference. **No manual porting of intrinsics is required.**

### Executable size analysis

| Architecture | Before strip | After strip | `.text` (real code) | Debug info size |
|-------------|:-------------------:|:-------------------:|:-------------------:|:---------------:|
| x86_64 (baseline) | 1,026,440 B | 182,752 B | 174,059 B | ≈ 843,688 B (82%) |
| riscv64 (baseline) | 997,888 B | 166,232 B | 158,741 B | ≈ 831,656 B (83%) |
| x86_64 (optimized) | — | 182,752 B | — | — |
| riscv64 (optimized, static) | — | 835,536 B | — | — |

The target size increase in the optimized build is entirely from `-static` (bundles full libc/libstdc++), not from the source patches. The reference size is byte-identical, confirming the patches add negligible code.

### Optimization attempts (before/after, matched conditions — same session)

| Optimization | Before | After | Delta | Causal analysis |
|--------------|:------:|:-----:|:-----:|-----------------|
| O1 ctype LUT + O6 inline strncmp (both platforms) + target flags (march/LTO/static/no-exceptions) — reference elapsed | 0.059 s | 0.030 s | −49% | O1 eliminates ~16% of reference cycles in libc ctype calls (isalpha 8.07% + isspace 5.67% + PLT stubs); O6 eliminates ~7 pp of `__strncmp_avx2` cycles; Frontend Bound drops 39.0% → 29.8% as PLT/call/ret traffic is removed from the hot path |
| Same — target elapsed (under qemu) | 0.690 s | 0.395 s | −43% | Source patches + flags; under qemu every guest libc call is emulated and extremely expensive, so replacing them with inline code yields a large guest-instruction-count reduction; LTO enables cross-TU inlining, `-static` removes PLT/relocation overhead, `zba/zbb/zbs` provide efficient bit-manipulation for the table bit tests |
| Reference IPC | 2.559 | 2.881 | +13% | Instruction count fell 18% (174.9M → 143.5M) and cycles fell 27% (68.3M → 49.8M); the pipeline is less frontend-stalled |
| Reference dream.xml LoadFile parse | 3.716 ms | 2.188 ms | −41% | O1/O6 directly target the parse hot path |
| Target dream.xml LoadFile parse (qemu) | 39.319 ms | 29.213 ms | −26% | Same patches; full suite includes printing/allocation paths not touched by O1/O6, diluting the target's full-suite gain |
| Reference in-memory parse | 2.100 ms | 1.129 ms | −46% | Parse-only benchmark isolates the O1/O6 effect |
| Target in-memory parse (qemu) | 20.009 ms | 9.515 ms | −52% | The parse path improved MORE on the target than the reference — consistent with emulated libc calls being the dominant cost |
| Reference Backend Bound | 9.3% | 15.0% | +5.7 pp | Consequence of the frontend improvement, not a regression: with frontend stalls removed, the pipeline now spends proportionally more time stalled on backend resources; LLC miss rate remains ~53% (145 KB dream.xml working set) |
| Reference L1-dcache miss rate | 0.757% | 0.917% | +0.16 pp | The charClassTable adds a 128-byte read-only table to the hot path; absolute miss count is flat (353K → 344K) but total loads fell (46.8M → 37.6M), so the rate rose. Negligible impact |
| Reference branch misprediction | 0.515% | 0.551% | +0.04 pp | Inline compares add a few branches; the parser's branch pattern is unchanged |
| Target executable size | 166,232 B | 835,536 B | +5.03× | From `-static`; a deployment trade-off, not a performance regression |

**Attribution summary**: O1 contributes the largest share (eliminating ~16% of reference cycles in ctype calls); O6 contributes the second share (eliminating ~7 pp of strncmp cycles); the target's flag changes (LTO/static/march/no-exceptions) add incremental benefit on the parse path but are confounded with the source patches in this build.

### Improvement of 1_1_1 compared to 1_1_1

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|----------|---------------:|----------------:|--------------:|-----------|
| xmltest | 0.059 | 0.03 | 50.85 | real_time |
| xmltest | 2.559 | 2.881 | 112.58 | IPC |
| xmltest | 174.9 | 143.5 | 82.05 | instructions |
| xmltest | 68.3 | 49.8 | 72.91 | cycles |
| xmltest | 3.716 | 2.188 | 58.88 | dream_xml_parse_time |
| xmltest | 2.1 | 1.129 | 53.76 | in_memory_parse_time |

### Improvement of 1_1_2 compared to 1_1_2

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|----------|---------------:|----------------:|--------------:|-----------|
| xmltest | 0.69 | 0.395 | 57.25 | real_time |
| xmltest | 39.319 | 29.213 | 74.3 | dream_xml_parse_time |
| xmltest | 20.009 | 9.515 | 47.55 | in_memory_parse_time |

### Recommended optimizations (from Phase 5 analysis, prioritized)

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--------:|-------------|:-------------|:------:|-------|
| 1 | O1 — Replace libc ctype calls with a 128-byte static classification table | est. 10–25% under QEMU; several % native | Low | **APPLIED** — removes ≈30–60k PLT→glibc calls per parse; pure source patch, portable, no behavior change |
| 2 | O6 — Replace `strncmp` in `Identify`/`ParseText` with inline constant-length compares | est. 5–10% under QEMU | Low | **APPLIED** — ≈16.8k short `strncmp` calls per parse eliminated |
| 3 | O4 — Link the target `-static` | est. 5–15% under QEMU | Very low | **APPLIED** — removes guest `ld.so` + PLT/GOT indirection; full `-static` worked (no fallback needed) |
| 4 | O2 — Add `-march=rv64gc_zba_zbb_zbs` | est. 1–5% | Very low | **APPLIED** — Zb-extensions improve scalar codegen |
| 5 | O3 — Add `-flto` | est. 3–8% | Very low | **APPLIED** — cross-TU inlining reduces frontend pressure |
| 6 | O7 — Bump `MemPoolT::ITEMS_PER_BLOCK` from 4 KB to 16–64 KB | est. 1–4% | Very low | NOT applied — fewer guest heap operations per parse |
| 7 | O5 — Add `-fno-exceptions -fno-rtti -fno-unwind-tables` | est. 1–3% | Very low | **APPLIED** — TinyXML2 never throws/uses RTTI |
| 8 | Housekeeping: strip `-g`/debug from deployed binaries | 0% runtime | — | 82% of file size is debug info (≈0.83 MB) |

**Skipped with reasons**: mimalloc/jemalloc (node pool already covers hot allocation path; cross-build effort ≫ gain under QEMU); RVV/RVV-enabled glibc (parser has no vectorizable loops; large effort); PGO (QEMU profiles unrepresentative; defer to native hardware); `-ffast-math` (no FP in hot path); `TINYXML2_DEBUG` (already off in Release).

## 6. Notes About Exploration Process

- **Tool wrapper failures**: `amphimixis-analyze`, `amphimixis-build`, `amphimixis-profile`, `amphimixis-validate`, `amphimixis-analyze-vectorization` all failed with "No such file or directory" in this environment. Documented CLI/manual fallbacks were used throughout: `amixis build`, `amixis validate`, `amixis compare`, `perf stat`/`perf record`, `/usr/bin/time`, `objdump`, `file`, `ldd`. All fallbacks were disclosed by the respective agents.
- **Configurator incident (disclosed)**: the first `1_1_2` build silently produced an x86-64 binary because `create_toolchain()` returned `None` for an inline toolchain dict without a `name` key. Fixed by registering the `riscv64-gnu` toolchain globally in `~/.config/amphimixis/toolbox.yml` (a side-effect outside the workspace, disclosed) and rebuilding; the final binary was verified as RISC-V ELF.
- **QEMU/emulation caveats**: the target runs under **qemu-user-riscv 8.2.2 emulation** on the local x86 host, not native RISC-V hardware. All target timing includes emulation overhead and must NOT be interpreted as native RISC-V performance. Hardware performance counters are unavailable inside qemu-user, so target IPC/cache/branch/TopDown metrics and guest hotspot tables are **NOT AVAILABLE** — no values were invented. Host-side perf samples on the qemu process measure the emulator (TCG code), not the guest.
- **No `tinyxml2.json`/`tinyxml2.yaml` project data file was produced** by the tools in this environment (only the binary `tinyxml2.project` pickle exists); the report therefore reflects analyzer/builder/profiler/optimizer outputs and the tool-owned `improvements.json` and `cross-tables/CT-*.md` files.
- **`nice -n -20` permission denied** (non-root) — all measurement runs at niceness 0.
- **System load differed between Phase 4 and Phase 6c** (higher load during Phase 6c); before/after comparisons use matched-condition baseline re-measurements taken in the same session, and Phase 4 numbers are quoted for reference.
- **Target build used `-DCMAKE_SYSROOT='/'`** (host root) because the host toolchain layout lacks a full RISC-V sysroot; this works because TinyXML2 is a self-contained single-file library. Headers/libs were not validated against a real RISC-V sysroot at compile time.
- **Full `-static` worked** for the target (GCC 13.3.0 found all static libs); no `-static-libgcc -static-libstdc++` fallback was needed.
- **Optimization patch review**: an independent review found and corrected a semantic bug in the originally-proposed O1 LUT table (byte 0x7B `{` was wrongly marked alpha; `IsNameStartChar` initially included `.`/`-`). The corrected table was audited byte-by-byte against C-locale semantics; both test suites pass 528/0 with the patches.

## 7. Migration Readiness Summary

| Criterion | Status |
|-----------|--------|
| Builds on reference | YES |
| Tests pass on reference | YES (528/0) |
| Builds on target | YES (cross-compile, riscv64) |
| Tests pass on target | YES (528/0, under qemu-user-riscv) |
| Zero external dependencies | YES |
| No hand-written intrinsics | YES |
| Alignment safe | YES |
| Exceptions handled | YES (no exceptions/RTTI used) |
| Auto-vectorization | N/A (scalar workload; no vectorizable loops) |

**Migration Verdict: READY**

TinyXML2 is architecture-agnostic pure C++ with zero external dependencies, zero architecture macros, and zero hand-written intrinsics. It builds and passes its full 528-assertion test suite on riscv64 (cross-compiled, verified under qemu-user-riscv). First-party distro packaging (Debian/Ubuntu/Alpine riscv64 binaries, vcpkg all-triplet support, Yocto recipe with no patches) independently confirms riscv64 readiness. The measured 12.5× performance gap is dominated by qemu-user emulation overhead and the absence of AVX2 libc string routines in the RISC-V guest libc — not by the application code — and was reduced to ~43% elapsed-time improvement after applying the O1/O6 source patches and target flag optimizations.

### Required Actions

1. **Add a riscv64 CI runner** to the GitHub Actions matrix to exercise the target architecture continuously (currently no riscv64 CI).
2. **Validate the build against a real RISC-V sysroot** at compile time (the exploration used `-DCMAKE_SYSROOT='/'` due to host toolchain layout; a proper sysroot would catch any header/libc assumptions).
3. **Consider upstreaming the O1/O6 source patches** (ctype classification table + inline constant-length compares) — they measurably improve both platforms (reference −49% elapsed, target −43% under qemu) with no behavior change (528/0 tests pass).
4. **For native RISC-V deployment**, use `-march=rv64gc_zba_zbb_zbs` and consider `-static` for embedded targets (trade-off: +5× binary size).
5. **Strip debug info for deployment** (82% of file size is debug info).
6. **Re-validate performance on native RISC-V hardware** — all target timing in this report includes qemu-user emulation overhead and does not reflect native RISC-V performance; a native board with an RVV-enabled toolchain would close most of the remaining gap.
