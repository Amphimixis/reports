# libfuse — Migration Readiness Report

**Project:** libfuse
**Reference platform:** x86_64 (native)
**Target platform:** riscv64 (cross-compiled with `riscv64-linux-gnu-gcc`, executed under `qemu-riscv64-static` user-mode emulation)
**Workspace:** `/work/libfuse-workspace/` — **Cloned repository:** `/work/libfuse-workspace/libfuse`
**Resolved upstream clone URL:** `https://github.com/libfuse/libfuse.git`
**Analysis date:** 2026-10-04

> **Data provenance.** Numbers in this report come from: (a) subagent repository-analysis output, (b) `improvements.json`, (c) `cross-tables/CT-*.md`, (d) `libfuse.json` (written by `amixis profile`). Where the Amphimixis profiling tool failed to produce counters, a clearly labelled **manual fallback** (`perf stat -ddd -x, --repeat …`, outside the tool) was used. Nothing is fabricated; unmeasured values are marked **NOT AVAILABLE**.

---

## 1. Repository & Project Status

| Field | Value |
|---|---|
| Resolved clone URL | `https://github.com/libfuse/libfuse.git` (verified active upstream) |
| Local clone path | `/work/libfuse-workspace/libfuse` |
| Latest commit | `eb7bae84508c520bdcedcfd88175c5afe4c82ecb` (2026-10-02, "build(deps): bump the codeql-action group with 3 updates") |
| Total commits | 2,814 (first commit 2001-10-23) |
| Latest tag / release | `fuse-3.18.3` (2026-09-09) |
| HEAD description | `fuse-3.16.2-908-geb7bae84`; in-tree version `3.19.0-rc0` |
| Branches | `master` + 16 remote branches |
| Activity | 545 commits in last 12 months; 464 in last 6 months; actively maintained but "maintenance-only" |
| Stars / Forks / Watchers | ~6,100 / ~1,300 / 152 |
| Build system | **Meson + Ninja** (`meson.build`, `meson_options.txt`, requires `meson >= 0.60`); no autotools/CMake backend |
| Source size | 97 C/C++ source+header files, ~55,189 LOC |
| Tests | Meson test suite + `test/run-tests.py`; ~106 test cases (`ctests`, `examples`, `misc`, `mount`, `notify`, `unit`) |
| CI | GitHub Actions present (`.github/workflows/`), x86_64-only matrix (plus FreeBSD via qemu-system); **no RISC-V job** |
| Benchmarks | Absent (`amixis analyze`: `benchmarks: not found`); `xfstests/` is external config only |
| Documentation | `doc/` (14 READMEs + man pages), `dev-docs/`, Doxygen API docs |
| Distro packaging | Alpine `fuse3 3.18.3-r0` built for **riscv64**; Debian source `fuse3 3.18.3-1` (riscv64 is a Debian release arch) |
| External dependencies | Required: libc + pthreads. Optional: libdl, librt, liburing, iconv, systemd, udev, backtrace. Build/test: meson, ninja, python3, gdb. **No gettext/NLS usage.** |

---

## 2. Platform-Specific Code Analysis

### 2.1 Architecture macros (semantics verified — names can be misleading)

| Macro | File:Line | What it actually guards | riscv64 impact |
|---|---|---|---|
| `__powerpc__` / `__powerpc64__` | `lib/usdt.h:305,450` | USDT asm operand constraints for SystemTap probes | Not used on riscv64 |
| `__arm__` | `lib/usdt.h:307` | USDT operand constraint `g` | Not used |
| `__loongarch__` | `lib/usdt.h:309` | USDT operand constraint `nmr` | Not used; no explicit `__riscv` case (generic default used) |
| `__ia64__` / `__s390__` / `__s390x__` | `lib/usdt.h:317` | Selects `nop 0` vs `nop` for USDT probe | riscv64 gets generic `nop` (valid RISC-V instruction) |
| `__LP64__` | `lib/usdt.h:361` | `.8byte` vs `.4byte` SDT note address | Correct on riscv64 LP64 |
| `__i386__` | `lib/usdt.h:452` | USDT arg register alias workaround | Not used |
| `__ia64__` | `util/fusermount.c:365` | Itanium `__clone2()` vs generic `clone()` | riscv64 uses generic `clone()` — portable |
| `__SIZEOF_LONG__ > 4` | `util/fuser_conf.c:369` | 64-bit UFSD magic in whitelist | Correct on riscv64 LP64 |
| `__i386__`, `__powerpc64__`, `__s390x__`, `__sun__` | `checkpatch.pl:6810` | Regex in bundled style checker; **not compiled** | None |

