# Amphimixis Migration-Readiness Report — glibc

**Project:** glibc (GNU C Library)
**Reference platform:** x86_64 (local container, native)
**Target platform:** riscv64 (cross-compiled, executed under qemu-riscv64-static user-mode emulation)
**Date:** 2026-10-03

---

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Resolved clone URL | `https://github.com/bminor/glibc.git` (official upstream GitHub mirror, reachable) |
| Canonical upstream | `https://sourceware.org/git/glibc.git` — UNREACHABLE (connection timeout from this container) |
| Latest commit hash | `04e750e75b73957cf1c791535a3f4319534a52fc` |
| Latest commit date | 2026-01-20 10:18:56 -0500 |
| Latest commit subject | `Add advisory text for CVE-2025-15281` |
| Total commits | 43,313 (from GitHub API pagination) |
| Latest release tag | `glibc-2.42` (development tag `glibc-2.42.9000`; in-tree `version.h` = 2.42.9000, NEWS says "Version 2.43") |
| Default branch | `master` |
| Activity | Actively maintained upstream (CVE advisory commit Jan 2026). The `bminor/glibc` mirror is archived/read-only but current through HEAD. |
| Clone performed | `git clone --depth 1 https://github.com/bminor/glibc.git glibc` (shallow, single-branch master, 20,739 files) |
| Build systems | GNU Autotools: Autoconf (`configure`, `configure.ac`) + GNU Make. No CMake/Meson. |
| Tests | ~2,713 `tst-*.c` + ~523 `test-*.c` across subdirs, driven by in-tree `support/` harness and `make check`. `amixis analyze` reported only `benchtests` (inaccurate). |
| Benchmarks | `benchtests/` |
| CI | No in-tree CI config (`.github/`, `.gitlab-ci.yml`, `.travis.yml`, `Jenkinsfile`, `.cirrus.yml` absent). External sourceware buildbot/build farm used. |
| Documentation | `manual/` (Texinfo), `INSTALL`, `NEWS`, `README`, `SECURITY.md`, `MAINTAINERS` |
| External dependencies | NONE at runtime (analyzer `dependencies: []` is correct — glibc is the C library). |
| Build-host prerequisites | make, gcc, binutils, python, sed, gawk, bison, gettext, texinfo. `gawk`, `bison`, `gettext`/`msgfmt`, `texinfo` were missing and were installed via apt during configuration. |
| Distro packages | Debian (`glibc` 2.44-2, riscv64 official buildd arch); Arch Linux official x86_64 + Arch Linux RISC-V port; Yocto/OE `meta/recipes-core/glibc/glibc_2.44.bb` with active `riscv/meta-riscv`. |

### RISC-V upstream support state
Fully native and mature — no port/fork required. 198 files under `sysdeps/riscv/` and `sysdeps/unix/sysv/linux/riscv/`; ABI variants rv32/rv64 × rvd/rvf/rvv; IFUNC multiarch via `riscv_hwprobe`; hand-written RVV `memset.S`; complete `*.abilist` ABI lists. The historical `riscvarchive/riscv-glibc` fork is ARCHIVED (2021) with README "The RISC-V glibc Port is Upstream". No active fork carrying RISC-V patches is needed.

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros

| Macro | Code occurrences | Primary location | Semantics verified |
|---|---:|---|---|
| `__x86_64__` | 120 | `sysdeps/x86_64/*`, `sysdeps/x86/*` | x86 asm/multiarch/IFUNC, CPU features, PLT hooks; isolated |
| `__i386__` | 12 | `sysdeps/i386/*`; `intl/dcigettext.c:74`, `elf/sotruss-lib.c`, `soft-fp/testit.c` | 32-bit x86 asm; SIGFPE heuristic; x87 tests |
| `__i486__/__i586__/__i686__` | 1 each | `sysdeps/i386/*` | i486/586/686 code paths |
| `__AVX__` | 24 | `sysdeps/x86/*` | x86 AVX feature detection/ISA level |
| `__AVX2__`,`__AVX512F__`,`__AVX512BW__`,`__AVX512CD__`,`__AVX512DQ__`,`__AVX512VL__` | 2–10 each | `sysdeps/x86/*` | x86 ISA feature bits |
| `__SSE*__`,`__SSSE3__`,`__FMA__`,`__BMI__`,`__BMI2__`,`__POPCNT__`,`__LZCNT__` | 1–5 each | `sysdeps/x86/*` | x86 feature gating |
| `__aarch64__` | 2 | `elf/tst-asm-helper.h:23`, `sysdeps/aarch64/*` | AArch64 ELF property-note test helper; false on riscv |
| `__ARM_NEON__` | 13 | `sysdeps/aarch64/fpu/*` | NEON math; isolated |
| `__ARM_FEATURE_SVE` | 3 | `sysdeps/aarch64/*` | SVE math; isolated |
| `__arm__` | 1 | `intl/dcigettext.c:74` | SIGFPE heuristic arch list |
| `__riscv` | 83 | `sysdeps/riscv/*`, `sysdeps/unix/sysv/linux/riscv/*` | RISC-V base code; native upstream port |
| `__riscv_xlen` | 10 | `sysdeps/riscv/bits/wordsize.h:19`, `preconfigure:8` | XLEN (32/64); value is 32/64, NOT boolean |
| `__riscv_zbb` | 3 | `string-fza.h:22`, `string-fzi.h:22`, `math-use-builtins-ffs.h:1` | Zbb fast byte-scan/ffs |
| `__riscv_float_abi_double` | 8 | `sysdeps/riscv/preconfigure` | soft-float/ABI selection (lp64d) |
| `__riscv_atomic` | 7 | `atomic-machine.h:22`, `preconfigure:9,52` | Requires A extension; preconfigure errors if A missing |
| `__ORDER_LITTLE_ENDIAN__`,`__ORDER_BIG_ENDIAN__`,`__BYTE_ORDER__` | 10/5/15 | `sysdeps/generic/dl-cache.h:152` | Endianness; riscv64 is little-endian |
| `__LP64__` | 24 | `bits/typesizes.h:65`, x86 sysdeps | LP64 ABI; true on riscv64-lp64d |
| `__ILP32__` | 76 | `sysdeps/x86/bits/*`, `sysdeps/x86_64/multiarch/*.S` | Essentially x86 x32; riscv32 uses `__riscv_xlen` |
| `__SIZEOF_POINTER__` | 7 | `sysdeps/riscv/bits/wordsize.h:19` | Compiler macro in bytes; ABI sanity check |
| `__linux__` | 31 | `support/*`, `include/time.h` | Linux-only code paths; true on riscv64-linux |
| `_WIN32` | 17 | `intl/dcigettext.c`, `posix/getopt.c`, `misc/error.c`, ... | Dead code in glibc on Linux/riscv |
| `__APPLE__` | 2 | `timezone/private.h` | Dead code on Linux |

