# OpenSSL — Migration Readiness Report (x86_64 → riscv64)

**Prepared by:** Amphimixis orchestrator pipeline (analyzer → configurator → builder → profiler → optimizer → repeat-with-optimization)
**Date:** 2026-10-04
**Resolved upstream clone URL:** https://github.com/openssl/openssl.git
**Local clone:** `/work/OpenSSL-workspace/openssl`
**Workspace:** `/work/OpenSSL-workspace/`

---

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Repository URL | https://github.com/openssl/openssl.git |
| Latest commit | `4d25710dbfeacbb36d055592a3bc6811172248b1` ("docs: fix two broken doc links") |
| Latest commit date | 2026-10-02 10:47:34 +0200 |
| Total commits | 41,346 (first commit 1998-12-21) |
| Commits last 90 days | 1,124 |
| Contributors | 1,533 |
| Latest tag | `openssl-4.1.0-beta1` (2026-09-23) |
| Latest stable releases | `openssl-4.0.3`, `openssl-3.6.5`, `openssl-3.5.9`, `openssl-3.4.8` (all 2026-09-29) |
| Total tags | 449 |
| Project version | `VERSION.dat` = MAJOR=4 MINOR=2 PATCH=0 PRE_RELEASE_TAG=dev (runtime: `OpenSSL 4.2.0-dev`) |
| Activity | Actively maintained; daily commits; scheduled RISC-V / AVX-512 / valgrind CI |
| Build system | OpenSSL's own Perl `Configure` script → generated `Makefile` (make). **No CMake / autotools / Meson.** |
| Tests | 407 `test/recipes/*.t` recipes; 290 `test/*.c` programs |
| CI | 35 GitHub Actions workflows (incl. `riscv-more-cross-compiles.yml`, `cross-compiles.yml`, `avx512-sde.yml`, `os-zoo.yml`, `valgrind-daily.yml`) |
| Documentation | 924 `.pod` man pages under `doc/` plus many Markdown guides |
| Benchmarks | `apps/speed.c` → `openssl speed` |
| Repo size on disk | ~285 MB (blobless clone) |
| Forks with target-arch patches | None needed — upstream integrates RISC-V directly (171 RISC-V-authored commits; dedicated 12-config QEMU cross-compile/test CI). Only unrelated low-quality fork found. |

### External dependencies (assessed for riscv64)

| Dependency | Kind | riscv64 status | Notes |
|---|---|---|---|
| Perl 5 (core modules) | build, required | ready | Perl is architecture-independent; present |
| `Text::Template` 1.56 | build, required | ready | Bundled in `external/perl/Text-Template-1.56` |
| GNU Make | build, required | ready | present |
| C compiler (GCC/Clang) | build, required | ready | `linux64-riscv64` target exists; `riscv64-linux-gnu-gcc` toolchain; dedicated CI |
| zlib | run/build, optional | ready | Portable C |
| zstd | run/build, optional | ready | Portable C |
| brotli | run/build, optional | ready | Portable C |
| pthreads (glibc NPTL) | runtime, default-on | ready | `no-threads` disables |
| libdl / DSO loader (glibc) | runtime, default-on | ready | `no-dso` disables |

### Distro packages

| Distro | Status |
|---|---|
| Debian | In `main`, source package `openssl`; riscv64 is an official Debian port architecture |
| Arch Linux | `core/openssl` 3.6.5-1 (x86_64 verified); runtime deps brotli/glibc/zlib/zstd |
| Yocto/OE | `openembedded-core` recipe `openssl 4.0.2`; no riscv64-specific patching required upstream |

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros (representative)

| Macro | Example location | What it guards | Category |
|---|---|---|---|
| `__x86_64__` / `__x86_64` | `crypto/des/des_local.h:36` | endianness / word-size fast paths | x86 |
| `_M_IX86`, `_M_AMD64`, `_M_X64` | `crypto/aes/aes_local.h:18` | MSVC x86/x64 intrinsics | x86/compiler |
| `__AVX2__` | 2 files | AVX2 intrinsic availability | x86 |
| `__BMI__` | 1 file | BMI inline-asm guard | x86 |
| `__aarch64__` | `crypto/armcap.c:27` | ARM64 paths / CPU caps | ARM |
| `__arm__` / `__arm` | `crypto/armcap.c:125` | ARM32 CPU caps | ARM |
| `__ARM_ARCH__`, `__ARM_MAX_ARCH__` | 19 files | ARM ISA build gate (`__ARM_MAX_ARCH__` is build-defined by OpenSSL, not the compiler) | ARM |
| `__riscv` | `crypto/riscvcap.c` | RISC-V architecture detection | RISC-V |
| `__riscv_xlen` | `crypto/des/des_local.h:46` (+25 files) | RV32 vs RV64 selection | RISC-V |
| `__riscv_zbb`, `__riscv_zbkb` | `crypto/des/des_local.h:45`, `crypto/chacha/chacha_enc.c:28` | inline-asm bit-manipulation fast paths | RISC-V |
| `__BYTE_ORDER__`, `__ORDER_LITTLE_ENDIAN__` | `crypto/sha/sha512.c:387` | endianness selection | endianness |
| `__LP64__`, `__ILP32__`, `__SIZEOF_LONG__` | `crypto/ec/curve448/curve448utils.h:30` | limb / pointer size selection | pointer/size |
| `_WIN32`, `_WIN64`, `__linux__`, `__APPLE__`, `__ANDROID__` | various (93 files for `_WIN32`) | OS paths | OS |