### 2.2 Platform preprocessor guards

| Guard | Location (representative) | riscv64 impact |
|---|---|---|
| `__linux__` | `lib/fuse_lowlevel.c:25`, `lib/mount_util.c:16`, `util/fusermount.c:15`, several `test/*.c` | Active/expected on riscv64 Linux |
| `!__NetBSD__ && !__FreeBSD__ && !__DragonFly__ && !__FreeBSD_kernel__` | `include/fuse_mount_compat.h:14` | Active on Linux; only adds `#ifndef` fallbacks |
| `__NetBSD__` / `__FreeBSD__` / `__DragonFly__` / `__ANDROID__` | `lib/mount_util.c:30,47` | Not active on riscv64/Linux |
| BSD guards | `lib/fuse.c`, `lib/helper.c`, `example/*`, `test/*` | Not active |
| `__GLIBC__ >= 2 && __GLIBC_MINOR__ >= 32` | `lib/fuse_lowlevel.c:359` | Libc-version guard (not arch-specific); correct on riscv64 glibc ≥2.32 |
| `__UCLIBC__` / `__APPLE__` | `meson.build:355` | Portable; riscv64 glibc keeps symbol versioning |
| `_WIN32` | `example/cxxopts.hpp:271` | Not active |
| `__GNUC__` version | `example/passthrough_ll.c:59` | Portable |

### 2.3 Endianness / pointer-size

- No `__BYTE_ORDER__`, `__LITTLE_ENDIAN__`, or `__BIG_ENDIAN__` branching anywhere.
- `htobe64`/`<endian.h>` in `util/fusermount.c:25,1611` performs explicit, endian-independent conversion.
- `__LP64__` / `__SIZEOF_LONG__` guards evaluate correctly for riscv64 LP64.

### 2.4 Other arch-relevant constructs

- Hardcoded syscall numbers (`__NR_move_mount 429`, `__NR_fsopen 430`, `__NR_fsconfig 431`, `__NR_fsmount 432`) in `lib/mount_fsmount.c:38-49` are **asm-generic** values, identical on RISC-V, and are only `#ifndef` fallbacks.
- `SYS_listmount`/`SYS_statmount`, `SYS_capget`/`SYS_capset`, `SYS_mbind` come from headers — portable.
- `FUSE_SYMVER` `.symver` works with GNU as on RISC-V.
- Vendored `lib/usdt.h` has no `__riscv` case but defaults are valid; USDT is **disabled by default**.

### 2.5 Vectorization intrinsics in source

**NONE FOUND.** No `<immintrin.h>`/`<xmmintrin.h>`/`<arm_neon.h>`/`<riscv_vector.h>`, no `_mm*`/`__riscv_v*` intrinsics, no `__SSE*`/`__AVX*`/`__ARM_NEON` macros anywhere.

### 2.6 Portability verdict

| Aspect | Verdict |
|---|---|
| Exceptions | **No exceptions** (C library; error paths use return codes/errno) |
| Alignment safety | **Safe** — no alignment-sensitive constructs; `__LP64__`/`__SIZEOF_LONG__` handled |
| Embedded usability | **Good** — optional deps can all be disabled; no mandatory external libraries |
| Overall | **LOW concern / migration-ready** |

---

## 3. Build & Test Results

The canonical config `/work/input.yml` declares the riscv platform, but `amixis 0.2.0` has **no Meson build backend** and its `amixis build` command fails (`Failed to create build`) for two reasons documented in Section 6. The four builds were therefore produced with **real Meson + Ninja** invocations into the directories the profiler expects (`/work/<build_name>`).

| Build | Platform | Compiler flags | Build result | Tests |
|---|---|---|---|---|
| `1_1_1` | x86_64 reference (baseline) | `-O2 -g`, `debugoptimized`, shared | **SUCCESS** | 106 tests: **7 passed / 0 failed / 99 skipped** |
| `1_1_2` | x86_64 reference (optimized) | `-O3 -march=native -g`, `release` | **SUCCESS** | 106 tests: **7 passed / 0 failed / 99 skipped** |
| `1_2_3` | riscv64 target (baseline) | `-O2 -march=rv64gc -mabi=lp64d -g`, cross | **SUCCESS** | 2 no-mount unit binaries run under QEMU, exit 0; full suite not runnable under user-mode QEMU |
| `1_2_4` | riscv64 target (optimized) | `-O3 -march=rv64gcv -mabi=lp64d -g`, cross | **SUCCESS** | same as `1_2_3` |