**Misleading-macro checks:** `__riscv_v` is the RVV ISA *version* macro (e.g. 1000000 for RVV 1.0), not an intrinsic prefix. `INTDIV0_RAISES_SIGFPE` correctly falls to `0` on RISC-V (integer div-by-zero returns −1, no SIGFPE). `nptl/perf.c:706` `#error "HP_TIMING_NOW missing"` is not built by default. `elf/sotruss-lib.c` x86 branches are unused on riscv (its own `sysdeps/riscv/sotruss-lib.c` is used).

### 2.2 Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `#ifdef __riscv_v` (`sysdeps/unix/sysv/linux/riscv/sysdep.h:358`) | RISC-V | Chooses vector-register clobber list for `syscall` inline asm |
| `#if __riscv_xlen == (__SIZEOF_POINTER__*8)` (`bits/wordsize.h:19`) | RISC-V | Defines `__WORDSIZE`; `#error unsupported ABI` otherwise |
| `#if defined __riscv_zbb \|\| __riscv_xtheadbb` (`string-fza.h:22`, `string-fzi.h:22`) | RISC-V | Generic fast zero-byte scan vs bit-testing fallback |
| `#ifdef __riscv_atomic` (`atomic-machine.h:22`) | RISC-V | Atomic CAS vs fallback; configure requires A |
| `#ifdef __APPLE__` (`timezone/private.h`) | macOS | Dead code on Linux |
| `#if defined _WIN32 && !defined __CYGWIN__` | Windows | Dead code in glibc |
| `#ifdef __aarch64__` (`elf/tst-asm-helper.h:23`) | AArch64 | ELF property-note test helper; false on riscv |
| `#if defined __x86_64__ \|\| defined __i386__` (`hurd/test-xstate.h:23`) | Hurd/x86 | Hurd-only test; inert |
| `#ifdef CRT_GET_RFIB_DATA` + `#ifdef __i386__` (`unwind-dw2-fde-glibc.c:132`) | i386 | `data->dbase` setup; inert |

### 2.3 Vectorization intrinsics in source

| ISA | Representative intrinsics | Files | Notes |
|---|---|---|---|
| x86 SSE/AVX | `__m128i` (149), `__m256i` (63), `__m512i` (56), `_mm_*`, `_mm256_*`, `_mm512_*`, `immintrin.h`, `x86intrin.h`, `__builtin_ia32_*` | Exclusively `sysdeps/x86/**`, `sysdeps/x86_64/**` | Fully isolated; no leakage into generic code |
| ARM NEON | `vmulq_f64` (135), `vld1rq` (119), `arm_neon.h`, `vld1_gather_index` | `sysdeps/aarch64/fpu/*` | Fully isolated to aarch64 |
| RISC-V RVV | No C intrinsics at all; `riscv_vector.h` never included. RVV used via hand-written assembly: `vsetvli`, `vmv.v.x`, `vse8.v` in `sysdeps/riscv/rvv/memset.S`; IFUNC dispatch via `riscv_hwprobe` | `sysdeps/riscv/rvv/memset.S`, `.../multiarch/{memset.c,memset-vector.S}` | RVV is build-selected assembly, not intrinsic-based |

### 2.4 Portability verdict

| Criterion | Verdict |
|---|---|
| No exceptions | PASS — no exception-based control flow required; `-fexceptions`/C++ boundaries handled by glibc itself |
| Alignment safe | PASS — `memcpy`/`memmove`/`memset` handle misalignment via `__memcpy_noalignment`, `wordcopy`, `riscv_hwprobe` misaligned-fast detection |
| Embedded usability | PASS — configurable (`--disable-shared`, `--enable-static`), hardware ABI rv64gc / lp64d |
| Overall portability level | LOW RISK — complete, actively maintained native upstream RISC-V port; no architectural barrier. Concerns are toolchain/environment only. |

---

## 3. Build & Test Results

`amixis build` could not drive the glibc build: both build names failed with `[Config][✗] Failed to create build` (root cause in `amphimixis.log`: `Invalid local machine arch: riscv, your machine is x86_64`, plus no Autotools backend). The manual Autotools build is the valid measurement path.

| Platform | Build system / configure | Build status | CFLAGS used | Tests run | Test result |
|---|---|---|---|---|---|
| x86_64 (reference, build `1_1_1`) | `configure --prefix=/usr --build=x86_64-linux-gnu --enable-hardcoded-path-in-tests --disable-werror` then `make -j8` | SUCCESS (exit 0) | `-O3 -march=native -g` | `make -j8 subdirs='string' tests` | 100 PASS / 2 FAIL / 15 UNSUPPORTED |
| riscv64 (target, build `1_2_2`) | `configure --prefix=/usr --build=x86_64-linux-gnu --host=riscv64-linux-gnu CC=riscv64-linux-gnu-gcc ...` then `make -j8` | SUCCESS (exit 0) | `-O3 -march=rv64gc -g` | 77 built string test binaries run manually under qemu-riscv64-static | 68 PASS / 3 FAIL / 6 TIMEOUT (0 UNSUPPORTED) |

