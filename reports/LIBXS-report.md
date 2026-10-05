# LIBXS (Crossroads I/O) — Migration Readiness Report

**Project:** LIBXS (Crossroads I/O) · **Resolved repository:** https://github.com/PiotrSikora/libxs.git
**Reference platform:** x86_64 · **Target platform:** riscv64 (cross-compiled, executed under qemu-user emulation)
**Workspace:** `/work/LIBXS-workspace/` · **Date:** 2026-10-05

---

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Resolved project | Crossroads I/O ("libxs") messaging library |
| Repository URL | https://github.com/PiotrSikora/libxs.git |
| Clone path | /work/LIBXS-workspace/libxs |
| Analyzed revision | tag `v1.2.0` (matches the Debian source package version) |
| Latest commit | `1f6d8da319eb536782c0f9d0b30bcc085ede646f` (2012-06-13 11:08:53 +0200) |
| Total commits at v1.2.0 | 1662 |
| Latest tag / release | `v1.2.0` (2012-06-13); also v1.1.0, v1.0.1, v1.0.0 |
| Activity level | Dormant / abandoned since ~2013 (default branch HEAD `fc9dc49`, 2012-03-14) |
| License | LGPL-3.0+ with linking exception |
| Build systems | GNU Autotools (autoconf/automake/libtool); MSVC solution for Windows |
| Tests | 24 test executables under `tests/` (custom harness) |
| Benchmarks | 6 perf programs: `local_lat`, `remote_lat`, `local_thr`, `remote_thr`, `inproc_lat`, `inproc_thr` |
| Documentation | 33 AsciiDoc man pages |
| Size | ~23,374 LOC across `src/` + `include/` |
| CI | None |
| Distro packages | Debian source `libxs` 1.2.0-2 (stretch, buster; removed from testing); Ubuntu 14.04–20.04 (1.2.0); Fedora/EPEL 1.0.0; deepin/PureOS/Trisquel 1.2.0; AUR 1.0.0; MSYS2 1.0.0; Spack recipe; not present in Yocto/OE |
| External dependencies | POSIX pthreads; librt/`clock_gettime` (merged into glibc); optional OpenPGM/libpgm ≥ 5.1 (disabled with `--with-pgm=no`); pkg-config (build-time); asciidoc/xmlto (docs only); Windows-only ws2_32/rpcrt4/iphlpapi; internal bundled libzmq compat library |

**Repository choice rationale / ambiguity.** The source-package name "LIBXS" is ambiguous. The Debian source package `libxs` self-describes as Crossroads I/O ("a library for building scalable and high performance distributed applications") with watch URL `download.crossroads.io/libxs-*.tar.gz`, which unambiguously identifies Crossroads I/O, the ZeroMQ-derived project. The GitHub network root `PiotrSikora/libxs` (21 stars, 27 forks, the most-starred exact match) was chosen as origin; upstream is defunct, so the complete `v1.2.0` release that Debian packages survives in its network; it was fetched from fork `leapmotion/libxs` and checked out at `v1.2.0`. The only actively-developed repo literally named `hfp/libxs` is an unrelated algebra library created in 2026 and was excluded. If the intended target were `hfp/libxs`, all findings would differ; the distro evidence makes Crossroads I/O the correct match.

**Forks with target-architecture patches.** No RISC-V patches were found. The only target-arch fix in the fork network is `leapmotion/libxs` branch `fix-arm-compile` (commit `9a90957`, an `it eq` fix to the ARM `ldrex/strex` inline asm in `src/atomic_ptr.hpp`), which is ARM/Thumb only. Branch inventory contains no `riscv` branch anywhere; the original `crossroads-io` GitHub org is deleted; GitHub code/commit search for `libxs riscv` returned nothing relevant.

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros

| Macro | File : Line | What it guards | Category |
|---|---|---|---|
| `_M_IX86`, `_M_X64` | `src/clock.cpp:112` | `__rdtsc()` on MSVC x86/x64 | x86 |
| `__i386__`, `__x86_64__` | `src/clock.cpp:114` | inline `rdtsc` asm | x86 (GCC) |
| `__SUNPRO_CC` + `__i386`/`__amd64`/`__x86_64` | `src/clock.cpp:118` | Sun Studio x86 `rdtsc` asm | x86 (Solaris) |
| `__i386__`, `__x86_64__` | `src/atomic_counter.hpp:29,89,134` | backend include + `lock; xaddl` add/sub | x86 (GCC) |
| `__i386__`, `__x86_64__` | `src/atomic_ptr.hpp:30,98,147` | backend include + `lock; xchg`/`cmpxchg` | x86 (GCC) |
| `__ARM_ARCH_7A__` | `src/atomic_counter.hpp:30,72,115`; `src/atomic_ptr.hpp:31,73,127` | 32-bit ARMv7-A `ldrex/strex`/`dmb` asm | ARM |
| `__s390__` | `src/clock.cpp:126` | s390 `stck` asm | s390 |
| `__ia64` | `src/address.cpp:260,313,339` | OpenVMS/Itanium address handling (only when `XS_HAVE_OPENVMS`) | Itanium |
| `__INITIAL_POINTER_SIZE` | `src/address.cpp:313,339` | OpenVMS 64-bit pointer mode | OpenVMS |
| `__GNUC__`, `_MSC_VER`, `__SUNPRO_CC`, `__INTEL_COMPILER`, `__cplusplus` | multiple | compiler/feature selection | compiler |

> **Semantic checks:** `__ARM_ARCH_7A__` is defined only for 32-bit ARMv7-A — not on AArch64 and not on RISC-V, so the ARM asm block is correctly skipped on riscv64. `__s390__`, `__ia64` and `__INITIAL_POINTER_SIZE` are likewise inactive. There are **no** `__riscv`, endianness (`__BYTE_ORDER__` etc.) or pointer-size (`__LP64__`, `__SIZEOF_POINTER__`) macros anywhere in the source (verified by grep).

### 2.2 Platform preprocessor guards

| Guard | Platform | Scope |
|---|---|---|
| `XS_HAVE_WINDOWS` | Windows | sockets, threads, signaling, clock |
| `XS_HAVE_OPENVMS` | OpenVMS/Itanium | address handling, ypipe |
| `XS_HAVE_OPENPGM` | PGM transport | optional PGM engine |
| `XS_HAVE_LINUX` / `_ANDROID` / `_SOLARIS` / `_FREEBSD` / `_OSX` / `_NETBSD` / `_OPENBSD` / `_QNXNTO` / `_AIX` / `_HPUX` / `_CYGWIN` / `_MINGW32` | various OS | per-OS syscalls/headers |
| `XS_HAVE_EPOLL` / `_DEVPOLL` / `_KQUEUE` / `_POLL` / `_SELECT` | event loop | selects async poller (`src/polling.hpp`) |
| `XS_HAVE_EVENTFD`, `XS_HAVE_SOCK_CLOEXEC`, `XS_HAVE_IFADDRS`, `XS_HAVE_PLUGINS` | Linux features | signaling / sockets / plugins |
| `XS_ATOMIC_GCC_SYNC`, `XS_ATOMIC_SOLARIS`, `XS_ATOMIC_OVER_MUTEX` | atomics | selects atomic backend (`configure.ac:244-259`) |
| `HAVE_CLOCK_GETTIME`, `HAVE_GETHRTIME`, `HAVE_SYS_TYPES_H`, … | POSIX | time/headers |