**Build failure detail (Amphimixis tool):** `amixis build <project> --config /work/input.yml --build-name <name>` returned exit 1 with `[Config][✗] Failed to create build`. Root causes: (1) `_has_valid_arch()` rejects a **local** riscv platform on an x86_64 host; (2) `amphimixis 0.2.0` registers only `cmake` and `make` build systems, not Meson. The manual Meson builds succeeded, and `amixis profile` can run against `/work/input-profile.yml`.

**Test failure detail:** there were **no test failures**. On x86_64 the 99 skips are because `/dev/fuse` and the FUSE kernel module are unavailable and `fusermount3` is absent; on riscv64 only mount-free unit binaries can execute under `qemu-riscv64-static`.

**Architecture confirmation (readelf/objdump):** x86 binaries are `ELF64 … Advanced Micro Devices X86-64`; riscv binaries are `elf64-littleriscv architecture: riscv:rv64` with `Tag_RISCV_arch` `rv64gc` (baseline) and `…v1p0…` (optimized).

---

## 4. Performance Comparison

### 4.1 Experimental conditions

| Condition | Value |
|---|---|
| CPU | Intel 13th Gen Core i7-13620H (hybrid P/E cores), 16 threads |
| Pinning | `taskset -c 0` (a P-core; `cpu_atom` events `<not counted>` on CPU 0) |
| Priority | `nice -n -20` **NOT AVAILABLE** — container lacks `CAP_SYS_NICE`; runs at nice 0 |
| `perf_event_paranoid` | `-1` (sufficient) |
| `kptr_restrict` | `1` (kernel symbols unresolvable in `perf report`) |
| Warm-up | 2 runs per executable before profiling |
| Measurement runs (tool) | 1 per executable (tool has no repeats) |
| Measurement runs (manual fallback) | 10 repeats (`perf stat --repeat 10`) / 3×200 wall-clock batches |
| Frequency | governor `powersave`, turbo enabled, effective 3.46–4.51 GHz (**DVFS active/uncontrolled**) |
| QEMU | `qemu-riscv64-static` user-mode TCG; no KVM, no hardware V |

### 4.2 Tool-recorded statistics — `libfuse.json` (written by `amixis profile`)

The tool recorded `/bin/time` values only; every `perf stat` counter collection failed (see Section 6). For all 8 executable runs the JSON contains `real_time/user_time/kernel_time` = `0.00`/`0.00`/`0.00` (x86) or `0.01`/`0.01`/`0.00` (riscv), `executable_run_success = true`, `perf_record_name` set, `perf_script_name = null`, `perf_archive_name = null`, and `perf_stat` containing only the text `Error: switch \`x' requires a value`. The full recorded content is reproduced verbatim in a code block immediately below this table.

| Build | Executable | real (s) | user (s) | kernel (s) | perf_stat | perf_script_name | perf_archive_name |
|---|---|---:|---:|---:|---|---|---|
| `1_1_1` | `test/test_want_conversion` | 0.00 | 0.00 | 0.00 | ERROR (`switch x`) | null | null |
| `1_1_1` | `test/test_loop_config` | 0.00 | 0.00 | 0.00 | ERROR | null | null |
| `1_1_2` | `test/test_want_conversion` | 0.00 | 0.00 | 0.00 | ERROR | null | null |
| `1_1_2` | `test/test_loop_config` | 0.00 | 0.00 | 0.00 | ERROR | null | null |
| `1_2_3` | `qemu…test_want_conversion` | 0.01 | 0.01 | 0.00 | ERROR | null | null |
| `1_2_3` | `qemu…test_loop_config` | 0.01 | 0.01 | 0.00 | ERROR | null | null |
| `1_2_4` | `qemu…test_want_conversion` | 0.01 | 0.01 | 0.00 | ERROR | null | null |
| `1_2_4` | `qemu…test_loop_config` | 0.01 | 0.01 | 0.00 | ERROR | null | null |

**Tool-recorded `libfuse.json` (verbatim):**

