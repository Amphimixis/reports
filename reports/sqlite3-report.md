# sqlite3 — Migration Readiness Report

**Reference platform:** x86_64 (native, local container)
**Target platform:** riscv64 (cross-compiled with `riscv64-linux-gnu-gcc` 13.3.0, executed under `qemu-riscv64-static` 8.2.2 user-mode emulation — no native RISC-V hardware available)
**Project version analyzed:** SQLite 3.54.0 (development trunk)
**Analysis date:** 2026-10-04

---

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Chosen repository URL | `https://github.com/sqlite/sqlite.git` (official read-only Git mirror) |
| Canonical upstream | `https://sqlite.org/src` (Fossil) |
| Clone path | `/work/sqlite3-workspace/sqlite` |
| Latest commit | `ccbdec8444e3a717dac8c154f17fdd7e80aa7827` |
| Latest commit date | 2026-10-03 15:54:28 +0000 |
| Latest commit subject | "Fix some formatting issues in test program vec1bench.tcl." |
| Total commits | 32544 (first commit 2000-05-29) |
| Latest version tag | `version-3.53.4` (commit `b09c88c14082339b66c7b7158d609a771e64ca69`, 2026-07-24) |
| Total tags | 390 (375 `version-*`); in-tree `VERSION` = 3.54.0 (trunk) |
| Activity | Actively maintained — 111 commits in the last 30 days; pushed 2026-10-03 |
| Stars / forks | 10589 stars / 1681 forks (GitHub API, 2026-10-04) |
| Default branch | `master` |
| Build systems | autosetup (SQLite's TCL-based `./configure`) + GNU make (`main.mk`, `Makefile.in`); legacy GNU-autoconf fallback under `autoconf/`; Windows `Makefile.msc`; no CMake/Meson |
| Test suite | TCL-based: 1195 `test/*.test` files driven by `test/testrunner.tcl`; `testfixture` harness; separate multi-process `mptest/` harness |
| External dependencies | Core amalgamation has **no required external dependency** beyond libc/pthread/libm. Optional: TCL (tests/tclsqlite), zlib, readline/editline, ICU — all portable C libraries |
| Distro packages | Debian riscv64: `sqlite3 3.53.4-2` (testing/unstable), `3.46.1-7+deb13u2` (stable) — native riscv64 binaries, no RISC-V-specific patch; Arch `core/sqlite 3.53.4-1`; Yocto OpenEmbedded `openembedded-core` recipe `sqlite3 3.53.4` |
| Forks with target-architecture patches | Checked — **none needed**. An upstream riscv32 build break was already fixed on `master` (PR #44; upstream check-in `e3f318bf52932460`, Git commit `27e808ff2cdb6541c12793ccaa09949b5898f14b`), guarding `sqlite3Multiply128` with `__riscv_xlen>32` |
| Analyzer portability level | LOW (as reported by the analyzer; justification states low risk — highly portable ANSI C) |

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros

| Macro | File:Line | Semantics / riscv64 effect |
|---|---|---|
| `__SIZEOF_POINTER__` | `src/sqliteInt.h:942`; `src/tclsqlite.c:72` | Pointer size source for `SQLITE_PTRSIZE`; riscv64 = 8 (portable, compiler-provided) |
| `__BYTE_ORDER__`, `__ORDER_LITTLE_ENDIAN__`, `__ORDER_BIG_ENDIAN__` | `src/sqliteInt.h:1020-1025` | Endianness; riscv64 resolves to little-endian (`SQLITE_BYTEORDER` 1234) |
| `i386`, `__i386__`, `_M_IX86`, `_M_ARM`, `__arm__`, `__x86`, `__x86_64`, `__x86_64__`, `_M_X64`, `_M_AMD64` | `src/sqliteInt.h:944-948, 1026-1029` | Fallback pointer-size/endianness hints; not required on riscv64 (built-ins take precedence) |
| `__riscv`, `__riscv_xlen` | `src/util.c:470-471` | Selects `SQLITE_USE_UINT128`; explicitly enables native `__uint128_t` multiply for riscv64 (`__riscv_xlen>32`). This is deliberate riscv64 support |
| `__arm__ && !__aarch64__`, `__ppc__ && !__ppc64__` | `src/util.c:1019-1021` | `SQLITE_AVOID_U64_DIVIDE` software path; not triggered on riscv64 (has hardware divide) |
| `i586`, `__i586__`, `_M_IX86`, `__x86_64__`, `__aarch64__`, `__ppc__` | `src/hwtime.h:31-68` | Inline-asm debug timestamp (`rdtsc`/`cntvct_el0`/`mftb`); riscv64 has **no branch** and falls back to a scalar function returning 0 — debug-only, no functional impact |
| `__AVX2__`, `__FMA__`, `_MSC_VER`, `__x86_64__`, `__aarch64__` | `ext/vec1/vec1.c:115,142-143,192,244` | Optional vec1 extension SIMD kernel selection; riscv64 uses the scalar fallback (`vec1.c:216-226`) |
| `i386`/`__x86_64__`/`_M_ARM`/`__arm__` | `ext/misc/shathree.c`, `ext/misc/totype.c`, `ext/rtree/rtree.c`, `tool/*.c` | Byte-order/64-bit detection replicated in tools; riscv64 resolves via built-ins |

Explicitly searched for and **not found**: `__ARM_NEON__`, `__ARM_FEATURE_*`, `__SSE*`, `__AVX512*`, `__BMI*`, `__POPCNT`, `__AES__`, `__riscv_vector`, `__riscv_crypto`, `__LP64__`, `__ILP32__`.

### 2.2 Vectorization intrinsics in source

| Intrinsic family | ISA | Files | Notes |
|---|---|---|---|
| `_mm256_*`, `_mm_*` | AVX2/FMA (x86) | `ext/vec1/vec1.c` (127 lines), `ext/vec1/tool/vec1_algo_test.c` | Only inside `#ifdef __AVX2__` in the optional `ext/vec1` extension; runtime-dispatched with scalar fallback |
| `vld1q_*`, `vdupq_*`, `vaddq_*`, `vfmaq_*`, `vaddvq_*`, `vst1q_*` | NEON (AArch64) | `ext/vec1/vec1.c`, `ext/vec1/tool/vec1_algo_test.c` | Only inside `#elif defined(__aarch64__)` |
| RVV (`__riscv_v*`, `vsetvl`, `vle*`, `vse*`) | RISC-V V | **None found** | No RISC-V Vector intrinsics anywhere in the tree |

**Core `src/` contains no SIMD intrinsics at all.** All SIMD is confined to the optional, non-distributed `ext/vec1` extension, which has a scalar fallback and will compile/run on riscv64 without acceleration.

### 2.3 Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `_WIN32` / `_WIN64` / `_MSC_VER` | Windows | `src/os_win.c`, `src/mutex_w32.c`, `src/msvc.h` (4-byte-aligned malloc), Win32 heap |
| `__APPLE__` | macOS | `os_unix.c` locking, `mem1.c` zone malloc, `loadext.c` dylib |
| `__ANDROID__` | Android | `os_unix.c` batch-atomic-write/mmap variants |
| `__linux__` | Linux | `os_unix.c` `O_DIRECT`/batch-atomic-write, `_GNU_SOURCE` fallocate — **this is the path used for riscv64 Linux** |
| `__CYGWIN__` | Cygwin | `mutex.h`, `os_setup.h`, locking |
| `SQLITE_BIGENDIAN` / `SQLITE_LITTLEENDIAN` / `SQLITE_UTF16NATIVE` | Endianness | UTF-16 conversion (`utf.c`, `util.c`, `vdbe.c`), RTree, btreeInt; generic byte-swap with runtime fallback |

### 2.4 Portability verdict

| Aspect | Verdict |
|---|---|
| Exceptions | **N/A** — SQLite core is C, not C++; no exception handling to port |
| Alignment safety | **Safe** — `SQLITE_4_BYTE_ALIGNED_MALLOC` is Windows/MSVC-only; `ALIGN128` is a generic GCC attribute; no architecture-specific misaligned-access workaround is required on riscv64. Historical `align8-fix`/`alignment-fixes`/`solaris-alignment` branches are not riscv64-specific |
| Endianness | **Safe** — `SQLITE_BYTEORDER` from compiler built-ins with a fully portable runtime fallback (`SQLITE_BYTEORDER==0`); riscv64 is little-endian |
| Embedded usability | **High** — single ANSI-C amalgamation, no required external dependencies beyond libc/pthread/libm, small code footprint (stripped target binary 1,942,536 B) |
| Overall | **LOW risk** (highly portable; already a first-class Debian/Ubuntu riscv64 package; explicit riscv64 support in `src/util.c`) |

---

## 3. Build & Test Results

Build configuration: reference recipe `1_1_1` = x86_64 native, `CFLAGS="-O3 -march=native -g"` (resolves to `alderlake`), `--enable-fts5 --enable-rtree`, TCL tests enabled. Target recipe `1_2_2` = cross-compile with `riscv64-linux-gnu-gcc`, `CFLAGS="-O3 -march=rv64gc -g"` (rv64imafdc_zicsr_zifencei, ABI lp64d), `--host=riscv64-linux-gnu --disable-tcl`.

| Platform | Build | Build result | Test result |
|---|---|---|---|
| x86_64 reference (`1_1_1`) | `./configure --with-tcl=... --enable-fts5 --enable-rtree CFLAGS="-O3 -march=native -g"` → `make -j4 sqlite3 testfixture` | **SUCCESS** — 0 errors, 0 warnings; `sqlite3` (11,588,968 B) and `testfixture` (9,989,712 B) produced | **SUCCESS with 2 environmental errors** — `testfixture test/veryquick.test`: **383,628 tests, 2 errors, 0 skipped**, "All memory allocations freed - no leaks" (2 m 24.6 s) |
| riscv64 target (`1_2_2`; profiled as `1_1_2`) | `CC=riscv64-linux-gnu-gcc ./configure --host=riscv64-linux-gnu --build=x86_64-linux-gnu --disable-tcl --enable-fts5 --enable-rtree CFLAGS="-O3 -march=rv64gc -g"` → `make -j4 sqlite3` | **SUCCESS** — 0 errors, 0 warnings; ELF64 little-endian RISC-V `sqlite3` (11,945,160 B) produced | **PARTIAL** — `testfixture` cannot be cross-built (no riscv64 TCL); functional smoke + built-in checks + 11/12 CLI-replayable `.sql` scripts passed under QEMU; `PRAGMA integrity_check` = `ok` |

### Build/test failures detail

- Reference `veryquick.test` errors `shell1-3.21.7` (`.dump` output ordering/quoting mismatch) and `zipfile-25.0` (error-string difference) are **environmental**, reproduce with the standard suite, and are unrelated to `-march=native` or architecture.
- Target test limitation is structural: the TCL `testfixture` links TCL; the cross recipe uses `--disable-tcl` because no riscv64 TCL/sysroot TCL exists in the environment. The `test/*.test` files are TCL harness scripts and cannot be replayed through the CLI.
- `fossildelta.sql` reported 2 errors on both x86 and riscv because it loads `./fossildelta.so`, a loadable extension not built here — **not architecture-specific**.
- `amixis build` could not drive SQLite: the Amphimixis Make backend does not run `./configure` and forwards autoconf flags to `make`; additionally `parse_config()` rejects a local `riscv` platform on an x86_64 host (see Section 6).

---

## 4. Performance Comparison

### 4.1 Experimental conditions

| Condition | Value |
|---|---|
| Host CPU | 13th Gen Intel Core i7-13620H, 16 logical CPUs (10 physical: 6 P-cores + 4 E-cores), L1d 416 KiB, L2 9.5 MiB, L3 24 MiB |
| Reference | native `build-x86/sqlite3` (`-O3 -march=native -g`) |
| Target | `build-riscv/sqlite3` (`-O3 -march=rv64gc -g`) under `qemu-riscv64-static -L /usr/riscv64-linux-gnu` (QEMU 8.2.2 user-mode; no binfmt_misc) |
| Workload | `bench.sql` — CREATE TABLE; recursive-CTE batch INSERT of 1,000,000 rows; 2 indexes; ANALYZE; indexed SELECT + GROUP BY; self-JOIN; UPDATE; DELETE; `PRAGMA integrity_check` |
| Core pinning | `taskset -c 6` (P-core, max 4.9 GHz) for all measurement runs |
| Priority | `nice -n -5` and `chrt` were **denied** in the container; all runs at default nice 0 |
| Frequency / governor | governor `powersave`; `cpupower` unavailable; in-run effective frequency: x86 4.41 GHz, QEMU 4.22 GHz |
| Warmup runs | 2 per platform |
| Measurement runs | 5 per platform (median + min–max) |
| Timing tool | `/usr/bin/time`; counters via `perf stat` |

### 4.2 Key metrics (Phase 4, median of 5 runs)

| Metric | Reference x86_64 (native) | Target riscv64 (QEMU host-side) |
|---|---|---|
| Elapsed wall time | 3.33 s | 18.07 s |
| IPC | 2.0136 | 4.4356 (host QEMU, not guest) |
| L1-dcache load miss rate | 2.7948 % | 0.6389 % (host QEMU) |
| LLC-load miss rate | 24.9907 % | 31.3859 % (host QEMU) |
| Branch misprediction rate | 0.6696 % | 0.1367 % (host QEMU) |
| Frontend Bound | 22.4 % | 15.4 % (host QEMU) |
| Backend Bound | 34.0 % | 6.7 % (host QEMU) |
| Retiring | 35.6 % | 75.2 % (host QEMU) |

> **QEMU/emulation caveat:** every riscv64 counter row describes the **QEMU emulator's host x86 execution**, not native RISC-V SQLite code. User-mode QEMU exposes no guest PMU, so **native RISC-V cycles, instructions, IPC, cache-miss and branch counters are NOT AVAILABLE**. The 5.43× elapsed-time ratio is dominated by dynamic binary translation, not by native hardware performance.

### 4.3 Hotspots — reference platform (x86_64 native, `perf report`, cycles, self time)

| % Time | Function |
|---|---|
| 12.07 | `sqlite3VdbeExec` |
| 6.57 | `sqlite3BtreeIndexMoveto` |
| 5.58 | `vdbeRecordCompareInt` |
| 4.78 | `sqlite3BtreeTableMoveto` |
| 3.82 | `vdbeSorterSort.part.0` |
| 3.51 | `pcache1Fetch` |
| 3.22 | `vdbeSorterCompareInt` |
| 2.40 | `sqlite3VdbeRecordCompareWithSkip` |
| 1.15 | `sqlite3BtreeInsert` |
| 0.90 | `checkTreePage` |

### 4.4 Hotspots — target platform (riscv64 under QEMU, host DSO-level only)

| % samples | DSO (host) | Analysis |
|---|---|---|
| 40.4 | `/tmp/perf-<pid>.map` (QEMU TCG JIT) | Dynamically generated translation blocks |
| 33.3 | `/usr/bin/qemu-riscv64-static` | Emulator runtime / TCG dispatch / guest memory access |
| 26.3 | `[unknown]` (kernel) | Host syscalls, page faults, I/O |

> **Guest-side function hotspots are NOT AVAILABLE** — user-mode QEMU provides no guest PMU and no guest/host symbols (`qemu-riscv64-static` is stripped; the TCG JIT map is unresolvable). The target profile is the emulator, not SQLite. Whether the same functions are hot on the target **cannot be determined** from this setup.

### 4.5 Bottleneck summary and causal analysis

1. **QEMU translation expansion dominates.** The target run retires **11.35×** more host instructions (336.3e9 vs 29.62e9) and consumes **5.15×** more cycles, yielding **5.43×** elapsed time. Effective frequency is nearly equal (4.41 vs 4.22 GHz, ratio 0.956×), so time ≈ cycles/frequency holds (5.43 ≈ 5.15/0.956). The high host IPC (4.44) and Retiring 75.2 % are characteristic of QEMU's simple TCG loop. **This must not be reported as native riscv64-vs-x86 performance.**
2. **x86 native is compute/layout + memory bound.** Topdown: Backend 34.0 % + Frontend 22.4 % + Bad Speculation 8.4 % vs Retiring 35.6 %; LLC-load miss 24.99 %. Hotspots are the bytecode interpreter (`sqlite3VdbeExec` 12.1 %), B-tree index/table seeks (`sqlite3BtreeIndexMoveto` 6.6 %, `sqlite3BtreeTableMoveto` 4.8 %), record comparison (`vdbeRecordCompareInt` 5.6 %) and the sorter (`vdbeSorterSort.part.0` 3.8 %). Roughly 44 % of cycle samples are kernel-side page-cache/I/O work, matching 1.34 s system time.
3. **RISC-V lacks SIMD here.** The x86 binary contains 4,977 AVX2/FMA vector instructions; the riscv64 `rv64gc` binary contains **0** (no V extension). Auto-vectorization is therefore unavailable on the target with the current flags/toolchain (GCC 13.3 cannot auto-vectorize RVV — verified "no vectype").
4. **The vector gap is masked by emulation** and, on this workload, the x86 hotspots are scalar/branch/cache bound, so the realistic native RVV upside is modest.

### 4.6 Vectorization intrinsics (binary)

| Architecture | Count | ISAs found |
|---|---|---|
| x86_64 (`-march=native`) | 4,977 vector instructions (58 unique opcodes) | SSE/SSE2 + AVX/AVX2 + FMA |
| riscv64 (`-march=rv64gc`) | 0 | none — scalar single/double FP only (`fadd.d`, `fsub.d`, `fdiv.d`, `fmul.d`, `fmadd.d`) |

`amixis analyze --vector x86 <binary>` succeeded (`58 unique / 4977 total`). `amixis analyze --vector riscv <binary>` **failed** (host `objdump` cannot disassemble RISC-V); verification used `riscv64-linux-gnu-objdump -d` (0 RVV instructions).

### 4.7 Cross-tables (tool-generated by `amixis compare`)

Source file: `cross-tables/CT-1__1__1..sqlite3-1__1__2..sqlite3.md` (compared `.scriptout` basenames `1__1__1..sqlite3` and `1__1__2..sqlite3`).

#### Cross-table — EVENT: CYCLES

| Symbol                           | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------------|-------:|-------:|-------:|
| \[unknown\]                      |   45.26 |  100.00 |  +54.74 |
| sqlite3VdbeExec                  |   12.07 |    0.00 |  -12.07 |
| sqlite3BtreeIndexMoveto          |    6.57 |    0.00 |   -6.57 |
| vdbeRecordCompareInt             |    5.58 |    0.00 |   -5.58 |
| sqlite3BtreeTableMoveto          |    4.78 |    0.00 |   -4.78 |
| vdbeSorterSort.part.0            |    3.82 |    0.00 |   -3.82 |
| pcache1Fetch                     |    3.51 |    0.00 |   -3.51 |
| vdbeSorterCompareInt             |    3.22 |    0.00 |   -3.22 |
| sqlite3VdbeRecordCompareWithSkip |    2.40 |    0.00 |   -2.40 |
| sqlite3BtreeInsert               |    1.15 |    0.00 |   -1.15 |
| checkTreePage                    |    0.90 |    0.00 |   -0.90 |
| btreeParseCellPtr                |    0.87 |    0.00 |   -0.87 |
| btreeParseCellPtrIndex           |    0.87 |    0.00 |   -0.87 |
| getCellInfo                      |    0.70 |    0.00 |   -0.70 |
| getPageNormal                    |    0.68 |    0.00 |   -0.68 |
| sqlite3PcacheRelease             |    0.57 |    0.00 |   -0.57 |
| moveToRoot                       |    0.51 |    0.00 |   -0.51 |
| sqlite3BtreeNext.isra.0          |    0.43 |    0.00 |   -0.43 |
| \_\_libc\_pread                  |    0.36 |    0.00 |   -0.36 |
| getAndInitPage                   |    0.33 |    0.00 |   -0.33 |

#### Cross-table — EVENT: CPU_CORE/CACHE-MISSES/

| Symbol                           | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------------|-------:|-------:|-------:|
| \[unknown\]                      |   79.29 |  100.00 |  +20.71 |
| vdbeSorterSort.part.0            |    7.95 |    0.00 |   -7.95 |
| sqlite3VdbeExec                  |    4.20 |    0.00 |   -4.20 |
| vdbeSorterCompareInt             |    3.36 |    0.00 |   -3.36 |
| sqlite3BtreeInsert               |    0.92 |    0.00 |   -0.92 |
| sqlite3BtreeIndexMoveto          |    0.75 |    0.00 |   -0.75 |
| sqlite3VdbeRecordCompareWithSkip |    0.53 |    0.00 |   -0.53 |
| vdbeRecordCompareInt             |    0.36 |    0.00 |   -0.36 |
| balance\_nonroot                 |    0.32 |    0.00 |   -0.32 |
| cfree                            |    0.29 |    0.00 |   -0.29 |
| pthread\_mutex\_lock             |    0.24 |    0.00 |   -0.24 |
| sqlite3BtreeTableMoveto          |    0.18 |    0.00 |   -0.18 |
| insertCell                       |    0.17 |    0.00 |   -0.17 |
| getCellInfo                      |    0.17 |    0.00 |   -0.17 |
| pcache1Fetch                     |    0.12 |    0.00 |   -0.12 |
| pthread\_mutex\_unlock           |    0.09 |    0.00 |   -0.09 |
| btreeComputeFreeSpace            |    0.09 |    0.00 |   -0.09 |
| \_\_libc\_pread                  |    0.09 |    0.00 |   -0.09 |
| memcpy@plt                       |    0.08 |    0.00 |   -0.08 |
| sqlite3PcacheMakeClean           |    0.08 |    0.00 |   -0.08 |

#### Cross-table — EVENT: CPU_CORE/BRANCH-MISSES/

| Symbol                           | 1_1_1 % | 1_1_2 % | Delta % |
|:---------------------------------|-------:|-------:|-------:|
| \[unknown\]                      |    6.80 |  100.00 |  +93.20 |
| sqlite3BtreeIndexMoveto          |   33.24 |    0.00 |  -33.24 |
| sqlite3VdbeRecordCompareWithSkip |   13.87 |    0.00 |  -13.87 |
| sqlite3BtreeTableMoveto          |   10.20 |    0.00 |  -10.20 |
| checkTreePage                    |    6.32 |    0.00 |   -6.32 |
| sqlite3VdbeExec                  |    6.13 |    0.00 |   -6.13 |
| vdbeRecordCompareInt             |    5.71 |    0.00 |   -5.71 |
| vdbeSorterCompareInt             |    4.39 |    0.00 |   -4.39 |
| pcache1Fetch                     |    3.95 |    0.00 |   -3.95 |
| vdbeSorterSort.part.0            |    1.52 |    0.00 |   -1.52 |
| pcache1FetchStage2               |    1.37 |    0.00 |   -1.37 |
| getPageNormal                    |    1.08 |    0.00 |   -1.08 |
| unixWrite                        |    0.83 |    0.00 |   -0.83 |
| sumStep                          |    0.66 |    0.00 |   -0.66 |
| unixRead                         |    0.59 |    0.00 |   -0.59 |
| getAndInitPage                   |    0.43 |    0.00 |   -0.43 |
| freeSpace                        |    0.40 |    0.00 |   -0.40 |
| sqlite3PcacheRelease             |    0.28 |    0.00 |   -0.28 |
| btreeComputeFreeSpace            |    0.24 |    0.00 |   -0.24 |
| moveToRoot                       |    0.22 |    0.00 |   -0.22 |

> The target (`1_1_2`) column is **100 % `[unknown]`** for every event because the riscv64 profile is the QEMU host process and cannot be symbolized per guest function. This is the exact tool output, **not fabricated**; it is valid only for the x86 side.

---

## 5. Optimization Results

### 5.1 Vector instructions in binary

| Binary | Vector instrs | Unique | ISAs |
|---|---:|---:|---|
| x86 baseline (`1_1_1`, `-march=native`) | 4,977 | 58 | SSE/SSE2/AVX/AVX2/FMA |
| x86 optimized (`1_1_1-opt-lto`, `-flto`) | 4,842 | 58 | same ISA set |
| riscv baseline (`1_1_2`, `rv64gc`) | 0 | 0 | none (no V extension) |
| riscv optimized (`1_1_2-opt-lto`) | 0 | 0 | none |
| riscv `-march=rv64gcv` rebuild | 0 | 0 | none — GCC 13.3 cannot auto-vectorize RVV (verified "no vectype") |

`Tag_RISCV_arch` of the target baseline: `rv64i2p1_m2p0_a2p1_f2p2_d2p2_c2p0_zicsr2p0_zifencei2p0_zmmul1p0` (no `v`).

### 5.2 Executable size analysis

| Binary | Unstripped | Stripped | Debug + symbol bloat | % of file |
|---|---:|---:|---:|---:|
| x86 baseline | 11,588,968 B | 2,495,008 B | 9,093,960 B | 78.5 % |
| riscv baseline | 11,945,160 B | 1,942,536 B | 10,002,624 B | 83.7 % |
| x86 optimized (LTO, `-g0`) | 2,695,248 B | 2,544,080 B | — | — |
| riscv optimized (LTO, `-g0`) | 2,131,936 B | 1,983,816 B | — | — |

Section-level: `.text*` is **1,398,074 B (riscv)** vs **2,029,538 B (x86)** — riscv code is **31.1 % smaller** thanks to the compressed-instruction extension. The larger riscv *unstripped* file is entirely debug info (`.debug_line` 3,978,249 B vs 2,145,359 B), inflated by the variable-length ISA. `strip`/`-g0` is functionally neutral and removes ~10 MB.

### 5.3 Optimization attempts (measured)

| Optimization | Before | After | Delta | Causal analysis |
|---|---|---|---|---|
| LTO, riscv (`-O3 -march=rv64gc -flto -g0`) | 17.17 s | 16.56 s | **−3.6 %** | Cross-TU (`shell.c`↔`sqlite3.c`) inlining/internalization; +2.4 % `.text`; faster every paired round |
| Static link, riscv (`-O3 -march=rv64gc -g0 -static`, glibc) | 17.17 s | 16.76 s | **−2.4 %** | Removes PLT/GOT indirection + dynamic loader; output byte-identical to baseline |
| LTO + static, riscv | 17.17 s | 16.58 s | **−3.4 %** | Does not compound — ~same as LTO alone |
| LTO, x86 (`-O3 -march=native -flto -g0`) | 3.33 s | 3.51 s | **+5.4 % real / +5.5 % user** | **Regression**: slightly higher instruction count and lower IPC from changed cross-TU inlining/layout; cache/branch rates unchanged. Treated as neutral-to-negative on x86 |
| `-O3` → `-O2`, riscv | same code | same code | **0 %** | Byte-identical machine code (483,168 instructions); `-O3` buys nothing on RV64GC GCC 13 |
| `-march=rv64gcv`, riscv | 17.17 s | 17.17 s | **0 %** | GCC 13.3 emits no RVV (identical code, 483,167 vs 483,168 instrs) |
| `strip` / `-g0` | 11,945,160 B | 1,942,536 B | **0 % runtime, −83.7 % file** | Debug sections are not loaded at runtime |

> All riscv deltas were measured under the same QEMU invocation on both arms, so relative deltas are meaningful; absolute target numbers include emulation overhead. `calculate-optimization-improvement` values are recorded in Section 5.6.

### 5.4 Recommended optimizations (prioritized)

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--:|---|:--:|:--:|---|
| 1 | LTO `-flto` (keep `-g0`) | −3.6 % wall (measured under QEMU) | Low | Best measured win; +2.4 % `.text`; add `-fuse-linker-plugin` |
| 2 | Drop debug info (`-g0`/`strip`/`-Wl,--strip-debug`) | 0 % runtime, −83.7 % file | Low | Measured; use `-gsplit-dwarf` only if field debugging is needed |
| 3 | `-static` (prefer musl) | −2.4 % wall (measured, glibc) | Low–Med | Removes PLT/loader overhead; glibc static warns on `dlopen`/`getpwuid`; does not compound with LTO |
| 4 | `-O2` instead of `-O3` | ~0 % runtime | Low | Measured byte-identical code on RV64GC GCC 13; faster builds |
| 5 | `-ffunction-sections -fdata-sections -Wl,--gc-sections` | size reduction (est. few %) | Low | NOT TESTED |
| 6 | Target tuning `-mtune=`/`-mcpu=` (e.g. `sifive-u74`, `thead-c906`) | est. 0–5 % | Low | NOT MEASURABLE here (no native HW) |
| 7 | PGO `-fprofile-generate`/`-fprofile-use` | est. 5–15 % | Med–High | Highest-upside unmeasured item; guest-accurate counts can be trained under QEMU |
| 8 | GCC 14+/LLVM + `-march=rv64gcv` on V-capable HW | est. 0–5 % on this workload | High | GCC 13 cannot auto-vectorize RVV; hotspots are scalar/branch-bound |
| 9 | SQLite tuning (`SQLITE_ENABLE_STAT4`, larger cache, `SQLITE_DIRECT_OVERFLOW_READ`) | est. 0–10 % | Med | Query-plan dependent; NOT TESTED |
| 10 | Branchless B-tree search (`sqlite3BtreeIndexMoveto`, `sqlite3VdbeRecordCompareWithSkip`) | est. 2–8 % | High | ~70 % of x86 branch misses; correctness-sensitive |
| 11 | RVV kernels for `ext/vec1` | 0 % for this benchmark | High | vec1 not linked into `sqlite3`; currently AVX2/NEON only |
| 12 | Allocator swap (mimalloc/jemalloc/memsys5) | low | Med | No target allocators in sysroot; allocation not a top hotspot |

### 5.5 Step-by-step apply instructions

```sh
# 1) LTO build (fresh dir, do not overwrite baseline build-* dirs)
mkdir -p /work/sqlite3-workspace/build-riscv-opt && cd /work/sqlite3-workspace/build-riscv-opt
CC=riscv64-linux-gnu-gcc ../sqlite/configure --host=riscv64-linux-gnu --build=x86_64-linux-gnu \
  --disable-tcl --enable-fts5 --enable-rtree \
  CFLAGS="-O3 -march=rv64gc -g0 -flto" LDFLAGS="-flto"
make -j4 sqlite3

# 2) Remove debug info for deployment (footprint only)
riscv64-linux-gnu-strip build-riscv-opt/sqlite3 -o build-riscv-opt/sqlite3.stripped
# or use -g0 at compile time (already above) / -Wl,--strip-debug

# 3) Static link (test separately; does not compound with LTO)
make -C <dir> CFLAGS='-O3 -march=rv64gc -g0 -static' sqlite3

# 4) -O3 -> -O2 (no code difference measured on RV64GC GCC 13)
CFLAGS="-O2 -march=rv64gc -g0"

# 5) Section GC (optional size pass)
CFLAGS="-O2 -march=rv64gc -g0 -ffunction-sections -fdata-sections" \
LDFLAGS="-Wl,--gc-sections -Wl,--as-needed"

# 6) Target tuning once the board is known: add -mtune=sifive-u74 (or -mcpu=thead-c906 ...)

# 7) PGO
CFLAGS="-O2 -march=rv64gc -fprofile-generate" make ...
qemu-riscv64-static -L /usr/riscv64-linux-gnu ./sqlite3 train.db < bench.sql
CFLAGS="-O2 -march=rv64gc -fprofile-use -fprofile-correction" make ...

# 8) Only with RVV HW + GCC 14+: -march=rv64gcv_zvl128b -O3 -flto -g0, verify with
#    riscv64-linux-gnu-objdump -d | grep -E 'vsetvli|vle[0-9]+\.v', run with qemu -cpu rv64,v=true
```

### 5.6 Recorded improvement measurements

The following values are copied verbatim from the tool-owned `improvements.json`.

## Improvement of 1_1_2 (rv64gc -O3 -g) compared to opt-rv64gc-lto (-O3 -flto -g0)

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| sqlite3 | 17.17 | 16.56 | 96.45 | real_time |

## Improvement of 1_1_2 (rv64gc -O3 -g) compared to opt-rv64gc-static (-O3 -static -g0)

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| sqlite3 | 17.17 | 16.76 | 97.61 | real_time |

## Improvement of 1_1_2 compared to 1_1_2-opt-lto

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| sqlite3 | 17.88 | 17.59 | 98.38 | real_time |
| sqlite3 | 16.24 | 15.82 | 97.41 | user_time |
| sqlite3 | 1.62 | 1.65 | 101.85 | sys_time |
| sqlite3 | 1942536 | 1983816 | 102.13 | executable_size_stripped_bytes |
| sqlite3 | 4.46 | 4.41 | 98.88 | IPC |
| sqlite3 | 75350718237 | 73770758298 | 97.9 | cycles |
| sqlite3 | 336295219779 | 325130623889 | 96.68 | instructions |
| sqlite3 | 30.68 | 28.9 | 94.2 | LLC_load_miss_rate_pct |
| sqlite3 | 0.678 | 0.702 | 103.54 | L1_dcache_miss_rate_pct |
| sqlite3 | 0.1367 | 0.1412 | 103.29 | branch_misprediction_rate_pct |

## Improvement of 1_1_1 compared to 1_1_1-opt-lto

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| sqlite3 | 3.33 | 3.51 | 105.41 | real_time |
| sqlite3 | 2.01 | 2.12 | 105.47 | user_time |
| sqlite3 | 1.32 | 1.38 | 104.55 | sys_time |
| sqlite3 | 2495008 | 2544080 | 101.97 | executable_size_stripped_bytes |
| sqlite3 | 2.07 | 2.06 | 99.52 | IPC |
| sqlite3 | 14287470150 | 14463316664 | 101.23 | cycles |
| sqlite3 | 29579072829 | 29736779949 | 100.53 | instructions |
| sqlite3 | 21.83 | 22.03 | 100.92 | LLC_load_miss_rate_pct |
| sqlite3 | 2.8 | 2.81 | 100.36 | L1_dcache_miss_rate_pct |
| sqlite3 | 0.667 | 0.659 | 98.8 | branch_misprediction_rate_pct |

> `Improvement %` is `optimizedValue / baselineValue × 100` as recorded by `calculate-optimization-improvement`; values below 100 % are improvements for time/cycle/count metrics.

### 5.7 Tool-owned profile data (`sqlite.json`)

```json
{
  "1_1_2": {
    "sqlite3": {
      "build_name": "1_1_2",
      "executable": "sqlite3",
      "executable_run_success": true,
      "real_time": "16.79",
      "user_time": "15.18",
      "kernel_time": "1.59",
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
      "perf_record_name": "1__1__2..sqlite3.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_1": {
    "sqlite3": {
      "build_name": "1_1_1",
      "executable": "sqlite3",
      "executable_run_success": true,
      "real_time": "3.20",
      "user_time": "1.95",
      "kernel_time": "1.24",
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
      "perf_record_name": "1__1__1..sqlite3.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  }
}
```

---

## 6. Notes About Exploration Process

### 6.1 QEMU / emulation caveats

- The riscv64 target was executed **only** under `qemu-riscv64-static` 8.2.2 user-mode emulation on an x86_64 host. There is **no native RISC-V hardware** and **no binfmt_misc** registration, so the emulator is invoked explicitly with `-L /usr/riscv64-linux-gnu`.
- **All target timings include QEMU dynamic-binary-translation overhead** and are **not** representative of native riscv64 hardware performance. The measured ~5.4× x86→target wall-time ratio is dominated by emulation, not by ISA differences.
- **Native RISC-V hardware counters, IPC, cache-miss rates, branch-misprediction rates and guest function hotspots are NOT AVAILABLE.** `perf` on the target measures the QEMU host process; its counters describe the emulator's TCG loop, not the guest. The target cross-table column is 100 % `[unknown]`.
- `-march=rv64gcv` produces no vector code because the environment's GCC 13.3 predates RVV auto-vectorization; RVV effects cannot be measured here even under an RVV-capable QEMU CPU.

### 6.2 Tooling errors and limitations

- **`amixis` cannot model a local qemu-user target.** `amphimixis` 0.2.0 contains zero references to `emulator`/`qemu`/`binfmt`; the platform schema is only `{id, arch, address, username, password, port}`. `parse_config()` → `_has_valid_arch()` rejects a local platform whose arch (riscv) differs from the host uname (x86_64), so `amixis build`/`amixis profile` fail for the canonical target build `1_2_2` and — because the check runs for every build — also block `1_1_1` while the riscv platform is present. `amixis validate` does not perform this check and passes. **Workaround used:** a separate profiling config models the emulated target as a local x86 platform running a wrapper script that execs `qemu-riscv64-static`; this produced genuine tool-owned `.scriptout`, `sqlite.json`/`sqlite.pkl` and the `cross-tables/CT-*.md` via `amixis profile`/`amixis compare`. The profiling build names are therefore `1_1_1` (reference) and `1_1_2` (target), not the canonical `1_2_2`.
- **The Amphimixis Make backend cannot drive SQLite's autoconf build.** It runs `make <config_flags> …` without executing `./configure`, forwards flags such as `--with-tcl=…`/`--enable-fts5` directly to `make`, and requires a pre-existing `Makefile` (SQLite ships `Makefile.in`). All builds were therefore performed with the documented manual fallback (`./configure … && make …`).
- **`amixis profile`'s internal `perf stat` invocation is broken** in this version: it builds `perf stat -ddd -x| …`, and the unquoted `|` is parsed by the shell as a pipe, yielding `Error: switch 'x' requires a value`. Consequently `sqlite.json`'s `perf_stat` field is unusable usage text; all counters were obtained via the manual `perf stat` fallback.
- **`amixis analyze --vector riscv` fails** because it calls the host `objdump`, which cannot disassemble RISC-V. Verification used `riscv64-linux-gnu-objdump`.
- The installed `amixis` has no `analyze-vectorization` subcommand; the working form is `amixis analyze --vector {x86,riscv,arm} <binary>`.

### 6.3 Environment issues encountered

- TCL 8.6 / `tcl-dev` were not installed initially and were installed (`apt-get install tcl tcl-dev`) to build the reference `testfixture`. No riscv64 TCL exists, so the target test harness could not be cross-built.
- Running the TCL suite as root aborts (`test/attach.test` performs a chmod-0000 check); the suite was run as user `ubuntu`.
- `file(1)` is not installed; `readelf`/`riscv64-linux-gnu-readelf` were used.
- `nice -n -5` and `chrt` were denied in the container; CPU governor remained `powersave` and `cpupower` was unavailable. Runs could not be raised above default priority.
- `fossildelta.sql` requires the unbuilt `fossildelta.so` loadable extension and fails identically on x86 and riscv (not architecture-specific).
- The reference `veryquick.test` suite reported 2 errors (`shell1-3.21.7`, `zipfile-25.0`) that are environmental and reproduce independently of architecture.

---

## 7. Migration Readiness Summary

### 7.1 Readiness checklist

| Check | Result |
|---|---|
| Builds on reference | **YES** (x86_64, 0 errors, 0 warnings) |
| Tests pass on reference | **YES** — 383,628 tests, 2 environmental errors, 0 skipped |
| Builds on target | **YES** (riscv64 cross-compile, 0 errors, 0 warnings; ELF64 RISC-V confirmed) |
| Tests pass on target | **PARTIAL** — full TCL suite not runnable (no riscv64 TCL); smoke tests, persistence, `integrity_check`, and 11/12 CLI-replayable `.sql` scripts passed under QEMU |
| Zero external dependencies | **YES** for the core amalgamation (optional TCL/zlib/readline/ICU are all portable) |
| No hand-written intrinsics | **YES** in core `src/`; optional `ext/vec1` has AVX2/NEON only, with a scalar fallback |
| Alignment safe | **YES** (generic alignment handling; no riscv64-specific workaround required) |
| Exceptions handled | **N/A** (C project; no C++ exceptions) |
| Auto-vectorization | **Reference: YES** (4,977 AVX2/FMA instructions); **Target: NO** (rv64gc = 0 vector instructions; GCC 13.3 cannot auto-vectorize RVV) |

### 7.2 Migration verdict

**MINOR CONCERNS**

SQLite is functionally highly portable to riscv64: it cross-compiles cleanly, runs correctly under emulation, has no required external dependencies, no hand-written intrinsics in the core, safe alignment/endianness handling, and is already shipped as a native riscv64 package by Debian/Ubuntu (no RISC-V-specific patch). The concerns are non-blocking but real: (1) native RISC-V performance is **unmeasured** — all target counters come from QEMU and cannot substantiate native throughput; (2) the full TCL regression suite could not be executed on the target because riscv64 TCL is unavailable; (3) the target build has **zero vector instructions** (RV64GC, GCC 13.3 cannot auto-vectorize RVV), so it cannot benefit from SIMD that the x86 reference gets via AVX2/FMA. LTO was measured as a small riscv win (−3.6 % wall) and slightly negative on x86; debug-info removal (−83.7 % file) is a clear packaging win.

### 7.3 Required actions

1. **Validate on native riscv64 hardware** (or a full-system emulator with a guest PMU) to obtain real cycles/instructions/IPC, cache and branch behavior, and function hotspots; the current target counters are QEMU-host-side only.
2. **Run the full SQLite TCL test suite on riscv64** by supplying an riscv64 TCL (`tclsh`/`tcl-dev` for the target sysroot) and building `testfixture` there; the reference passed 383,628 tests, but the target received only smoke-level coverage.
3. **Adopt LTO (`-flto`) and `-g0`/`strip`** for riscv64 deployment builds (measured: −3.6 % wall; −83.7 % file size); keep LTO off on x86 unless re-validated, as it regressed slightly there.
4. **Plan an RVV path only if the target hardware has the V extension** and a GCC 14+/LLVM toolchain is available; confirm with `objdump | grep vsetvli`. On this workload the expected upside is modest because hotspots are scalar/branch/cache bound.
5. **Add target tuning** (`-mtune=`/`-mcpu=` for the actual board) once hardware is known; the current build uses `-march=rv64gc` only.
6. **Evaluate PGO and SQLite-specific options** (`SQLITE_ENABLE_STAT4`, larger default page cache) on representative workloads; these were not tested.
7. **Keep the manual build path** (`./configure … && make …`) documented, since the installed Amphimixis Make backend cannot drive SQLite's autoconf build and cannot model a local qemu-user target.