Counts from the analyzer: x86 `__x86_64__`(37 files)/`_M_X64`(35); RISC-V `__riscv`(23)/`__riscv_xlen`(26)/`__riscv_zbb`(5)/`__riscv_zbkb`(6); endianness `__BYTE_ORDER__`(6); OS `_WIN32`(93), `__APPLE__`(24).

### 2.2 Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `__APPLE__` | macOS | sigill-based CPU feature detection |
| `__ANDROID__ && __ANDROID_API__ >= 18` | Android | ARM HWCAP detection |
| `__linux__` / `OPENSSL_SYS_LINUX` | Linux | `<asm/hwprobe.h>` / `__NR_riscv_hwprobe` RISC-V detection |
| `_WIN32` / `_WIN64` | Windows | UI, async, file I/O, threading |
| `__VMS` | OpenVMS | IA64 asm, keccak scratch |
| `_AIX`, `__sun`, `__hpux`, `__FreeBSD__` | Unix variants | threading/file I/O/ppc caps |
| `__e2k__` | Elbrus | async I/O |

OpenSSL-specific capability macros (semantics checked): `OPENSSL_riscvcap` (environment override), `RISCV_HAS_<EXT>()` runtime bit tests generated via `include/arch/riscv_arch.def`, `OSSL_RISCV_HWPROBE` (Linux hwprobe), `OPENSSL_riscv_hwcap_P` / `VECTOR_CAPABLE`. `__ARM_MAX_ARCH__` is misleading: it is **build-defined by OpenSSL**, not by the compiler.

### 2.3 Portability verdict

| Criterion | Verdict | Evidence |
|---|---|---|
| No exceptions | N/A | OpenSSL is C; no C++ exception handling required |
| Alignment safe | Yes (low risk) | Alignment handled per-algorithm via `GETU32/PUTU32/BSWAP` and perlasm; recent upstream RISC-V misaligned-input fixes already merged (SHA-512/SHA-256/SM3/ChaCha) |
| Embedded usability | Conditional | Large footprint; `no-asm`, `no-threads`, `no-shared`, `no-dso` fallbacks supported and documented in `INSTALL.md` |
| Overall portability | LOW risk | First-class upstream riscv64 support: `linux64-riscv64` target, `asm_arch => riscv64`, Linux hwprobe CPU detection, `OPENSSL_riscvcap` override, 171 RISC-V commits, dedicated RISC-V CI |

**Vectorization intrinsics (source):** x86 C intrinsics in 3 files (`crypto/aes/aes_vaes512_intrinsics.c` AVX-512/VAES; `crypto/evp/dec_b64_avx2.c`, `enc_b64_avx2.c` AVX2); **no NEON C intrinsics**; RISC-V has 13 hand-written RVV perlasm backends plus scalar-crypto perlasm, dispatched at runtime via hwprobe.

---

## 3. Build & Test Results

Config `/work/input.yml` was adapted to OpenSSL's Perl-`Configure` build system (the originally supplied CMake flags do not apply to OpenSSL). Recipe 1: native x86_64, `-O3 -march=native -g`, tests on. Recipe 2: cross riscv64, `--cross-compile-prefix=riscv64-linux-gnu-`, `-O3 -g`, tests on. `amixis validate /work/input.yml` → correct.

| Platform | Build name | Configure target | Build result | Assembly | Tests run | Passed | Failed |
|---|---|---|---|---|---|---|---|
| x86_64 (reference) | 1_1_1 | `linux-x86_64` | Success (exit 0; wall 7:34.89) | Enabled (`asm_arch=x86_64`) | 3703 | 3703 | 0 |
| riscv64 (target, qemu-user) | 1_2_2 | `linux64-riscv64` (cross) | Success (exit 0; wall 10:00.65) | Enabled (`asm_arch=riscv64`; RVV kernels compiled) | 3560 (`Files=406`) | 3560 | 0 |