```json
{
  "1_2_4": {
    "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_4/test/test_want_conversion": {
      "build_name": "1_2_4",
      "executable": "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_4/test/test_want_conversion",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__2__4.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_s1__2__4_stest_stest__want__conversion.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_4/test/test_loop_config": {
      "build_name": "1_2_4",
      "executable": "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_4/test/test_loop_config",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__2__4.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_s1__2__4_stest_stest__loop__config.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_2_3": {
    "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_3/test/test_want_conversion": {
      "build_name": "1_2_3",
      "executable": "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_3/test/test_want_conversion",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__2__3.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_s1__2__3_stest_stest__want__conversion.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_3/test/test_loop_config": {
      "build_name": "1_2_3",
      "executable": "/usr/bin/qemu-riscv64-static -L /usr/riscv64-linux-gnu /work/1_2_3/test/test_loop_config",
      "executable_run_success": true,
      "real_time": "0.01",
      "user_time": "0.01",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__2__3.._susr_sbin_sqemu-riscv64-static_x20_-L_x20__susr_sriscv64-linux-gnu_x20__swork_s1__2__3_stest_stest__loop__config.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_2": {
    "test/test_want_conversion": {
      "build_name": "1_1_2",
      "executable": "test/test_want_conversion",
      "executable_run_success": true,
      "real_time": "0.00",
      "user_time": "0.00",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__1__2..test_stest__want__conversion.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test/test_loop_config": {
      "build_name": "1_1_2",
      "executable": "test/test_loop_config",
      "executable_run_success": true,
      "real_time": "0.00",
      "user_time": "0.00",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__1__2..test_stest__loop__config.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  },
  "1_1_1": {
    "test/test_want_conversion": {
      "build_name": "1_1_1",
      "executable": "test/test_want_conversion",
      "executable_run_success": true,
      "real_time": "0.00",
      "user_time": "0.00",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__1__1..test_stest__want__conversion.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    },
    "test/test_loop_config": {
      "build_name": "1_1_1",
      "executable": "test/test_loop_config",
      "executable_run_success": true,
      "real_time": "0.00",
      "user_time": "0.00",
      "kernel_time": "0.00",
      "perf_stat": [
        { "counter_value": "Error: switch `x' requires a value" },
        { "counter_value": "Usage: perf stat [<options>] [<command>]" },
        { "counter_value": "-x, --field-separator <separator>" },
        { "counter_value": "print counts with custom separator" }
      ],
      "perf_record_name": "1__1__1..test_stest__loop__config.perfdata",
      "perf_script_name": null,
      "perf_archive_name": null
    }
  }
}
```

### 4.3 Key metrics — manual fallback (REAL MEASURED, outside the tool)

Because the tool's counters failed, the profiler performed a manual `perf stat -ddd -x, --repeat 10` fallback. **All riscv columns are HOST x86_64 counters for the `qemu-riscv64-static` process, NOT guest RISC-V execution.** Not comparable across architectures.

`test_loop_config`:

| Metric | `1_1_1` x86 -O2 | `1_1_2` x86 -O3 native | `1_2_3` riscv (host QEMU) | `1_2_4` riscv (host QEMU) |
|---|---:|---:|---:|---:|
| Elapsed time | 0.868 ms | 0.912 ms | 14.053 ms | 13.529 ms |
| IPC | 1.09 | 1.17 | 2.20 | 2.21 |
| L1-dcache miss rate | 2.66 % | 2.65 % | 0.54 % | 0.53 % |
| LLC miss rate | 9.43 % | 7.34 % | 38.69 % | 18.88 % |
| Branch misprediction rate | 2.84 % | 2.81 % | 1.99 % | 2.00 % |
| Frontend Bound | 41.8 % | 41.5 % | 38.7 % | 39.1 % |
| Backend Bound | 27.0 % | 26.5 % | 10.6 % | 9.6 % |
| Retiring | 21.0 % | 21.6 % | 35.6 % | 36.1 % |

`test_want_conversion`:

| Metric | `1_1_1` x86 -O2 | `1_1_2` x86 -O3 native | `1_2_3` riscv (host QEMU) | `1_2_4` riscv (host QEMU) |
|---|---:|---:|---:|---:|
| Elapsed time | 0.956 ms | 1.027 ms | 14.112 ms | 13.509 ms |
| IPC | 1.21 | 1.16 | 2.15 | 2.13 |
| L1-dcache miss rate | 2.67 % | 2.66 % | 0.54 % | 0.53 % |
| LLC miss rate | 5.64 % | 7.21 % | 34.07 % | 26.27 % |
| Branch misprediction rate | 2.70 % | 2.69 % | 2.02 % | 2.02 % |
| Frontend Bound | 42.3 % | 42.9 % | 38.9 % | 39.0 % |
| Backend Bound | 24.8 % | 25.5 % | 10.6 % | 10.5 % |
| Retiring | 22.5 % | 21.7 % | 35.7 % | 35.5 % |

### 4.4 Hotspot analysis

**Function-level hotspots: NOT AVAILABLE** on either platform. The workloads are microsecond-scale; `perf record -F 1000` collected only 8 (x86) and 31–33 (riscv) samples; `kptr_restrict=1` blocked kernel symbol resolution and the test-binary offsets could not be resolved. On riscv the samples fall in the QEMU JIT/translated-code region (not guest code).

| Platform | Observed sample distribution (tool `.scriptout`) | Usable hotspots? |
|---|---|---|
| x86_64 reference | Mostly `[unknown]` / `[unknown] (/ (deleted))`; only `strncmp` ×1 and `pthread_rwlock_wrlock` ×1 named | **NOT AVAILABLE** |
| riscv64 under QEMU | ~88.8 % `[unknown]` in `qemu-riscv64-static` TCG/JIT regions, remainder `/tmp/perf-*.map` | **NOT AVAILABLE** (host QEMU code, not guest) |

### 4.5 Bottleneck summary and QEMU/emulation caveats

- **Dominant riscv bottleneck = `qemu-riscv64-static` TCG emulation**, not libfuse code: ~16× wall-clock, ~29× host cycles, ~58× host instructions vs x86 native.
- x86 test binaries are **process-startup / dynamic-linker bound** (sub-millisecond compute, Frontend Bound ≈ 42 %, IPC ≈ 1.1).
- **QEMU caveat (repeated):** all riscv timing and counters include emulation overhead and measure the host QEMU process. They do **not** reflect native RISC-V hardware performance. No native RISC-V hardware was available; the target was cross-built and run under user-mode emulation only.
- No SIMD bottleneck exists to fix (see Section 5).

### 4.6 Cross-tables (produced by `amixis compare`, copied verbatim from `cross-tables/CT-*.md`)

> The tool labels the comparison columns with the build paths it recorded (`/work/1_1_1 %`, etc.). Tables are reproduced exactly as written by the tool; several rows are degenerate (`[unknown]`) because symbol resolution failed.

## Cross-table: `1__1__1..test_stest__loop__config` vs `1__1__2..test_stest__loop__config`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_loop\_\_config and 1\_\_1\_\_2..test\_stest\_\_loop\_\_config

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol      | /work/1_1_1 % | /work/1_1_2 % | Delta % |
|:------------|-------------:|-------------:|-------:|
| \[unknown\] |        100.00 |        100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol         | /work/1_1_1 % | /work/1_1_2 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\] (/ |         32.43 |         15.41 |  -17.02 |
| \[unknown\]    |         67.57 |         84.59 |  +17.02 |

## Cross-table: `1__1__1..test_stest__want__conversion` vs `1__1__2..test_stest__want__conversion`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_want\_\_conversion and 1\_\_1\_\_2..test\_stest\_\_want\_\_conversion

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol      | /work/1_1_1 % | /work/1_1_2 % | Delta % |
|:------------|-------------:|-------------:|-------:|
| \[unknown\] |        100.00 |        100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                  | /work/1_1_1 % | /work/1_1_2 % | Delta % |
|:------------------------|-------------:|-------------:|-------:|
| \[unknown\] (/          |          0.00 |         17.07 |  +17.07 |
| pthread\_rwlock\_wrlock |         15.21 |          0.00 |  -15.21 |
| strncmp                 |          8.44 |          0.00 |   -8.44 |
| \[unknown\]             |         76.35 |         82.93 |   +6.58 |

## Cross-table: `1__2__3..qemu-riscv64-static…loop__config` vs `1__2__4..qemu-riscv64-static…loop__config`

# Cross-tables for 1\_\_2\_\_3..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_s1\_\_2\_\_3\_stest\_stest\_\_loop\_\_config and 1\_\_2\_\_4..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_s1\_\_2\_\_4\_stest\_stest\_\_loop\_\_config

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol         | /work/1_2_3 % | /work/1_2_4 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\] (/ |        100.00 |        100.00 |   +0.00 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol         | /work/1_2_3 % | /work/1_2_4 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\] (/ |         29.00 |         23.77 |   -5.23 |
| \[unknown\]    |         71.00 |         76.23 |   +5.23 |

## EVENT: CYCLES

| Symbol         | /work/1_2_3 % | /work/1_2_4 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\] (/ |         79.07 |         86.75 |   +7.68 |
| \[unknown\]    |         20.93 |         13.25 |   -7.68 |

## Cross-table: `1__1__1..test_stest__loop__config` vs `1__2__3..qemu-riscv64-static…loop__config`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_loop\_\_config and 1\_\_2\_\_3..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_s1\_\_2\_\_3\_stest\_stest\_\_loop\_\_config

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol         | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\] (/ |          0.00 |        100.00 | +100.00 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol         | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\]    |        100.00 |         71.00 |  -29.00 |
| \[unknown\] (/ |          0.00 |         29.00 |  +29.00 |

## EVENT: CYCLES

| Symbol         | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:---------------|-------------:|-------------:|-------:|
| \[unknown\]    |         67.57 |         20.93 |  -46.64 |
| \[unknown\] (/ |         32.43 |         79.07 |  +46.64 |

## Cross-table: `1__1__1..test_stest__want__conversion` vs `1__2__3..qemu-riscv64-static…want__conversion`

# Cross-tables for 1\_\_1\_\_1..test\_stest\_\_want\_\_conversion and 1\_\_2\_\_3..\_susr\_sbin\_sqemu-riscv64-static\_x20\_-L\_x20\_\_susr\_sriscv64-linux-gnu\_x20\_\_swork\_s1\_\_2\_\_3\_stest\_stest\_\_want\_\_conversion

## EVENT: CPU\_CORE/BRANCH-MISSES/

| Symbol      | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:------------|-------------:|-------------:|-------:|
| \[unknown\] |          0.00 |        100.00 | +100.00 |

## EVENT: CPU\_CORE/CACHE-MISSES/

| Symbol      | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:------------|-------------:|-------------:|-------:|
| \[unknown\] |        100.00 |        100.00 |   +0.00 |

## EVENT: CYCLES

| Symbol                  | /work/1_1_1 % | /work/1_2_3 % | Delta % |
|:------------------------|-------------:|-------------:|-------:|
| \[unknown\]             |         76.35 |         97.06 |  +20.71 |
| pthread\_rwlock\_wrlock |         15.21 |          0.00 |  -15.21 |
| strncmp                 |          8.44 |          0.00 |   -8.44 |
| \[unknown\] (/          |          0.00 |          2.94 |   +2.94 |

> A sixth intended cross-table — riscv `test_want_conversion` `1_2_3` vs `1_2_4` — is **NOT AVAILABLE**: `amixis compare` crashed with `OSError: [Errno 36] File name too long` when writing the CT file (QEMU command strings embedded in the generated filename exceed NAME_MAX).

---

## Improvement of 1_2_4 compared to 1_2_5

Source: `improvements.json` (verbatim values). `1_2_5` = riscv64 `-O3 -march=rv64gc -mabi=lp64d`, `-Ddebug=false -Db_lto=true -Ddefault_library=both`. Debug-info removal + LTO reduces artifact size only; runtime is unchanged/not meaningful (non-alloc debug sections).

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---:|---:|---:|---|
| libfuse3.so.4 | 1363304 | 320000 | 23.47 | file_size_bytes |
| test_loop_config | 30632 | 13560 | 44.27 | file_size_bytes |
| libfuse3.so.4 | 1363304 | 320000 | 23.47 | file_size_bytes |
| test_loop_config | 30632 | 13560 | 44.27 | file_size_bytes |

## Improvement of 1_2_3 compared to 1_2_6

Source: `improvements.json` (verbatim values). `1_2_6` = riscv64 `-O3 -march=rv64gc`, fully static test binaries (`-Ddefault_library=static -Dc_link_args='-static' -Ddebug=false`). Static linking removes emulated dynamic-loader work, giving a real ~38–40 % wall-time reduction under QEMU.

| Measured | Baseline value | Optimized value | Improvement % | Parameter |
|---|---:|---:|---:|---|
| test_loop_config | 13.4039 | 8.2893 | 61.84 | real_wall_time_ms |
| test_want_conversion | 13.0913 | 7.8598 | 60.04 | real_wall_time_ms |
| test_loop_config | 13.4039 | 8.2893 | 61.84 | real_time_ms |
| test_want_conversion | 13.0913 | 7.8598 | 60.04 | real_time_ms |

---

## 5. Optimization Results

### 5.1 Vector instructions in binaries

`amixis analyze -v riscv` failed (it invokes the host `objdump`). Manual `riscv64-linux-gnu-objdump` was used for RISC-V.

| Build | Binary | Arch | SIMD/vector refs | Real packed arithmetic | ymm | zmm |
|---|---|---|---:|---:|---:|---:|
| `1_1_1` (`-O2`) | test_loop_config | x86 | 4 | 0 | 0 | 0 |
| `1_1_1` | test_want_conversion | x86 | 6 | 0 | 0 | 0 |
| `1_1_1` | libfuse3.so.4 | x86 | 685 | 2 (`paddd`) | 0 | 0 |
| `1_1_2` (`-O3 -march=native`) | test_loop_config | x86 | 3 | 1 (`xorps`) | 0 | 0 |
| `1_1_2` | test_want_conversion | x86 | 29 | 0 | 20 | 0 |
| `1_1_2` | libfuse3.so.4 | x86 | 890 | 34 | 499 | 0 |
| `1_2_3` (`-O2 -march=rv64gc`) | test_loop_config / test_want_conversion / libfuse3.so.4 | riscv | 0 | 0 | — | — |
| `1_2_4` (`-O3 -march=rv64gcv`) | test_loop_config / test_want_conversion / libfuse3.so.4 | riscv | **0 RVV** | 0 | — | — |

`-march=rv64gcv` was accepted (ELF `Tag_RISCV_arch` lists `v1p0`/`zve*`) but emitted **zero RVV instructions**; GCC 13.3.0 could not auto-vectorize even a probe SAXPY loop ("no vectype"). No hand-written intrinsics exist in libfuse.

### 5.2 Optimization attempts (Before / After / Delta / Causal analysis)

| Optimization applied | Metric | Before (`1_2_3`/`1_2_4`) | After (`1_2_5`/`1_2_6`) | Delta | Causal analysis |
|---|---|---:|---:|---:|---|
| Static-link riscv tests (`-Ddefault_library=static -static`) | test_loop_config wall time | 13.4039 ms | 8.2893 ms | **−38.2 %** (improvement 61.84 %) | Removes emulated dynamic-loader + relocation work under TCG; host cycles drop ~59.9M→32.6M |
| Static-link riscv tests | test_want_conversion wall time | 13.0913 ms | 7.8598 ms | **−40.0 %** (improvement 60.04 %) | Same cause |
| No-debug + LTO (`-Ddebug=false -Db_lto=true`) | libfuse3.so.4 size | 1,363,304 B | 320,000 B | **−76.5 %** (improvement 23.47 %) | ~67–73 % of the file was DWARF debug info; LTO/GC also removes dead code |
| No-debug + LTO | test_loop_config size | 30,632 B | 13,560 B | **−55.7 %** (improvement 44.27 %) | Debug sections removed; non-alloc, so **runtime speed NOT MEASURABLE** |

### 5.3 Recommended optimizations

| Priority | Optimization | Expected gain | Effort | Notes |
|---|---|---|---|---|
| 1 | Fix Amphimixis `perf stat` invocation (`-x,` quoted + matching parser delimiter; use argv list) | Measurement fidelity (enables real counters) | Low | Without it every tool counter is an error |
| 2 | Statically link libfuse/tests for embedded riscv | Real ~38–40 % wall-time reduction under QEMU (measured) | Medium | Biggest real lever; removes emulated loader work |
| 3 | Drop forced debug info (`-Ddebug=false`) | ~67–76 % smaller library/test files | Low | Size only; runtime NOT MEASURABLE |
| 4 | Enable LTO (`-Db_lto=true`) | Smaller `.text`; cross-TU inlining | Low | Pair with `-Ddebug=false` |
| 5 | `-fno-semantic-interposition -fno-plt -fvisibility=hidden -ffunction-sections -fdata-sections -Wl,--gc-sections` | Fewer PLT relocations / smaller size | Low–Med | Marginal under startup-bound workloads |
| 6 | `-mtune=sifive-u74` / `-mcpu=…` tuning | Uncertain (0–5 % on glue code) | Low | Test with `perf stat --repeat` |
| 7 | PGO (`-Db_pgo=generate/use`) | Low | High | **NOT MEASURABLE** meaningfully (QEMU floor, µs workload) |
| 8 | RVV intrinsics / vector loops | Zero | High | No vectorizable kernels; GCC 13 emits none. Not worth it |
| 9 | Fix `amixis analyze -v riscv` to use the cross `objdump` | Tooling correctness | Low | Currently fails with "can't disassemble for architecture UNKNOWN" |

---

## 6. Notes About Exploration Process

Errors, limitations and special circumstances encountered (none of these are libfuse defects):

1. **Amphimixis 0.2.0 has no Meson build backend** — only `cmake` and `make` are registered (`core/build_systems/build_systems.py`). `amixis build` failed for all four builds; real Meson+Ninja builds were used instead. `meson` was installed with `pip3 install --break-system-packages meson` (1.12.1).
2. **Local-architecture validation blocks cross targets** — `_has_valid_arch()` rejects a local riscv platform on an x86_64 host. To profile, `/work/input-profile.yml` was created declaring platform 2 as `arch: x86` (schema-only workaround) with absolute QEMU command strings. The canonical `/work/input.yml` retains `arch: riscv`.
3. **Tool `perf stat` is broken** — it builds `perf stat -ddd -x| …` without quoting, so the shell interprets `|` as a pipe; perf errors with ``Error: switch `x' requires a value``. Consequently `libfuse.json` contains no counters. This is a host/tool issue, not a project issue. A manual, correctly-quoted fallback was used for all real metrics.
4. **`perf archive` does not exist in perf 6.8.12** — no `.tar.bz2`; `perf_script_name`/`perf_archive_name` are `null`.
5. **`amixis compare` filename overflow** — one riscv cross-table crashed with `OSError: [Errno 36] File name too long` because the QEMU command string is embedded in the generated filename.
6. **`amixis analyze -v riscv`** fails on cross binaries (host `objdump`); manual `riscv64-linux-gnu-objdump` was substituted.
7. **Optional dependencies absent** — `liburing`, `libsystemd`, `libudev` headers are not installed, so `io_uring`/systemd/udev support were disabled by Meson (non-fatal). No network/apt available.
8. **Test environment limitations** — `/dev/fuse` and the FUSE kernel module are unavailable and `fusermount3` is not installed, so 99/106 tests skipped on x86_64; riscv tests were limited to two mount-free unit binaries.
9. **`file` is not installed** — architectures were confirmed via `readelf`/`objdump`.
10. **Uncontrolled frequency/nice** — DVFS active and `nice -n -20` not permitted (no `CAP_SYS_NICE`); sub-10 % timing deltas are within noise.
11. **QEMU/emulation caveat** — all riscv measurements are user-mode TCG emulation; timing includes emulation overhead and may not reflect native RISC-V hardware. perf counters for riscv count the host QEMU process, not guest RISC-V instructions.
12. **Repeat profiling** — the Phase-6 re-measurement was performed manually (same `perf stat` fallback) because the tool's counter path is broken; no tool-based counters were obtainable for the optimized builds either.