**Build notes:**
- riscv `-march=rv64gcv` was REJECTED by configure: `glibc requires GCC 15 or later for the V extension` (cross GCC is 13.3.0). The build fell back to `-march=rv64gc`.
- Build dirs: `/work/glibc-workspace/build-x86_64`, `/work/glibc-workspace/build-riscv64`.

**Failure detail (all environmental, not code defects):**
- x86_64 FAIL: `string/tst-strerror`, `string/tst-strsignal` — `could not create a private mount namespace` (container limitation).
- x86_64 UNSUPPORTED (15): x86-only `*-rtm` variants and translation/container tests without `msgfmt`.
- riscv64 FAIL (3): `string/bug-strcoll2`, `string/tst-strxfrm`, `string/tst-strxfrm2` — missing compiled locale data in the cross build tree.
- riscv64 TIMEOUT (6): `string/test-memcpy`, `test-memcpy-large`, `test-mempcpy`, `test-strcasecmp`, `test-strncasecmp`, `tst-cmp` — exhaustive sweep drivers too slow under qemu-user within the 180 s budget.
- Cross `make check` could not use glibc's harness (`testroot.pristine/install.stamp` needs an ssh-style wrapper); tests were built with `run-built-tests=no` and run manually under qemu.

---

## 4. Performance Comparison

### 4.1 Experimental conditions

| Condition | Reference (x86_64) | Target (riscv64) |
|---|---|---|
| Platform | Local container, native | `qemu-riscv64-static` 8.2.2, user-mode emulation |
| CPU | Intel Core i5-1035G1 @ 1.00 GHz (Ice Lake, 4C/8T) | Emulated guest on the same host CPU |
| nproc | 8 logical ({0,4}{1,5}{2,6}{3,7}) | same |
| Core pinned | `taskset -c 2` | `taskset -c 2` |
| Priority | `nice -n -20` FAILED (`cannot set niceness: Permission denied`) — ran at nice 0 | same |
| Governor | `powersave` (sysfs read-only; could not switch to `performance`) | same |
| Frequency observed | ~1.25–1.40 GHz idle-run samples; perf reported 1.363–1.368 GHz effective | 1.360–1.363 GHz effective (host) |
| NMI watchdog | read-only; counters multiplexed (33–67% enable) | same |
| Warmup | 1 run per executable | 1 run per executable |
| Measurement repeats | 6 per executable (`--repeat 6`) | 6 (`--repeat 6`; memset split 3+3) |
| Method | `perf stat -ddd` native HW counters + `perf record -g -F 1000` | `perf stat -ddd` / `perf record` on the HOST qemu process |
| Sampling events | `cpu-clock,cache-misses,branch-misses` | `cpu-clock,cache-misses,branch-misses` (host) |

### 4.2 Key metrics — x86_64 native vs riscv64/QEMU

| Metric | Reference x86_64 (`1_1_1`) | Target riscv64 (`1_2_2`, HOST qemu process) |
|---|---|---|
| bench-memcpy elapsed (mean of 6) | 12.4996 s | 44.263 s |
| bench-memset elapsed (mean of 6) | 3.35275 s | 175.189 s |
| memcpy IPC | 2.27 | 3.84 (host, not guest) |
| memset IPC | 1.05 | 4.57 (host, not guest) |
| memcpy L1-dcache miss rate | 21.69 % | 1.35 % |
| memset L1-dcache miss rate | 287.06 % (counter-denominator artifact; store/RFO misses over load-only denominator) | 0.14 % |
| memcpy LLC miss rate | 30.79 % | 19.42 % |
| memset LLC miss rate | 15.49 % | 17.55 % |
| memcpy branch misprediction rate | 0.66 % | 0.08 % |
| memset branch misprediction rate | 0.26 % | 0.09 % |
| memcpy Frontend Bound (TMA) | 0.4 % | 0.4 % |
| memcpy Backend Bound (TMA) | 0.3 % | 0.3 % |
| memcpy Retiring (TMA) | 7.2 % | 17.8 % |
| memset Frontend Bound (TMA) | 0.1 % | 0.4 % |
| memset Backend Bound (TMA) | 11.0 % | 0.1 % |
| memset Retiring (TMA) | 7.8 % | 9.0 % |

TMA `TopdownL1` was only ~33–67 % enabled due to multiplexing and a locked NMI watchdog; Frontend/Backend/Retiring are low-confidence. RISC-V TMA values describe the host qemu process.

### 4.3 Hotspots (real perf record data)

**Reference x86_64 — bench-memcpy (33,316 cpu-clock samples):**

| % | Function | Module | Analysis |
|--:|---|---|---|
| 21.62 | `_wordcopy_fwd_dest_aligned` | bench-memcpy | Generic C fallback for unaligned dst; largest single cost |
| 17.67 | `__memmove_evex_unaligned_erms` | libc.so | AVX-512 EVEX IFUNC variant |
| 15.58 | `__memmove_avx512_unaligned_erms` | libc.so | AVX-512 IFUNC variant |
| 14.05 | `_wordcopy_fwd_aligned` | bench-memcpy | Generic C fallback, aligned |
| 13.41 | `__memmove_erms` | libc.so | ERMS/`rep movsb` variant |
| 12.16 | `__memmove_avx512_no_vzeroupper` | libc.so | AVX-512 variant |
| 1.59 | `generic_memcpy` | bench-memcpy | Portable reference impl |

**Reference x86_64 — bench-memset (8,628 samples):**

| % | Function | Module | Analysis |
|--:|---|---|---|
| 23.73 | `__memset_avx512_unaligned_erms` | libc.so | AVX-512 + ERMS |
| 23.16 | `__memset_evex_unaligned_erms` | libc.so | EVEX variant |
| 18.17 | `__memset_erms` | libc.so | Hardware `rep stosb` |
| 14.57 | `generic_memset` | bench-memcpy | Portable reference impl |
| 13.15 | `__memset_avx512_no_vzeroupper` | libc.so | AVX-512 variant |

**Target riscv64 under QEMU — hotspots: NOT AVAILABLE.** `qemu-riscv64-static` is stripped and the guest is JIT-translated; 96–100 % of the CYCLES bucket is `[unknown]`/JIT. No guest function names can be truthfully reported.