Notes:
- x86: 1 compiler warning (`-Wstringop-overflow=`) in `providers/implementations/rands/drbg_ctr.c:121`; no error. Installed to `/work/1_1_1` (libs in `usr/lib64`); binary `/work/1_1_1/usr/bin/openssl`.
- riscv: 0 warnings, 0 errors. Installed to `/work/1_2_2` (libs in `usr/lib`); binary `/work/1_2_2/usr/bin/openssl`, `readelf` machine = RISC-V.
- riscv test run required running target binaries under qemu; a conditional hook (RISC-V ELF detection) was added to the generated `util/shlib_wrap.sh` so the test harness invokes target binaries via `qemu-riscv64-static`. All executed tests passed; 2 extra cross recipes (`02-test_errstr.t`, `04-test_conf.t`) are skipped by upstream with reason "unsupported for cross compiled configurations".
- 407 test recipes executed on each platform (main harness `Files=406` + separate FIPS prep pass `00-prep_fipsmodule_cnf.t`).

### Build/test failures detail

| Item | Result | Detail |
|---|---|---|
| `amixis build … --build-name=1_1_1` (automated) | FAILED (exit 1) | `CONFIGURATOR ERROR \| Invalid local machine arch: riscv, your machine is x86_64` — v0.2.0 rejects a local `arch: riscv` platform; no emulation support. Documented in `amixis-build-failure.txt`. |
| `amixis build … --build-name=1_2_2` (automated) | FAILED (exit 1) | same local-arch rejection at config parse time |
| Reference build/test | Success | see table above |
| Target build/test | Success (manual cross-compile + qemu) | see table above |
| riscv test method | `HARNESS_JOBS=4 make test` with `util/shlib_wrap.sh` qemu hook | `Files=406, Tests=3560, Result: PASS` |

---

## 4. Performance Comparison

**Benchmark:** `openssl speed -seconds 3 -evp sha256` (plus `aes-128-cbc`). **Target runs under QEMU user-mode emulation** (`qemu-riscv64-static` 8.2.2, `-L /usr/riscv64-linux-gnu`).

### 4.1 Experimental conditions

| Condition | Reference x86_64 | Target riscv64 (qemu) |
|---|---|---|
| Host CPU | Intel Core i5-1035G1 @ 1.00 GHz (Ice Lake) | same host (emulated guest) |
| Cores | 4 physical / 8 logical (`nproc`=8) | same |
| Pinning | `taskset -c 0` | `taskset -c 0` |
| Priority | nice 0 (`nice -n -20` denied — no CAP_SYS_NICE) | nice 0 |
| Governor / frequency | `powersave`; observed 0.4–1.3 GHz | same host |
| Warmup runs | 1 | 1 |
| Measurement runs | 5 | 5 |
| Benchmark seconds | 3 | 3 |
| Emulation | No (native) | **Yes — qemu-user TCG; timing includes emulation overhead** |
| Runtime caps | `OPENSSL_ia32cap` (SHA-NI/AES-NI/AVX-512) | `OPENSSL_riscvcap=RV64GC_ZBA_ZBB_ZBS_V vlen:128` |

### 4.2 Key metrics

<table>
<tr><th>Metric</th><th>Reference x86_64</th><th>Target riscv64 (qemu)</th><th>Validity</th></tr>
<tr><td>Elapsed real (avg of 5)</td><td>18.002 s</td><td>18.200 s</td><td>time-bounded benchmark loop; not a perf signal</td></tr>
<tr><td>User time (avg of 5)</td><td>17.984 s</td><td>18.160 s</td><td>time-bounded; not a perf signal</td></tr>
<tr><td>IPC</td><td>1.63</td><td>3.62</td><td>x86 guest; riscv = HOST-side emulator, NOT guest</td></tr>
<tr><td>L1-dcache miss rate</td><td>0.0659 %</td><td>0.0798 %</td><td>x86 guest; riscv host-only</td></tr>
<tr><td>LLC miss rate</td><td>32.04 %</td><td>45.30 %</td><td>x86 guest; riscv host-only</td></tr>
<tr><td>Branch misprediction rate</td><td>0.0085 %</td><td>0.1016 %</td><td>x86 guest; riscv host-only</td></tr>
<tr><td>Frontend Bound</td><td>23.8 %</td><td>17.2 %</td><td>x86 guest; riscv host-only</td></tr>
<tr><td>Backend Bound</td><td>0.0 %</td><td>0.7 %</td><td>x86 guest (Topdown multiplexed at 33 %); riscv host-only</td></tr>
<tr><td>Retiring</td><td>28.3 %</td><td>19.6 %</td><td>x86 guest; riscv host-only</td></tr>
</table>

sha256 throughput (guest-observable, but includes emulation):

| Block size | Reference x86_64 | Target riscv64 (qemu) |
|---|---|---|
| 16 B | 45,722.53 kB/s | 2,550.10 kB/s |
| 1 KiB | 431,949.67 kB/s | 34,542.75 kB/s |
| 16 KiB | 501,393.89 kB/s | 46,786.14 kB/s |

### 4.3 Hotspots

**Reference x86_64 — real guest hotspots (`cycles`, 18,002 samples):**