**riscv64-relevant semantics:** on Linux riscv64 the build selects `XS_HAVE_LINUX` + `XS_HAVE_EPOLL` (portable epoll backend) and `XS_ATOMIC_GCC_SYNC` (the `__sync_bool_compare_and_swap` link test succeeds with GCC on riscv64), so the portable `__sync_fetch_and_add` / `__sync_bool_compare_and_swap` paths are used and no arch asm is compiled. `clock_t::rdtsc()` has no RISC-V branch and returns 0, but `now_ms()` explicitly falls back to `now_us()` (`clock_gettime(CLOCK_MONOTONIC)`), so there is no functional break.

### 2.3 Vectorization intrinsics

**None.** There are no `immintrin.h`/`xmmintrin.h`/`emmintrin.h`, no `_mm_*`/`_mm256_*`/`_mm512_*`, no `__m128`/`__m256`, no NEON (`arm_neon.h`/`vld1`), and no RVV (`riscv_vector.h`/`__riscv_v`). The only architecture-specific assembly is scalar (`rdtsc`, atomics).

### 2.4 Portability verdict

| Dimension | Result |
|---|---|
| Architecture-specific macros | x86: 5 sites · ARM: 3 headers · s390: 1 · Itanium: 1 · **RISC-V: 0** — foreign-arch asm correctly `#if`-guarded and inert on riscv64 |
| OS/platform guards | POSIX/Linux/epoll path fully supported; Windows/OpenVMS paths irrelevant |
| Vectorization intrinsics | 0 (none) |
| Dependencies ready / partial / unknown / missing | 4 ready / 0 partial / 0 unknown / 0 missing (plus 1 optional dependency confirmed available for riscv64) |
| Forks with target patches | ARM/Thumb fix only; no RISC-V patches |
| No exceptions | Code uses `new (std::nothrow)`-style allocation; grep found no `throw`/`catch`/`<stdexcept>`/`XS_HAVE_EXCEPTIONS`/`-fexceptions` — no C++ exception dependency |
| Alignment safe | No endianness or pointer-size assumptions; only `htonl` for standard network byte order |
| Embedded usability | Small (~23k LOC) and no mandatory third-party dependency besides libc/pthreads |

**Overall portability level: LOW risk — "alignment safe / no exceptions."** Expected to compile and run on riscv64 with no source changes. Build-system caveats: `rdtsc()` returning 0 disables a clock fast path (functional, lower precision); the 2012 Autotools snapshot may need `config.guess`/`config.sub` regeneration; PGM multicast is optional and off by default.

**Dependency handling:** every applicable dependency was verified riscv64-ready; no dependency warranted a separate analysis/build/profile cycle.

---

## 3. Build & Test Results

| Build | Platform | Build result | Compiler / notes | Programs built | Tests run | Passed | Failed |
|---|---|---|---|---|---|---|---|
| `1_1_1` | x86_64 (reference) | SUCCESS (manual Autotools fallback) | `/usr/bin/g++`; `-O3 -march=native -g` (+`-Wno-error`) | 30 (6 perf + 24 tests) | 24 | 24 | 0 |
| `1_2_2` | riscv64 (target, statically linked) | SUCCESS (manual cross-compile fallback) | `/usr/bin/riscv64-linux-gnu-g++`; `-O3 -g -march=default(full-V)` (+`-Wno-error`), `-static -all-static` | 30 (6 perf + 24 tests) | 24 | 24 | 0 (run under qemu) |
| `1_2_5` | riscv64 (optimized, statically linked) | SUCCESS (Phase 6 optimization) | same cross toolchain; `-O3 -g -march=rv64gc` (+`-Wno-error`), `-static -all-static` | 30 (6 perf + 24 tests) | 24 | 24 | 0 (run under qemu) |

**Test execution.** Reference tests were run natively via `make check`: `# TOTAL: 24  # PASS: 24  # FAIL: 0  # ERROR: 0`. Target tests (both baseline and optimized) were run individually under qemu with `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu <absolute-binary>`; all 24 exited 0 for each build.

**Build/test failure details encountered and resolved (all documented in Section 6):**
- `amphimixis-build`/`amixis build` failed immediately at config load: `CONFIGURATOR | ERROR | Invalid local machine arch: riscv, your machine is x86_64`. Both builds were therefore produced with a manual Autotools fallback using the exact recipe flags.
- GNU Autotools was absent from the container and installed via apt (`autoconf automake libtool pkg-config file`).
- `./autogen.sh` fails spuriously (guards on `/usr/bin/libtool`, which modern libtool no longer installs); its payload `mkdir config && autoreconf --install --force -I config` was used instead.
- The 2012 codebase force-adds `-Werror` on Linux; modern GCC's `-Wstringop-truncation` failed on `address.cpp:432` (`strncpy (un->sun_path, name_, sizeof (un->sun_path))`). `-Wno-error` was added to both platforms' CFLAGS/CXXFLAGS (optimization-neutral).
- libtool silently discarded the recipe's plain `LDFLAGS=-static` for programs (produced dynamically-linked RISC-V ELFs); libtool's `-all-static` was injected at make time (`make -j4 LDFLAGS="-static -all-static"`), after cleaning `perf`/`tests` to force relink. Static linkage was then confirmed with `file` (`ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, ... statically linked`).
- With `--disable-shared`, libtool places program binaries in `perf/`/`tests/` rather than `.libs/`; binaries were copied into the `.libs/` paths required by the config.

---

## 4. Performance Comparison

### 4.1 Experimental conditions

| Condition | Reference (`1_1_1`) | Target (`1_2_2` / `1_2_5`) |
|---|---|---|
| CPU / arch | Intel Core i5-1035G1 @ 1.00 GHz (Ice Lake), x86_64, 8 logical CPUs (4 cores × HT) | Same physical x86_64 host; riscv64 guest executed under qemu-user |
| L1d / L2 / L3 | 192 KiB / 2 MiB / 6 MiB | Host caches (guest has none of its own under user-mode emulation) |
| Governor / frequency | `powersave`; `scaling_cur_freq` ≈ 1.3 GHz during runs (max 3.6 GHz); frequency not pinnable (sysfs read-only) | Identical (same host) |
| Core pinning | `taskset -c 0` for every warmup/measurement/record run | Identical (`taskset -c 0`) |
| Priority | `nice -n -20` attempted but DENIED (`setpriority: Permission denied`, no CAP_SYS_NICE) → default nice | Identical (also denied), to preserve matched conditions |
| Warmup runs | 2 per benchmark | 2 per benchmark |
| Measured runs | 7 per benchmark (median reported) | 7 per benchmark (median reported) |
| `perf record` | 1 representative recording per benchmark | 1 representative recording per benchmark |
| Emulator | — | `/usr/bin/qemu-riscv64-static` (→ `/usr/bin/qemu-riscv64`, QEMU 10.2.1), `-L /usr/riscv64-linux-gnu` |
| Benchmark args | `inproc_lat 64 100000`; `inproc_thr 64 1000000` | identical args |
| Linkage | libxs static; libc/libstdc++ dynamic | fully static (glibc + libstdc++ embedded) |