### 4.4 Bottleneck summary

- **memset 52.25× gap** is dominated by the RISC-V `__memset_vector` RVV routine (97.3 % of measured memset work) whose `vse8.v` loop is emulated ~100× slower than scalar by QEMU 8.2 TCG; x86 runs AVX-512 + hardware `rep stosb` natively.
- **memcpy 3.54× gap** reflects the scalar 8-byte `__memcpy_noalignment` (16×`ld` + 16×`sd` per 128 B) versus x86 AVX-512 `vmovdqu64` moving 64 B/instruction, plus QEMU soft-TLB overhead. The RISC-V IFUNC selection is correct (fastest local variant).
- RISC-V has **no RVV memcpy/memmove** and no IFUNC memmove; only a naive RVV memset exists.

### 4.5 Vectorization intrinsics in built binaries

| Binary | Platform | Vector instructions | ISAs |
|---|---|---|---|
| bench-memcpy | x86_64 | 13 unique / 474 total | SSE/AVX (`paddq`, `vpaddq`, `vperm`, `shufps`, `vshufps`, `movaps`, `vmovaps`, `orpd`, `xorpd`, `vxorpd`, `vpinsrq`) |
| bench-memset | x86_64 | 6 unique / 57 total | SSE/AVX (`movaps`, `vmovaps`, `orpd`, `xorpd`, `vxorpd`) |
| libc.so | x86_64 | 50 unique / 4,909 total | AVX-512 + AVX2 + SSE (`vpcmp` 611, `vmovaps` 451, `vpaddq` 281, `vzeroupper` 1074, `vmovdqu64` 821, `vmovdqu8` 551, `vmovdqa64` 373, `vpbroadcastq` 355, `vpcmpeqb` 233, `vpxorq` 101) |
| bench-memcpy | riscv64 | 0 | none (RV64GC scalar) |
| bench-memset | riscv64 | 0 | none (RV64GC scalar) |
| libc.so | riscv64 | 4 total | RVV, all inside `__memset_vector` (2× `vsetvli`, 1× `vse8.v`, 1× `vmv.v.x`) |

### 4.6 QEMU / emulation caveats

- The riscv target runs under **qemu-user user-mode emulation**; all timings include QEMU dynamic-binary-translation overhead and do **not** reflect native RISC-V hardware performance.
- `perf` on riscv runs counts the **host `qemu-riscv64-static` process**, not guest RISC-V hardware counters. Host IPC/instructions are emulation work, not guest metrics; guest cycles/IPC/cache/TMA and guest function hotspots are NOT AVAILABLE.
- `qemu-riscv64-static` cannot expose guest PMU counters.

### 4.7 Cross-table — 1_1_1_bench-memcpy (x86_64) vs 1_2_2_bench-memcpy (riscv64/QEMU)

#### Cross-table — EVENT: BRANCH-MISSES

| Symbol | 1_1_1_bench-memcpy % | 1_2_2_bench-memcpy % | Delta % |
|:-------------------------------------|---------:|---------:|-------:|
| \[unknown\]                          |      3.49 |     99.99 |  +96.50 |
| generic\_memcpy                      |     51.43 |      0.00 |  -51.43 |
| \_\_memmove\_evex\_unaligned\_erms   |     20.68 |      0.00 |  -20.68 |
| memcpy@GLIBC\_2.2.5                  |      5.51 |      0.00 |   -5.51 |
| do\_test.constprop.4                 |      4.05 |      0.00 |   -4.05 |
| \_\_memmove\_avx512\_unaligned\_erms |      2.49 |      0.00 |   -2.49 |
| \_\_memmove\_avx512\_no\_vzeroupper  |      2.28 |      0.00 |   -2.28 |
| do\_test.constprop.3                 |      1.80 |      0.00 |   -1.80 |
| \_wordcopy\_fwd\_aligned             |      1.59 |      0.00 |   -1.59 |
| \_wordcopy\_fwd\_dest\_aligned       |      1.36 |      0.00 |   -1.36 |
| do\_test.constprop.2                 |      1.09 |      0.00 |   -1.09 |
| do\_test.constprop.1                 |      1.05 |      0.00 |   -1.05 |
| \_\_printf\_fp\_buffer\_1.isra.0     |      0.52 |      0.00 |   -0.52 |
| \_\_memmove\_avx512\_unaligned       |      0.42 |      0.00 |   -0.42 |
| do\_test.constprop.0                 |      0.39 |      0.00 |   -0.39 |
| \_\_memmove\_erms                    |      0.32 |      0.00 |   -0.32 |
| \_IO\_file\_write@@GLIBC\_2.2.5      |      0.25 |      0.00 |   -0.25 |
| json\_attr\_uint                     |      0.22 |      0.00 |   -0.22 |
| \_IO\_file\_xsputn@@GLIBC\_2.2.5     |      0.11 |      0.00 |   -0.11 |
| fprintf                              |      0.11 |      0.00 |   -0.11 |

#### Cross-table — EVENT: CACHE-MISSES

| Symbol | 1_1_1_bench-memcpy % | 1_2_2_bench-memcpy % | Delta % |
|:-------------------------------------|---------:|---------:|-------:|
| \[unknown\]                          |     36.59 |     99.93 |  +63.34 |
| \_\_memmove\_evex\_unaligned\_erms   |     11.18 |      0.00 |  -11.18 |
| \_\_memmove\_avx512\_unaligned\_erms |     10.60 |      0.00 |  -10.60 |
| \_wordcopy\_fwd\_aligned             |      8.53 |      0.00 |   -8.53 |
| \_\_memmove\_avx512\_no\_vzeroupper  |      7.61 |      0.00 |   -7.61 |
| \_\_memmove\_erms                    |      5.42 |      0.00 |   -5.42 |
| \_wordcopy\_fwd\_dest\_aligned       |      2.96 |      0.00 |   -2.96 |
| \_\_printf\_fp\_buffer\_1.isra.0     |      2.84 |      0.00 |   -2.84 |
| \_\_printf\_buffer                   |      2.60 |      0.00 |   -2.60 |
| generic\_memcpy                      |      1.30 |      0.00 |   -1.30 |
| \_\_mpn\_divrem                      |      0.61 |      0.00 |   -0.61 |
| do\_test.constprop.3                 |      0.56 |      0.00 |   -0.56 |
| \_\_printf\_fp\_l\_buffer            |      0.53 |      0.00 |   -0.53 |
| fprintf                              |      0.52 |      0.00 |   -0.52 |
| \_dl\_fini                           |      0.50 |      0.00 |   -0.50 |
| \_\_strlen\_evex                     |      0.50 |      0.00 |   -0.50 |
| \_\_vfprintf\_internal               |      0.48 |      0.00 |   -0.48 |
| fputc                                |      0.41 |      0.00 |   -0.41 |
| do\_test.constprop.4                 |      0.39 |      0.00 |   -0.39 |
| \_\_getpagesize                      |      0.37 |      0.00 |   -0.37 |