---

## 7. Migration Readiness Summary

| Check | Status |
|---|---|
| Builds on reference platform (x86_64) | ✅ **Yes** (`1_1_1`, `1_1_2`) |
| Tests pass on reference platform | ⚠️ **Partial** — 7 passed / 0 failed / 99 skipped (no `/dev/fuse`) |
| Builds on target platform (riscv64) | ✅ **Yes** (`1_2_3`, `1_2_4`, cross-built; also `1_2_5`, `1_2_6`) |
| Tests pass on target platform | ⚠️ **Partial** — 2 mount-free unit binaries pass under QEMU; full suite needs real RISC-V hardware/kernel |
| Zero external dependencies | ✅ **Yes (core)** — only libc + pthreads required; all other deps optional |
| No hand-written intrinsics | ✅ **Yes** — 0 SIMD intrinsics in source; 0 RVV emitted |
| Alignment safe | ✅ **Yes** — LP64/`__SIZEOF_LONG__` handled; no alignment-sensitive constructs |
| Exceptions handled | ✅ **Yes** — C error codes/errno; no exceptions cross the libfuse ABI |
| Auto-vectorization | ⚠️ **Not applicable** — no vectorizable code; x86 emitted scalar AVX only, GCC 13 emitted no RVV |

### Migration Verdict: **MINOR CONCERNS**