### 4.2 Key metrics

**`inproc_lat` — 64 B / 100,000 roundtrips (median of 7):**

| Metric | Reference x86_64 (`1_1_1`) | Target riscv64 (`1_2_2`) | Target optimized (`1_2_5`) |
|---|---|---|---|
| Elapsed time (s) | 1.548 | 6.696 | 6.128 |
| IPC | 1.20 | 1.80 | 1.50 |
| L1-dcache miss rate (%) | 0.60 | 3.30 | 3.70 |
| LLC miss rate (%) | 29.30 | 8.40 | 7.30 |
| Branch misprediction rate (%) | 1.20 | 0.30 | 0.40 |
| Frontend Bound (%) | 51.3 | 29.6 | 30.1 |
| Backend Bound (%) | 9.5 | 10.1 | 12.1 |
| Retiring (%) | 25.9 | 29.2 | 26.7 |
| Bad Speculation (%) | 13.2 | 30.8 | 31.1 |
| Average latency (µs) | 7.734 | 33.039 | 30.238 |
| Executable size, stripped (B) | 338,520 | 2,036,704 | 2,032,608 |

**`inproc_thr` — 64 B / 1,000,000 messages (median of 7):**

| Metric | Reference x86_64 (`1_1_1`) | Target riscv64 (`1_2_2`) | Target optimized (`1_2_5`) |
|---|---|---|---|
| Elapsed time (s) | 0.324 | 17.015 | 13.145 |
| IPC | 2.60 | 1.70 | 1.60 |
| L1-dcache miss rate (%) | 1.70 | 1.40 | 1.30 |
| LLC miss rate (%) | 18.50 | 12.10 | 20.40 |
| Branch misprediction rate (%) | 0.20 | 0.20 | 0.20 |
| Frontend Bound (%) | 23.0 | 15.6 | 14.9 |
| Backend Bound (%) | 12.3 | 7.8 | 10.6 |
| Retiring (%) | 52.1 | 21.4 | 22.5 |
| Bad Speculation (%) | 12.4 | 54.7 | 51.8 |
| Throughput (msg/s) | 3,111,436 | 59,117 | 76,613 |
| Throughput (Mb/s) | NOT AVAILABLE (not measured for x86) | 30.268 | 39.226 |
| Executable size, stripped (B) | 338,520 | 2,036,704 | 2,032,608 |


### 4.3 Hotspots

**Reference platform x86_64 (`perf report`; kernel symbols unresolved because `kptr_restrict=1`):**

`inproc_lat`:
| % Time | Function | Module | Analysis |
|---|---|---|---|
| 2.98 | `__syscall_cancel_arch_end` | libc | syscall return path |
| 1.34 | `__syscall_cancel` | libc | syscall wrapper |
| 1.22 | `xs::pipe_t::read` | inproc_lat | inproc pipe read |
| 1.02 | `xs::xrep_t::xsend` | inproc_lat | REQ/REP send path |
| 0.88 | `xs::socket_base_t::send` | inproc_lat | base send |
| 0.61 | `xs::pipe_t::write` | inproc_lat | pipe write |
| 0.54 | `xs::socket_base_t::process_commands` | inproc_lat | command processing |
| 0.47 | `xs::mailbox_recv` | inproc_lat | mailbox |
| 0.41 | `pthread_mutex_lock` | libc | lock contention |
| 0.40 | `xs::signaler_wait` | inproc_lat | signaler poll/wait |

`inproc_thr`:
| % Time | Function | Module | Analysis |
|---|---|---|---|
| 10.82 | `__libc_malloc2` | libc | per-message allocation |
| 9.82 | `_int_free_chunk` | libc | per-message free |
| 8.58 | `xs::pipe_t::flush` | inproc_thr | pipe flush on send |
| 4.33 | `_int_malloc` | libc | allocator |
| 3.96 | `xs::pipe_t::write` | inproc_thr | pipe write |
| 3.09 | `xs::clock_t::rdtsc()` | inproc_thr | timestamp read per message |
| 2.78 | `worker(void*)` | inproc_thr | benchmark sender loop |
| 2.78 | `xs::lb_t::send` | inproc_thr | load-balancer send |
| 2.47 | `cfree` | libc | free wrapper |
| 2.16 | `_int_free_merge_chunk` | libc | allocator |
| 1.86 | `xs::msg_t::close()` | inproc_thr | message teardown |

**Target platform riscv64 (baseline and optimized): NOT AVAILABLE.** Under qemu user-mode emulation the guest ELF symbols are unresolvable: 0 `xs::` symbols; all samples land on the host qemu binary's stripped addresses, `[k] 0x…`, and `[JIT]`/anonymous regions. Host-side top frames are qemu translation/dispatch code (e.g. `qemu-riscv64 +0x8fd84`, `[JIT] tid N`), which are not libxs hotspots. This is inherent to user-mode emulation, not a tool failure.

### 4.4 Vectorization intrinsics

No hand-written SIMD intrinsics exist (Section 2.3). Both platforms rely on compiler auto-vectorization: x86 uses SSE/AVX (VEX integer/move instructions), and the default riscv64 toolchain `-march` enables the V extension, producing RVV (see Section 5.1). There is no vectorization migration gap.

### 4.5 Bottleneck summary and causal analysis

1. **QEMU TCG emulation dominates the target cost, not the RISC-V ISA.** Host instruction counts inflate 6.78× (`inproc_lat`) and 35.08× (`inproc_thr`) while wall time inflates 4.32× and 52.47×. The TopDown evidence points to QEMU's translate/dispatch loop: Bad Speculation rises from 13.2%→30.8% (lat) and 12.4%→54.7% (thr), and Retiring falls from 52.1%→21.4% (thr). These are emulation artifacts of the host process.
2. **Latency and throughput diverge because the benchmarks stress different paths.** `inproc_lat` is syscall/poll/mutex/signaler-bound; `inproc_thr` is allocation/pipe-bound (`__libc_malloc2` 10.8%, `_int_free_chunk` 9.8%, `xs::pipe_t::flush` 8.6%, `xs::clock_t::rdtsc` 3.1%). Under QEMU each guest syscall/allocation/flush expands, so throughput degrades far more (52×) than latency (4.3×).
3. **Memory behaviour is distorted by emulation.** The target shows 38.5× more L1-dcache load misses on `inproc_lat` (QEMU softmmu TLBs, guest memory, and JIT buffers on top of the 192 KiB L1d); LLC load traffic rises 11.6×/18.8×. Rates are not uniformly worse because QEMU streams a larger, more sequential working set. This describes the emulator, not native RISC-V caches.
4. **Vectorization is not a migration blocker.** Both builds auto-vectorize; the measured target slowness is caused by the translation layer, not missing SIMD.
5. **Static linking is a deliberate target choice with a size cost.** The fully static riscv stripped binary is 2,036,704 B vs the dynamically-linked x86 338,520 B (6.02×); ~70% of the stripped RISC-V image is statically linked libc+libstdc++. Debug info is non-`SHF_ALLOC` and does not occupy the runtime image.
6. **Absolute numbers were taken at ~1.3 GHz** (`powersave` governor, sysfs read-only). Both platforms ran under the same policy on core 0, so ratios are internally consistent but not peak-frequency figures.