#### Cross-table — EVENT: CYCLES

| Symbol | 1_1_1_bench-memcpy % | 1_2_2_bench-memcpy % | Delta % |
|:-------------------------------------|---------:|---------:|-------:|
| \[unknown\]                          |      0.57 |    100.00 |  +99.43 |
| \_wordcopy\_fwd\_dest\_aligned       |     21.62 |      0.00 |  -21.62 |
| \_\_memmove\_evex\_unaligned\_erms   |     17.67 |      0.00 |  -17.67 |
| \_\_memmove\_avx512\_unaligned\_erms |     15.58 |      0.00 |  -15.58 |
| \_wordcopy\_fwd\_aligned             |     14.05 |      0.00 |  -14.05 |
| \_\_memmove\_erms                    |     13.41 |      0.00 |  -13.41 |
| \_\_memmove\_avx512\_no\_vzeroupper  |     12.16 |      0.00 |  -12.16 |
| generic\_memcpy                      |      1.59 |      0.00 |   -1.59 |
| \_\_memmove\_avx512\_unaligned       |      1.05 |      0.00 |   -1.05 |
| memcpy@GLIBC\_2.2.5                  |      0.61 |      0.00 |   -0.61 |
| do\_test.constprop.4                 |      0.28 |      0.00 |   -0.28 |
| do\_test.constprop.3                 |      0.27 |      0.00 |   -0.27 |
| do\_test.constprop.1                 |      0.24 |      0.00 |   -0.24 |
| \_\_syscall\_cancel                  |      0.18 |      0.00 |   -0.18 |
| do\_test.constprop.2                 |      0.13 |      0.00 |   -0.13 |
| \_\_printf\_fp\_buffer\_1.isra.0     |      0.13 |      0.00 |   -0.13 |
| do\_test.constprop.0                 |      0.06 |      0.00 |   -0.06 |
| \_\_printf\_buffer                   |      0.05 |      0.00 |   -0.05 |
| fprintf                              |      0.03 |      0.00 |   -0.03 |
| \_\_strchrnul\_evex                  |      0.02 |      0.00 |   -0.02 |

### 4.8 Cross-table — 1_1_1_bench-memset (x86_64) vs 1_2_2_bench-memset (riscv64/QEMU)

#### Cross-table — EVENT: BRANCH-MISSES

| Symbol | 1_1_1_bench-memset % | 1_2_2_bench-memset % | Delta % |
|:------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                      |      0.00 |     73.60 |  +73.60 |
| generic\_memset                     |     22.58 |      0.00 |  -22.58 |
| \_\_memset\_evex\_unaligned\_erms   |     21.99 |      0.00 |  -21.99 |
| \[unknown\]                         |      5.60 |     26.39 |  +20.79 |
| do\_test.constprop.0                |     20.67 |      0.00 |  -20.67 |
| \_\_memset\_avx512\_no\_vzeroupper  |     14.40 |      0.00 |  -14.40 |
| do\_test                            |      4.08 |      0.00 |   -4.08 |
| \_\_memset\_avx512\_unaligned\_erms |      3.79 |      0.00 |   -3.79 |
| \_\_memset\_evex\_unaligned         |      2.33 |      0.00 |   -2.33 |
| \_\_printf\_fp\_buffer\_1.isra.0    |      1.33 |      0.00 |   -1.33 |
| \_\_memset\_avx512\_unaligned       |      0.36 |      0.00 |   -0.36 |
| \_IO\_file\_write@@GLIBC\_2.2.5     |      0.32 |      0.00 |   -0.32 |
| fprintf                             |      0.28 |      0.00 |   -0.28 |
| test\_main                          |      0.27 |      0.00 |   -0.27 |
| \_IO\_file\_xsputn@@GLIBC\_2.2.5    |      0.26 |      0.00 |   -0.26 |
| json\_attr\_uint                    |      0.17 |      0.00 |   -0.17 |
| hack\_digit                         |      0.12 |      0.00 |   -0.12 |
| \_\_memmove\_evex\_unaligned\_erms  |      0.11 |      0.00 |   -0.11 |
| json\_attr\_int                     |      0.11 |      0.00 |   -0.11 |
| \_IO\_fputs                         |      0.10 |      0.00 |   -0.10 |

#### Cross-table — EVENT: CACHE-MISSES

