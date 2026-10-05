# Amphimixis Migration Readiness Report — expat

**Project:** expat (libexpat) — streaming XML parser (C99)
**Reference platform:** x86_64 (local container, native)
**Target platform:** riscv64 (cross-compiled locally; executed under `qemu-riscv64-static` user-mode emulation)
**Workspace:** `/work/expat-workspace/`
**Config:** `/work/input.yml` (baseline builds `1_1_1`, `1_2_2`); `/work/expat-workspace/input_opt.yml` (optimized builds `1_1_3`, `1_2_4`)
**Analysis date:** 2026-10-03

---

## 1. Repository & Project Status

| Item | Value |
|---|---|
| Resolved clone URL | `https://github.com/libexpat/libexpat.git` (canonical upstream) |
| Latest commit | `a824e920f3d3d77930d7897c80b6925a6733aebc` — 2026-10-02 (~1 day before analysis; merge of PR #1405) |
| Total commits | 5565 (first commit 1997-11-04) |
| Latest tag / release | `R_2_8_5` (tagged 2026-09-22) |
| Recent tag cadence | R_2_8_0 … R_2_8_5 (2026-04-24 … 2026-09-22) |
| Activity | Actively maintained — near-daily commits; 2888/5565 commits by lead maintainer |
| Stars / forks / open issues | 1386 / 553 / 23 (GitHub API, 2026-10-03) |
| License | MIT (SPDX: MIT) |
| Primary language | C (C99); C++ only for optional fuzz harness |
| Build systems | CMake (primary, `CMakeLists.txt`) and Autotools (`configure.ac`); Windows Inno Setup scripts |
| Test framework | Bundled "minicheck" (in-tree, no external Check library) |
| Test count | 433 unique `START_TEST` functions / 438 registrations across 9 test cases; `runtests` reports 5196 checks |
| Benchmarks | `tests/benchmark/benchmark.c` (parse timing utility) |
| Documentation | `doc/` (reference.html, xmlwf.xml), `README.md`, `Changes`, `CMake.README`, `CONTRIBUTING.md`, `SECURITY.md` |
| CI | 26 GitHub Actions workflows, including `qemu.yml` covering **riscv64** and s390x |
| External dependencies (required) | **0** — C99 standard library only |
| Distro packages | Widely packaged: Debian/Ubuntu (`libexpat1`, `libexpat1-dev`), Arch (`expat`), Yocto/OE, FreeBSD, etc. No distro-specific target-arch patches identified |

### Forks with Target-Architecture Patches
Checked — **not needed / already merged upstream**. Upstream `master` already contains RISC-V work:
- **PR #1337** "Make CI cover compilation and execution on riscv64" (merged 2026-08-28) — QEMU workflow builds and runs the full test suite + `run-xmltest` on riscv64 Linux.
- **PR #1339** "Cover s390x in addition to riscv64…" (merged 2026-08-28) — adds big-endian s390x.
- **PR #1371** "lib: Do not assume `long long` has the strictest member alignment" (merged 2026-09-14) — fixes CHERI RISC-V alignment (`union expat_align` now includes `void *`).
- No forks with unmerged target-architecture patches found (GitHub searches returned 0 repos).

### Dependency Portability Assessment
The core library has **zero external dependencies**. Every dependency-like item was checked for riscv64:

| Dependency | Required? | riscv64 verdict | Notes |
|---|---|---|---|
| C99 standard library | Yes | ready | Provided by riscv64 glibc 2.39 / GCC 13.3.0 |
| `getrandom` (libc entropy) | Auto | ready | glibc riscv64 provides it; `SYS_getrandom` is arch-neutral |
| `getentropy` | Auto | ready | glibc ≥2.25 provides it |
| `arc4random` / `arc4random_buf` | Auto | ready | glibc ≥2.36 |
| `/dev/urandom` fallback | Fallback | ready | Available |
| minicheck test framework | Yes (tests) | ready | Bundled in-tree |
| CMake / GNU make | Yes (build host) | ready | Architecture-independent |
| Protobuf / fuzzer deps | Optional | not applicable | Not enabled in this build |

**Count:** required external deps = 0; all checked items ready. No dependency sub-pipeline required.

---

## 2. Platform-Specific Code Analysis

### Architecture Macros

| Macro | File:Line | What it guards | Category |
|---|---|---|---|
| `__i386` | `lib/expat_external.h:74` | `XMLCALL` → `__attribute__((cdecl))` for 32-bit GNU/x86 only; empty on all other targets. **No-op on riscv64 and x86_64.** | x86 (32-bit, harmless) |

No x86-64, AVX/SSE, ARM/NEON, or RISC-V architecture macros exist in the library code.

### Platform Preprocessor Guards

| Guard | File:Line (examples) | Platform | Semantics |
|---|---|---|---|
| `_WIN32` | `xmlparse.c:112,122,151,1087…`; `xmltok.c:61` | Windows | entropy `rand_s`, binary file modes, I/O variants |
| `_WIN64` | `internal.h:87`; `readfilemap.c:49` | Windows 64-bit | printf width formats |
| `__MINGW32__` | `random_rand_s.c:48,64`; `basic_tests.c:3933` | MinGW | `rand_s` versions; format specifiers |
| `__APPLE__` | `random_getentropy.c:39` | macOS | `sys/random.h` |
| `__CYGWIN__` / `__BEOS__` | `expat_external.h:95` | Cygwin/BeOS | excludes `dllimport` |
| `__wasi__` | `xmlparse.c:1158` | WASI | excludes `/dev/urandom` logic |
| `__SIZEOF_WCHAR_T__` | `expat_external.h:150` | compiler | guards optional `XML_UNICODE_WCHAR_T` (requires 2-byte `wchar_t`); riscv64 has 4-byte `wchar_t`, default UTF-8 mode unaffected |
| `BYTEORDER` (1234/4321) | `expat_config.h.cmake`; used `xmltok.c:847…` | endianness | riscv64 is little-endian → `BYTEORDER=1234`, identical path to x86_64; big-endian covered by s390x CI |
| `WORDS_BIGENDIAN` | `expat_config.h.cmake:90` | endianness | defined but unused in library source |
| libc feature macros (`HAVE_GETENTROPY`, `HAVE_GETRANDOM`, `HAVE_ARC4RANDOM`, `XML_DEV_URANDOM`) | `xmlparse.c:116–180` | libc detection | selects entropy backend; all available on riscv64 glibc |

### Vectorization Intrinsics (Source)
**None found.** Searches for `_mm_`, `_mm256_`, `_mm512_`, `<immintrin.h>`, x86intrin, `arm_neon`, `__riscv_v`/`riscv_vector`, Altivec, and WASM SIMD all returned zero matches. Expat is scalar C; only compiler auto-vectorization is possible.

### Portability Verdict

| Aspect | Verdict |
|---|---|
| Exceptions needed | **None** — no patches required; upstream explicitly tests riscv64 |
| Alignment safe | **Yes** — `long long`-only alignment assumption fixed (PR #1371), `union expat_align` includes `void *` |
| Embedded usability | **High** — small, dependency-free C99; heap-based, no threads required; entropy can fall back to `/dev/urandom` |
| Overall portability level | **LOW** (very high readiness) — expected to build and pass tests on riscv64 without modification |

---

## 3. Build & Test Results

Build tool: `amixis build` (CMake backend). Both baseline builds compiled on the local x86_64 host; the riscv64 binary ran under `qemu-riscv64-static -L /usr/riscv64-linux-gnu`. No manual CMake fallback was required for the baseline builds.

| Platform | Build | Command | Result | Build time |
|---|---|---|---|---|
| x86_64 native | `1_1_1` | `amixis build … --build-name 1_1_1` | **PASS** (exit 0) | ~16.0 s |
| riscv64 cross (run via qemu) | `1_2_2` | `amixis build … --build-name 1_2_2` | **PASS** (exit 0) | ~17.5 s |

| Platform | Test | Command | Result |
|---|---|---|---|
| x86_64 | `runtests` (minicheck) | `./tests/runtests` | **5196 checks, 0 failed** |
| x86_64 | `ctest` | `ctest --output-on-failure` | **1/1 passed** (42.55 s) |
| x86_64 | XML conformance (`run-xmltest`) | `make run-xmltest` | **Passed: 1801, Failed: 8** (8 are committed expected deviations; `diff` vs expected = clean) |
| riscv64 (qemu) | `runtests` (minicheck) | `qemu-riscv64-static -L … ./tests/runtests` | **5196 checks, 0 failed** |
| riscv64 (qemu) | XML conformance | `tests/xmltest.sh` with qemu `xmlwf` + `fix-xmltest-log.sh` + `diff` | **Passed: 1801, Failed: 8** — identical to x86; diff clean |

**Build/test failures detail:** none. Binaries confirmed `readelf`: `1_1_1` = Advanced Micro Devices X86-64; `1_2_2` = RISC-V. RISC-V ELF `Tag_RISCV_arch` includes `v1p0` (RVV) and `zba/zbb/zbs`.

Noteworthy observations (not failures):
- `RelWithDebInfo` appends `-O2` after the recipe `-O3`, so the baseline's effective optimization was **`-O2`** (identical on both platforms; addressed in Section 5).
- The provided `-march=rv64gcvb` is rejected by GCC 13.3/binutils 2.42; the canonical equivalent `-march=rv64gcv_zba_zbb_zbs` was used.
- `BUILD_TESTING` is unused by expat's CMake; `-DEXPAT_BUILD_TESTS=ON` is the effective option.

---

## 4. Performance Comparison

### Experimental Conditions

| Item | Reference (`1_1_1`) | Target (`1_2_2`) |
|---|---|---|
| Host CPU | Intel Core i5-1035G1 @ 1.00 GHz (4 cores / 8 threads, Ice Lake) | same host (x86_64) |
| Architecture | x86_64 native | riscv64 guest emulated by qemu-riscv64-static 8.2.2 |
| Governor / frequency | `powersave`; observed ≈1.1–1.4 GHz (max 3.6 GHz) | same host |
| Cores / pinning | `taskset -c 0` | `taskset -c 0` |
| Priority | nice 0 (setting `nice -n -20` **denied** — container lacks `CAP_SYS_NICE`) | nice 0 |
| Warmup runs | 2 | 2 |
| Measurement runs | 10 (timing) + 10 (perf stat) | 10 (timing) + 10 (host perf stat) |
| Compilers | gcc 13.3.0 `-O3 -march=native -g` (effective -O2) | riscv64-linux-gnu-gcc 13.3.0 `-O3 -march=rv64gcv_zba_zbb_zbs -g` (effective -O2) |
| Workload | `benchmark <pr-xml-little-endian.xml> 4096 100` (313,076 B UTF-16LE) | same file & args |

### Key Metrics (baseline builds)

Label: **[NATIVE]** = measured on x86_64 silicon; **[HOST-SIDE/QEMU]** = perf counters of the x86 qemu process on the target run, **not native RISC-V**; **N/A** = not obtainable.

| Metric | Reference x86_64 [NATIVE] | Target riscv64 (QEMU) | Label |
|---|---:|---:|---|
| Benchmark per-loop time (mean of 10) | 3.675 ms | 27.052 ms | GUEST-TIMING (target includes QEMU overhead) |
| task-clock | 374.2 ms | 2824.1 ms | HOST-SIDE/QEMU (target) |
| Cycles | 499.95 M | 3.860 G | HOST-SIDE/QEMU |
| Instructions | 1.2432 G | 13.507 G | HOST-SIDE/QEMU |
| IPC | 2.487 | 3.499 | HOST-SIDE/QEMU (emulator IPC) |
| Branch misprediction rate | 1.430% | 0.430% | HOST-SIDE/QEMU |
| L1-dcache miss rate | 0.520% | 0.239% | HOST-SIDE/QEMU |
| LLC miss rate | 26.43% | 17.02% | HOST-SIDE/QEMU |
| Frontend Bound (TopdownL1) | 23.98% | 13.70% | HOST-SIDE/QEMU |
| Backend Bound (TopdownL1) | 7.15% | 0.82% | HOST-SIDE/QEMU |
| Retiring (TopdownL1) | 40.80% | 38.64% | HOST-SIDE/QEMU |
| Bad Speculation (TopdownL1) | 28.10% | 46.86% | HOST-SIDE/QEMU |
| Native RISC-V cycles/IPC/cache/branch | — | **N/A** | NOT AVAILABLE (no RISC-V PMU; QEMU user-mode does not emulate PMU) |

### Hotspots

**Reference platform x86_64 [NATIVE, ACTUAL perf record]** — 1886 cycles samples, ~94% in `libexpat.so`:

| % Time | Function | Analysis |
|---:|---|---|
| 21.51% | `little2_updatePosition` | UTF-16(LE) line/column counter — byte-serial scan |
| 19.90% | `little2_contentTok` | UTF-16 content tokenizer — branchy state machine |
| 9.24% | `accountingDiffTolerated.part.0` | size accounting on each parse call |
| 5.40% | `doContent` | main content dispatch loop |
| 4.93% | `little2_scanComment` | comment scanner |
| 4.33% | `sip24_final` | SipHash finalization (attribute hashing) |
| 4.21% | `little2_getAtts` | attribute extraction |
| 3.48% | `little2_toUtf8` | charset conversion |
| 2.97% | `little2_nameLength` | name length scan |
| 2.81% | `normal_contentTok` | 8-bit content tokenizer |

**Target platform riscv64 — native hotspots: NOT AVAILABLE.** Host-side perf record of the qemu process resolves only `qemu-riscv64-static` TCG/JIT frames and `[unknown]` addresses (no guest `libexpat` symbols). No target hotspot estimate is provided (QEMU user-mode does not expose guest PMU samples).

### Bottleneck Summary
1. **QEMU dynamic binary translation is the dominant target bottleneck.** The emulator executes far more host instructions than the native x86 run for the same workload, producing the measured slowdown. This is an **emulation** cost, not a demonstrated RISC-V ISA cost.
2. **The application bottleneck (both platforms) is the branch-heavy, byte-serial UTF-16 parser** in `libexpat` (`little2_updatePosition`, `little2_contentTok`, `doContent`). These state machines are not amenable to SIMD; RVV was enabled by the toolchain but not emitted.
3. **No native RISC-V microarchitectural evidence** exists — target cache/branch/Topdown/IPC values are host-emulator artifacts, so no target-side microarchitectural bottleneck can be responsibly identified.

### Vectorization Intrinsics (Source)
No hand-written SIMD intrinsics exist in expat (see Section 2). Any SIMD in the binaries is compiler-generated.

### QEMU / Emulation Caveats (also see Section 6)
- All riscv64 results were produced by `qemu-riscv64-static` 8.2.2 user-mode emulation on x86_64; `binfmt_misc` is unavailable, so binaries were invoked explicitly with `-L /usr/riscv64-linux-gnu`. `-cpu rv64` was avoided (SIGILL); the default CPU supports RVV.
- **Timing** includes QEMU translation/JIT overhead → not representative of native RISC-V hardware.
- **`perf` on the target is host-side only** — it counts the x86 emulator, not native RISC-V PMU events. Native RISC-V counters are **NOT AVAILABLE**.
- Guest symbols are absent under QEMU user-mode profiling, so target function-level hotspot attribution is **NOT AVAILABLE**.

### Cross-table: x86_benchmark vs riscv_qemu

Source file: `cross-tables/CT-x86_benchmark-riscv_qemu.md` (generated by `amixis compare`). **This table is NOT a valid same-workload cross-architecture comparison**: the target `.scriptout` contains only QEMU-emulator/JIT/kernel symbols, so every target row is `[unknown]`/emulator. It is reproduced below for completeness and **must not be read as a migration performance comparison**.

#### EVENT: BRANCH-MISSES

| Symbol | x86_benchmark % | riscv_qemu % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 0.23 | 100.00 | +99.77 |
| little2\_contentTok | 38.85 | 0.00 | -38.85 |
| little2\_updatePosition | 23.97 | 0.00 | -23.97 |
| little2\_getAtts | 7.14 | 0.00 | -7.14 |
| doContent | 4.85 | 0.00 | -4.85 |
| little2\_scanComment | 3.23 | 0.00 | -3.23 |
| little2\_scanRef | 3.22 | 0.00 | -3.22 |
| little2\_nameLength | 3.17 | 0.00 | -3.17 |

#### EVENT: CACHE-MISSES

| Symbol | x86_benchmark % | riscv_qemu % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 32.48 | 100.00 | +67.52 |
| \[unknown\] (/ | 42.44 | 0.00 | -42.44 |
| little2\_contentTok | 5.74 | 0.00 | -5.74 |
| doContent | 2.90 | 0.00 | -2.90 |
| little2\_getAtts | 1.72 | 0.00 | -1.72 |
| accountingDiffTolerated.part.0 | 1.62 | 0.00 | -1.62 |
| little2\_nameLength | 1.32 | 0.00 | -1.32 |
| normal\_contentTok | 1.21 | 0.00 | -1.21 |

#### EVENT: CYCLES

| Symbol | x86_benchmark % | riscv_qemu % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 0.68 | 100.00 | +99.32 |
| little2\_updatePosition | 21.51 | 0.00 | -21.51 |
| little2\_contentTok | 19.90 | 0.00 | -19.90 |
| accountingDiffTolerated.part.0 | 9.24 | 0.00 | -9.24 |
| doContent | 5.40 | 0.00 | -5.40 |
| \[unknown\] (/ | 5.28 | 0.00 | -5.28 |
| little2\_scanComment | 4.93 | 0.00 | -4.93 |
| sip24\_final | 4.33 | 0.00 | -4.33 |

### Causal Analysis for the Cross-table
The x86 side reflects genuine `libexpat` parser symbols; the riscv side shows 100% `[unknown]` because QEMU user-mode does not expose guest symbols. Therefore the cross-table **cannot** be used to attribute differences to the RISC-V ISA. The only rigorous cross-platform result is the guest wall-clock slowdown under emulation.

---

## 5. Optimization Results

### Vector Instructions in Binary

| Binary | Arch | Count | ISAs found |
|---|---|---:|---|
| `1_1_1/tests/benchmark/benchmark` | x86 | 6 total | `ordpd`/`xorpd`/`vxorpd` (startup/libc artifacts) |
| `1_1_1/libexpat.so.1.12.5` | x86 | 60 total / 11 unique | `vpinsrq`×26, `orps`/`xorps`/`vxorps`×24, `vpaddq`/`paddq`/`vpaddd`/`paddd`×6, `vpinsrd`×1 |
| `1_2_2/tests/benchmark/benchmark` | riscv | 0 | none |
| `1_2_2/libexpat.so.1.12.5` | riscv | **0** | none (no `vset*`, `vle*`, `vse*`, `vadd*`, `vfmadd`) |
| `1_1_3/libexpat.so` (optimized) | x86 | 185 total / 11 unique | same SSE/AVX scalar-move classes |
| `1_2_4/libexpat.so` (optimized) | riscv | **0** | none |

Tool status: `amixis analyze -v x86` worked; `amixis analyze -v riscv` **failed** on the RISC-V ELF (generic `objdump` cannot decode RISC-V). Counts were obtained with the `riscv64-linux-gnu-objdump` / `objdump` fallback. Hot parser functions contain **0 SIMD** on x86; the RISC-V build emits **0 RVV** despite `Tag_RISCV_arch` enabling `v1p0`. Expat's hot loops are serial byte/word state machines and do not auto-vectorize.

### Executable Size Analysis

| Binary | Unstripped | Stripped | Debug-info share |
|---|---:|---:|---:|
| x86 `benchmark` | 30,224 B | 14,472 B | ~52% |
| riscv `benchmark` | 26,352 B | 10,360 B | ~61% |
| x86 `libexpat.so.1.12.5` | 590,800 B | 178,504 B | ~70% |
| riscv `libexpat.so.1.12.5` | 626,992 B | 145,656 B | ~77% |
| optimized x86 `libexpat.so` (1_1_3) | — | 223,488 B | +25.2% vs baseline |
| optimized riscv `libexpat.so` (1_2_4) | — | 178,352 B | +22.4% vs baseline |

Debug info is 52–77% of unstripped size but is not mapped at run time; it does not cause runtime i-cache pressure. Optimized `-funroll-loops` grew `.text` by +35.2% (x86) and +38.6% (riscv), which is the direct cause of the RISC-V regression below.

### Optimization Attempts (before/after)

Applied (via `input_opt.yml`, recipes 3/4): **P1** enforce effective `-O3` (`-DCMAKE_C_FLAGS_RELWITHDEBINFO="-O3 -g -DNDEBUG"`), **P2** `-fno-semantic-interposition`, **P8** `-funroll-loops`. All values below are from `improvements.json` (measured; mean of 2 warmup + 10 runs, `taskset -c 0`, nice 0).

| Platform | Optimization Attempted | Baseline build | Optimized build | Before | After | Delta % | Causal Analysis |
|---|---|---|---|---|---|---:|---|
| x86_64 native | P1+P2+P8 | `1_1_1` | `1_1_3` | 3.675 ms/loop | 3.697 ms/loop | 100.6 | No measurable effect. Workload is branch/retire-bound (Retiring 40.0%, Bad Speculation 29.3%); unrolling has nothing to exploit. `.text` grew +35.2% but 6 MiB L3 / 256 KiB L1i absorb it. |
| riscv64 (QEMU) | P1+P2+P8 | `1_2_2` | `1_2_4` | 27.052 ms/loop | 29.076 ms/loop | 107.48 | **Regression under emulation** (~19σ). Host-side TCG instructions rose ~+6.8% and `.text` +38.6%; QEMU translates the inflated unrolled code, increasing JIT/fetch footprint. This is an emulation artifact; it was not possible to confirm native RISC-V behaviour (no native PMU). |
| x86_64 native | P1+P2+P8 (wall clock) | `1_1_1` | `1_1_3` | 0.366 s | 0.368 s | 100.55 | Flat; within frequency-scaling noise. |
| riscv64 (QEMU) | P1+P2+P8 (wall clock) | `1_2_2` | `1_2_4` | 2.765 s | 2.963 s | 107.16 | Same emulation-driven regression. |

Optimized-build correctness: `runtests` = **5196 checks / 0 failed** on both `1_1_3` (x86) and `1_2_4` (riscv under qemu). Effective flags confirmed: both builds end in `-O3` with no trailing `-O2`.

### Recommended Optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|---:|---|---|---|---|
| 1 | Enforce effective `-O3` (fix `RelWithDebInfo` ordering) | 2–8% (est.) | Low | Build bug: recipe `-O3` overridden by `-O2`. **Tested — no measurable effect on this branch-bound workload.** |
| 2 | `-fno-semantic-interposition` | 1–4% (est.) | Low | Removes internal PLT hops. **Tested together with P1/P8 — no x86 gain.** |
| 3 | LTO (`CMAKE_INTERPROCEDURAL_OPTIMIZATION=ON`) | 1–5% (est.) | Low–Med | Cross-TU inlining; not yet tested. |
| 4 | PGO (`-fprofile-generate`/`-fprofile-use`) | 5–15% (est.) | High | Best fit for the branchy parser (Bad Speculation 28–29%); not yet tested. |
| 5 | Static link library (`EXPAT_SHARED_LIBS=OFF`, `-static`) | 2–6% (est.) | Med | Removes PLT/GOT + dynamic-loader emulation; not yet tested. |
| 6 | Code: lazy amplification in `accountingDiffTolerated` | up to ~5–9% (est.) | Med | Skips unconditional float divide (default 8 MiB threshold; workload is 313 KB). Upstream-worthy; not applied. |
| 7 | Code: word-wise `little2_updatePosition` fast path | up to ~10–15% (est.) | High | Attacks the #1 hotspot; invasive, requires full-suite regression testing. Not applied. |
| 8 | `-funroll-loops -fomit-frame-pointer -ftree-vectorize` | 0–3% (est.) | Low | **Tested (`-funroll-loops`) — regressed riscv under QEMU, flat on x86.** |
| 9 | Newer RISC-V toolchain (GCC 14/15) | unknown | Low–Med | `-mtune` is invisible under QEMU. |
| 10 | Allocator swap (mimalloc/jemalloc) | 0–3% (est.) | Med | Not present for riscv sysroot; memory not the bottleneck. |

### Improvement of 1_1_1 compared to 1_1_3

Data copied verbatim from `improvements.json`.

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| tests/benchmark/benchmark | 3.675 | 3.697 | 100.6 | avg_time_per_loop_ms |
| tests/benchmark/benchmark | 0.366 | 0.368 | 100.55 | wall_clock_s |

### Improvement of 1_2_2 compared to 1_2_4

Data copied verbatim from `improvements.json`.

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---|---|---|
| tests/benchmark/benchmark | 27.052 | 29.076 | 107.48 | avg_time_per_loop_ms |
| tests/benchmark/benchmark | 2.765 | 2.963 | 107.16 | wall_clock_s |

(For time metrics, values < 100% indicate a speedup; > 100% indicate a slowdown/regression. Both x86 entries are ≈100% (flat); both riscv entries are > 100% (QEMU-emulation regression).)

### Cross-table: x86opt_benchmark vs riscvopt_qemu

Source file: `cross-tables/CT-x86opt_benchmark-riscvopt_qemu.md` (optimized builds). As above, the riscv side is **host-QEMU symbols only** and is not a native RISC-V comparison.

#### EVENT: CACHE-MISSES

| Symbol | x86opt_benchmark % | riscvopt_qemu % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] (/ | 16.19 | 25.52 | +9.34 |
| \[unknown\] | 70.65 | 74.48 | +3.82 |
| little2\_contentTok | 3.59 | 0.00 | -3.59 |
| doContent | 1.47 | 0.00 | -1.47 |
| lookupWithLength | 1.44 | 0.00 | -1.44 |
| contentProcessor | 0.94 | 0.00 | -0.94 |
| doProlog | 0.91 | 0.00 | -0.91 |
| XML\_ParseBuffer | 0.91 | 0.00 | -0.91 |
| sip24\_update.isra.0 | 0.89 | 0.00 | -0.89 |
| XML\_GetBuffer | 0.76 | 0.00 | -0.76 |
| storeAtts | 0.61 | 0.00 | -0.61 |
| sip24\_final | 0.53 | 0.00 | -0.53 |
| little2\_updatePosition | 0.36 | 0.00 | -0.36 |
| doParseXmlDecl | 0.24 | 0.00 | -0.24 |
| XmlPrologStateInit | 0.23 | 0.00 | -0.23 |
| accountingDiffTolerated.part.0 | 0.12 | 0.00 | -0.12 |
| memcmp@plt | 0.07 | 0.00 | -0.07 |
| storeRawNames | 0.04 | 0.00 | -0.04 |
| little2\_toUtf8 | 0.03 | 0.00 | -0.03 |
| XML\_Parse | 0.01 | 0.00 | -0.01 |

#### EVENT: BRANCH-MISSES

| Symbol | x86opt_benchmark % | riscvopt_qemu % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 0.23 | 87.66 | +87.42 |
| little2\_contentTok | 37.26 | 0.00 | -37.26 |
| little2\_updatePosition | 23.47 | 0.00 | -23.47 |
| \[unknown\] (/ | 2.72 | 12.34 | +9.63 |
| little2\_getAtts | 7.22 | 0.00 | -7.22 |
| doContent | 5.83 | 0.00 | -5.83 |
| little2\_scanComment | 3.30 | 0.00 | -3.30 |
| little2\_scanRef | 2.52 | 0.00 | -2.52 |
| lookupWithLength | 2.26 | 0.00 | -2.26 |
| little2\_nameLength | 1.89 | 0.00 | -1.89 |
| storeAtts | 1.89 | 0.00 | -1.89 |
| normal\_contentTok | 1.53 | 0.00 | -1.53 |
| accountingDiffTolerated.part.0 | 1.12 | 0.00 | -1.12 |
| little2\_cdataSectionTok | 1.10 | 0.00 | -1.10 |
| XML\_Parse | 0.92 | 0.00 | -0.92 |
| callProcessor.constprop.0 | 0.84 | 0.00 | -0.84 |
| storeRawNames | 0.83 | 0.00 | -0.83 |
| little2\_scanLit | 0.83 | 0.00 | -0.83 |
| sip24\_update.isra.0 | 0.82 | 0.00 | -0.82 |
| hashTableClear | 0.57 | 0.00 | -0.57 |

### Recorded amixis profile data (`expat.json`)

`amixis profile` could not profile the RISC-V build or the benchmark/xmlwf executables (the tool cannot wrap QEMU, and it runs executables without the benchmark's required arguments). Recorded values:

```json
{
  "1_2_2": {
    "xmlwf/xmlwf": { "build_name": "1_2_2", "executable": "xmlwf/xmlwf", "executable_run_success": false, "real_time": null, "user_time": null, "kernel_time": null, "perf_stat": null, "perf_record_name": null, "perf_script_name": null, "perf_archive_name": null },
    "tests/runtests": { "build_name": "1_2_2", "executable": "tests/runtests", "executable_run_success": false, "real_time": null, "user_time": null, "kernel_time": null, "perf_stat": null, "perf_record_name": null, "perf_script_name": null, "perf_archive_name": null },
    "tests/benchmark/benchmark": { "build_name": "1_2_2", "executable": "tests/benchmark/benchmark", "executable_run_success": false, "real_time": null, "user_time": null, "kernel_time": null, "perf_stat": null, "perf_record_name": null, "perf_script_name": null, "perf_archive_name": null }
  },
  "1_1_1": {
    "xmlwf/xmlwf": { "build_name": "1_1_1", "executable": "xmlwf/xmlwf", "executable_run_success": false, "real_time": null, "user_time": null, "kernel_time": null, "perf_stat": null, "perf_record_name": null, "perf_script_name": null, "perf_archive_name": null },
    "tests/runtests": { "build_name": "1_1_1", "executable": "tests/runtests", "executable_run_success": true, "real_time": "43.43", "user_time": "43.08", "kernel_time": "0.20", "perf_stat": [ { "counter_value": "Error: switch `x' requires a value" }, { "counter_value": "Usage: perf stat [<options>] [<command>]" }, { "counter_value": "-x, --field-separator <separator>" }, { "counter_value": "print counts with custom separator" } ], "perf_record_name": "1__1__1..tests_sruntests.perfdata", "perf_script_name": null, "perf_archive_name": null },
    "tests/benchmark/benchmark": { "build_name": "1_1_1", "executable": "tests/benchmark/benchmark", "executable_run_success": false, "real_time": null, "user_time": null, "kernel_time": null, "perf_stat": null, "perf_record_name": null, "perf_script_name": null, "perf_archive_name": null }
  }
}
```

The real measured performance data used in this report came from the profiler's manual pipeline (`perf stat`, `perf record`, `/usr/bin/time`, and the benchmark's own per-loop timing), saved under `/work/expat-workspace/profiling-data/` and `/work/expat-workspace/profiling-data-opt/`.

---

## 6. Notes About Exploration Process

1. **Provided RISC-V `-march` string invalid.** The supplied config used `-march=rv64gcvb`, which GCC 13.3/binutils 2.42 rejects (`ISA string is not in canonical order. 'b'`). The canonical, assemble-able equivalent `-march=rv64gcv_zba_zbb_zbs` was used (keeps V + Zba/Zbb/Zbs = B intent).
2. **amixis 0.2.0 rejects address-less non-native platforms.** `core/configurator.py:_has_valid_arch()` rejected the riscv platform (which has no address), so no build was created. A minimal environment patch accepting a local arch when a matching `qemu-<arch>` emulator exists was applied. Backup: `/work/expat-workspace/configurator.py.orig`. This is an environment change, not a config change.
3. **`binfmt_misc` unavailable** (read-only kernel mount), so RISC-V binaries could not be executed transparently; every run used `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu`.
4. **QEMU / emulation caveats:** all riscv timings include QEMU translation overhead and are not representative of native RISC-V silicon; target `perf` data is host-side (x86 emulator) only; native RISC-V PMU counters are NOT AVAILABLE; guest symbols are absent from QEMU user-mode samples. `-cpu rv64` caused SIGILL and was avoided.
5. **amixis Profiler limitations.** It cannot wrap QEMU (it joins `build_path`+executable and runs via `sh -c`), and it executes each executable without arguments, so `xmlwf`/`benchmark` smoke tests failed and `perf stat -x|` errored. Consequently `expat.json` is mostly `null`/`executable_run_success:false`. A manual profiler fallback was used for all measurements, with full warmup/repeat/pinning discipline.
6. **`nice -n -20` not permitted** (container lacks `CAP_SYS_NICE`), so both platforms ran at nice 0 (identical on both sides).
7. **CPU frequency was not boosted** (powersave governor, observed ≈1.1–1.4 GHz vs 3.6 GHz max), adding variance; this affects both platforms equally.
8. **`file` command unavailable** — `readelf`/`objdump` used instead.
9. **`wget` unavailable and the first `curl` download stalled** — the XML conformance suite (`xmlts20080827.zip`) was fetched with `curl` directly; the riscv conformance suite was run manually with qemu as the `xmlwf` runner because `make run-xmltest` does not inject the emulator.
10. **`ctest` in the cross build failed** to open `/lib/ld-linux-riscv64-lp64d.so.1` (binfmt handler omits `-L`); riscv tests were run explicitly via qemu instead.
11. **`amixis analyze -v riscv` fails on RISC-V ELF** (generic `objdump` cannot decode RISC-V); `riscv64-linux-gnu-objdump` was used as fallback.
12. **`perf archive` returned rc=1** (debug-link objects not all resolvable); `perf script`/`.scriptout` still succeeded.
13. **Build-directory cwd issue:** an early wrapper invocation created `/work/1_1_1`; it was removed and `amixis build` was re-run from `/work/expat-workspace/`, so builds live at `/work/expat-workspace/1_1_1` and `/work/expat-workspace/1_2_2`.
14. **Optimized variants** `1_1_3`/`1_2_4` were built with a separate config `/work/expat-workspace/input_opt.yml` (recipes 3/4); `/work/input.yml` was not modified.
15. **Cross-tables are not meaningful cross-architecture comparisons** because the target side contains only QEMU-emulator symbols (documented in Sections 4 and 5).

---

## 7. Migration Readiness Summary

| Check | Result |
|---|---|
| Builds on reference (x86_64) | **YES** — `1_1_1` builds successfully |
| Tests pass on reference | **YES** — `runtests` 5196 checks / 0 failed; XML conformance 1801 passed / 8 expected deviations |
| Builds on target (riscv64, cross) | **YES** — `1_2_2` builds successfully (cross-compiled, ELF RISC-V with RVV + Zba/Zbb/Zbs) |
| Tests pass on target (qemu) | **YES** — `runtests` 5196 checks / 0 failed; XML conformance 1801 passed / 8 expected deviations (identical to reference) |
| Zero external dependencies | **YES** — required external deps = 0 (C99 stdlib only) |
| No hand-written intrinsics | **YES** — 0 SIMD intrinsics in source |
| Alignment safe | **YES** — `long long` alignment assumption fixed (PR #1371; `union expat_align` includes `void *`) |
| Exceptions handled | **None required** — no source changes/patches needed |
| Auto-vectorization | **Not applicable / none** — 0 RVV emitted on riscv64; x86 SIMD is scalar-move class, absent from hot parser loops |

### Migration Verdict: **READY**

Expat builds and passes its full test suite (5196 checks, 0 failed) and the XML conformance suite on both the x86_64 reference platform and the riscv64 target (under QEMU user-mode emulation), with **zero required external dependencies, no hand-written intrinsics, no alignment exceptions, and no source patches**. Upstream already runs the complete test + conformance suite on riscv64 in CI and has merged a RISC-V alignment fix. The performance gap measured here is dominated by QEMU emulation overhead and is not attributable to the RISC-V ISA; no native RISC-V PMU was available to quantify native performance.

### Required Actions
1. **None blocking.** The project is ready for riscv64 migration as-is.
2. *(Recommended)* Validate performance on **native RISC-V hardware**; all riscv timings and counters here are QEMU-emulated and cannot represent native silicon.
3. *(Optional)* Re-evaluate the untested optimizations (LTO, PGO, static linking) and the two code-level suggestions (lazy amplification in `accountingDiffTolerated`; word-wise `little2_updatePosition` fast path) on native riscv64, since the tested `-O3`/`-fno-semantic-interposition`/`-funroll-loops` combination gave no benefit and regressed under QEMU.
4. *(Recommended)* Run the project's own `make run-xmltest` under a properly configured emulator (with `-L`), or on native RISC-V, as the CI does.

---

*Report generated by the Amphimixis orchestrator. Tool-owned data files (`improvements.json`, `cross-tables/CT-*.md`, `expat.json`, `expat.pkl`) were read but not modified.*