### 4.6 QEMU / emulation caveat

The RISC-V target is **not running on RISC-V hardware**. Timing and every hardware counter reported for the target are **host x86 events measuring the `qemu-riscv64` process**; guest instructions are dynamically translated by QEMU TCG. Target timings therefore include emulation overhead and do **not** reflect native riscv64 hardware performance. Guest function-level hotspots are NOT AVAILABLE because JIT-translated host PCs never map back to guest ELFs.

---

## 5. Optimization Results

### 5.1 Vector instructions in binary

| Architecture | Binary | RVV/vector count | Notes |
|---|---|---:|---|
| x86_64 | `bin-x86/perf/inproc_thr` (and `inproc_lat`) | 12 unique / 290 total | SSE/AVX: `vpinsrq` 152, `vpcmp` 49, `paddq`/`vpaddq`, `paddw`/`vpaddw`, `vperm`, `orps`, `xorps`, `vxorps`, `vpinsrd` |
| riscv64 baseline | `bin-riscv/perf/inproc_thr` | app 367 / total 2756 (narrow pattern); broad 482 / 3506 | RVV 1.0; default full-V `-march` auto-vectorizes libxs + benchmark code |
| riscv64 optimized | `bin-riscv-opt/perf/inproc_thr` | app 0 / total 2330 (narrow pattern); broad 0 / 2879 | `-march=rv64gc` eliminates all app RVV; residual RVV is in statically linked glibc (present in both builds) |

`amphimixis-analyze-vectorization` succeeded for x86 but **failed for RISC-V** (exit 1/2) because it calls the native `objdump` (`can't disassemble for architecture UNKNOWN!`); the documented fallback `/usr/bin/riscv64-linux-gnu-objdump` was used.

### 5.2 Optimization attempts

Two optimizations were applied together in build `1_2_5` (they are therefore measured in combination, not individually isolated).

| Optimization | Before (`1_2_2`) | After (`1_2_5`) | Delta | Causal analysis |
|---|---|---|---|---|
| Inline small messages: raise `max_vsm_size` 29 → 64 in `src/msg.hpp` (with companion opaque-buffer ABI bump `include/xs/xs.h` `xs_msg_t` `_[32]`→`_[72]`, enforced by `xs_msg_size_check`) | `inproc_thr` throughput 59,117 msg/s; elapsed 17.015 s; host instructions 39,408,055,195; L1-dcache load misses 141,158,017; LLC loads 3,144,001 | throughput 76,613 msg/s; elapsed 13.145 s; host instructions 28,497,391,113; L1-dcache load misses 98,918,743; LLC loads 1,639,355 | throughput +29.6%; elapsed −22.7%; host instructions −27.7%; L1d load misses −29.9%; LLC loads −47.9% | The 64 B benchmark payload exceeded the 29 B VSM threshold, forcing a heap `malloc`+`free` per message (`msg_t::init_size`); the native x86 profile showed ~31% of `inproc_thr` cycles in the allocator. Inline storage removes the allocation at its cause. `inproc_lat` reuses one message and had no per-message allocation, hence a smaller gain. |
| Narrow target ISA: compile riscv64 with `-march=rv64gc` instead of the toolchain default full-V `-march` | `inproc_lat` elapsed 6.696 s; latency 33.039 µs; host instructions 15,877,660,842; app RVV 367 | elapsed 6.128 s; latency 30.238 µs; host instructions 12,339,957,787; app RVV 0 | elapsed −8.5%; latency −8.5%; host instructions −22.3%; app RVV 367→0 | GCC 15's default RISC-V `-march` enables the V extension, so `-O3` emitted RVV in libxs/benchmark code; QEMU TCG emulates RVV instruction-by-instruction (a controlled micro-check measured ~5× slowdown for a vectorized vs scalar loop). Removing app RVV removes avoidable translation work; residual cost is QEMU's dispatch loop. `inproc_lat` has no allocation, so its gain is dominated by this change. |

### 5.3 Recommended optimizations

| Priority | Optimization | Expected Gain | Effort | Notes |
|---|---|---|---|---|
| High | Raise `max_vsm_size` 29 → 64 (inline small messages) | Removes the measured ~31% native allocator share on `inproc_thr`; small effect on `inproc_lat` | Low (1-line source) | Targets the #1 hotspot at cause. Changes `msg_t` ABI/layout; the companion `xs_msg_t` opaque-buffer size must be updated. Applied in `1_2_5`. |
| High | Build riscv recipe with `-march=rv64gc` (or `-fno-tree-vectorize`) | 0–15% target, workload-dependent; micro-check shows ~5× RVV-vs-scalar under TCG, but only ~13% of RVV instructions were app code | Low (flag only) | Removes avoidable RVV TCG emulation in app loops; static libc still contains RVV. Applied in `1_2_5`. |
| Medium | `-O2` instead of `-O3` for riscv | 0–10%; smaller hot code → less TCG i-cache pressure | Low | Combine-test separately from `-march`. Not applied. |
| Medium | `-flto` (riscv) | 0–10%; cross-module inlining, fewer guest instructions | Medium | May grow code or break 2012 libtool; needs `-Wno-error`. Not applied. |
| Medium | Strip debug (`-Wl,--strip-debug` / post-link `strip`) | ≈0% runtime (debug is non-alloc); file 6,268,904→2,752,512 (`--strip-debug`) / 2,036,704 (`--strip-all`) | Low | Footprint only; validates "debug bloat ≠ runtime". |
| Low | Implement RISC-V `rdtsc` (e.g. `rdtime`) so `process_commands` throttling works | <1–2% | Medium (source) | `process_commands` measured <1% on x86. |
| Low | `-Wl,--gc-sections` (+ `-ffunction-sections -fdata-sections`) | Small size reduction; no steady-state speed | Low | Prebuilt libc/libstdc++ objects lack function sections, so limited. |
| Low | Dynamic link instead of `-all-static` | Unknown sign; probably neutral-to-negative (adds `ld.so` translation) | High (reconfigure) | Shared riscv64 sysroot libs exist. |
| Low | Alternative allocator (mimalloc/jemalloc/tcmalloc) | Potentially 5–15% native | High (cross-compile dep) | Not available for riscv64 in this container. |
| Low | QEMU runtime `-cpu max` / `-cpu rva23u64` | Unknown; may improve TCG host codegen | Low (run cmd) | No rebuild; trial only. |