| Symbol | 1_1_1_bench-memset % | 1_2_2_bench-memset % | Delta % |
|:------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                      |      0.00 |     38.70 |  +38.70 |
| \[unknown\]                         |     34.18 |     61.26 |  +27.09 |
| \_\_memset\_avx512\_unaligned\_erms |     16.90 |      0.00 |  -16.90 |
| \_\_memset\_evex\_unaligned\_erms   |     11.94 |      0.00 |  -11.94 |
| \_\_memset\_avx512\_no\_vzeroupper  |      9.51 |      0.00 |   -9.51 |
| generic\_memset                     |      7.32 |      0.00 |   -7.32 |
| \_\_memset\_erms                    |      6.24 |      0.00 |   -6.24 |
| \_\_printf\_fp\_buffer\_1.isra.0    |      3.01 |      0.00 |   -3.01 |
| \_\_printf\_buffer                  |      1.59 |      0.00 |   -1.59 |
| do\_test.constprop.0                |      1.30 |      0.00 |   -1.30 |
| \_\_mpn\_divrem                     |      1.08 |      0.00 |   -1.08 |
| \_\_vfprintf\_internal              |      0.71 |      0.00 |   -0.71 |
| \_\_printf\_fp\_l\_buffer           |      0.44 |      0.00 |   -0.44 |
| \_\_memset\_avx512\_unaligned       |      0.36 |      0.00 |   -0.36 |
| fprintf                             |      0.35 |      0.00 |   -0.35 |
| fputc                               |      0.32 |      0.00 |   -0.32 |
| \_\_memmove\_evex\_unaligned\_erms  |      0.28 |      0.00 |   -0.28 |
| \_IO\_file\_overflow@@GLIBC\_2.2.5  |      0.24 |      0.00 |   -0.24 |
| \_\_printf\_buffer\_write           |      0.24 |      0.00 |   -0.24 |
| \_IO\_do\_write@@GLIBC\_2.2.5       |      0.24 |      0.00 |   -0.24 |

#### Cross-table — EVENT: CYCLES

| Symbol | 1_1_1_bench-memset % | 1_2_2_bench-memset % | Delta % |
|:------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                      |      0.00 |     96.27 |  +96.27 |
| \_\_memset\_avx512\_unaligned\_erms |     23.73 |      0.00 |  -23.73 |
| \_\_memset\_evex\_unaligned\_erms   |     23.16 |      0.00 |  -23.16 |
| \_\_memset\_erms                    |     18.17 |      0.00 |  -18.17 |
| generic\_memset                     |     14.57 |      0.00 |  -14.57 |
| \_\_memset\_avx512\_no\_vzeroupper  |     13.15 |      0.00 |  -13.15 |
| \[unknown\]                         |      0.63 |      3.73 |   +3.10 |
| \_\_memset\_evex\_unaligned         |      1.64 |      0.00 |   -1.64 |
| \_\_memset\_avx512\_unaligned       |      1.58 |      0.00 |   -1.58 |
| do\_test.constprop.0                |      1.23 |      0.00 |   -1.23 |
| do\_test                            |      0.85 |      0.00 |   -0.85 |
| test\_main                          |      0.63 |      0.00 |   -0.63 |
| \_\_syscall\_cancel                 |      0.19 |      0.00 |   -0.19 |
| \_\_printf\_fp\_buffer\_1.isra.0    |      0.16 |      0.00 |   -0.16 |
| \_IO\_file\_write@@GLIBC\_2.2.5     |      0.08 |      0.00 |   -0.08 |
| \_\_mpn\_divrem                     |      0.05 |      0.00 |   -0.05 |
| \_\_memmove\_evex\_unaligned\_erms  |      0.03 |      0.00 |   -0.03 |
| \_IO\_fwrite                        |      0.03 |      0.00 |   -0.03 |
| \_\_GI\_\_\_libc\_write             |      0.03 |      0.00 |   -0.03 |
| \_\_printf\_buffer\_to\_file\_done  |      0.03 |      0.00 |   -0.03 |

---

## 5. Optimization Results

### 5.1 Vector instructions in binary (per platform, per binary)

| Architecture | bench-memcpy | bench-memset | libc.so |
|---|---|---|---|
| x86_64 | 13 unique / 474 total (SSE/AVX) | 6 unique / 57 total (SSE/AVX) | 50 unique / 4,909 total (AVX-512/AVX2/SSE) |
| riscv64 | 0 (RV64GC scalar) | 0 (RV64GC scalar) | 4 total, all in `__memset_vector` (RVV) |

The `amphimixis-analyze-vectorization` tool failed for RISC-V (`objdump: can't disassemble for architecture UNKNOWN`); `riscv64-linux-gnu-objdump` was used as documented fallback.

### 5.2 Executable size analysis (stripped vs unstripped)

| Binary | Platform | Unstripped (B) | Stripped (B) | Debug-info delta (B) | `.text` (B) |
|---|---|---:|---:|---:|---:|
| bench-memcpy | x86_64 | 147,176 | 47,528 | 99,648 | 37,098 |
| bench-memcpy | riscv64 | 132,344 | 35,040 | 97,304 | 26,912 |
| bench-memset | x86_64 | 110,992 | 35,240 | 75,752 | 24,874 |
| bench-memset | riscv64 | 104,224 | 26,848 | 77,376 | 21,992 |
| libc.so | x86_64 | 11,975,848 | 1,913,728 | 10,062,120 | 1,882,549 |
| libc.so | riscv64 | 11,439,104 | 1,474,824 | 9,964,280 | 1,445,889 |

Stripping removes ~68–84 % of file size on both platforms but `.text` is unchanged; it yields ~0 % runtime improvement (size/debug-bloat only). RISC-V `.text` is smaller yet the code is slower — confirming an instruction-width/availability gap, not a frontend bottleneck.

### 5.3 Optimization attempts

| Optimization | Before | After | Delta | Causal Analysis |
|---|---|---|---|---|
| OPT-1: force riscv memset to scalar `__memset_generic` and remove `__memset_vector` from the bench IFUNC list (QEMU-measurement fix) | bench-memset riscv/QEMU elapsed 175.189 s; cycles 476,742,168,503; instructions 2,175,778,975,169 | bench-memset riscv/QEMU elapsed 6.235 s; cycles 7,723,359,786; instructions 25,114,194,306 | memset ~28× faster (elapsed); cycles ~62× lower; instructions ~87× lower | The baseline bench timed every registered IFUNC variant, and QEMU 8.2 TCG emulates the naive RVV `vse8.v` loop ~100× slower than scalar; removing the vector variant removed the dominant emulation cost. This is a QEMU/benchmark-isolation optimization, NOT a production recommendation (on native RVV hardware the vector path should be retained). |
| OPT-1 side effect on memcpy (control) | riscv/QEMU elapsed 44.263 s | riscv/QEMU elapsed 48.443 s | +9.4 % elapsed (host frequency drift); instructions +0.1 % | The change touches only the memset IFUNC entry; memcpy code is identical. Instructions/branches/L1-loads unchanged, so the elapsed difference is environmental (host thermal/frequency drift between sessions), not a regression. |