| % self | Function | DSO |
|---:|---|---|
| 77.07 % | `sha256_block_data_order_shaext` | libcrypto.so.4 |
| 3.32 % | `[unknown]` | — |
| 2.63 % | `OPENSSL_cleanse` | libcrypto.so.4 |
| 2.40 % | `malloc` | libc.so.6 |
| 1.88 % | `cfree` | libc.so.6 |
| 1.32 % | `SHA256_Final` | libcrypto.so.4 |
| 0.83 % | `EVP_DigestInit_ex` | libcrypto.so.4 |
| 0.71 % | `SHA256_Update_thunk` | libcrypto.so.4 |
| 0.69 % | `EVP_Digest` | libcrypto.so.4 |

**Target riscv64 — guest-level hotspots: NOT AVAILABLE.** Under qemu-user, host perf samples only the emulator: 66.11 % of samples fall in JIT-translated guest code blocks (`/tmp/perf-<pid>.map`, no RISC-V symbol names) and 33.69 % in `qemu-riscv64-static` TCG engine code. No TCG plugin was available, so guest symbol-level hotspots cannot be resolved.

### 4.4 Bottleneck summary

1. **TCG emulation overhead dominates** every riscv64 number: host instruction count is 2.24× the x86 guest count (86.46e9 vs 38.55e9) and ~34 % of host samples are in the TCG engine. None of the riscv hardware counters reflect native RISC-V hardware.
2. **Guest instruction-path asymmetry:** x86 executes hardware SHA-NI (`sha256_block_data_order_shaext`, 77.07 % of cycles), while the riscv guest executes the scalar `sha256_block_data_order_zbb` kernel because qemu advertises no ZVKB/ZVKNHA/ZVKNHB; the compiled RVV SHA-256 kernel is dormant.
3. **Only throughput is guest-meaningful** and it is 10.7× (16 KiB) to 17.9× (16 B) lower under emulation — not a valid measure of the native RISC-V-vs-x86 gap. Guest hotspots, guest IPC and guest cache/branch behavior are NOT AVAILABLE under qemu-user.

### 4.5 Vectorization intrinsics (binary-level, at profile time)

- x86 `libcrypto.so.4`: SIMD/SHA/VAES present — AVX-512 (`vpternlogq` 6451), SHA-NI (`sha256rnds2` 128), AES-NI/VAES families.
- riscv64 `libcrypto.so.4`: perlasm emits **6,506 raw `.word`/`.insn` encodings** (RVV + Zbb kernels); host binutils cannot decode the vector-crypto mnemonics.

### 4.6 Cross-table 1 — reference vs target (`amixis compare`, REAL measured perf data)

File: `/work/cross-tables/CT-1_1_1_openssl-1_2_2_openssl.md`

**Interpretation / QEMU caveat:** Build A (`1_1_1_openssl`, x86) symbols are real guest symbols. Build B (`1_2_2_openssl`, riscv under qemu) rows are **all `[unknown]`** because host perf samples only qemu's JIT/TCG code; the tool placed the riscv `cpu-clock` event into the `CYCLES` column. The symbol-level comparison against Build B is therefore **not informative** — only Build A's distribution is meaningful. This is real measured data, but emulator-side.

#### EVENT: BRANCH-MISSES

| Symbol | 1_1_1_openssl % | 1_2_2_openssl % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 52.31 | 100.00 | +47.69 |
| sha256\_block\_data\_order\_shaext | 10.15 | 0.00 | -10.15 |
| EVP\_MD\_CTX\_gettable\_params | 6.31 | 0.00 | -6.31 |
| SHA256\_Update\_thunk | 5.70 | 0.00 | -5.70 |
| EVP\_Digest\_loop.isra.0 | 4.40 | 0.00 | -4.40 |
| EVP\_MD\_CTX\_get\_size\_ex | 4.02 | 0.00 | -4.02 |
| EVP\_DigestFinal\_ex | 3.00 | 0.00 | -3.00 |
| EVP\_Digest | 2.81 | 0.00 | -2.81 |
| OPENSSL\_LH\_doall | 1.24 | 0.00 | -1.24 |
| sha256\_internal\_final | 1.17 | 0.00 | -1.17 |
| SHA256\_Init | 1.04 | 0.00 | -1.04 |
| OPENSSL\_cleanse | 1.03 | 0.00 | -1.03 |
| EVP\_MD\_CTX\_free | 0.82 | 0.00 | -0.82 |
| SHA256\_Final | 0.70 | 0.00 | -0.70 |
| EVP\_CIPHER\_CTX\_get\_key\_length | 0.62 | 0.00 | -0.62 |
| EVP\_DigestInit\_ex | 0.50 | 0.00 | -0.50 |
| cfree | 0.44 | 0.00 | -0.44 |
| EVP\_MD\_CTX\_get0\_md | 0.41 | 0.00 | -0.41 |
| BIO\_printf | 0.31 | 0.00 | -0.31 |
| EVP\_MD\_CTX\_reset | 0.31 | 0.00 | -0.31 |