---

## Improvement of 1_2_2 compared to 1_2_5

| Measured | Baseline value | Optimized value | Improvement % | Parameter | Baseline build | Optimized build |
|---|---|---|---|---|---|---|
| inproc_thr | 32.956 | 42.903 | 130.18 | throughput_Mb_per_s | 1_2_2 | 1_2_5 |
| inproc_lat | 29.917 | 27.133 | 90.69 | average_latency_us | 1_2_2 | 1_2_5 |
| inproc_lat | 6.695885457 | 6.12778591 | 91.52 | real_time_s | 1_2_2 | 1_2_5 |
| inproc_lat | 6694.55 | 6129.34 | 91.56 | task_clock_ms | 1_2_2 | 1_2_5 |
| inproc_lat | 9014174501 | 8238869730 | 91.4 | cpu_cycles_host | 1_2_2 | 1_2_5 |
| inproc_lat | 15877660842 | 12339957787 | 77.72 | instructions_host | 1_2_2 | 1_2_5 |
| inproc_lat | 1.8 | 1.5 | 83.33 | IPC | 1_2_2 | 1_2_5 |
| inproc_lat | 3.3 | 3.7 | 112.12 | L1_dcache_miss_rate_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 8.4 | 7.3 | 86.9 | LLC_miss_rate_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 0.3 | 0.4 | 133.33 | branch_misprediction_rate_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 29.6 | 30.1 | 101.69 | frontend_bound_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 10.1 | 12.1 | 119.8 | backend_bound_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 29.2 | 26.7 | 91.44 | retiring_pct | 1_2_2 | 1_2_5 |
| inproc_lat | 30.8 | 31.1 | 100.97 | bad_speculation_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 17.014948704 | 13.144610747 | 77.25 | real_time_s | 1_2_2 | 1_2_5 |
| inproc_thr | 17003.48 | 13141.34 | 77.29 | task_clock_ms | 1_2_2 | 1_2_5 |
| inproc_thr | 23093035354 | 17761273281 | 76.91 | cpu_cycles_host | 1_2_2 | 1_2_5 |
| inproc_thr | 39408055195 | 28497391113 | 72.31 | instructions_host | 1_2_2 | 1_2_5 |
| inproc_thr | 1.7 | 1.6 | 94.12 | IPC | 1_2_2 | 1_2_5 |
| inproc_thr | 1.4 | 1.3 | 92.86 | L1_dcache_miss_rate_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 12.1 | 20.4 | 168.6 | LLC_miss_rate_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 0.2 | 0.2 | 100 | branch_misprediction_rate_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 15.6 | 14.9 | 95.51 | frontend_bound_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 7.8 | 10.6 | 135.9 | backend_bound_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 21.4 | 22.5 | 105.14 | retiring_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 54.7 | 51.8 | 94.7 | bad_speculation_pct | 1_2_2 | 1_2_5 |
| inproc_thr | 59117 | 76613 | 129.6 | throughput_msg_s | 1_2_2 | 1_2_5 |
| inproc_thr | 2036704 | 2032608 | 99.8 | executable_size_stripped_bytes | 1_2_2 | 1_2_5 |
| inproc_lat | 2036704 | 2032608 | 99.8 | executable_size_stripped_bytes | 1_2_2 | 1_2_5 |

---

## Cross-Tables

### Cross-table A1 — CT-1_1_1_inproc_lat-1_2_2_inproc_lat — EVENT: BRANCH-MISSES