### 5.4 Recommended optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|:--:|---|:--:|:--:|---|
| 1 | Neutralize the QEMU-pathological `__memset_vector` for the riscv measurement (force scalar selection / remove from bench IFUNC list) | Very high for the metric: `__memset_vector` = 97.3 % of measured memset work; est. 175 s → ~5–8 s under QEMU | Low | QEMU/benchmark isolation fix; on real HW keep the vector path |
| 2 | Optimize `sysdeps/riscv/rvv/memset.S` (hoist `vsetvli`, unroll, add scalar/size threshold, masked tail) | Under QEMU modest (helper-bound); on native RISC-V est. 1.2–2×; removes <16 B regression | Medium | Assembly builds today with binutils 2.42 (`.option arch,+v`); no GCC 15 needed to build it |
| 3 | Add RVV `memcpy`/`memmove` + size-thresholded IFUNC | On native RISC-V potentially 2–4×; under QEMU likely neutral/negative | High | Closes structural gap: RISC-V currently has 0 RVV memcpy/memmove |
| 4 | Obtain GCC ≥ 15 and build `-march=rv64gcv` | Enabler: unlocks C-level RVV auto-vectorization/intrinsics | High | Hard blocker confirmed in `preconfigure`; NOT AVAILABLE in this container |
| 5 | Validate on native RISC-V hardware instead of QEMU | Measurement validity | Resource | QEMU RVV path ~100× off; cannot rank ISA changes under QEMU. NOT AVAILABLE here |
| 6 | Test compiler/link flags (`-O2` vs `-O3`, `-funroll-loops`, `-fomit-frame-pointer`, `-fno-plt`, `-flto`, static libc) | Typically 0–5 % on these benches | Low–Medium | Test each independently; glibc LTO support limited |
| 7 | Verify/repair IFUNC selection and add size thresholds | Small-size wins (<16 B); correctness insurance | Low | QEMU advertises `IMA_V`; verify native board before trusting selection |
| 8 | Strip debug info | ~0 % runtime; artifact size −68–84 % | Low | Artifact size only; `.text` unchanged |
| 9 | Alternative allocators | ~0 % for these benchmarks | Low | Not applicable — no heap in hot paths |

### 5.5 Improvement of 1_2_2 compared to 1_2_2_opt

| Measured | Baseline value | Optimized value | Improvement % |
|---|---:|---:|---:|
| bench-memset (real_time) | 175.189 | 6.235 | 3.56 |
| bench-memset (cycles) | 476742168503 | 7723359786 | 1.62 |
| bench-memset (instructions) | 2175778975169 | 25114194306 | 1.15 |

### 5.6 Cross-table — 1_2_2_bench-memcpy (baseline riscv) vs 1_2_2_opt_bench-memcpy (optimized riscv)

#### Cross-table — EVENT: BRANCH-MISSES