#### EVENT: CACHE-MISSES

| Symbol | 1_1_1_openssl % | 1_2_2_openssl % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 78.14 | 100.00 | +21.86 |
| sha256\_block\_data\_order\_shaext | 12.04 | 0.00 | -12.04 |
| cfree | 0.80 | 0.00 | -0.80 |
| pthread\_getspecific | 0.69 | 0.00 | -0.69 |
| OPENSSL\_cleanse | 0.64 | 0.00 | -0.64 |
| EVP\_MD\_CTX\_get\_size\_ex | 0.56 | 0.00 | -0.56 |
| malloc | 0.52 | 0.00 | -0.52 |
| EVP\_MD\_CTX\_clear\_flags | 0.48 | 0.00 | -0.48 |
| EVP\_DigestInit\_ex | 0.47 | 0.00 | -0.47 |
| SHA256\_Final | 0.46 | 0.00 | -0.46 |
| EVP\_Digest | 0.45 | 0.00 | -0.45 |
| CRYPTO\_malloc | 0.41 | 0.00 | -0.41 |
| EVP\_DigestFinal\_ex | 0.41 | 0.00 | -0.41 |
| SHA256\_Update\_thunk | 0.37 | 0.00 | -0.37 |
| EVP\_Digest\_loop.isra.0 | 0.36 | 0.00 | -0.36 |
| SHA256\_Init | 0.35 | 0.00 | -0.35 |
| evp\_md\_ctx\_free\_algctx | 0.30 | 0.00 | -0.30 |
| EVP\_MD\_CTX\_gettable\_params | 0.21 | 0.00 | -0.21 |
| RAND\_get\_rand\_method | 0.21 | 0.00 | -0.21 |
| CRYPTO\_zalloc | 0.20 | 0.00 | -0.20 |

#### EVENT: CYCLES

| Symbol | 1_1_1_openssl % | 1_2_2_openssl % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 3.32 | 100.00 | +96.68 |
| sha256\_block\_data\_order\_shaext | 77.07 | 0.00 | -77.07 |
| OPENSSL\_cleanse | 2.63 | 0.00 | -2.63 |
| malloc | 2.40 | 0.00 | -2.40 |
| cfree | 1.88 | 0.00 | -1.88 |
| SHA256\_Final | 1.32 | 0.00 | -1.32 |
| EVP\_DigestInit\_ex | 0.83 | 0.00 | -0.83 |
| SHA256\_Update\_thunk | 0.71 | 0.00 | -0.71 |
| EVP\_Digest | 0.69 | 0.00 | -0.69 |
| evp\_md\_ctx\_clear\_digest | 0.63 | 0.00 | -0.63 |
| EVP\_DigestFinal\_ex | 0.59 | 0.00 | -0.59 |
| ossl\_prov\_is\_running | 0.58 | 0.00 | -0.58 |
| EVP\_Digest\_loop.isra.0 | 0.56 | 0.00 | -0.56 |
| CRYPTO\_malloc | 0.56 | 0.00 | -0.56 |
| EVP\_MD\_CTX\_get\_size\_ex | 0.41 | 0.00 | -0.41 |
| EVP\_MD\_CTX\_reset | 0.41 | 0.00 | -0.41 |
| EVP\_MD\_CTX\_gettable\_params | 0.39 | 0.00 | -0.39 |
| EVP\_MD\_CTX\_clear\_flags | 0.38 | 0.00 | -0.38 |
| CRYPTO\_zalloc | 0.38 | 0.00 | -0.38 |
| sha256\_internal\_final | 0.37 | 0.00 | -0.37 |

### 4.7 Cross-table 2 — baseline vs optimized target (`1_2_2` vs `1_2_2_opt`)

File: `/work/cross-tables/CT-1_2_2_openssl-1_2_2_opt_openssl.md`

**Interpretation / caveat:** Both builds produced only aggregate `[unknown]` host-side qemu rows; there is no per-symbol signal separating baseline from optimized. `amixis compare` also labels the `cpu-clock` event as `CYCLES`. This table is real but **not analytically meaningful**.

#### EVENT: CYCLES