| Symbol | 1_1_1_inproc_lat % | 1_2_2_inproc_lat % | Delta % |
|:-----------------------------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                                             |      0.00 |     43.85 |  +43.85 |
| \_\_syscall\_cancel (/                                     |      5.09 |      0.00 |   -5.09 |
| \[unknown\]                                                |     51.33 |     56.15 |   +4.82 |
| xs::object\_t::send\_activate\_read(xs::pipe\_t\*)         |      3.52 |      0.00 |   -3.52 |
| xs\_sendmsg                                                |      2.80 |      0.00 |   -2.80 |
| worker(void\*)                                             |      2.64 |      0.00 |   -2.64 |
| \_\_poll (/                                                |      2.64 |      0.00 |   -2.64 |
| xs::mailbox\_recv(xs::mailbox\_t\*, xs::command\_t\*, int) |      2.60 |      0.00 |   -2.60 |
| xs::signaler\_wait(xs::signaler\_t\*, int)                 |      2.56 |      0.00 |   -2.56 |
| main                                                       |      2.46 |      0.00 |   -2.46 |
| xs::socket\_base\_t::recv(xs::msg\_t\*, int)               |      2.38 |      0.00 |   -2.38 |
| xs::signaler\_send(xs::signaler\_t\*)                      |      2.31 |      0.00 |   -2.31 |
| \_\_GI\_\_\_libc\_write (/                                 |      2.09 |      0.00 |   -2.09 |
| xs::msg\_t::size()                                         |      1.97 |      0.00 |   -1.97 |
| xs::socket\_base\_t::send(xs::msg\_t\*, int)               |      1.97 |      0.00 |   -1.97 |
| xs::socket\_base\_t::process\_commands(int, bool)          |      1.82 |      0.00 |   -1.82 |
| xs::xrep\_t::xsend(xs::msg\_t\*, int)                      |      1.70 |      0.00 |   -1.70 |
| xs::lb\_t::send(xs::msg\_t\*, int)                         |      1.50 |      0.00 |   -1.50 |
| xs::rep\_t::xsend(xs::msg\_t\*, int)                       |      1.35 |      0.00 |   -1.35 |
| xs::req\_t::xrecv(xs::msg\_t\*, int)                       |      0.96 |      0.00 |   -0.96 |

### Cross-table A2 — CT-1_1_1_inproc_lat-1_2_2_inproc_lat — EVENT: CACHE-MISSES

| Symbol | 1_1_1_inproc_lat % | 1_2_2_inproc_lat % | Delta % |
|:---------------------------------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                                                 |      0.00 |     19.56 |  +19.56 |
| \[unknown\]                                                    |     97.78 |     80.44 |  -17.33 |
| \_\_syscall\_cancel\_arch\_end (/                              |      0.41 |      0.00 |   -0.41 |
| xs::clock\_t::rdtsc()                                          |      0.30 |      0.00 |   -0.30 |
| xs::ctx\_t::send\_command(unsigned int, xs::command\_t const&) |      0.24 |      0.00 |   -0.24 |
| xs::mailbox\_recv(xs::mailbox\_t\*, xs::command\_t\*, int)     |      0.22 |      0.00 |   -0.22 |
| \_\_GI\_\_\_libc\_write (/                                     |      0.21 |      0.00 |   -0.21 |
| xs::pipe\_t::read(xs::msg\_t\*)                                |      0.18 |      0.00 |   -0.18 |
| xs::rep\_t::xrecv(xs::msg\_t\*, int)                           |      0.15 |      0.00 |   -0.15 |
| \_\_poll (/                                                    |      0.12 |      0.00 |   -0.12 |
| xs::socket\_base\_t::process\_commands(int, bool)              |      0.12 |      0.00 |   -0.12 |
| worker(void\*)                                                 |      0.10 |      0.00 |   -0.10 |
| xs\_sendmsg                                                    |      0.07 |      0.00 |   -0.07 |
| xs::signaler\_send(xs::signaler\_t\*)                          |      0.06 |      0.00 |   -0.06 |
| \_\_syscall\_cancel (/                                         |      0.03 |      0.00 |   -0.03 |

### Cross-table A3 — CT-1_1_1_inproc_lat-1_2_2_inproc_lat — EVENT: CYCLES

| Symbol | 1_1_1_inproc_lat % | 1_2_2_inproc_lat % | Delta % |
|:-----------------------------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                                             |      0.00 |     42.10 |  +42.10 |
| \[unknown\]                                                |     83.82 |     57.90 |  -25.92 |
| \_\_syscall\_cancel\_arch\_end (/                          |      2.98 |      0.00 |   -2.98 |
| \_\_syscall\_cancel (/                                     |      1.34 |      0.00 |   -1.34 |
| xs::pipe\_t::read(xs::msg\_t\*)                            |      1.22 |      0.00 |   -1.22 |
| xs::xrep\_t::xsend(xs::msg\_t\*, int)                      |      1.02 |      0.00 |   -1.02 |
| xs::socket\_base\_t::send(xs::msg\_t\*, int)               |      0.88 |      0.00 |   -0.88 |
| xs::pipe\_t::write(xs::msg\_t\*)                           |      0.61 |      0.00 |   -0.61 |
| xs::socket\_base\_t::process\_commands(int, bool)          |      0.54 |      0.00 |   -0.54 |
| xs::mailbox\_recv(xs::mailbox\_t\*, xs::command\_t\*, int) |      0.47 |      0.00 |   -0.47 |
| pthread\_mutex\_lock@@GLIBC\_2.2.5 (/                      |      0.41 |      0.00 |   -0.41 |
| xs::signaler\_wait(xs::signaler\_t\*, int)                 |      0.40 |      0.00 |   -0.40 |
| main                                                       |      0.34 |      0.00 |   -0.34 |
| \_\_poll (/                                                |      0.34 |      0.00 |   -0.34 |
| \_\_GI\_\_\_libc\_write (/                                 |      0.34 |      0.00 |   -0.34 |
| std::\_Rb\_tree<std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> >, std::pair<std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> > const, xs::xrep\_t::outpipe\_t>, std::\_Select1st<std::pair<std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> > const, xs::xrep\_t::outpipe\_t> >, std::less<std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> > >, std::allocator<std::pair<std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> > const, xs::xrep\_t::outpipe\_t> > >::find(std::\_\_cxx11::basic\_string<unsigned char, std::char\_traits<unsigned char>, std::allocator<unsigned char> > const&) \[clone .isra.0\] |      0.34 |      0.00 |   -0.34 |
| xs::object\_t::send\_activate\_read(xs::pipe\_t\*)         |      0.34 |      0.00 |   -0.34 |
| xs::socket\_base\_t::recv(xs::msg\_t\*, int)               |      0.27 |      0.00 |   -0.27 |
| xs::pipe\_t::flush()                                       |      0.27 |      0.00 |   -0.27 |
| xs::lb\_t::send(xs::msg\_t\*, int)                         |      0.27 |      0.00 |   -0.27 |

### Cross-table B1 — CT-1_1_1_inproc_thr-1_2_2_inproc_thr — EVENT: BRANCH-MISSES

| Symbol | 1_1_1_inproc_thr % | 1_2_2_inproc_thr % | Delta % |
|:-----------------------------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                                             |      0.00 |     44.91 |  +44.91 |
| \[unknown\]                                                |     48.13 |     55.09 |   +6.96 |
| \_\_syscall\_cancel (/                                     |      6.95 |      0.00 |   -6.95 |
| \_dl\_lookup\_symbol\_x (/                                 |      3.67 |      0.00 |   -3.67 |
| xs::signaler\_wait(xs::signaler\_t\*, int)                 |      3.56 |      0.00 |   -3.56 |
| \_int\_malloc (/                                           |      2.80 |      0.00 |   -2.80 |
| \_\_GI\_\_\_libc\_write (/                                 |      2.37 |      0.00 |   -2.37 |
| xs\_sendmsg                                                |      2.22 |      0.00 |   -2.22 |
| xs::pipe\_t::write(xs::msg\_t\*)                           |      2.17 |      0.00 |   -2.17 |
| \_int\_free\_chunk (/                                      |      2.15 |      0.00 |   -2.15 |
| \_\_poll (/                                                |      2.09 |      0.00 |   -2.09 |
| xs::mailbox\_recv(xs::mailbox\_t\*, xs::command\_t\*, int) |      1.97 |      0.00 |   -1.97 |
| worker(void\*)                                             |      1.83 |      0.00 |   -1.83 |
| xs::object\_t::send\_activate\_read(xs::pipe\_t\*)         |      1.79 |      0.00 |   -1.79 |
| main                                                       |      1.53 |      0.00 |   -1.53 |
| xs::socket\_base\_t::process\_commands(int, bool)          |      1.48 |      0.00 |   -1.48 |
| \_\_errno\_location (/                                     |      1.40 |      0.00 |   -1.40 |
| xs::lb\_t::send(xs::msg\_t\*, int)                         |      1.24 |      0.00 |   -1.24 |
| xs\_recvmsg                                                |      1.12 |      0.00 |   -1.12 |
| cfree@GLIBC\_2.2.5 (/                                      |      1.02 |      0.00 |   -1.02 |

### Cross-table B2 — CT-1_1_1_inproc_thr-1_2_2_inproc_thr — EVENT: CACHE-MISSES

| Symbol | 1_1_1_inproc_thr % | 1_2_2_inproc_thr % | Delta % |
|:---------------------------------------------|---------:|---------:|-------:|
| \[unknown\] (/                               |      0.00 |     19.07 |  +19.07 |
| \_int\_free\_chunk (/                        |      3.15 |      0.00 |   -3.15 |
| \_int\_free\_merge\_chunk (/                 |      2.20 |      0.00 |   -2.20 |
| xs::socket\_base\_t::recv(xs::msg\_t\*, int) |      2.15 |      0.00 |   -2.15 |
| xs::msg\_t::flags()                          |      1.90 |      0.00 |   -1.90 |
| xs::msg\_t::close()                          |      1.57 |      0.00 |   -1.57 |
| \_\_syscall\_cancel\_arch\_end (/            |      1.54 |      0.00 |   -1.54 |
| \_\_libc\_malloc2 (/                         |      1.48 |      0.00 |   -1.48 |
| xs::pipe\_t::write(xs::msg\_t\*)             |      1.48 |      0.00 |   -1.48 |
| xs\_msg\_init\_size                          |      1.31 |      0.00 |   -1.31 |
| worker(void\*)                               |      1.06 |      0.00 |   -1.06 |
| xs::clock\_t::rdtsc()                        |      0.82 |      0.00 |   -0.82 |
| \[unknown\]                                  |     81.36 |     80.93 |   -0.42 |

### Cross-table B3 — CT-1_1_1_inproc_thr-1_2_2_inproc_thr — EVENT: CYCLES

| Symbol | 1_1_1_inproc_thr % | 1_2_2_inproc_thr % | Delta % |
|:---------------------------------------------|---------:|---------:|-------:|
| \[unknown\]                                  |     23.66 |     62.58 |  +38.93 |
| \[unknown\] (/                               |      0.00 |     37.42 |  +37.42 |
| \_\_libc\_malloc2 (/                         |     10.82 |      0.00 |  -10.82 |
| \_int\_free\_chunk (/                        |      9.82 |      0.00 |   -9.82 |
| xs::pipe\_t::flush()                         |      8.58 |      0.00 |   -8.58 |
| \_int\_malloc (/                             |      4.33 |      0.00 |   -4.33 |
| xs::pipe\_t::write(xs::msg\_t\*)             |      3.96 |      0.00 |   -3.96 |
| xs::clock\_t::rdtsc()                        |      3.09 |      0.00 |   -3.09 |
| worker(void\*)                               |      2.78 |      0.00 |   -2.78 |
| xs::lb\_t::send(xs::msg\_t\*, int)           |      2.78 |      0.00 |   -2.78 |
| cfree@GLIBC\_2.2.5 (/                        |      2.47 |      0.00 |   -2.47 |
| \_int\_free\_merge\_chunk (/                 |      2.16 |      0.00 |   -2.16 |
| xs::msg\_t::close()                          |      1.86 |      0.00 |   -1.86 |
| xs\_recvmsg                                  |      1.85 |      0.00 |   -1.85 |
| xs::msg\_t::flags()                          |      1.85 |      0.00 |   -1.85 |
| xs::socket\_base\_t::recv(xs::msg\_t\*, int) |      1.85 |      0.00 |   -1.85 |
| xs\_sendmsg                                  |      1.55 |      0.00 |   -1.55 |
| malloc@plt                                   |      1.55 |      0.00 |   -1.55 |
| xs::pipe\_t::read(xs::msg\_t\*)              |      1.54 |      0.00 |   -1.54 |
| \_\_syscall\_cancel\_arch\_end (/            |      1.53 |      0.00 |   -1.53 |

### Cross-table C1 — CT-1_2_2_inproc_lat-1_2_5_inproc_lat — EVENT: BRANCH-MISSES

| Symbol | 1_2_2_inproc_lat % | 1_2_5_inproc_lat % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |     56.15 |     54.36 |   -1.79 |
| \[unknown\] (/ |     43.85 |     45.64 |   +1.79 |

### Cross-table C2 — CT-1_2_2_inproc_lat-1_2_5_inproc_lat — EVENT: CACHE-MISSES

| Symbol | 1_2_2_inproc_lat % | 1_2_5_inproc_lat % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |     80.44 |     83.77 |   +3.32 |
| \[unknown\] (/ |     19.56 |     16.23 |   -3.32 |

### Cross-table C3 — CT-1_2_2_inproc_lat-1_2_5_inproc_lat — EVENT: CYCLES

| Symbol | 1_2_2_inproc_lat % | 1_2_5_inproc_lat % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |     57.90 |     66.70 |   +8.80 |
| \[unknown\] (/ |     42.10 |     33.30 |   -8.80 |

### Cross-table D1 — CT-1_2_2_inproc_thr-1_2_5_inproc_thr — EVENT: BRANCH-MISSES

| Symbol | 1_2_2_inproc_thr % | 1_2_5_inproc_thr % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |     55.09 |     57.38 |   +2.29 |
| \[unknown\] (/ |     44.91 |     42.62 |   -2.29 |

### Cross-table D2 — CT-1_2_2_inproc_thr-1_2_5_inproc_thr — EVENT: CACHE-MISSES

| Symbol | 1_2_2_inproc_thr % | 1_2_5_inproc_thr % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\] (/ |     19.07 |     10.96 |   -8.10 |
| \[unknown\]    |     80.93 |     89.04 |   +8.10 |

### Cross-table D3 — CT-1_2_2_inproc_thr-1_2_5_inproc_thr — EVENT: CYCLES

| Symbol | 1_2_2_inproc_thr % | 1_2_5_inproc_thr % | Delta % |
|:---------------|---------:|---------:|-------:|
| \[unknown\]    |     62.58 |     67.66 |   +5.08 |
| \[unknown\] (/ |     37.42 |     32.34 |   -5.08 |

> Cross-table caveat: for RISC-V recordings the `perf record` event was `cpu-clock` (software event), which `amixis compare` labels under `CYCLES`; Build B of the `CYCLES` tables is therefore cpu-clock, not a hardware cycle count. For CT-C/CT-D every symbol is `[unknown]` because qemu-user guest symbols are unresolvable, so these tables describe the host qemu/JIT sample distribution, not guest function hotspots.

---

## 6. Notes About Exploration Process

1. **`amixis` (Amphimixis) build/profile could not be used.** Both `amphimixis-build` (`amixis build`) and `amphimixis-profile` failed at config load with `CONFIGURATOR | ERROR | Invalid local machine arch: riscv, your machine is x86_64`, because the target platform (riscv, no remote address) is treated as a local host and the tool requires a local platform's arch to match `uname -m`. `amixis validate` passes (structural validation does not exercise the arch check). Consequently both builds and all profiling used documented manual fallbacks replicating the recipe flags and the amixis perf pipeline. `amixis compare` DID work and produced all `cross-tables/CT-*.md` files.
2. **No `<project>.json` / `<project>.yaml` / `<project>.pkl` profile file exists** because `amixis profile`/`amixis run` never succeeded. That tool-owned structured profile data is **NOT AVAILABLE**; the report relies on the raw measured `perf stat` medians and the tool-owned cross-tables instead.
3. **Missing toolchain.** `autoconf`, `automake`, `libtool`, `pkg-config` and `file` were absent and installed via `apt-get` (network available).
4. **`./autogen.sh` fails spuriously** because it guards on `/usr/bin/libtool`, which modern libtool no longer installs (only `libtoolize`); its payload `mkdir config && autoreconf --install --force -I config` was run directly.
5. **Project forces `-Werror`.** The 2012 codebase sets `libxs_werror=yes` on Linux with no toggle; modern GCC's `-Wstringop-truncation` fails `address.cpp:432`. `-Wno-error` was added to CFLAGS/CXXFLAGS for both platforms (optimization-neutral, parity preserved).
6. **libtool discarded plain `-static`** for programs; libtool's `-all-static` was injected at make time (and `perf`/`tests` cleaned to force relink), after which all target programs were confirmed statically linked.
7. **`/usr/bin/qemu-riscv64-static` did not exist**; only `/usr/bin/qemu-riscv64` (statically linked). A compatibility symlink `/usr/bin/qemu-riscv64-static -> /usr/bin/qemu-riscv` was created outside the workspace; if the container is recreated, recreate it or substitute `/usr/bin/qemu-riscv64`. All target executables were run as absolute qemu command lines: `/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu <absolute-binary>`.
8. **`nice -n -20` was denied** (`setpriority: Permission denied`, no CAP_SYS_NICE) even as root; no elevated priority was applied to either platform (matched conditions).
9. **`/proc/sys/kernel/kptr_restrict` and `nmi_watchdog` are on a read-only filesystem** (could not lower `kptr_restrict` from 1), so kernel symbols resolve as `[k] unknown`, most x86 samples are unresolved kernel, and `L1-icache-load-misses` is `<not counted>`.
10. **RISC-V guest symbols are not resolvable under qemu-user** (0 `xs::` symbols); target hotspots are therefore NOT AVAILABLE, with host qemu/JIT addresses reported only. This is inherent to user-mode emulation.
11. **`amphimixis-analyze-vectorization` fails on RISC-V binaries** (exit 1/2) because it calls the native `objdump`; fallback `/usr/bin/riscv64-linux-gnu-objdump` was used.
12. **RISC-V has no native `cycles` counter in this pipeline**; recordings used the `cpu-clock` software event, which `amixis compare` labels as `CYCLES` (not the same physical quantity as x86 hardware cycles).
13. **CPU frequency is not controllable** (`powersave` governor, read-only sysfs); absolute timings are at ~1.3 GHz. Both platforms ran under the same policy on core 0, so the comparisons are internally consistent but not peak-frequency figures.
14. **Builds `1_1_1` and `1_2_2` share identical in-tree executable paths** in `input.yml`; only one architecture can occupy them at a time. `stage-x86.sh` / `stage-riscv.sh` / `stage-riscv-opt.sh` were used so each measurement ran the correct binaries (verified by SHA-256 in the optimized re-profile).
15. **The two Phase-6 optimizations were applied together in one build (`1_2_5`)**, so their individual contributions are inferred (from the x86 allocator hotspot and the RVV count change), not separately measured. The `max_vsm_size` change required a companion `xs_msg_t` opaque-buffer ABI bump (32→72 bytes) enforced by `xs_msg_size_check`.
16. **QEMU/emulation caveat (repeated for emphasis):** all target timings and counters are host x86 measurements of the qemu process under TCG dynamic translation; they include emulation overhead and do not represent native riscv64 hardware performance.
17. No `/tmp` was used for any purpose; no tool-owned file (`improvements.json`, `cross-tables/CT-*.md`, profile JSON/YAML/pkl) was edited by an agent.
18. In the initial config, the provided recipes used CMake flags (`-DCMAKE_*`) for an Autotools project and defined only one platform; these were corrected during Phase 2 to Autotools flags, a second (riscv) platform was added, and `run_machine` for the cross build was set to the riscv platform.

---

## 7. Migration Readiness Summary

| Criterion | Result |
|---|---|
| Builds on reference platform (x86_64) | YES — `1_1_1` builds successfully (24/24 tests) |
| Tests pass on reference platform | YES — 24/24 via `make check` |
| Builds on target platform (riscv64) | YES — `1_2_2` (and optimized `1_2_5`) cross-compile statically; all 30 programs built |
| Tests pass on target platform | YES — 24/24 under qemu-user (`qemu-riscv64-static -L /usr/riscv64-linux-gnu`); native riscv64 hardware NOT tested |
| Zero external dependencies | NO mandatory third-party dependency; POSIX pthreads/librt come from glibc; optional OpenPGM ≥ 5.1 is disabled (`--with-pgm=no`) and is available as a riscv64 distro package |
| No hand-written intrinsics | YES — zero SIMD intrinsics; only scalar arch asm (`rdtsc`, atomics), all `#if`-guarded and inert on riscv64 |
| Alignment safe | YES — no endianness or pointer-size assumptions; only standard `htonl` network byte order |
| Exceptions handled | Not used — code relies on `nothrow` allocation; no `throw`/`catch`/`<stdexcept>`/`-fexceptions` dependency |
| Auto-vectorization | YES on both — x86 SSE/AVX; riscv RVV via default full-V `-march` (app RVV removed by `-march=rv64gc` in `1_2_5`) |

### Migration Verdict: **MINOR CONCERNS**

The LIBXS (Crossroads I/O) codebase is inherently portable: it has no SIMD intrinsics, no endianness/pointer-size assumptions, and all architecture-specific assembly is cleanly fenced off so that none of it compiles on riscv64. It builds and passes its full test suite (24/24) on both the reference x86_64 platform and the riscv64 target, and dependency portability is clean. The concerns are environmental/build-system rather than code: the upstream project is dormant (last release 2012), the 2012 Autotools snapshot needs `autoreconf`/`config.guess` regeneration and a `-Wno-error` workaround under modern GCC, static target linking required a libtool `-all-static` workaround, and — most importantly — **the riscv64 target was only validated under qemu-user emulation, not on native riscv64 hardware**. All target performance numbers carry QEMU emulation overhead and are not native RISC-V performance figures.

### Required Actions

1. **Validate on native riscv64 hardware.** All target results here are under qemu-user TCG; run the build and the 24-test suite on real RV64 silicon to confirm behaviour and obtain meaningful performance baselines.
2. **Modernize the build system.** Regenerate `config.guess`/`config.sub` (or refresh the Autotools snapshot) so the `riscv64-linux-gnu` triplet is recognized; consider replacing the 2012 `autogen.sh` libtool check.
3. **Address the `-Werror` incompatibility** by fixing (or suppressing) the `strncpy` truncation warning in `src/address.cpp:432`, so the project builds without `-Wno-error` on modern toolchains.
4. **Adopt the validated optimizations** (`max_vsm_size` 29→64 with the `xs_msg_t` ABI bump, and `-march=rv64gc` for target builds), which together gave +29.6% throughput and −8.5% latency on the emulated target; re-validate the ABI change for downstream consumers before release.
5. **Keep PGM optional** (`--with-pgm=no`) unless multicast is required; if needed, test the riscv64 system `libpgm` rather than the bundled 5.1.118.
6. **Retain static linking for qemu-user deployments**, but strip debug info for distribution (`--strip-debug` reduces the riscv binary from 6,268,904 B to 2,752,512 B with no runtime effect); note that fully static images are ~6× larger than the dynamic x86 build due to embedded glibc/libstdc++.