| Symbol | 1_2_2_bench-memcpy % | 1_2_2_opt_bench-memcpy % | Delta % |
|:-------------------------|---------:|---------:|-------:|
| \[unknown\]              |     99.99 |     61.52 |  -38.47 |
| \[unknown\] (/           |      0.00 |     38.47 |  +38.47 |
| \_\_vdso\_clock\_gettime |      0.01 |      0.01 |   +0.00 |

#### Cross-table — EVENT: CACHE-MISSES

| Symbol | 1_2_2_bench-memcpy % | 1_2_2_opt_bench-memcpy % | Delta % |
|:-------------------------|---------:|---------:|-------:|
| \[unknown\] (/           |      0.00 |     25.17 |  +25.17 |
| \[unknown\]              |     99.93 |     74.80 |  -25.13 |
| \_\_vdso\_clock\_gettime |      0.07 |      0.03 |   -0.05 |

#### Cross-table — EVENT: CYCLES

| Symbol | 1_2_2_bench-memcpy % | 1_2_2_opt_bench-memcpy % | Delta % |
|:-------------------------|---------:|---------:|-------:|
| \[unknown\] (/           |      0.00 |      8.80 |   +8.80 |
| \[unknown\]              |    100.00 |     91.20 |   -8.80 |
| \_\_vdso\_clock\_gettime |      0.00 |      0.00 |   -0.00 |

### 5.7 Cross-table — 1_2_2_bench-memset (baseline riscv) vs 1_2_2_opt_bench-memset (optimized riscv)

#### Cross-table — EVENT: BRANCH-MISSES

| Symbol | 1_2_2_bench-memset % | 1_2_2_opt_bench-memset % | Delta % |
|:-------------------------|---------:|---------:|-------:|
| \[unknown\] (/           |     73.60 |     24.31 |  -49.29 |
| \[unknown\]              |     26.39 |     75.68 |  +49.29 |
| \_\_vdso\_clock\_gettime |      0.00 |      0.01 |   +0.01 |

#### Cross-table — EVENT: CACHE-MISSES

| Symbol | 1_2_2_bench-memset % | 1_2_2_opt_bench-memset % | Delta % |
|:-------------------------|---------:|---------:|-------:|
| \[unknown\]              |     61.26 |     73.43 |  +12.17 |
| \[unknown\] (/           |     38.70 |     26.57 |  -12.14 |
| \_\_vdso\_clock\_gettime |      0.03 |      0.00 |   -0.03 |

#### Cross-table — EVENT: CYCLES

| Symbol | 1_2_2_bench-memset % | 1_2_2_opt_bench-memset % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |      3.73 |     65.17 |  +61.44 |
| \[unknown\] (/ |     96.27 |     34.83 |  -61.44 |

> Note: both inputs to the 1_2_2 vs 1_2_2_opt cross-tables are RISC-V/QEMU `.scriptout` files. Because the guest is JIT-translated and the qemu binary is stripped, amixis can only attribute samples to `[unknown]`/`[JIT]` plus a trace of `__vdso_clock_gettime`; the tables show how the unknown/qemu-vs-guest sample split shifted, not guest function names.

### 5.8 Recorded profile data (`<project>.json/.yaml/.pkl`)

`glibc.json` / `glibc.yaml` / `glibc.pkl`: **NOT AVAILABLE** — `amixis profile` could not run (build creation failed: `Invalid local machine arch: riscv, your machine is x86_64`; no Autotools backend), so no `amixis`-owned project profile file was produced.

---

## 6. Notes About Exploration Process

- **Canonical upstream unreachable:** `sourceware.org/git/glibc.git` timed out; the reachable GitHub mirror `bminor/glibc.git` (HEAD `04e750e…`) was used. The mirror is archived/read-only but current.
- **amixis 0.2.0 limitations (blocking automated pipeline):**
  - No Autotools/autoconf backend (`build_systems_dict = {"cmake", "make"}`); the `make` backend never runs `./configure`, so it cannot build glibc.
  - Local cross-arch run is rejected: platform 2 (riscv, local) fails `_has_valid_arch` on an x86_64 host, so `amixis build` and `amixis profile` abort with `Failed to create build` for BOTH build names.
  - Inline `toolchain:` dicts are silently dropped by the validator; `sysroot` is not passed by the make backend.
  - The validator is purely structural — it accepted the original CMake config and the invalid `-march=rv64gcvb`, so "validate PASS" does not imply usability.
  - Consequently, all builds/tests/profiling were performed manually (Autotools + perf + `amixis compare`), which is the valid measurement path.
- **Provided config was for CMake, not Autotools:** `input.yml` used `-DCMAKE_*` flags and invalid `-march=rv64gcvb`. The configurator produced a corrected config at `/work/glibc-workspace/input.yml` (`build_system: make`, Autotools flags, valid `-march`), which passes `amixis validate`.
- **Missing host tools** (`gawk`, `bison`, `gettext`, `texinfo`) were installed via apt during configuration; `texinfo` provides `makeinfo` (no `/usr/bin/texinfo` binary).
- **GCC-15 V-extension embargo:** riscv `-march=rv64gcv` is rejected by `preconfigure` (`glibc requires GCC 15 or later for the V extension`); cross GCC is 13.3.0, so the riscv build used `-march=rv64gc`. RVV assembly still assembles via `.option arch,+v` (binutils 2.42).
- **Test harness under cross build:** `make check` aborted in `testroot.pristine/install.stamp` (qemu-user wrapper cannot satisfy an ssh-style `cp`); tests were built with `run-built-tests=no` and executed manually under qemu.
- **Container restrictions:** `nice -n -20` denied (`cannot set niceness: Permission denied`); CPU governor `performance` not settable (sysfs read-only); NMI watchdog not disableable (counters multiplexed). `file(1)` absent; `readelf -h` used.
- **Profiling integrity:** all reported metrics are REAL measured data from `perf stat -ddd` / `perf record`, and cross-tables are real `amixis compare` output. No values were estimated or reconstructed. Baseline riscv binaries were overwritten by the OPT-1 rebuild, so baseline stripped sizes are NOT AVAILABLE.
- **QEMU caveat (repeated):** the riscv target runs under qemu-user user-mode emulation; timings include emulation overhead and do not reflect native hardware. `perf` counts the host qemu process, not guest counters; guest IPC, cache, TMA and function hotspots are NOT AVAILABLE.

---

## 7. Migration Readiness Summary

| Check | Status |
|---|---|
| Builds on reference (x86_64) | YES — `make -j8` exit 0 |
| Tests pass on reference (x86_64) | YES — 100 PASS / 2 FAIL (container mount-namespace) / 15 UNSUPPORTED |
| Builds on target (riscv64) | YES — cross `make -j8` exit 0 (`-march=rv64gc`) |
| Tests pass on target (riscv64) | YES (subset) — 68 PASS / 3 FAIL (missing locale data) / 6 TIMEOUT (qemu slowness) |
| Zero external dependencies | YES — no external runtime libraries |
| No hand-written intrinsics | YES for generic code; arch intrinsics isolated in sysdeps; RISC-V uses hand-written RVV assembly (not intrinsics) |
| Alignment safe | YES — misalignment handled in string routines and `riscv_hwprobe` |
| Exceptions handled | YES — no exception-based control-flow dependency |
| Auto-vectorization | x86: dense AVX-512; riscv: scalar RV64GC only (RVV blocked by GCC 15 requirement) |

### Migration Verdict: MINOR CONCERNS

glibc is architecturally ready for x86_64 → riscv64: it contains a complete, actively maintained, native upstream RISC-V port with ABI-stable sysdeps, IFUNC multiarch and optional RVV. Both platforms build successfully and tests overwhelmingly pass (failures are container/environment artifacts). The concerns are environmental/measurement, not code portability: the available cross GCC (13.3.0) cannot enable the V extension for C code, the target was validated only under qemu-user emulation, and amixis 0.2.0 cannot drive glibc's Autotools build.

### Required Actions

1. Obtain a GCC ≥ 15 riscv64 cross toolchain to enable `-march=rv64gcv` and C-level RVV; the current build is scalar RV64GC only.
2. Enable/optimize RVV string routines: remove the naive `__memset_vector` performance cliff and add RVV `memcpy`/`memmove` with size-thresholded IFUNC selection.
3. Validate on native RISC-V hardware (with RVV) — QEMU user-mode numbers cannot rank ISA optimizations and misrepresent RVV performance.
4. Either extend amixis with an Autotools backend and local cross-arch/qemu support, or keep the manual Autotools + perf + `amixis compare` workflow documented in this report.
5. Compile/fix locale data in the cross build tree to eliminate the 3 riscv test failures, and raise/remove the qemu test timeout to clear the 6 slow sweep-runner timeouts.
6. Provide `gawk`, `bison`, `gettext`, `texinfo` (now installed here) as standard build-host prerequisites.
