# Amphimixis Migration Readiness Report — libuv1

**Project:** libuv (Debian source package `libuv1`)
**Repository:** https://github.com/libuv/libuv
**Reference platform:** x86_64
**Target platform:** x86_64
**Workspace:** `/work/libuv1-workspace/`
**Config:** `/work/input.yml`
**Date:** 2026-10-05

> **Scope note.** The provided Amphimixis configuration defines a single local platform (`id: 1, arch: x86`) and native local builds. The target platform therefore equals the reference platform (x86_64); no cross-compilation and no emulation (QEMU) were used. This run is a native x86_64 build and optimization study. RISC-V-relevant source findings are reported as forward-looking context only.

---

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Repository URL | https://github.com/libuv/libuv |
| Clone URL | https://github.com/libuv/libuv.git |
| Default branch | v1.x |
| Latest commit | `49b1c064714b412d7f7a2f5e7146c977554e57f9` (2026-10-01) — `linux: don't use posix_spawn before glibc 2.24 (#5319)` |
| Total commits | 5791 |
| Latest tag | v1.53.0 (2026-09-24) |
| Activity | 208 commits in the last 12 months; actively maintained; 27,225 stars, 3,940 forks |
| License | MIT (`LICENSE`); `LICENSE-extra` for legacy Joyent parts; docs CC BY 4.0 |
| Build systems | CMake (`CMakeLists.txt`), Autotools (`configure.ac`, `Makefile.am`), Make |
| Test framework | In-tree runner (`test/run-tests.c`, `test/runner.c`, `test/test-list.h`) with CTest integration |
| Tests | 540 executed (TAP plan `1..540`); analyzer counted 478 `TEST_DECLARE` macros (per-OS subset) |
| Benchmarks | In-tree (`test/benchmark-list.h`, 55 `BENCHMARK_DECLARE`; `test/run-benchmarks.c`). Note: `amixis analyze` reported "benchmarks: not found" (false negative) |
| CI | GitHub Actions: 9 workflows (CI-unix/win/freebsd/netbsd/openbsd/solaris/docs/sample, sanitizer) |
| Docs | Sphinx (`docs/src/`, 33 `.rst`), README, CONTRIBUTING, SUPPORTED_PLATFORMS, ChangeLog |
| External dependencies | None third-party. Links only system libs: `pthread`, `dl`, `rt` (Linux); platform libs (kstat/nsl/ws2_32/…) are OS-conditional |
| Distro package | Debian source package `libuv1` (tracker.debian.org/pkg/libuv1); Debian carries a patch adjusting `test-sizeof` expectations. Other distro package metadata: NOT AVAILABLE (snapshot scraper blocked) |
| Forks with target-arch patches | None found. GitHub searches for `libuv riscv` / fork-with-riscv returned 0; RISC-V work is upstream (PR #5019 merged 2026-02-04; PR #5177 merged 2026-07-02) |

---

## 2. Platform-Specific Code Analysis

### Architecture macros

| Macro | File:Line | Meaning / what it guards | Category |
|---|---|---|---|
| `__x86_64__` | `src/unix/linux.c:81,101,119` | Syscall numbers: `__NR_copy_file_range`=326, `__NR_statx`=332, `__NR_getrandom`=318 | x86 |
| `__x86_64__` | `src/uv-common.c:908` | `uv__cpu_relax()` → `rep; nop` (PAUSE) | x86 |
| `__x86_64__` | `test/test-sizeof.c:32,262,612` | Expected struct ABI sizes (x86_64 layout) | x86 (test) |
| `__i386__` | `src/unix/linux.c:83,103,121`; `src/uv-common.c:908`; `test/test-sizeof.c:200,318,659` | 32-bit x86 syscall numbers, PAUSE, ABI sizes | x86 |
| `__arm__` | `src/unix/linux.c:87,107,125,513,1757`; `src/uv-common.c:910`; `include/uv/unix.h:419` | ARM syscall numbers; `isb` relax (`__ARM_ARCH>=7`); disable io_uring on 32-bit ARM; `/proc/cpuinfo` marker; `O_DIRECT`=0x10000 | ARM |
| `__aarch64__` | `src/unix/linux.c:89,105,123,1760`; `src/uv-common.c:910`; `include/uv/unix.h:419` | AArch64 syscall numbers; `isb`; cpuinfo "CPU part"; `O_DIRECT`=0x10000 | ARM |
| `__ARM_ARCH` | `src/uv-common.c:910` | Selects `isb` relax only for ARMv7+ | ARM |
| `__powerpc__`/`__ppc__`/`__ppc64__`/`__powerpc64__`/`__PPC__`/`__PPC64__` | `src/unix/linux.c:91,109,127,513,1751`; `src/uv-common.c:912–914` | PPC syscall numbers; disable io_uring on ppc64; `or 1,1,1` relax; cpuinfo marker; `O_DIRECT` value | PowerPC |
| `__s390__` | `src/unix/linux.c:85,111,129` | s390 syscall numbers | s390 |
| `__riscv` | `src/unix/linux.c:95,113,131` | RISC-V syscall numbers: `copy_file_range`=285, `statx`=291, `getrandom`=278 | RISC-V |
| `__riscv_xlen` | `src/uv-common.c:916`; `test/test-sizeof.c:430` | Gate 64-bit RISC-V FENCE relax and rv64 ABI struct sizes | RISC-V |
| `__mips__` | `src/unix/linux.c:1764`; `include/uv/unix.h` | cpuinfo marker / `O_DIRECT`=0x08000 | MIPS |
| `__loongarch__`, `__arc__`, `__hppa__`, `__sparc__`, `__m68k__` | `src/unix/linux.c:1767`; `include/uv/unix.h:421–427` | cpuinfo marker, `O_DIRECT`, syscall numbers | other |
| `__SIZEOF_POINTER__` | `src/unix/linux.c:513` | 32-bit ARM io_uring disable | pointer size |
| `_LP64` | `src/unix/core.c:604` | 64-bit data model branch (with `TARGET_OS_IPHONE`) | pointer size |

Semantics were checked (names can mislead): `__arm__` is 32-bit ARM only and is always paired with `__aarch64__`; `__riscv` is paired with `__riscv_xlen == 64` where 64-bit is required; the RISC-V relax `.insn 0x0100000f` is a valid FENCE encoding. No misleading macro use found.

### Platform preprocessor guards (OS)

629 OS-macro guard occurrences in `src/` + `include/` (non-Windows). Principal guards:

| Guard | Platform | Scope |
|---|---|---|
| `__linux__` | Linux | syscall layer (`linux.c`), `uv/linux.h`, `O_DIRECT`, io_uring/epoll backend |
| `__APPLE__`, `__MACH__` | macOS/iOS | kqueue backend, proctitle, fs events |
| `_WIN32`/`_WIN64` | Windows | entire `src/win` tree (IOCP) |
| `__FreeBSD__`/`__NetBSD__`/`__OpenBSD__`/`__DragonFly__` | BSDs | kqueue, proctitle, `bsd-ifaddrs` |
| `__sun`/`__sun__`/`__illumos__` | Solaris/illumos | event ports, `port_create` |
| `_AIX`/`_AIX71`/`_AIX73`/`__PASE__` | AIX / IBM i | `uv/aix.h` / `uv/posix.h` |
| `__MVS__` | z/OS | os390 syscalls, `uv/os390.h` |
| `__QNX__`, `__HAIKU__`, `__CYGWIN__`, `__MSYS__`, `__GNU__` | QNX/Haiku/Cygwin/MSYS/Hurd | posix backend |
| `__ANDROID__`/`__ANDROID_API__` | Android | API-level gates |
| `__GLIBC__`/`__UCLIBC__` | libc variant | `thread.c`/`process.c` symbol choices |
| `_POSIX_VERSION` | POSIX level | fallbacks when < 200809L |

### Vectorization intrinsics (source)

None. No `_mm_*`/`_mm256_*`/`_mm512_*`, no `immintrin.h`/`xmmintrin.h`, no NEON (`vld1`/`vadd`), no RISC-V `__riscv_v`. The only SIMD-looking match is a comment in `src/win/thread.c:27` (`__mm_mfence`), not used code. Only scalar `asm` exists, all in `src/uv-common.c` `uv__cpu_relax()` (`rep; nop` x86 PAUSE; `isb` ARMv7+/AArch64; `or 1,1,1` PowerPC; empty barrier Apple PPC; `.insn 0x0100000f` RISC-V FENCE). These are synchronization barriers, not data-parallel vector code. Any vectorization is compiler auto-vectorization.

### Portability verdict

| Aspect | Verdict |
|---|---|
| No exceptions / compiler barriers | No C++ exceptions (pure C). The only inline asm is optional, arch-guarded scalar relax/barrier code with a no-op fallback; no portability barrier |
| Alignment safe | Yes, explicitly verified. `test/test-sizeof.c` is a compile-time ABI database asserting exact `sizeof`/`offsetof` of public types per architecture, including a dedicated `__riscv && __riscv_xlen == 64` block and x86_64 blocks. A Debian patch also adjusts `test-sizeof` expectations |
| Embedded usability | Limited / not a target. Requires a full OS ABI (epoll/io_uring/kqueue/IOCP, POSIX threads, process spawning, `/proc`, filesystem). Tier 1: GNU/Linux glibc >=2.17 / Linux >=3.10; usable on musl (Tier 2), Android (Tier 3); not RTOS/bare-metal. RISC-V is source-supported but not a named tier and has no native CI runner |
| Overall portability level | LOW concern for the configured x86_64 target (native, Tier 1, CI-tested, no third-party deps, no intrinsics). Forward-looking RISC-V: LOW–MEDIUM — arch plumbing (`__riscv` syscall numbers, rv64 FENCE relax, rv64 ABI size checks, `O_DIRECT`) is already upstream as of v1.53.0, but RISC-V is not in libuv's CI matrix |

---

## 3. Build & Test Results

All builds are native x86_64 (target == reference). Build names are `<build_machine>_<run_machine>_<recipe_id>`. Build trees are under `/work/libuv1-workspace/`.

| Platform | Build | Recipe / flags | Status | Build time | Tests (declared / passed / failed / skipped) |
|---|---|---|---|---|---|
| x86_64 (local) | 1_1_1 | recipe 1: RelWithDebInfo (`-O2 -g -DNDEBUG`), BUILD_TESTING=ON | OK | ~29 s | 540 / 540 / 0 / 11 |
| x86_64 (local) | 1_1_2 | recipe 2: RelWithDebInfo + c/cxx `-O3 -march=native -g` (effective `-O2 -march=native -g` due to CMake ordering) | OK | ~30 s | 540 / 540 / 0 / 11 |
| x86_64 (local) | 1_1_3 | recipe 3: RelWithDebInfo with `CMAKE_C_FLAGS_RELWITHDEBINFO="-O3 -march=native -g -DNDEBUG"` (true `-O3`) | OK | ~29 s | 540 / 540 / 0 / 11 |

- Both the static (`uv_run_tests_a`) and shared (`uv_run_tests`) test runners were used for 1_1_1 and 1_1_2; 540 ok / 0 not-ok / exit 0 for each. For 1_1_3, `uv_run_tests_a` passed 540/540. The 11 skips are platform-conditional (macOS FSEvents, Windows-only, io_uring unavailable, no external IPv6, no `/dev/tty`, long unix paths) and identical across builds.
- Produced binaries per build directory (`/work/libuv1-workspace/<build>/`): `uv_run_benchmarks_a`, `uv_run_tests_a`, `uv_run_tests`, `libuv.a`, `libuv.so.1.0.0`.
- 1_1_3 `.text` sizes vs earlier builds (from `size -A`): `uv_run_benchmarks_a` 163,472 B (1_1_1) / 165,104 B (1_1_2) / 203,568 B (1_1_3); `uv_run_tests_a` 706,512 / 706,640 / 743,824 B; `libuv.a` 122,265 / 124,108 / 147,334 B. The larger `1_1_3` `.text` is consistent with genuine `-O3` + `-march=native` inlining/vectorization taking effect.

### Build / test failures detail

- **CTest as root:** `ctest` exits 8 because libuv refuses to run as root (`test/run-tests.c` checks `geteuid()==0`). Resolved with the documented `UV_RUN_AS_ROOT=1` escape hatch; all 540 tests passed with assertions executing. This is an environment constraint, not a build/product failure.
- **No compiler errors, no missing dependencies, no timeouts** for any build.
- **Flag-ordering defect (discovered, then fixed):** recipe 2's compile line was `-O3 -march=native -g -O2 -g -DNDEBUG`; GCC honors the last `-O`, so recipe 2 was effectively `-O2`. Recipe 3 fixes this and was verified (`C_FLAGS = -O3 -march=native -g -DNDEBUG ...`, zero occurrences of `-O2`).

---

## 4. Performance Comparison

### Experimental conditions

| Condition | Value |
|---|---|
| Machine | local container, Linux x86_64; Intel Core i7-13620H (6 P-cores + 4 E-cores, 16 threads; AVX2, FMA, no AVX-512) |
| Core pinning | `taskset -c 4` (P-core; SMT sibling 5) for every warmup and measurement run |
| Priority | `nice -n -20` requested but DENIED by container policy (`setpriority: Permission denied`, CAP_SYS_NICE absent); effective `nice 0` for both builds |
| Governor | `powersave` (all 16 policies; `/sys` read-only, cannot change). Turbo enabled but uncontrolled (`intel_pstate/no_turbo = 0`, `status = active`). Frequency dynamic; CPU4 observed 0.40–2.20 GHz |
| Warmup runs | 2 per build |
| Measurement runs | 10 per build (identical methodology per build) |
| perf_event_paranoid | `-1` (perf permitted) |
| Workload | `uv_run_benchmarks_a loop_count` → 2,000,000 idle-handle ticks; runner calls `uv_sleep(1000)` after the benchmark, so wall time includes a fixed ~1.0 s sleep |
| Emulation | None — native x86_64. No QEMU and no emulation caveat applies |
| perf limitation | Kernel symbols unresolved (`kptr_restrict = 1`), so flat `[unknown]` kernel attribution dominates the cycle profile |

### Key metrics (baseline 1_1_1 vs optimized 1_1_3)

Means over 10 pinned runs. Source: profiler campaign (phase 6), consistent between both builds; values also recorded in `improvements.json`.

| Metric | 1_1_1 (baseline) | 1_1_3 (true -O3 -march=native) |
|---|---|---|
| Elapsed wall time (s) | 1.565 | 1.532 |
| User time (s) | 0.266 | 0.254 |
| System/kernel time (s) | 0.292 | 0.272 |
| Benchmark self-time (s) | 0.546 | 0.517 |
| Throughput (ticks/s) | 3659800 | 3869300 |
| Cycles | 1181300000 | 1122000000 |
| Instructions | 2524000000 | 2471400000 |
| IPC | 2.1381 | 2.2028 |
| L1-dcache miss rate (%) | 0.01278 | 0.0109 |
| LLC miss rate (%) | 9.039 | 8.217 |
| Branch misprediction rate (%) | 0.006675 | 0.006056 |
| Frontend Bound (%) | 29.12 | 28.53 |
| Backend Bound (%) | 27.67 | 28.14 |
| Retiring (%) | 41.95 | 42.02 |

### Key metrics (baseline 1_1_1 vs attempted optimization 1_1_2)

Means over 10 pinned runs from the first profiling campaign. 1_1_2 is effectively `-O2 -march=native` (its `-O3` was overridden).

| Metric | 1_1_1 (baseline) | 1_1_2 (effective -O2, -march=native) |
|---|---|---|
| Elapsed wall time (s) | 1.565 | 1.554 |
| User time (s) | 0.266 | 0.262 |
| Cycles | 1.1812e9 | 1.1586e9 |
| Instructions | 2.5240e9 | 2.5198e9 |
| IPC | 2.138 | 2.176 |
| L1-dcache miss rate (%) | 0.0128 | 0.0115 |
| LLC miss rate (%) | 9.04 | 8.19 |
| Branch misprediction rate (%) | 0.0067 | 0.0061 |
| Frontend Bound (%) | 30.4 | 27.8 |
| Backend Bound (%) | 25.4 | 27.9 |
| Retiring (%) | 42.6 | 41.8 |

Note: the baseline TopdownL1 values in this second table come from the first campaign (`libuv.json`); TopdownL1 for 1_1_1 was re-measured in phase 6 and is the 29.12/27.67/41.95 shown in the first table.

### Hotspots

Hotspot percentages are perf `cycles` sample shares. Kernel symbols cannot be resolved (`kptr_restrict`), so the kernel `[unknown]` row dominates.

**Baseline 1_1_1 — top self symbols:**

| % cycles | Function | Module | Analysis |
|---|---|---|---|
| 64.79 | `[unknown]` (kernel) | kernel | epoll/poll-wait kernel path; unresolved |
| 11.24 | `[unknown]` | libc | libc syscall glue |
| 3.37 | `uv__hrtime` | uv_run_benchmarks_a | monotonic-clock read per loop iteration |
| 2.62 | `uv__run_idle` | uv_run_benchmarks_a | idle-handle processing (2M iterations) |
| 2.43 | `uv__io_poll` | uv_run_benchmarks_a | poll with timeout 0 each iteration |
| 1.87 | `uv__io_poll_check` | uv_run_benchmarks_a | watcher scan |
| 1.50 | `uv_run` | uv_run_benchmarks_a | main loop |
| 1.50 | `clock_gettime` | libc | backing `uv__hrtime` |
| 2.81 | `[unknown]` | [vdso] | vDSO clock time |

**Optimized 1_1_3 — top self symbols:**

| % cycles | Function | Module | Analysis |
|---|---|---|---|
| 62.65 | `[unknown]` (kernel) | kernel | epoll/poll-wait kernel path; unresolved |
| 13.62 | `[unknown]` | libc | libc syscall glue |
| 4.67 | `uv__hrtime` | uv_run_benchmarks_a | timestamp read per tick (vDSO `clock_gettime`) |
| 2.53 | `uv__io_poll_check` | uv_run_benchmarks_a | loop bookkeeping around poll |
| 1.56 | `uv__run_idle` | uv_run_benchmarks_a | idle-handle dispatch |
| 1.56 | `clock_gettime` | libc | vDSO/libc timestamp |
| 1.56 | `epoll_pwait` | libc | syscall wrapper |
| 1.36 | `__errno_location` | libc | TLS errno access |
| 1.36 | `uv_run` | uv_run_benchmarks_a | event-loop driver |
| 3.31 | `[unknown]` | [vdso] | vDSO clock time |

Same hot functions on both builds (identical source/workload); only proportions and libc/PLT attribution shift. No new hotspot appears.

### Bottleneck summary and causal analysis

1. **Per-iteration `epoll_pwait` syscall (dominant).** `loop_count` runs a `uv_idle` handle 2,000,000 times with no file descriptors registered, so `uv__io_poll` issues `epoll_pwait` with timeout 0 every iteration; `uv__update_time()` reads the clock twice per iteration. Result: ~52% of CPU time is kernel (sys 0.292 s of ~0.55 s) and the kernel `[unknown]` path is the largest cycle share. This cost is compiler-independent and caps any build-level gain at a low single-digit percentage.
2. **True `-O3` + `-march=native` gives a real but small ~5% improvement (1_1_1 to 1_1_3).** Benchmark self-time 0.546 s → 0.517 s; throughput 3.66M → 3.87M ticks/s; cycles 1.1813e9 → 1.1220e9; instructions 2.5240e9 → 2.4714e9; IPC 2.1381 → 2.2028. Only the user-space share (~48%) is addressable; the measured ~5% is consistent with that ceiling.
3. **The earlier 1_1_2 attempt produced almost no change because `-O3` never applied.** Its compile line ended in `-O2`; instructions retired differ by only 0.17% vs baseline. The small IPC/throughput shifts there are from `-march=native` alone.
4. **Front-end bound dominates the user-space pipeline (~28–30%); branch prediction is excellent (~0.006% misprediction).** The front-end cost comes from the tight event loop plus syscall-entry footprint, not from unpredictable branches. The small Frontend Bound reduction on 1_1_3 matches a shorter/tighter instruction stream.
5. **Cache behavior is not a bottleneck.** L1-dcache miss rate is ~0.01% (working set fits L1) and LLC miss rate is ~8–9% of only ~30–40k LLC accesses. Changes track code layout and are near the noise floor.
6. **Wall-clock time is a harness artifact.** `runner.c` executes `uv_sleep(1000)` in benchmark mode, so the 1.5 s elapsed includes ~1 s fixed sleep; CPU time, cycles and throughput are the valid discriminators.

### Vectorization / intrinsics

- Source: no hand-written SIMD intrinsics (see Section 2).
- Binary (from `amphimixis-analyze-vectorization`, x86): 1_1_1 (`-O2` generic) = 7 unique classes / 288 instructions, SSE/SSE2 only (`movups` ×211, `movaps` ×66, `movapd` ×4, `paddq` ×3, `paddd` ×2, `mulpd` ×1, `psubq` ×1). 1_1_2 (`-march=native`, effective `-O2`) = 30 unique classes / 250 instructions, AVX/VEX (`vpinsrq`, `vxorps`, `vxorpd`, 256-bit moves, `vpaddq`, `vpaddd`, `vmulpd`, `vpsubq`, `vpsll`, `vpcmp`). 1_1_3 (`-O3 -march=native`) = 37 unique classes / 417 instructions, AVX/VEX (`vpinsrq` ×58, `vxorps` ×38, `vxorpd` ×27, `vpcmp` ×10, `vpaddd` ×9, `vpaddq` ×9, `vmovaps` ×8, `vpinsrd` ×8, `vmovupd` ×6, `vshufps` ×5, `vpermd`/`vperm` ×2, `vmulpd` ×1, `vpsubq` ×1, plus SSE forms). No FMA and no hot vectorizable loop: these are auto-vectorized/codegen changes in non-hot code.

---

## Cross-table: baseline 1_1_1 vs attempted-optimized 1_1_2

Source file: `/work/libuv1-workspace/cross-tables/CT-1__1__1..uv__run__benchmarks__a_x20_loop__count-1__1__2..uv__run__benchmarks__a_x20_loop__count.md` (copied verbatim).

### EVENT: CYCLES

| Symbol                            | ./1_1_1 % | ./1_1_2 % | Delta % |
|:----------------------------------|---------:|---------:|-------:|
| \[unknown\]                       |     80.28 |     67.52 |  -12.75 |
| \[unknown\] (/                    |      0.00 |     12.06 |  +12.06 |
| clock\_gettime (/                 |      0.00 |      1.75 |   +1.75 |
| clock\_gettime                    |      1.50 |      0.00 |   -1.50 |
| \_\_errno\_location (/            |      0.00 |      1.33 |   +1.33 |
| \_\_errno\_location               |      1.33 |      0.00 |   -1.33 |
| epoll\_pwait (/                   |      0.00 |      1.16 |   +1.16 |
| uv\_\_metrics\_update\_idle\_time |      0.00 |      1.13 |   +1.13 |
| epoll\_pwait                      |      0.93 |      0.00 |   -0.93 |
| idle\_cb                          |      0.74 |      0.00 |   -0.74 |
| uv\_run                           |      1.50 |      2.14 |   +0.63 |
| uv\_\_hrtime                      |      3.39 |      2.93 |   -0.46 |
| uv\_\_io\_poll                    |      2.40 |      1.97 |   -0.43 |
| uv\_\_io\_poll\_prepare           |      0.19 |      0.57 |   +0.37 |
| uv\_\_run\_prepare                |      0.54 |      0.20 |   -0.35 |
| uv\_\_run\_idle                   |      2.61 |      2.94 |   +0.33 |
| uv\_\_run\_timers                 |      0.19 |      0.39 |   +0.20 |
| \_\_vdso\_clock\_gettime          |      0.38 |      0.20 |   -0.19 |
| uv\_\_run\_pending                |      0.96 |      0.78 |   -0.18 |
| uv\_\_io\_poll\_check             |      1.90 |      1.75 |   -0.15 |

### EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol                            | ./1_1_1 % | ./1_1_2 % | Delta % |
|:----------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                    |      0.00 |     21.35 |  +21.35 |
| \[unknown\]                       |     96.88 |     78.49 |  -18.38 |
| uv\_run                           |      2.55 |      0.00 |   -2.55 |
| uv\_\_backend\_timeout            |      0.35 |      0.00 |   -0.35 |
| epoll\_pwait                      |      0.17 |      0.00 |   -0.17 |
| clock\_gettime (/                 |      0.00 |      0.04 |   +0.04 |
| uv\_\_hrtime                      |      0.02 |      0.05 |   +0.04 |
| clock\_gettime                    |      0.04 |      0.00 |   -0.04 |
| uv\_\_metrics\_update\_idle\_time |      0.00 |      0.02 |   +0.02 |
| uv\_\_io\_poll\_check             |      0.00 |      0.01 |   +0.01 |
| \_\_errno\_location (/            |      0.00 |      0.01 |   +0.01 |
| uv\_\_run\_idle                   |      0.01 |      0.00 |   -0.01 |
| epoll\_pwait (/                   |      0.00 |      0.01 |   +0.01 |
| uv\_\_io\_poll                    |      0.00 |      0.01 |   +0.01 |
| uv\_\_run\_timers                 |      0.00 |      0.01 |   +0.01 |

### EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                 | ./1_1_1 % | ./1_1_2 % | Delta % |
|:-----------------------|---------:|---------:|-------:|
| \[unknown\] (/         |      0.00 |     45.38 |  +45.38 |
| \[unknown\]            |     98.60 |     53.75 |  -44.84 |
| uv\_\_run\_check       |      0.70 |      0.00 |   -0.70 |
| epoll\_pwait (/        |      0.00 |      0.41 |   +0.41 |
| epoll\_pwait           |      0.19 |      0.00 |   -0.19 |
| clock\_gettime         |      0.12 |      0.00 |   -0.12 |
| clock\_gettime (/      |      0.00 |      0.08 |   +0.08 |
| uv\_\_io\_poll\_check  |      0.10 |      0.03 |   -0.07 |
| uv\_\_run\_pending     |      0.00 |      0.07 |   +0.07 |
| \_\_errno\_location (/ |      0.00 |      0.04 |   +0.04 |
| uv\_\_io\_poll         |      0.03 |      0.00 |   -0.03 |
| uv\_\_run\_idle        |      0.09 |      0.08 |   -0.01 |
| uv\_run                |      0.05 |      0.05 |   -0.01 |
| uv\_\_hrtime           |      0.11 |      0.11 |   +0.01 |

## Cross-table: baseline 1_1_1 vs optimized 1_1_3

Source file: `/work/libuv1-workspace/cross-tables/CT-1__1__1..uv__run__benchmarks__a_x20_loop__count-1__1__3..uv__run__benchmarks__a_x20_loop__count.md` (copied verbatim).

### EVENT: CYCLES

| Symbol                            | 1_1_1 % | 1_1_3 % | Delta % |
|:----------------------------------|-------:|-------:|-------:|
| \[unknown\]                       |   80.28 |   66.10 |  -14.18 |
| \[unknown\] (/                    |    0.00 |   13.89 |  +13.89 |
| uv\_\_io\_poll                    |    2.40 |    0.79 |   -1.61 |
| epoll\_pwait (/                   |    0.00 |    1.58 |   +1.58 |
| clock\_gettime (/                 |    0.00 |    1.55 |   +1.55 |
| clock\_gettime                    |    1.50 |    0.00 |   -1.50 |
| \_\_errno\_location (/            |    0.00 |    1.38 |   +1.38 |
| \_\_errno\_location               |    1.33 |    0.00 |   -1.33 |
| uv\_\_hrtime                      |    3.39 |    4.70 |   +1.31 |
| uv\_\_run\_idle                   |    2.61 |    1.57 |   -1.05 |
| uv\_\_metrics\_update\_idle\_time |    0.00 |    0.97 |   +0.97 |
| uv\_\_run\_pending                |    0.96 |    0.00 |   -0.96 |
| epoll\_pwait                      |    0.93 |    0.00 |   -0.93 |
| uv\_\_io\_poll\_check             |    1.90 |    2.55 |   +0.65 |
| idle\_cb                          |    0.74 |    0.20 |   -0.54 |
| epoll\_pwait@plt                  |    0.00 |    0.39 |   +0.39 |
| uv\_\_run\_check                  |    0.58 |    0.97 |   +0.39 |
| uv\_\_run\_timers                 |    0.19 |    0.39 |   +0.20 |
| \_\_errno\_location@plt           |    0.00 |    0.20 |   +0.20 |
| uv\_\_backend\_timeout            |    0.19 |    0.00 |   -0.19 |

### EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol                 | 1_1_1 % | 1_1_3 % | Delta % |
|:-----------------------|-------:|-------:|-------:|
| uv\_run                |    2.55 |    0.00 |   -2.55 |
| \[unknown\] (/         |    0.00 |    2.08 |   +2.08 |
| cfree (/               |    0.00 |    1.27 |   +1.27 |
| uv\_\_backend\_timeout |    0.35 |    0.00 |   -0.35 |
| \[unknown\]            |   96.88 |   96.64 |   -0.24 |
| epoll\_pwait           |    0.17 |    0.00 |   -0.17 |
| clock\_gettime         |    0.04 |    0.00 |   -0.04 |
| uv\_\_hrtime           |    0.02 |    0.00 |   -0.02 |
| uv\_\_run\_idle        |    0.01 |    0.00 |   -0.01 |
| clock\_gettime (/      |    0.00 |    0.00 |   +0.00 |
| uv\_\_run\_timers      |    0.00 |    0.00 |   +0.00 |

### EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol                            | 1_1_1 % | 1_1_3 % | Delta % |
|:----------------------------------|-------:|-------:|-------:|
| \[unknown\]                       |   98.60 |   77.55 |  -21.05 |
| \[unknown\] (/                    |    0.00 |   20.86 |  +20.86 |
| epoll\_pwait (/                   |    0.00 |    0.77 |   +0.77 |
| uv\_\_run\_check                  |    0.70 |    0.04 |   -0.67 |
| clock\_gettime (/                 |    0.00 |    0.21 |   +0.21 |
| epoll\_pwait                      |    0.19 |    0.00 |   -0.19 |
| clock\_gettime                    |    0.12 |    0.00 |   -0.12 |
| \_\_errno\_location (/            |    0.00 |    0.09 |   +0.09 |
| uv\_\_metrics\_update\_idle\_time |    0.00 |    0.08 |   +0.08 |
| uv\_\_run\_idle                   |    0.09 |    0.16 |   +0.07 |
| uv\_\_io\_poll\_check             |    0.10 |    0.05 |   -0.06 |
| uv\_run                           |    0.05 |    0.10 |   +0.04 |
| uv\_\_io\_poll                    |    0.03 |    0.00 |   -0.03 |
| uv\_\_hrtime                      |    0.11 |    0.10 |   -0.00 |

---

## 5. Optimization Results

### Vector instructions in binary

| Binary | Unique instruction classes | Total vector instructions | ISA summary |
|---|---:|---:|---|
| 1_1_1 (`-O2` generic x86-64) | 7 | 288 | SSE/SSE2 legacy: `movups`, `movaps`, `movapd`, `paddq`, `paddd`, `mulpd`, `psubq` |
| 1_1_2 (`-march=native`, effective `-O2`) | 30 | 250 | AVX/VEX: `vpinsrq`, `vxorps`, `vxorpd`, `vmovaps`/`vmovapd`/`vmovupd`, `vpaddq`, `vpaddd`, `vmulpd`, `vpsubq`, `vpsll`, `vpcmp` |
| 1_1_3 (true `-O3 -march=native`) | 37 | 417 | AVX/VEX (as above) plus `vpermd`/`vperm`, `vshufps`, `vpinsrd`; no FMA |

### Optimization attempts

| Attempt | Before | After | Delta (measured) | Causal analysis |
|---|---|---|---|---|
| Recipe 2: `-O3 -march=native -g` under RelWithDebInfo | 1_1_1 (`-O2` generic) | 1_1_2 (effective `-O2 -march=native`) | throughput +1.6%, cycles -1.9%, IPC +1.8% | CMake emitted `-O2` after `-O3`; GCC honored the last `-O`, so `-O3` never applied. Instructions retired differ by only 0.17%; the small gain is from `-march=native` (AVX2/VEX) alone |
| Recipe 3: populating `CMAKE_C_FLAGS_RELWITHDEBINFO` so true `-O3` applies | 1_1_1 (`-O2` generic) | 1_1_3 (true `-O3 -march=native`) | throughput +5.7%, cycles -5.0%, IPC +3.0% | Genuine `-O3` codegen plus `-march=native`; improvement capped by the syscall-bound workload (only the ~48% user-space share is addressable) |
| `strip` debug info (analysis only) | unstripped `uv_run_benchmarks_a` 1_329_752 B (1_1_1) | stripped 263_064 B (1_1_1) | on-disk size only | ~80% of the file is DWARF debug info, which is not mapped at runtime; no effect on the benchmark |

### Recommended optimizations

| Priority | Optimization | Expected gain (this workload) | Effort | Notes |
|:--:|---|---|:--:|---|
| 1 | Fix flag ordering so `-O3` actually applies (validity fix; already applied as recipe 3) | ~0–2% on this workload; essential for a valid `-O3` experiment | Low | Current recipe 2 never compiled with `-O3`; recipe 3 fixes it (`-DCMAKE_C_FLAGS_RELWITHDEBINFO` override) |
| 2 | Keep `-march=native` (build host == run host) | Observed throughput +5.7% / cycles -5.0% combined with true `-O3` | Low | Portability caveat: emitted AVX2/VEX binary requires an AVX2 host; use `-march=x86-64-v3` for portable CI artifacts |
| 3 | LTO (`-flto` / `CMAKE_INTERPROCEDURAL_OPTIMIZATION=ON`) | Marginal (~0–2%, user-space only) | Low–Med | Improves layout/inlining of the ~48% user portion; cannot touch kernel |
| 4 | PGO (`-fprofile-generate` then `-fprofile-use`) | Marginal (~0–3%) | Med | Best chance to reduce the 28–30% frontend bound, bounded by syscall share |
| 5 | Static libc / `-static-libgcc` (test independently) | ~0% on a 2M-tick loop | Low | Affects startup only; generic packaging benefit |
| 6 | Allocator (mimalloc/jemalloc) | ~0% on `loop_count` | Low | No per-iteration allocation in this benchmark |
| 7 | `strip` | ~0% runtime | Low | Reduces disk/transport size only |
| 8 | `-funroll-loops` / `-ffast-math` | ~0%, possible frontier regression | Low | No math or hot vectorizable loop; not recommended |

**Step-by-step instructions (as recommended by the optimizer):**

- **Fix flag ordering (applied):** add a recipe with `config_flags: '-DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_C_FLAGS_RELWITHDEBINFO="-O3 -march=native -g -DNDEBUG" -DCMAKE_CXX_FLAGS_RELWITHDEBINFO="-O3 -march=native -g -DNDEBUG" -DBUILD_TESTING=ON'`, add a build `{build_machine: 1, run_machine: 1, recipe_id: 3, executables: ["uv_run_benchmarks_a loop_count"]}`, then verify `grep '^C_FLAGS' <build>/CMakeFiles/uv_a.dir/flags.make` shows no trailing `-O2`.
- **LTO:** add `-DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON` to a recipe's `config_flags` (keep separate from static-linking changes), rebuild and compare `.text` via `size -A` and cycles/instructions.
- **PGO:** build with `-fprofile-generate=/work/libuv1-workspace/pgo`, run `./1_1_X/uv_run_benchmarks_a loop_count`, then rebuild with `-fprofile-use=/work/libuv1-workspace/pgo -fprofile-correction -march=native`; keep training/use on the same host.
- **Static linking:** test `-static` or `-DCMAKE_EXE_LINKER_FLAGS="-static-libgcc -static-libstdc++"` independently from LTO, then measure before combining.
- **Allocator:** `LD_PRELOAD=.../libmimalloc.so` / `libjemalloc.so` (record NOT AVAILABLE if not installed); a null result is expected for this workload.
- **Strip:** `cp <build>/uv_run_benchmarks_a ...stripped && strip ...` (no runtime change).

Honest ceiling: because ~52% of CPU time and the majority of cycles are in the kernel `epoll_pwait`/`clock_gettime` path, no build-level optimization can yield more than a low-single-digit win on this specific benchmark. The most important action taken was fixing the flag ordering so recipe 3 truly tests `-O3`.

## Improvement of 1_1_1 compared to 1_1_3

Source: `/work/improvements.json` (tool-owned; copied verbatim).

| Measured | Baseline value | Optimized value | Improvement % |
|---|---|---|---|
| cycles | 1181300000 | 1122000000 | 94.98 |
| instructions | 2524000000 | 2471400000 | 97.92 |
| IPC | 2.1381 | 2.2028 | 103.03 |
| real_time | 1.565 | 1.532 | 97.89 |
| user_time | 0.266 | 0.254 | 95.49 |
| kernel_time | 0.292 | 0.272 | 93.15 |
| throughput | 3659800 | 3869300 | 105.72 |
| L1-dcache_miss_rate | 0.01278 | 0.0109 | 85.29 |
| LLC_miss_rate | 9.039 | 8.217 | 90.91 |
| branch_miss_rate | 0.006675 | 0.006056 | 90.73 |
| frontend_bound | 29.12 | 28.53 | 97.97 |
| backend_bound | 27.67 | 28.14 | 101.7 |
| retiring | 41.95 | 42.02 | 100.17 |

### Recorded profile JSON (`/work/libuv1-workspace/libuv.json`)

The authoritative profile JSON contains one record per build for executable `uv_run_benchmarks_a loop_count`. Summary (values copied from the file):

| Build | run_success | real_time (s) | user_time (s) | kernel_time (s) | task-clock (msec) | cpu-cycles | instructions | branches | branch-misses | L1-dcache-load-misses | LLC-loads | dTLB-loads | tma_frontend_bound (%) | tma_backend_bound (%) | tma_retiring (%) | tma_bad_speculation (%) |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1_1_1 | true | 1.52 | 0.25 | 0.26 | 524.67 | 1121912552 | 2505292356 | 461907006 | 159454 | 118959 | 63946 | 678545951 | 30.4 | 25.4 | 42.6 | 1.6 |
| 1_1_2 | true | 1.52 | 0.26 | 0.26 | 530.01 | 1125392629 | 2505052141 | 462163479 | 138572 | 129890 | 66265 | 675014204 | 27.8 | 27.9 | 41.8 | 2.4 |
| 1_1_3 | true | 1.51 | 0.22 | 0.28 | 510.04 | 1091481802 | 2467305553 | 450971996 | 95234 | 145215 | 67334 | 663317994 | 29.9 | 25.7 | 42.8 | 1.6 |

The JSON also records `context-switches`, `cpu-migrations`, `page-faults`, and derived `metric-value`s (`llc_miss_rate`: 16.0% / 12.4% / 25.7% for 1_1_1 / 1_1_2 / 1_1_3; `branch_miss_rate` rounds to 0.0%; `insn_per_cycle`: 2.2 / 2.2 / 2.3). Kernel/pmu counters for `cpu_atom/*` are `<not counted>` because the workload ran on a P-core.

---

## 6. Notes About Exploration Process

- **No cross-ISA migration configured.** The provided `/work/input.yml` defines exactly one platform (`id: 1, arch: x86`) and three native local builds. Therefore target == reference (x86_64) and **no QEMU/emulation was used**; there is no emulation caveat. Cross-build artifacts are native x86_64.
- **RISC-V toolchain present but unused.** The container has `riscv64-linux-gnu-gcc/g++`, a riscv64 sysroot, and `qemu-riscv64`, but no riscv platform/recipe exists in the config, so no riscv build was attempted. RISC-V is covered only as forward-looking source analysis (Section 2).
- **CTest cannot run as root.** `ctest` exits 8 (`The libuv test suite cannot be run as root`). Used the documented `UV_RUN_AS_ROOT=1` override; all 540 tests passed with assertions executing.
- **Recipe 2 `-O3` was silently defeated by CMake flag ordering.** Verified via `flags.make`/`compile_commands.json` (`... -O3 -march=native -g -O2 -g -DNDEBUG`); GCC used the last `-O` = `-O2`. Fixed by adding recipe 3 using `CMAKE_C_FLAGS_RELWITHDEBINFO`; verified `1_1_3` has no `-O2` and true `-O3`.
- **`amphimixis-profile` wrapper failed** (exit code 1, no diagnostics). The underlying `amixis profile` CLI succeeded when run from `/work/libuv1-workspace` and produced `1__1__{1,2,3}...perfdata/scriptout` and updated the tool-owned `libuv.json`/`libuv.pkl`. A manual pinned `perf stat` campaign (2 warmup + 10 measurement runs) was used as the statistically rigorous source for the key-metric tables.
- **Stale `/work/libuv.json`.** An earlier failed profiling attempt launched from `/work` left a stale `/work/libuv.json` (records `executable_run_success: false`, null metrics). The authoritative profile JSON is `/work/libuv1-workspace/libuv.json`, used in this report.
- **Profiling environment limitations (documented, not fabricated):** `nice -n -20` denied by container policy (effective nice 0 for both builds); CPU governor is `powersave` and cannot be changed (`/sys` read-only); turbo is enabled but uncontrolled; frequency is dynamic (0.40–2.20 GHz observed). These add run-to-run variance but affect all builds identically.
- **Kernel symbols unresolved** (`kptr_restrict = 1`), so the perf cycle profile has a large `[unknown]` kernel component; this is marked as unresolved, not estimated.
- **`amixis analyze` false negative for benchmarks**: it reported "benchmarks: not found"; libuv's benchmarks live under `test/`. The analyzer confirmed 55 `BENCHMARK_DECLARE` entries.
- **Debian snapshot scraper blocked** (anti-scraper page); distro data was taken from `tracker.debian.org/pkg/libuv1`. Other distro metadata: NOT AVAILABLE.
- **No `/tmp` was used** for any artifact. `/work/pipeline.log` was not read by any agent. Tool-owned files (`improvements.json`, `cross-tables/CT-*.md`, `libuv.json`/`libuv.pkl`) were created only via the Amphimixis tools and were not hand-edited.

---

## 7. Migration Readiness Summary

| Check | Result |
|---|---|
| Builds on reference platform (x86_64) | YES — 1_1_1, 1_1_2, 1_1_3 |
| Tests pass on reference platform | YES — 540/540 (11 platform skips) |
| Builds on target platform (x86_64) | YES — target == reference; no cross-build required |
| Tests pass on target platform | YES — 540/540 (11 platform skips) |
| Zero external dependencies | YES — no third-party deps; links only system libs (`pthread`, `dl`, `rt` on Linux) |
| No hand-written intrinsics | YES — no SIMD intrinsics in source; only arch-guarded scalar CPU-relax asm |
| Alignment safe | YES — compile-time ABI checks in `test/test-sizeof.c`, incl. x86_64 and rv64 blocks |
| Exceptions handled | N/A — pure C; no C++ exceptions |
| Auto-vectorization | YES — SSE2 baseline (`1_1_1`); AVX2/VEX with `-march=native` (`1_1_2`, `1_1_3`); no hot vectorizable loop |

**Migration Verdict: READY**

For the configured target (x86_64, identical to the reference platform), libuv builds cleanly and passes its full test suite; it has no third-party dependencies, no hand-written SIMD intrinsics, and compile-time ABI/alignment checks. The primary finding of this run is a build-flag validity defect (recipe 2's `-O3` was overridden by a trailing `-O2`), which was identified and fixed (recipe 3) with a measured ~5% throughput gain.

Forward-looking note (proxy only, not configured here): for an actual RISC-V target, libuv's arch plumbing is already upstream, but RISC-V is not in its CI matrix — a dedicated cross-config + qemu-user run would be required to validate.

### Required Actions

1. If a genuine cross-ISA (RISC-V) migration is intended, add a second platform + cross-toolchain recipe to the config and re-run the build/profile pipeline under qemu-user (not configured in this run).
2. For any future `-O3` experiment, keep recipe 3 (or equivalent `CMAKE_C_FLAGS_<CONFIG>` override) so the optimization level is not overridden by CMake's config flags.
3. For portable/reproducible artifacts, replace `-march=native` with a fixed baseline (e.g. `-march=x86-64-v3`) since 1_1_2/1_1_3 emit AVX2/VEX code that requires an AVX2 host.
4. Run the test suite as a non-root user, or continue using `UV_RUN_AS_ROOT=1` where root execution is unavoidable (container constraint).
5. Optionally `strip` debug info for distribution artifacts (no runtime effect).