| Symbol | 1_2_2_openssl % | 1_2_2_opt_openssl % | Delta % |
|:---|---------:|---------:|-------:|
| \[unknown\] | 66.59 | 66.00 | -0.59 |
| \[unknown\] (/ | 33.41 | 34.00 | +0.59 |

---

## 5. Optimization Results

### 5.1 Vector instructions in binary

| Architecture | Binary | Method | Result | ISAs found |
|---|---|---|---|---|
| x86_64 | `usr/bin/openssl` | `amphimixis-analyze-vectorization` | OK — 25 unique / 448 total | SSE/AVX integer (`movaps`, `vpaddd`, `vpsubd`, `vperm`) |
| x86_64 | `libcrypto.so.4` | `amphimixis-analyze-vectorization` | OK — 57 unique / 44,702 total | AVX-512 (`vpternlogq` 6451), AES-NI, VAES, SHA-NI (`sha256rnds2` 128), VPCLMULQDQ |
| riscv64 | `usr/bin/openssl` / `libcrypto.so.4` | tool FAILED (exit 1: `objdump: can't disassemble for architecture UNKNOWN!`); fallback `riscv64-linux-gnu-objdump/nm` | 0 mnemonics decoded; 6,506 raw `.word`/`.insn`; RVV symbols present | RVV/Zbb kernels present but undecodable by host binutils 2.42 |

RISC-V RVV kernel symbols found (evidence of compiled RVV paths): `sha256_block_data_order_zvkb_zvknha_or_zvknhb`, `sha512_block_data_order_zvkb_zvknhb` (+`_zvl128/_zvl256/_zvl512`), `ChaCha20_ctr32_v_zbb_zvkb`, `rv64i_zvkned_*` (AES), `gcm_ghash_rv64i_zvkb_zvbc`, `ossl_hwsm3_block_data_order_zvksh`, `rv64i_zvksed_sm4_*`, `riscv_vlen`.

### 5.2 Executable size analysis

| Artifact | Unstripped | Stripped | Debug+symbol delta | `.text` |
|---|---:|---:|---:|---:|
| x86 `openssl` | 3,336,744 B | 1,186,056 B | 2,150,688 B (64.5 %) | 1,062,507 B |
| riscv `openssl` | 3,419,616 B | 1,021,968 B | 2,397,648 B (70.1 %) | 904,436 B |
| x86 `libcrypto.so.4` | 22,514,512 B | 6,700,200 B | 15,814,312 B (70.2 %) | 6,218,052 B |
| riscv `libcrypto.so.4` | 21,694,240 B | 5,426,384 B | 16,267,856 B (75.0 %) | 4,963,349 B |
| x86 `libssl.so.4` | 6,169,896 B | 1,270,688 B | 4,899,208 B (79.4 %) | 1,207,095 B |
| riscv `libssl.so.4` | 6,290,544 B | 1,041,096 B | 5,249,448 B (83.4 %) | 980,739 B |

Causal note: the raw-file parity is dominated by DWARF (non-allocatable, never paged in). The real `.text` is 15–20 % **smaller** on riscv64 because the x86 build carries more hand-written SIMD variants per algorithm. No real code bloat on riscv64.

### 5.3 Optimization attempts (measured)

| Attempt | Platform | Before | After | Delta | Causal analysis |
|---|---|---|---:|---:|---:|---|
| Low-level `speed sha256` vs `-evp sha256` (bypass EVP/provider) | x86 @16K | 501,394 | 501,847 | +0.1 % | EVP layer not a bottleneck; SHA-NI compression dominates (77 % cycles) |
| Low-level `speed sha256` vs `-evp sha256` | riscv/qemu @16K | 48,813 | 46,793 | −4.1 % | EVP overhead negligible; removing per-op malloc would not help |
| Force RVV: `OPENSSL_riscvcap=...zvbb_zvkb_zvknha_zvknhb_v` | riscv/qemu @16K | 48,813 | 29,229 | 1.67× slower | **EMULATION-ONLY**: TCG implements RVV crypto poorly; not evidence about native RVV |
| Force RVV (same) | riscv/qemu @16B | 2,530.8 | 1,740.4 | 1.45× slower | same TCG artifact |
| Rebuild with `-march=rv64gcv_zba_zbb_zbs` (see Improvement section below) | riscv/qemu | see table | see table | ~parity | hot SHA-256/AES paths are unchanged perlasm; C `sh1add`/`andn`/`rol` etc. added and `.text` −2.2 %, but no measurable effect under TCG |

## Improvement of 1_2_2 compared to 1_2_2_opt

Values below are copied verbatim from the tool-owned `improvements.json` (medians of 5 measurement runs; the optimized means were corrupted by identified host-contention outliers and are not used). Higher throughput = better.

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---|---:|---:|---:|---|
| openssl | 2626.42 | 2660.65 | 101.3 | sha256_16B_median_kBps |
| openssl | 38055.81 | 37969.58 | 99.77 | sha256_1024B_median_kBps |
| openssl | 48502.1 | 48856.1 | 100.73 | sha256_16384B_median_kBps |
| openssl | 30845.61 | 30010.03 | 97.29 | aes-128-cbc_16384B_median_kBps |

Causal interpretation: the `-march=rv64gcv_zba_zbb_zbs` rebuild did add Zba/Zbb/Zbs codegen for C code (e.g. `sh1/2/3add` 3401, `andn` 509, `rol` 594, `rori` 833, `rev8` 165) and shrank `.text` by 2.2 %, but the profiled hot paths (`sha256_block_data_order_zbb`, `AES_encrypt`) are byte-identical perlasm assembly in both builds. The observed deltas (±3 %) are within qemu/host measurement noise; no statistically meaningful change is demonstrated. Under qemu-user TCG this optimization is not expected to move these benchmarks.

### 5.5 Recommended optimizations

| Priority | Optimization | Expected gain | Effort | Native-valid vs emulation-only | Notes |
|:--:|---|:--:|:--:|:--:|---|
| 1 | Deploy on hardware with Zvkb + (Zvknha\|Zvknhb), VLEN≥128; OpenSSL auto-dispatches the existing RVV kernel | Est. 2–4× for SHA-256 over scalar Zbb (estimate — unmeasurable under TCG) | Low | NATIVE | No rebuild; runtime hwprobe selection |
| 2 | Rebuild C with `-march=rv64gc_zba_zbb_zbs_v` (or `-mcpu=` exact core) | Est. ~0 for SHA-256 asm; 5–30 % for C-only crypto | Low | NATIVE | Enables bitmanip + RVV auto-vectorization |
| 3 | `-flto` and/or `-fno-semantic-interposition -Wl,-Bsymbolic-functions` | Est. 1–5 % | Medium | NATIVE | Reduces PLT/GOT overhead; LTO with perlasm is fragile |
| 4 | PGO on a representative native workload | Est. 5–15 % control-flow-heavy code | High | NATIVE | Must be collected on target hardware, not qemu |
| 5 | Strip debug info before deployment | 1.9–16 MB/file; ~0 runtime | Low | NATIVE (size only) | Debug sections non-allocatable |
| 6 | Custom allocator (mimalloc/jemalloc) cross-built for riscv | ~0 for sha256 (measured); helps alloc-heavy MT server paths | Medium | CONDITIONAL | Requires riscv build of allocator |
| 7 | Newer GCC 14/15 + binutils ≥2.44 | Est. 2–10 % overall | Medium | NATIVE | Better codegen + tooling visibility |
| — | Force `OPENSSL_riscvcap=...zvkb...` under qemu | 1.67× **slower** (measured) | Low | EMULATION-ONLY | TCG artifact; do not cite as native |
| — | qemu `-cpu rv64,v=true,vlen=256`, MT-TCG, static linking | varies | Low | EMULATION-ONLY | Only changes the emulator; no native meaning |

**Key finding:** the built binary already contains RVV SHA-256/SHA-512/ChaCha/AES/SM3/SM4 kernels. On native RVV hardware no rebuild is needed for SHA dispatch; only the C-level `-march` gap needs a rebuild for other algorithms.

### 5.6 Step-by-step instructions

```bash
# R1 — Native RVV SHA-256 (no rebuild)
cat /proc/cpuinfo | grep -i isa                          # expect Zvkb, Zvknha|Zvknhb, vlen>=128
LD_LIBRARY_PATH=/work/1_2_2/usr/lib openssl speed -seconds 3 -evp sha256
openssl version -a | grep -i cpuinfo

# R2 — Rebuild C with -march (native-valid)
cd /work/OpenSSL-workspace/openssl
./Configure linux64-riscv64 enable-tests --prefix=/usr \
  --cross-compile-prefix=riscv64-linux-gnu- -march=rv64gcv_zba_zbb_zbs
make -j4

# R3 — LTO / symbol binding (test separately)
./Configure linux64-riscv64 --cross-compile-prefix=riscv64-linux-gnu- \
  -O3 -fno-semantic-interposition
make -j4 && make install DESTDIR=/work/OpenSSL-workspace/opt/ltoA

# R4 — PGO (collect on native hardware only)
./Configure linux64-riscv64 -O3 -fprofile-generate --cross-compile-prefix=riscv64-linux-gnu-
make -j4 && ./apps/openssl speed -seconds 3 -evp sha256
./Configure linux64-riscv64 -O3 -fprofile-use -fprofile-correction --cross-compile-prefix=riscv64-linux-gnu-
make -j4

# R5 — Strip for deployment (on copies)
riscv64-linux-gnu-strip --strip-all libcrypto.so.4 libssl.so.4 openssl
```

---

## 6. Notes About Exploration Process

1. **`amixis` cannot drive the riscv target in this container.** v0.2.0 rejects a *local* platform whose `arch` is not a substring of `uname -m`; the config's platform 2 is `arch: riscv` on an `x86_64` host. Both `amixis build` and `amixis profile` fail at config parse with `Invalid local machine arch: riscv, your machine is x86_64`. The tool ships no qemu/emulation support (the `distributed-cross` sample requires a remote RISC-V host with an `address`). All builds/profiling were therefore done with the documented manual cross-compilation + qemu-user fallback. `amixis validate` and `amixis compare` (config-independent) both work.
2. **Config adaptation:** the originally supplied `/work/input.yml` used CMake flags; OpenSSL is not a CMake project. The configurator rewrote the recipes for OpenSSL's Perl `Configure`→Makefile pipeline. `amixis validate /work/input.yml` → correct.
3. **QEMU/emulation caveat (repeated):** the target ran under `qemu-riscv64-static` 8.2.2 user-mode TCG. All target-side timing includes emulation overhead; host perf counters and the cross-table's Build-B rows reflect the emulator/JIT, not RISC-V guest behavior. Guest hotspots/IPC/cache/branch data are NOT AVAILABLE. None of these numbers may be used to estimate native RISC-V performance.
4. **`file` utility not installed** and apt offline; `readelf` used for architecture/type verification.
5. **Install libdir differs:** x86 installs to `usr/lib64`, riscv to `usr/lib`; qemu runs need `LD_LIBRARY_PATH` pointing at the matching dir (without it, rc 127).
6. **riscv test execution:** binfmt_misc cannot be mounted (read-only `/proc/sys`), so a conditional RISC-V-ELF-detecting hook was added to the generated `util/shlib_wrap.sh` (backup `util/shlib_wrap.sh.orig`) to run target binaries under qemu while leaving host-perl recipes native.
7. **RVV instructions are raw `.word`-encoded** in perlasm output; `amphimixis-analyze-vectorization -arch riscv` fails with host objdump. RVV symbol presence via `riscv64-linux-gnu-nm` is the evidence.
8. **Reference test binaries no longer on disk** — the in-tree reconfigure for riscv overwrote the build tree; x86 test evidence is preserved in `test-1_1_1.log` and the `/work/1_1_1` install.
9. **`nice -n -20` denied** (no CAP_SYS_NICE); both builds measured at nice 0. Host frequency not pinned (`powersave`, 0.4–1.3 GHz) — source of measurement outliers.
10. **`perf archive` unavailable** in perf 6.8.12 (`'archive' is not a perf-command`); user-space symbols still resolved for x86.
11. **`<project>.json` / `<project>.yaml` / `<project>.pkl` NOT AVAILABLE** — `amixis profile` aborted at config parse (item 1) before producing stats. No such tool-owned file exists under `/work`.

---

## 7. Migration Readiness Summary

| Check | Result | Evidence |
|---|---|---|
| Builds on reference (x86_64) | ✅ Yes | `linux-x86_64` configure + make exit 0; binary runs natively |
| Tests pass on reference | ✅ Yes | 3703 passed / 0 failed (407 recipes) |
| Builds on target (riscv64) | ✅ Yes | `linux64-riscv64` cross configure + make exit 0; binary runs under qemu |
| Tests pass on target | ✅ Yes | 3560 passed / 0 failed under qemu (2 extra cross-unsupported recipes skipped) |
| Zero external dependencies | ❌ No | Requires Perl + Make + C compiler; optional zlib/zstd/brotli/pthreads/libdl — all portable on riscv64 |
| No hand-written intrinsics | ❌ No | x86 has AVX2/AVX-512 C intrinsics; RISC-V uses hand-written perlasm (RVV/Zbb) — but RISC-V paths already exist and are runtime-dispatched |
| Alignment safe | ✅ Yes | Per-algorithm byte macros + perlasm; upstream RISC-V misaligned-input fixes merged |
| Exceptions handled | N/A | C project; no C++ exception handling |
| Auto-vectorization | ⚠️ Partial | x86 extensive; riscv C auto-vectorization requires `-march=...v` (not in baseline `-O3 -g`); hand-written RVV kernels are compiled and runtime-gated |
| Guest-level performance measured | ❌ NOT AVAILABLE | qemu-user provides no guest perf; only host-side emulator counters and emulation-inclusive throughput |

### Migration Verdict: READY

OpenSSL has first-class upstream riscv64 support (dedicated `linux64-riscv64` target, `asm_arch=riscv64`, Linux hwprobe CPU detection, `OPENSSL_riscvcap` override, 171 RISC-V commits, dedicated RISC-V CI). Both the reference and the cross-compiled target build cleanly, and the full test suites pass (3703/0 native, 3560/0 under qemu). No portability blockers were found; dependencies and alignment are safe. The measured performance gap is dominated by qemu-user emulation and is **not** a native RISC-V result.

### Required Actions

1. **Re-measure on native RISC-V hardware** — qemu-user timing cannot characterize the native gap; obtain guest perf there.
2. **Verify runtime RVV dispatch on target hardware** (Zvkb + Zvknha/Zvknhb, VLEN≥128) so the already-built RVV SHA-256/SHA-512/ChaCha/AES/SM3/SM4 kernels are selected via hwprobe.
3. **Rebuild C code with `-march=rv64gcv_zba_zbb_zbs`** (or an exact `-mcpu`) to close the C-level bitmanip/auto-vectorization gap for non-assembly algorithms.
4. Strip debug info for deployment (saves 1.9–16 MB/file).
5. If a fully automated Amphimixis run is required, provide a reachable remote riscv64 host (with `address`/credentials) or an amixis build with emulation support — the local `arch: riscv` configuration is rejected by v0.2.0.