libfuse is architecture-agnostic C with no hard RISC-V portability blockers: it cross-compiles cleanly, the mount-free unit tests pass under emulation, there are no intrinsics to port, and all external dependencies are riscv64-available (Alpine already package `fuse3 3.18.3` for riscv64). The concerns are **validation and tooling**, not code portability: the full functional (mount) suite could not run here, CI has no RISC-V coverage, and the measurement tooling had to be worked around.

### Required Actions

1. **Add a riscv64 job to CI** (the current matrix is x86_64-only) so future changes are continuously validated on the target.
2. **Run the full mount test suite on real RISC-V hardware** with the FUSE kernel module, root privileges and `fusermount3`; user-mode QEMU cannot exercise mount paths.
3. **Fix the Amphimixis profiling tool** for cross targets: use a correctly quoted/argv `perf stat -x,` invocation, select the toolchain's target `objdump`, and shorten/encode cross-table filenames.
4. **For embedded riscv deployments, prefer static linking** of libfuse/tests — measured ~38–40 % lower wall time under emulation — and drop forced `-g`/enable LTO to shrink artifacts (~67–76 % smaller).
5. **Do not rely on auto-vectorization** for riscv with GCC 13 (no RVV emitted); any future SIMD work must use hand-written RVV intrinsics, and there is currently no vectorizable hot path in libfuse.
6. **Treat all riscv performance figures in this report as emulation-bound**; re-measure on native RISC-V hardware before making performance claims.
