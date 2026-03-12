# Rotifer Python-to-Rust Porting Guide

A complete reference for porting Python/Perl bioinformatics tools to Rust using PyO3, based on the rotifer project's production experience porting 215+ functions across 5 crates.

---

> **Executive Summary — Five Things You Must Know Before You Start**
>
> 1. **The hard bugs are at boundaries, not in Rust.** PyO3's type conversions, GIL management, and numpy interop are mature and well-documented. The bugs that will consume your time live at the edges: shell compatibility between bash and zsh, numpy import cascades killing unrelated modules, TTY detection failing under Python's stdin setup, and race conditions in filesystem operations. Budget 70% of your debugging time for boundary issues.
>
> 2. **Fault isolation is everything.** One broken crate must never kill others. One broken submodule must never cascade-kill the package. Catch `Exception` (not just `ImportError`) around every submodule import, and give every crate a `NATIVE_AVAILABLE` flag so callers can degrade gracefully.
>
> 3. **Exit-code contracts enable zero-risk incremental adoption.** Exit code `0` = success, `1` = error, `2` = "I don't handle this, fall through." With this contract, the Rust layer is completely invisible to users and to downstream scripts. You can port 5 functions or 500 without changing a single caller.
>
> 4. **`OnceLock` and `py.allow_threads()` are the two patterns you will use on every function.** `allow_threads` releases the GIL for CPU work — without it, Rust blocks all Python threads. `OnceLock` initializes expensive singletons once — but it must never panic inside `get_or_init`, or the lock is permanently poisoned. Match these two patterns exactly and you will avoid the most common classes of bugs.
>
> 5. **Test through Python (pytest), not through Rust (cargo test), for cdylib crates.** `cargo test` conflicts with PyO3's linker requirements for shared libraries. Write `#[cfg(test)]` only for pure-Rust logic, and drive all integration tests from pytest. This also gives you functional parity tests against the Python originals for free.

---

## Table of Contents

1. [Introduction](#1-introduction)
   - [1.1 Principles for Any Python-to-Rust Port](#11-principles-for-any-python-to-rust-port)
2. [Architecture Overview](#2-architecture-overview)
   - [When NOT to Port: A Decision Framework](#when-not-to-port-a-decision-framework)
3. [Build System and Toolchain](#3-build-system-and-toolchain)
4. [Patterns That Worked](#4-patterns-that-worked)
5. [Bugs and How They Were Fixed](#5-bugs-and-how-they-were-fixed)
   - [Bug Taxonomy: Where Things Break](#bug-taxonomy-where-things-break)
6. [The Shell Dispatch Layer](#6-the-shell-dispatch-layer)
7. [Testing Strategy](#7-testing-strategy)
8. [Migration Playbook: Step by Step](#8-migration-playbook-step-by-step)
9. [Performance Patterns](#9-performance-patterns)
10. [Common Pitfalls](#10-common-pitfalls)
11. [Lessons Learned](#11-lessons-learned)
12. [Appendix A: File Inventory](#appendix-a-file-inventory)
13. [Appendix B: Quick Reference Card](#appendix-b-quick-reference-card)

---

## 1. Introduction

This document records every design decision, bug, and workaround encountered while porting the rotifer bioinformatics framework from Python and Perl to Rust. The port spans 5 production PyO3 crates, 215+ exported functions, 67 CLI tools with transparent dispatch, and a zsh+bash-compatible shell layer.

**Who this is for**: Anyone attempting a large-scale Python-to-Rust port using PyO3, especially for scientific computing codebases with deep shell integration, numpy interop, and CLI tools that must remain backward-compatible.

**Honest summary**: The Rust code itself was the easy part. PyO3's ergonomics for functions, numpy arrays, and type conversions are mature and well-documented. The hard bugs were almost entirely at boundaries: shell compatibility between bash and zsh, numpy import cascades killing unrelated modules, TTY detection failing under PyO3's embedded Python, and race conditions in file scanning. If you are planning a similar port, budget 70% of your debugging time for these boundary issues and 30% for actual algorithm porting.

### 1.1 Principles for Any Python-to-Rust Port

These principles are distilled from the rotifer port but apply to any project moving Python functions to Rust via PyO3. Each one encodes a lesson that cost real debugging time to learn.

1. **Port the computation, not the CLI.** Keep argparse, stdin handling, and output formatting in Python. Rust functions should accept plain data (strings, lists, numpy arrays) and return plain data. This division keeps Rust functions clean, testable, and reusable across different frontends — shell, Jupyter, web API — without change.

2. **Fail gracefully at every boundary.** Use `try/except` in Python, `match` in Rust, and exit-code fallback in the shell. No layer should be able to crash the layer above it. A Rust crate that fails to load should silently degrade to Python; a Python dispatch handler that gets bad arguments should return exit code `1`, not raise an unhandled exception.

3. **The GIL is a resource, not a lock.** Release it for CPU work (`allow_threads`), but remember that anything touching Python objects needs it back. The pattern is: extract Python data into Rust types before `allow_threads`, do pure Rust work inside, convert results back to Python types after. The borrow checker enforces most of this, but deadlocks from calling Python inside `allow_threads` are invisible to the compiler — they only manifest at runtime as a hung process.

4. **Singletons need fallbacks.** `OnceLock`, `lazy_static`, and `once_cell` all have poisoning semantics — a panic inside initialization permanently disables the singleton for the lifetime of the process. Never use `.expect()` inside `get_or_init`. Always use `match` with a safe degraded-mode value that lets the program continue operating at reduced capability.

5. **Shell compatibility is a testing surface.** Bash and zsh have different semantics for `export -f`, `BASH_SOURCE`, and array syntax. If your shell layer must work in both, test it explicitly in both from the very first line of code. Fix the incompatibilities in Phase 0, before you write a single line of Rust. Shell bugs that surface in Phase 3 are expensive to debug because they look like Rust or Python bugs.

6. **Cache expensive probes.** Import checks, file existence tests, and capability detection should be cached per session. A `python3 -c "import crate"` probe costs ~200ms. Paid once per session it is negligible; paid on every tool invocation it is intolerable. Use an associative array in the shell layer with a `$crate+x` sentinel for missing-key detection.

7. **Match output format exactly.** Users pipe tool output. If your Rust version adds a column header that the Python version omitted, or changes column order, or formats floats differently, downstream `awk` and `cut` scripts break silently. The safest approach: run both versions on the same input and diff the output before shipping. Treat output format as part of the API contract, not an implementation detail.

8. **Return Python-native types.** Dicts, lists, strings, numpy arrays. Custom `#[pyclass]` structs couple the Rust and Python implementations — the Python fallback must also produce the same class, which defeats the fallback pattern entirely. Python-native types have no such coupling: a dict is a dict regardless of which side produced it.

9. **Profile before parallelizing.** Rayon has overhead (~1-5 microseconds per task dispatch). A batch of 100 trivial transforms may be slower in parallel than sequential. Set empirical thresholds, not theoretical ones. Bisection-benchmark your specific workload: start with `PARALLEL_THRESHOLD = 10_000` and adjust based on measured crossover. Parallel code that is slower than serial code is the worst kind of complexity.

10. **Document every bug.** The bugs you hit are the bugs your users will hit. Each bug is a paragraph in your porting guide. Write it down immediately after the fix, while the root cause is fresh. Six months later you will not remember why you are calling `symlink_metadata` before `metadata`, or why there is a `ZSH_VERSION` guard before every `export -f`. The documentation is the institutional memory.

**Scale of the port**:

| Crate | Functions | Role |
|-------|-----------|------|
| `rotifer_bio_native` | 150 | Bioinformatics kernels (alignment, clustering, network, parsing) |
| `rotifer_av_native` | 24 | Antivirus scanning (entropy, patterns, hashing, Mach-O) |
| `rotifer_cli_native` | 18 | CLI tool dispatch (table ops, alignment viewer, format conversion) |
| `trouter_core` | 12 | Agent routing (tier classification, model mapping) |
| `rotifer_core` | 11 | Core parsing (FASTA, similarity, architecture) |

Total: **215+ exported `#[pyfunction]`s**, 67 CLI tools with transparent Rust acceleration, and a shell layer that works identically in bash and zsh.

---

## 2. Architecture Overview

*The architecture is three layers deep — shell, Python, Rust — and each layer can fail independently without breaking the others. The Python layer is not a shim; it is the boundary that owns argparse, stdin, and exit-code translation so that Rust functions can remain pure computation.*

### Three-Layer Dispatch

Every ported tool follows the same three-layer architecture:

```
User types "seqstats foo.fa"
        │
        ▼
┌─────────────────────┐
│  Shell Layer         │  rust_functions.sh
│  _rust_dispatch()    │  Checks crate availability, calls Python
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Python Dispatch     │  wrappers/cli_native.py
│  TOOL_REGISTRY       │  Argparse, stdin handling, exit codes
│  dispatch_tool()     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Rust Extension      │  rotifer_bio_native, rotifer_cli_native
│  #[pyfunction]       │  Actual computation
└─────────────────────┘
```

**Exit code contract**: Every dispatch handler returns an integer: `0` = success, `1` = error, `2` = fallback to Python. The shell layer checks the exit code; if it receives `2`, it falls through to the original Python/Perl implementation. This makes the Rust layer completely opt-in and invisible to users.

### Crate Workspace Layout

The workspace is defined in `rust/Cargo.toml`:

```toml
[workspace]
members = [
    "rotifer_av_native",
    "rotifer_bio_native",
    "rotifer_core",
    "trouter_core",
    "rotifer_cli_native",
]
resolver = "2"

[workspace.dependencies]
pyo3 = { version = "0.22", features = ["extension-module"] }
rayon = "1.10"
regex = "1.10"
aho-corasick = "1.1"
numpy = "0.22"
ndarray = "0.16"
```

All crates share dependency versions via `workspace.dependencies`. Each crate is a `cdylib` — it compiles to a shared library (`.so` / `.dylib`) that Python loads directly.

### Workspace Dependency Details

The `rust/Cargo.toml` workspace root (70 lines) declares all shared dependencies so individual crates inherit exact versions without drift. Complete dependency list with purposes:

| Dependency | Version | Purpose |
|-----------|---------|---------|
| `pyo3` | 0.22 | Python extension module, GIL management, type conversions |
| `rayon` | 1.10 | Work-stealing thread pool for CPU-bound parallel loops |
| `regex` | 1.10 | Pattern matching (threat definitions, taxonomy parsing) |
| `aho-corasick` | 1.1 | Multi-pattern search automaton (tier classification, malware signatures) |
| `sha2` | 0.10 | SHA-256 file hashing for quarantine integrity verification |
| `hex` | 0.4 | Hex encoding of hash digests for human-readable output |
| `ahash` | 0.8 | Fast non-cryptographic hashing for internal hash maps |
| `goblin` | 0.8 | Mach-O / ELF binary parsing for AV scanning |
| `memmap2` | 0.9 | Memory-mapped I/O for large FASTA file parsing |
| `memchr` | 2.7 | SIMD-accelerated byte search (newline scanning in text parsing) |
| `numpy` | 0.22 | ndarray ↔ numpy bridge (must match PyO3 version exactly) |
| `ndarray` | 0.16 | n-dimensional arrays for DP matrices and similarity matrices |
| `serde` + `serde_json` | 1 | JSON (de)serialization for config, results, taxonomy |
| `serde_yaml` | 0.9 | YAML parsing for threat definition files |
| `clap` | 4 | Argument parsing in `rotifer_cli_native` subcommands |
| `notify` | 7 | File system event watching (trouter_core, AV file watcher) |
| `log` | 0.4 | Structured logging via the `log` facade |
| `pyo3-build-config` | 0.22 | Build script configuration for PyO3 (used in `build.rs`) |

The workspace also excludes three non-production crates from builds via the `exclude` key: `d3d12_gpu_chain`, `exo`, and `codex_rs`. These experimental crates share the repository but are not part of the production build pipeline.

### `TOOL_REGISTRY` Exact Count

`wrappers/cli_native.py` (1658 lines) contains a `TOOL_REGISTRY` dict with exactly **63 entries** (verified by `grep -c '"_dispatch_'`). The test suite asserts `len(registry) >= 43` — the actual count is 63, giving headroom for new tools without requiring test updates. Each entry maps a tool name string to a handler function name string (e.g., `"tfilter": "_dispatch_tfilter"`).

### Python Wrapper Layer

The `wrappers/` package provides fault-tolerant imports. A broken submodule (e.g., `operon_fold` needing numpy) must never cascade-kill the entire package:

```python
# wrappers/__init__.py
_SUBMODULES = ["av_native", "bio_native", "cli_native", "core", "operon_fold", "trouter"]
_loaded = {}

for _name in _SUBMODULES:
    try:
        _loaded[_name] = importlib.import_module(f"wrappers.{_name}")
    except Exception as _exc:
        _logger.debug("wrappers.%s failed to import: %s", _name, _exc)
        _loaded[_name] = None
```

Each per-crate wrapper follows the `NATIVE_AVAILABLE` pattern:

```python
# wrappers/cli_native.py
NATIVE_AVAILABLE = False
try:
    import rotifer_cli_native as _mod
    NATIVE_AVAILABLE = True
except ImportError:
    _mod = None
```

The `status()` function returns a dict of crate availability for diagnostics:

```python
>>> from wrappers import status
>>> status()
{'rotifer_core': True, 'rotifer_av_native': True, 'rotifer_bio_native': True, ...}
```

### Why Three Layers Instead of Two?

The extra Python layer (between shell and Rust) exists because it handles three concerns that Rust cannot easily handle: argparse argument parsing, stdin/TTY detection, and exit code translation. If the shell called Rust directly, every CLI tool would need its own argument parser in Rust, TTY detection would break under PyO3 (see Bug #5), and error messages would lose the human-readable context that Python's `argparse` provides. The Python layer absorbs these concerns cleanly, so the Rust functions can be pure computation with simple signatures.

Crucially, each layer can fail independently. If the Rust crate fails to load, the Python layer still runs and returns exit code `2` to signal fallback. If the Python dispatch layer fails (e.g., a bad argument), it returns exit code `1` and the shell propagates the error directly. The original Python or Perl tool is only reached if the Rust path explicitly signals it cannot handle the request. This isolation means a broken Rust build never causes the tool to silently misbehave — it always falls back to a known-good implementation.

**Trade-off**: The Python dispatch layer adds approximately 5ms overhead per tool invocation — a Python subprocess startup cost. For tools processing megabytes of FASTA data, this 5ms is completely negligible. For tools called thousands of times in a tight shell loop (e.g., `for f in *.fa; do seqstats "$f"; done`), the overhead accumulates. In that case, prefer calling the Python API directly in a single batch invocation rather than looping through the shell wrapper.

### Decision Log

| Decision | Alternative considered | Why we chose this |
|----------|----------------------|-------------------|
| PyO3 0.22 | cffi, SWIG, ctypes | Type safety at the Rust/Python boundary, first-class GIL control, native numpy ndarray interop via the `numpy` crate, and active maintenance with a clear upgrade path |
| `cdylib` not `rlib` | Static linking | Python's import system uses `dlopen` to load extension modules at runtime; `cdylib` is the only crate type that produces a compatible shared library (`.so` / `.dylib`) |
| Cargo workspace | Separate repos | Shared dependency versions prevent version drift, atomic builds compile all crates together, and workspace-level release profiles apply uniformly |
| maturin not setuptools-rust | setuptools-rust | maturin handles `pyproject.toml` natively, provides `maturin develop` for fast iteration, and integrates cleanly with conda environments without manual `setup.py` maintenance |
| Feature flags | Separate crates per domain | One crate per domain (`bio`, `av`, `cli`) with internal feature gates reduces the number of `pip install` / `maturin develop` steps; features allow building subsets (e.g., `--features core`) without splitting the Python import namespace |

### What Gets Ported vs What Stays in Python

**Port**: CPU-bound loops over sequences, matrices, or large text. Anything that runs >10ms per call in Python and can benefit from compiled code.

**Keep in Python**: Network I/O (NCBI fetches), argparse CLIs, config/YAML loading, one-off scripts, anything already fast enough.

### When NOT to Port: A Decision Framework

The instinct to port everything to Rust is natural after seeing the first benchmarks. Resist it. The following categories of code cost more to port than they return:

- **I/O-bound code.** Network calls, database queries, file downloads, NCBI API fetches. The bottleneck is latency, not CPU. Rust will not make an HTTP request faster than Python — both wait the same number of milliseconds for the server to respond. Port the code that processes the results, not the code that fetches them.

- **Glue code.** Config parsing, CLI wrappers, argument validation, format conversion one-liners. These are already fast in Python and rarely CPU-bound. Porting them adds maintenance cost (two implementations to keep in sync) without measurable benefit.

- **Already-fast code.** If the Python version runs in under 10ms, the porting overhead — build time, ABI version coupling, two-language maintenance — is not justified by the speedup. Profile before you port.

- **Rapidly-changing code.** If the algorithm is still being iterated on, porting it to Rust freezes the iteration cycle. You will be fixing Rust borrow checker errors instead of exploring the algorithm space. Port code that has stabilized, not code that is still in flux.

- **Code with complex Python dependencies.** If a function calls pandas, scikit-learn, or matplotlib internally, porting it to Rust means reimplementing those libraries. It is almost never worth it. Port the tight inner loops that take data from these libraries, not the orchestration code that calls them.

**Rule of thumb**: if a function spends more than 80% of its wall-clock time in a tight loop over data (sequence processing, matrix fill, pattern matching, entropy calculation), it is a port candidate. If it spends more than 80% of its time in I/O, network calls, or calling other Python libraries, leave it in Python.

---

## 3. Build System and Toolchain

*Build once with maturin, optimize aggressively with LTO, and always use `--release` for anything you plan to measure. Debug builds are 10-50x slower and will make your benchmark numbers meaningless.*

### Prerequisites

- Rust stable (1.75+), via `rustup`
- `maturin` (`pip install maturin`)
- conda environment with Python 3.10+ and numpy

### Per-Crate File Structure

Each crate needs three files:

**`Cargo.toml`** — declares the crate as a `cdylib`:
```toml
[package]
name = "rotifer_bio_native"
version = "0.2.0"
edition.workspace = true

[lib]
name = "rotifer_bio_native"
crate-type = ["cdylib"]

[features]
default = ["full"]
core = []
ml-aligner = ["core"]
clustering = ["core"]
network = ["core"]
parsing = ["core"]
full = ["ml-aligner", "clustering", "network", "parsing"]

[dependencies]
pyo3 = { workspace = true }
numpy = { workspace = true }
ndarray = { workspace = true }
rayon = { workspace = true }
```

**`pyproject.toml`** — tells maturin how to build:
```toml
[build-system]
requires = ["maturin>=1.0,<2.0"]
build-backend = "maturin"

[tool.maturin]
features = ["pyo3/extension-module"]
```

**`build.rs`** — configures PyO3 at build time:
```rust
fn main() {
    pyo3_build_config::use_pyo3_cfgs();
}
```

### Build Workflow

```bash
cd rust/rotifer_bio_native
maturin develop --release
```

This compiles the crate and installs the `.so`/`.dylib` directly into the active conda environment's `site-packages`. Always use `--release` — debug builds are 10-50x slower and give misleading benchmark results.

### Release Profile

The workspace-level release profile in `rust/Cargo.toml` maximizes performance:

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

`lto = true` enables link-time optimization across all dependencies. `codegen-units = 1` forces single-threaded codegen for better inlining at the cost of slower compilation. This is the right trade-off for scientific computing extensions where runtime performance matters more than build time.

**Why `opt-level=3, lto=true, codegen-units=1`?** These extensions are built once per environment setup and run millions of times across analyses. A 5-minute build time is negligible compared to the cumulative runtime saved. LTO allows the compiler to inline across crate boundaries — for example, inlining `memchr` byte-search calls into the FASTA parser eliminates function call overhead in the innermost parsing loop. In benchmarks across the rotifer crates, LTO + codegen-units=1 delivered a consistent 1.5–2x speedup over the default profile on the Needleman-Wunsch DP fill and the entropy calculation inner loops.

**Trade-off**: Build time increases approximately 2–3x compared to the default 16 codegen units. For development cycles where you are iterating on logic (not tuning performance), omit the release profile overrides to get faster rebuilds: `maturin develop` (without `--release`) uses the default debug profile and rebuilds in seconds. Only switch to `--release` when you need accurate benchmark numbers or are preparing a production build.

### Feature Flags for Selective Compilation

The `rotifer_bio_native` crate uses Cargo features to allow building subsets:

```bash
maturin develop --release                    # full (default) — all 150 functions
maturin develop --release --features core    # core only — NW, encoding, scoring
```

Feature-gated modules in `lib.rs`:

```rust
#[cfg(feature = "ml-aligner")]
mod conservation;
#[cfg(feature = "ml-aligner")]
mod similarity;
#[cfg(feature = "ml-aligner")]
mod tree;

#[cfg(feature = "network")]
mod graph;

#[cfg(feature = "clustering")]
mod clustering;
```

Registration in the `#[pymodule]` function is also feature-gated:

```rust
#[cfg(feature = "clustering")]
{
    m.add_function(wrap_pyfunction!(clustering::hierarchical_cluster, m)?)?;
    m.add_function(wrap_pyfunction!(clustering::assign_clusters, m)?)?;
    // ...
}
```

This lets users build only what they need, reducing compile time and binary size.

---

## 4. Patterns That Worked

*Seven patterns recur across every crate: GIL release, safe singletons, fault-tolerant imports, native return types, exit-code fallback, crate caching, and TOCTOU-safe I/O. Master these seven and you will handle 95% of what the port throws at you.*

### 4.1 `py.allow_threads()` for GIL Release

The single most important pattern. Without it, Rust code holds the GIL and blocks all other Python threads. With it, CPU-bound Rust work runs truly parallel.

From `rotifer_bio_native/src/nw.rs:27-108` — the exact `nw_dp` function signature with all generic lifetimes and return types:

```rust
// rotifer_bio_native/src/nw.rs:26-109
#[pyfunction]
pub fn nw_dp<'py>(
    py: Python<'py>,
    score_matrix: PyReadonlyArray2<'py, f64>,
    gap_open: f64,
    gap_extend: f64,
) -> PyResult<(
    Bound<'py, PyArray2<f64>>,
    Bound<'py, PyArray2<f64>>,
    Bound<'py, PyArray2<f64>>,
    Bound<'py, PyArray2<i8>>,
)> {
```

The lifetime `'py` is a **PyO3 GIL lifetime** — it proves that Python objects can only exist while the GIL is held. The four return matrix types are: M matrix (`f64`), Ix matrix (`f64`), Iy matrix (`f64`), and traceback (`i8`, using 3 states: 0=diagonal, 1=up, 2=left). The `Bound<'py, PyArray2<T>>` return type replaces the older `PyArray2<T>` (PyO3 0.21) — it borrows the GIL token rather than requiring an unsafe conversion.

The full DP fill (nw.rs:44-101):

```rust
    let scores = score_matrix.as_array();   // borrow array data BEFORE allow_threads
    let n = scores.nrows();
    let m = scores.ncols();

    let mut mat_m = Array2::<f64>::from_elem((n + 1, m + 1), f64::NEG_INFINITY);
    let mut mat_ix = Array2::<f64>::from_elem((n + 1, m + 1), f64::NEG_INFINITY);
    let mut mat_iy = Array2::<f64>::from_elem((n + 1, m + 1), f64::NEG_INFINITY);
    let mut traceback = Array2::<i8>::zeros((n + 1, m + 1));
    // ... boundary initialization ...

    py.allow_threads(|| {        // GIL released here
        for i in 1..=n {
            for j in 1..=m {
                let s = scores[[i - 1, j - 1]];
                // ... fill M, Ix, Iy, traceback ...
            }
        }
    });                          // GIL re-acquired here

    Ok((
        mat_m.into_pyarray_bound(py),
        mat_ix.into_pyarray_bound(py),
        mat_iy.into_pyarray_bound(py),
        traceback.into_pyarray_bound(py),
    ))
```

**Key rule**: Copy data *out* of Python objects (`as_array()`) *before* `allow_threads`, then convert results back *after* with `into_pyarray_bound(py)`. You cannot touch `PyObject`s or call `PyReadonlyArray2::as_array()` inside `allow_threads` — the borrow checker enforces this at compile time. The `scores` ndarray borrow is valid inside `allow_threads` because it's a pure Rust borrow of a buffer already extracted from the Python object.

**What happens when this fails**: If you forget `allow_threads`, the function works correctly but holds the GIL for the entire DP fill. Other Python threads — including the shell's signal handler — are blocked for the duration. On large alignment matrices (thousands of sequences), this manifests as a hung terminal that does not respond to Ctrl-C. The user must kill the process from another terminal. This failure mode is invisible in unit tests (single-threaded) and only appears under load.

**See also**: The GIL release pattern here has a subtle consequence when the function is called from the shell dispatch layer: `std::io::stdin().is_terminal()` returns incorrect results inside PyO3-hosted Python. This is the root cause documented in Bug #5 — TTY detection fails under PyO3 because the stdin file descriptor is configured by the Python interpreter before Rust code runs. Read stdin in Python before calling any Rust function that needs to know if it's connected to a terminal.

**Also see**: Bug #8 (4.8 below) for the related constraint that Python code (including Python's `logging` module) must never be called from within an `allow_threads` block — doing so deadlocks because the worker thread cannot acquire the GIL while the main thread is blocked waiting for `allow_threads` to complete.

### 4.2 `OnceLock` for Thread-Safe Singletons

From `trouter_core/src/models.rs:85-113` — pre-built Aho-Corasick automaton for tier classification.

```rust
use std::sync::OnceLock;
use aho_corasick::AhoCorasick;

struct TierMatcher {
    ac: AhoCorasick,
    pattern_to_tier: Vec<usize>,
}

static MATCHER: OnceLock<TierMatcher> = OnceLock::new();

fn get_matcher() -> &'static TierMatcher {
    MATCHER.get_or_init(|| {
        let mut patterns = Vec::new();
        let mut pattern_to_tier = Vec::new();
        for (tier_idx, (_name, _model, keywords)) in TIER_DEFS.iter().enumerate() {
            for keyword in *keywords {
                patterns.push(*keyword);
                pattern_to_tier.push(tier_idx);
            }
        }
        let ac = match AhoCorasick::new(&patterns) {
            Ok(ac) => ac,
            Err(e) => {
                log::error!("Failed to build Aho-Corasick automaton: {}", e);
                AhoCorasick::new(Vec::<&str>::new()).unwrap()
            }
        };
        TierMatcher { ac, pattern_to_tier }
    })
}
```

The `TIER_DEFS` constant at `models.rs:13-75` defines all 7 classification tiers. Here are all tiers in order (highest to lowest priority):

| # | Tier name | Model ID | Keywords |
|---|-----------|----------|----------|
| 0 | `xhigh-fast` | `gpt-5.3-codex-xhigh-fast` | "emergency", "urgent critical", "security hotfix", "production emergency" |
| 1 | `xhigh` | `gpt-5.3-codex-xhigh` | "critical", "security", "audit", "production", "safety" |
| 2 | `high-fast` | `gpt-5.3-codex-high-fast` | "urgent complex", "fast quality", "time-sensitive", "deadline", "hotfix" |
| 3 | `high` | `gpt-5.3-codex-high` | "refactor", "multi-file", "complex", "architecture", "design", "algorithm", "optimize" |
| 4 | `fast` | `composer-1.5` | "quick", "tweak", "parameter", "adjust", "iterate", "inline" |
| 5 | `low-fast` | `composer-1.5` | "prototype", "experiment", "throwaway", "one-liner", "trivial", "scratch" |
| 6 | `low` | `composer-1.5` | "fix", "typo", "rename", "simple", "boilerplate", "template" |

Tier 0 (`xhigh-fast`) uses multi-word phrases as keywords ("urgent critical") — the Aho-Corasick automaton handles substring matching, so a task description containing "this is urgent critical" matches on the two-word phrase. If no keyword matches, `classify_tier` returns the fallback `("standard", "composer-1.5")`.

**Critical**: Use `match` on the `AhoCorasick::new()` result, not `.expect()`. If you use `.expect()` and initialization panics, the `OnceLock` is permanently poisoned — every subsequent call panics too. The `match` pattern falls back to an empty automaton that simply returns the default tier. See Bug #7 below.

**Why the fallback pattern exists**: `OnceLock` does not have a "retry on failure" mode. Once a panic occurs inside `get_or_init`, Rust's standard library marks the cell as poisoned and all future calls will panic immediately without even attempting initialization again. The fallback to an empty automaton is not just defensive coding — it is the only way to prevent a transient initialization error from permanently disabling the tier classifier for the lifetime of the process. In production, an empty automaton simply returns the default tier for every query, which is safe and visible in logs (the `log::error!` call records the failure for post-mortem analysis). See Bug #7 for the full diagnosis.

### 4.3 Fault-Tolerant Module Imports

From `wrappers/__init__.py`:

```python
for _name in _SUBMODULES:
    try:
        _loaded[_name] = importlib.import_module(f"wrappers.{_name}")
    except Exception as _exc:
        _logger.debug("wrappers.%s failed to import: %s", _name, _exc)
        _loaded[_name] = None
```

This catches *all* exceptions, not just `ImportError`. Critical because numpy version mismatches throw `RuntimeError`, not `ImportError`. Without this, a numpy problem in `operon_fold` would make `from wrappers import bio_native` fail even though `bio_native` works fine.

**Why `Exception` and not `ImportError`?** The full exception hierarchy for C extension failures includes: `ImportError` (module not found, or explicit `raise ImportError`), `RuntimeError` (numpy ABI mismatch — the most common case: "module compiled against API version 0x10 but this version of numpy is 0xe"), `SystemError` (CPython internal error during initialization), `AttributeError` (module loaded but a required attribute is missing — common when a `.so` file is present but was compiled against a different version of the C extension API), and `OSError` (the `.so` file exists but cannot be loaded by the dynamic linker — typically a missing shared library dependency). Catching `ImportError` only would miss the last four categories. The catch-all `except Exception` is intentional, and the `_logger.debug(...)` call ensures the failure is visible in logs without disrupting the user.

**What happens when this fails**: If you catch only `ImportError`, a numpy ABI mismatch in `operon_fold` propagates upward and kills all subsequent submodule imports in the loop. The user sees `from wrappers import bio_native` fail even though `bio_native` has no numpy dependency — exactly as described in Bug #1. The fault-tolerant pattern prevents one broken submodule from cascading into a complete package failure.

### 4.4 Return Type Contract

Rust functions always return Python-native types: `str`, `int`, `float`, `bool`, `list`, `dict`, `tuple`, or numpy arrays. Never return custom PyO3 classes. This keeps the Python fallback path trivial — both Rust and Python return the same types.

**Why no custom PyO3 classes?** This is a serialization boundary decision. If a Rust function returns a `#[pyclass] struct AlignmentResult`, then the Python fallback path must also produce an `AlignmentResult` instance. That requires either duplicating the class definition in Python or importing it from the Rust extension — which defeats the purpose of a fallback that works when the Rust crate is unavailable. Python-native types have no such coupling: a dict returned by Rust and a dict returned by Python are identical from the caller's perspective. The fallback path is always a thin Python function that returns the same built-in type, with zero dependency on the Rust crate being present.

### 4.5 Shell Dispatch with Exit-Code Fallback

The shell layer wraps each tool:

```bash
seqstats() {
    if _rust_available "rotifer_cli_native"; then
        _rust_dispatch "seqstats" "$@"
        local rc=$?
        if [[ $rc -ne 2 ]]; then return $rc; fi
    fi
    # Fall through to original Python implementation
    "${ROTIFER_ROOT}/bin/seqstats" "$@"
}
```

Exit code `2` means "I don't handle this" — maybe the tool got unsupported flags, or the Rust implementation is partial. The original Python tool runs as if Rust didn't exist.

**What happens when this fails**: If a dispatch handler forgets to return `2` for unsupported features or flags, the shell layer receives exit code `0` or `1` and never falls through to the Python tool. The user gets either a silent no-op (exit code 0 with no output) or a cryptic Rust-level error message, instead of the working Python implementation. This is especially painful for flags that are common in scripts but not yet implemented in Rust — the tool breaks silently in production pipelines. Always return `2` as the default from any dispatch handler branch that doesn't fully handle the request.

### 4.6 Crate Availability Caching

Checking `python3 -c "import rotifer_cli_native"` on every tool invocation adds ~200ms overhead. The shell layer caches results per session:

```bash
declare -A _RUST_CRATE_CACHE 2>/dev/null || true

_rust_available() {
    local crate="$1"
    if [[ "${ROTIFER_RUST:-1}" == "0" ]]; then return 1; fi
    if [[ -n "${_RUST_CRATE_CACHE[$crate]+x}" ]]; then
        return "${_RUST_CRATE_CACHE[$crate]}"
    fi
    if PYTHONPATH="${_RST_PYTHONPATH}:${PYTHONPATH:-}" \
       "$_RST_PYTHON" -c "import $crate" 2>/dev/null; then
        _RUST_CRATE_CACHE[$crate]=0
        return 0
    else
        _RUST_CRATE_CACHE[$crate]=1
        return 1
    fi
}
```

First call per crate pays the import cost; all subsequent calls are instant associative-array lookups.

**Trade-off**: The cache lives in shell memory and is lost when the terminal session ends or the shell is restarted. Every new terminal session pays the ~200ms import probe cost for the first invocation of each crate. After a `maturin develop --release` rebuild, call `_rust_cache_clear` to force re-probing — otherwise the shell may cache a stale "available" result that reflects the old binary, though in practice this only matters if you rebuild a crate that was previously unavailable.

### 4.7 TOCTOU-Safe File Reading

From `rotifer_av_native/src/scanner.rs`:

```rust
fn read_file_contents_with_limit(path: &str, max_bytes: u64) -> std::io::Result<Vec<u8>> {
    // Pre-flight: reject symlinks without following them
    let input_meta = fs::symlink_metadata(Path::new(path))?;
    if input_meta.file_type().is_symlink() {
        return Err(Error::new(ErrorKind::InvalidInput, "Symlink inputs are not allowed"));
    }

    // Open the file descriptor first
    let mut file = fs::File::open(Path::new(path))?;

    // Check metadata on the open fd (fstat) — immune to TOCTOU swaps
    let fd_meta = file.metadata()?;
    if !fd_meta.is_file() {
        return Err(Error::new(ErrorKind::InvalidInput, "Input is not a regular file"));
    }
    if fd_meta.len() > max_bytes {
        return Err(Error::new(ErrorKind::InvalidData,
            format!("File exceeds maximum scan size ({} > {} bytes)", fd_meta.len(), max_bytes)));
    }

    let mut content = Vec::with_capacity(fd_meta.len() as usize);
    file.read_to_end(&mut content)?;
    Ok(content)
}
```

The key insight: call `symlink_metadata` (lstat) to reject symlinks, then open the fd, then call `file.metadata()` (fstat) on the *open fd*. An attacker who swaps the file between the symlink check and the open still can't bypass the fstat check, because fstat operates on the already-opened file descriptor.

**See also**: Bug #8 documents the specific TOCTOU vulnerability this pattern fixes, including the exact attack scenario. For the performance cost of this two-syscall pattern (lstat + fstat vs. a single stat): the overhead is two filesystem metadata calls per file. On Linux, `lstat` and `fstat` are typically under 1 microsecond each on warm inodes. For the AV scanner processing thousands of files in a batch, this is negligible compared to the cost of reading file contents. See Section 9 (Performance) for the batch parallelism pattern that amortizes this overhead across many files via rayon.

The three `ErrorKind` variants returned by `read_file_contents_with_limit` (`scanner.rs:250-285`) are:
- `ErrorKind::InvalidInput` — path is a symlink (`symlink_metadata` detects via `is_symlink()`)
- `ErrorKind::InvalidInput` — fd resolves to a non-regular file (e.g., a device, pipe, or directory after open)
- `ErrorKind::InvalidData` — file exceeds `max_bytes` (`DEFAULT_MAX_SCAN_FILE_BYTES = 100 * 1024 * 1024`, i.e. 100 MB)

The `ErrorKind::NotFound` case is handled one level up in `scan_file` (`scanner.rs:52-54`): a file that disappears between listing and scanning is silently skipped rather than propagated as an error.

### 4.8 Batch Parallelism with rayon

The `scan_files_batch` function in `rotifer_av_native/src/scanner.rs:144-212` demonstrates the full GIL-release + rayon pattern. The complete flow:

```rust
// scanner.rs:143-212
#[pyfunction]
#[pyo3(signature = (paths, patterns=None, entropy_threshold=7.0))]
pub fn scan_files_batch(
    py: Python<'_>,
    paths: Vec<String>,
    patterns: Option<Vec<Vec<u8>>>,
    entropy_threshold: f64,
) -> PyResult<Vec<PyObject>> {
    // Step 1: release GIL — rayon threads cannot hold it
    let all_threats: Vec<ThreatInfo> = py.allow_threads(|| {
        // Step 2: par_iter distributes paths across rayon thread pool
        paths
            .par_iter()
            .flat_map(|path| {
                // Step 3: read file in worker thread (no GIL)
                let content = match read_file_contents(path) {
                    Ok(c) => c,
                    Err(_) => return vec![],  // skip unreadable files
                };
                let mut threats = Vec::new();

                // Step 4: entropy analysis (pure math, no Python)
                let entropy = shannon_entropy(&content);
                if entropy > entropy_threshold {
                    threats.push(ThreatInfo { ... });
                }

                // Step 5: Aho-Corasick pattern scan
                if let Some(ref pats) = patterns {
                    if let Ok(ac) = aho_corasick::AhoCorasick::new(pats) {
                        for mat in ac.find_iter(&content) { ... }
                    }
                }
                threats
            })
            .collect()
    });
    // Step 6: GIL re-acquired; convert ThreatInfo structs to Python dicts
    all_threats.iter().map(|t| {
        let dict = PyDict::new_bound(py);
        dict.set_item("threat_name", &t.threat_name)?;
        // ...
        Ok(dict.into())
    }).collect()
}
```

**Why `flat_map` not `map`**: Each file produces a `Vec<ThreatInfo>` (zero or more threats). `flat_map` flattens the `Vec<Vec<ThreatInfo>>` into a single `Vec<ThreatInfo>`. This is idiomatic rayon for one-to-many parallel transforms.

**Why the GIL release wraps the entire rayon call**: rayon's thread pool is spawned outside of PyO3's GIL awareness. If any rayon worker thread tried to acquire the GIL (e.g., by calling Python code), it would deadlock — the main thread holds the GIL and is blocked waiting for `allow_threads` to return. The solution is to ensure all work inside `allow_threads` is pure Rust with no Python calls.

**What happens when this fails**: If you call Python code inside `allow_threads` — for example, calling Python's `logging` module to log a scan result, or calling any function that touches a `PyObject` — the program deadlocks. The worker thread blocks waiting to acquire the GIL; the main thread blocks inside `allow_threads` waiting for the worker thread to finish. There is no timeout and no error message — the process simply hangs. The Rust borrow checker cannot catch this at compile time because the `allow_threads` closure accepts any `FnOnce()`. The only guard is code review: audit every code path inside `allow_threads` to ensure it contains no Python calls, no `log` calls that route through Python, and no accesses to `PyObject` or `Py<T>` types.

---

## 5. Bugs and How They Were Fixed

*Twelve bugs, four root-cause categories: import isolation, shell compatibility, stdin/TTY handling, and API contracts. Most are boundary issues, not Rust issues. The Rust code was rarely the problem.*

This section documents every significant bug encountered during the port. Each entry follows the format: **Symptom** → **Root Cause** → **Fix** → **Lesson**.

### Bug 1: Numpy Cascade Failure

**File**: `rust/wrappers/__init__.py` — the `_SUBMODULES` import loop.

**Symptom**: `from wrappers import bio_native` fails with `RuntimeError` even though `rotifer_bio_native` is correctly built and importable on its own.

**Root Cause**: The `operon_fold` submodule depends on numpy. When numpy has a version mismatch or ABI incompatibility (common after conda environment updates), it throws `RuntimeError`, not `ImportError`. The specific exception message seen during debugging was: `RuntimeError: module compiled against API version 0x10 but this version of numpy is 0xe` (the version numbers vary, but the message pattern is "module compiled against API version"). The original `__init__.py` used a bare `import` inside a loop, and the exception propagated upward, killing the entire `wrappers` package. This meant `from wrappers import bio_native` failed even though `bio_native` had no numpy dependency.

**Fix**: Wrap each submodule import in a `try/except Exception` (not just `ImportError`), and set the failed module to `None`:

```python
try:
    _loaded[_name] = importlib.import_module(f"wrappers.{_name}")
except Exception as _exc:
    _logger.debug("wrappers.%s failed to import: %s", _name, _exc)
    _loaded[_name] = None
```

**Lesson**: In a multi-crate wrapper package, always catch `Exception`, never just `ImportError`. Numpy and other C extensions throw `RuntimeError`, `SystemError`, or even `AttributeError` on ABI mismatches.

### Bug 2: zsh `BASH_SOURCE[0]` Incompatibility

**File**: `rust/rust_functions.sh` lines 13-20.

**Symptom**: Sourcing `rust_functions.sh` in zsh fails silently — `_RST_ROOT` is empty, and all subsequent Python calls fail because `PYTHONPATH` is wrong.

**Root Cause**: zsh does not set `BASH_SOURCE`. The script was using `${BASH_SOURCE[0]}` to locate itself, which is undefined in zsh.

**Fix**: Dual-shell path detection at the top of `rust_functions.sh` (lines 14-20):

```bash
if [[ -n "${ZSH_VERSION:-}" ]]; then
    _RST_ROOT="${${(%):-%x}:A:h}"
elif [[ -n "${BASH_SOURCE[0]:-}" ]]; then
    _RST_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
else
    _RST_ROOT="$(cd "$(dirname "$0")" && pwd)"
fi
```

Token-by-token breakdown of the zsh idiom `${${(%):-%x}:A:h}`:
- `%x` — zsh prompt escape meaning "current script filename" (equivalent to `$0` when sourced)
- `(%)` — prompt expansion flag: tells zsh to interpret `%x` as a prompt escape
- `${...:-}` — standard parameter expansion with empty default (ensures the expansion doesn't fail if the flag returns empty)
- `:A` — zsh path modifier: resolve to absolute canonical path (follows symlinks, equivalent to `realpath`)
- `:h` — zsh path modifier: take the "head" (directory part), equivalent to `dirname`

So `${${(%):-%x}:A:h}` means: "get the current script path, make it absolute and canonical, then take the directory part."

**Lesson**: If your shell layer must work in both bash and zsh, test path detection explicitly in both. The `${ZSH_VERSION:-}` guard must come first.

### Bug 3: `export -f` Dumps Function Bodies in zsh

**Symptom**: Sourcing `rust_functions.sh` in zsh produces hundreds of lines of garbage output — the full bodies of every exported function are printed to the terminal.

**Root Cause**: bash's `export -f func_name` exports a function to subshells. zsh does not support `export -f` and instead interprets it as an attempt to export a variable, printing the function body.

**Fix**: Guard every `export -f` block:

```bash
if [[ -z "${ZSH_VERSION:-}" ]]; then
    export -f seqstats rcomplement nt2aa protein2dna
    export -f blast2table hmmer2table hhsuite2table genbank2table
    # ...
fi
```

**Lesson**: `export -f` is a bash-only feature. Always guard it with a zsh check. In zsh, functions defined in the current shell are automatically available to subshells run with the same shell.

### Bug 4: Stdin Hang When Called Without Arguments

**File**: `rust/rotifer_cli_native/src/aln_view_cli.rs` — stdin input branch.

**Symptom**: Typing `aln_view` at the shell (without any file argument) causes the terminal to hang indefinitely with no output.

**Root Cause**: The Rust CLI tool reads from stdin when no file is provided. When stdin is connected to a terminal (not piped), `stdin.read_to_string()` blocks forever waiting for input that will never come.

**Fix**: Check `is_terminal()` before reading stdin. The full import path is `std::io::IsTerminal` (introduced in Rust 1.70 / stable since 2023). This is a trait method, not a free function:

```rust
// rotifer_cli_native/src/aln_view_cli.rs
None => {
    use std::io::IsTerminal;          // trait from std::io, stable since 1.70
    if std::io::stdin().is_terminal() {
        print_help();
        return Ok(0);
    }
    let mut buf = String::new();
    std::io::stdin().read_to_string(&mut buf)?;
    buf
}
```

`IsTerminal` is implemented for `std::io::Stdin`, `std::io::Stdout`, `std::io::Stderr`, and their locked variants. On Unix it calls `isatty(fd)`. On Windows it calls `GetConsoleMode`.

**Lesson**: Any CLI tool that accepts stdin must check `is_terminal()` first. Show help or usage instead of blocking.

**See also**: Bug #5 is the deeper manifestation of the same underlying issue. Even after adding `is_terminal()` to the Rust code, it returns the wrong answer when the tool is invoked through the PyO3 dispatch layer, because Python's stdin setup changes the file descriptor state. Bug #4 (the hang) and Bug #5 (the wrong TTY result) are two symptoms of the same root cause: stdin handling must never be done in Rust when the tool is invoked through the Python dispatch chain. See the fix in Bug #5 for the definitive solution.

### Bug 5: Rust-Side TTY Detection Fails Under PyO3

**Symptom**: `std::io::stdin().is_terminal()` returns `false` even when the tool is run interactively from a terminal, because it is invoked via `python3 -c "..."` from the shell dispatch layer.

**Root Cause**: When the shell runs `python3 -c "from wrappers.cli_native import dispatch_tool; ..."`, Python's own stdin handling may have already consumed or redirected the file descriptor. The Rust code sees Python's stdin, not the user's terminal.

**Fix**: Perform TTY detection in the Python dispatch layer *before* calling Rust, and pass the result as an argument:

```python
# wrappers/cli_native.py
def _dispatch_tfilter(args):
    # ...
    if ns.input == "-":
        if sys.stdin.isatty():
            p.print_help()
            return 0
        data = sys.stdin.read()
```

The Python-side `sys.stdin.isatty()` correctly reflects the user's terminal. Read stdin in Python, pass the data as a string to Rust.

**Lesson**: Don't read stdin in Rust when called via PyO3. Do all stdin I/O in the Python dispatch layer and pass data as function arguments.

**See also**: Section 4.1 (`allow_threads`) explains why the GIL release pattern makes this worse — `allow_threads` only affects the GIL state, not the stdin file descriptor configuration. Python's stdin setup happens before `allow_threads` is entered, so even with GIL-released Rust code, `std::io::stdin().is_terminal()` still sees Python's view of the file descriptor. Section 6 (Shell Dispatch Layer) shows the `python3 -c "..."` invocation pattern that is the underlying cause: the shell spawns a Python subprocess which sets up its own stdin buffering before any Rust code runs. The correct fix — performing all TTY detection in Python before calling Rust — is shown in the dispatch handler pattern in `wrappers/cli_native.py`.

### Bug 6: cdylib Crates Cannot `cargo test`

**File**: Any crate with `crate-type = ["cdylib"]` in its `Cargo.toml`.

**Symptom**: Running `cargo test` in a crate produces linker errors about duplicate Python symbols, or the test binary crashes on startup. The exact linker error pattern on macOS is: `ld: symbol(s) not found for architecture arm64: _PyArg_ParseTupleAndKeywords` or `duplicate symbol '_Py_InitModule4'` depending on how the linker resolves the Python dynamic library. On Linux the error pattern is: `undefined reference to 'PyModule_Create2'` or `multiple definition of 'PyInit_<crate>'`.

**Root Cause**: cdylib crates compile to shared libraries, not executables. The test harness tries to build an executable that links against both PyO3 and the system Python, causing symbol conflicts.

**Fix**: Use pytest instead. Write `#[cfg(test)]` unit tests for pure-Rust logic only (no PyO3 types), and test all PyO3-facing functions through pytest:

```python
# rust/tests/test_transparent_dispatch.py
class TestFunctionalParity:
    def test_seqstats(self, bio_native, sample_fasta):
        if not bio_native.NATIVE_AVAILABLE:
            pytest.skip("Native crate not built")
        result = bio_native.seqstats(sample_fasta)
        assert isinstance(result, str)
        assert "seq1" in result
```

For pure-Rust unit tests (no PyO3 interaction), they work fine within `#[cfg(test)]` modules:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_nw_simple_match() {
        // Pure ndarray logic — no PyO3 types
        let scores = ndarray::array![[1.0, -1.0], [-1.0, 1.0]];
        // ...
        assert!((final_score - 2.0).abs() < 1e-10);
    }
}
```

**Lesson**: For cdylib crates, `cargo test` works only for pure-Rust code with no PyO3 types. All Python-facing tests must go through pytest.

### Bug 7: OnceLock Poisoning via `.expect()`

**Symptom**: After a one-time initialization failure (e.g., invalid regex in Aho-Corasick patterns), every subsequent call to `classify_tier()` panics with "OnceLock already initialized" or similar.

**Root Cause**: Using `.expect("message")` on the `AhoCorasick::new()` result inside `OnceLock::get_or_init()`. If the initialization closure panics, the `OnceLock` is permanently poisoned — `get_or_init` will panic on every subsequent call for the lifetime of the process.

**Fix**: Use `match` instead of `.expect()`, and fall back to a safe default:

```rust
let ac = match AhoCorasick::new(&patterns) {
    Ok(ac) => ac,
    Err(e) => {
        log::error!("Failed to build Aho-Corasick automaton: {}", e);
        AhoCorasick::new(Vec::<&str>::new()).unwrap()
    }
};
```

**Lesson**: Never panic inside `OnceLock::get_or_init()`. A panic poisons the lock permanently. Always handle errors with `match` and provide a safe fallback.

### Bug 8: TOCTOU in File Scanning

**File**: `rust/rotifer_av_native/src/scanner.rs:250-285` — `read_file_contents_with_limit`.

**Symptom**: Security scanner reports false negatives — a malicious file passes the scan despite matching known patterns.

**Root Cause**: The original code checked file metadata (size, type, symlink status) using `fs::metadata()`, then opened the file in a separate call. Between the check and the open, an attacker could swap the file (time-of-check to time-of-use race).

**The `symlink_metadata` vs `metadata` distinction**: `fs::metadata(path)` calls `stat(2)` on Unix — it follows symlinks and returns information about the target. `fs::symlink_metadata(path)` calls `lstat(2)` — it does NOT follow symlinks and returns information about the symlink itself. To detect symlinks you MUST use `symlink_metadata`. If you use `metadata` to check `is_symlink()`, it always returns `false` because `stat(2)` has already followed the symlink.

**Fix**: Open the file descriptor first, then inspect metadata on the open fd via `file.metadata()` (which calls `fstat(2)` on Unix):

```rust
let mut file = fs::File::open(Path::new(path))?;
let fd_meta = file.metadata()?;  // fstat on open fd — immune to swaps
```

`File::metadata()` (fstat) operates on the already-open file descriptor, not the path. Any path swap after `File::open()` does not affect the fd — it still points to the original inode. The symlink check using `symlink_metadata` happens before `open()` to prevent following a symlink to a sensitive file; the fstat check after open confirms the file type hasn't changed unexpectedly.

See the full implementation in Section 4.7 above.

**Lesson**: For security-sensitive file operations, always operate on open file descriptors, not paths. Path-based checks are inherently racy. Specifically: use `symlink_metadata` (lstat) to detect symlinks, then `File::metadata()` (fstat) on the open fd to verify file properties.

### Bug 9: `from rust.wrappers` Import Path Coupling

**Symptom**: Tests pass when run from the `rust/` directory but fail from the repository root, with `ModuleNotFoundError: No module named 'rust.wrappers'`.

**Root Cause**: Some imports used `from rust.wrappers import bio_native` (treating `rust/` as a package), while others used `from wrappers import bio_native` (treating `rust/` as a PYTHONPATH entry). The two are not equivalent and break depending on the working directory.

**Fix**: Standardize on `from wrappers import ...` everywhere, and ensure `PYTHONPATH` includes the `rust/` directory:

```bash
PYTHONPATH="${_RST_PYTHONPATH}:${PYTHONPATH:-}" "$_RST_PYTHON" -c "$code" "$@"
```

**Lesson**: Pick one import convention and enforce it everywhere. Use `PYTHONPATH` manipulation in the shell layer, never rely on relative imports from the working directory.

### Bug 10: Stale `__pycache__` After Module Renames

**Symptom**: After renaming a wrapper module (e.g., `bio.py` → `bio_native.py`), Python still imports the old module from `__pycache__/bio.cpython-310.pyc`.

**Root Cause**: Python's import system caches compiled bytecode in `__pycache__/`. After a rename, the old `.pyc` file still exists and may be found first.

**Fix**: Delete `__pycache__` directories after any module rename:

```bash
find rust/wrappers -name __pycache__ -type d -exec rm -rf {} +
```

**Lesson**: Module renames require `__pycache__` cleanup. Consider adding `find ... -name __pycache__ -exec rm -rf {} +` to your build script.

### Bug 11: Missing Conda Environment Mismatch

**File**: All crates — build-time vs. runtime Python ABI mismatch.

**Symptom**: `maturin develop --release` succeeds but `import rotifer_bio_native` fails with `ImportError: ... symbol not found`. The exact message pattern on macOS arm64 is: `ImportError: dlopen(.../rotifer_bio_native.cpython-310-darwin.so, 0x0002): symbol not found in flat namespace '__PyArg_ParseTupleAndKeywords_SizeT'`. On Linux the pattern is: `ImportError: .../rotifer_bio_native.cpython-310-linux-aarch64.so: undefined symbol: PyFloat_AsDouble`.

**Root Cause**: The Rust extension was built against one Python (e.g., system Python 3.12) but the user is running in a conda environment with Python 3.10. The ABI is incompatible.

**Fix**: Always activate the target conda environment *before* running maturin:

```bash
conda activate rotifer
cd rust/rotifer_bio_native
maturin develop --release
```

Maturin auto-detects the active Python and links against it. The shell layer uses `RST_PYTHON` to ensure the same interpreter is used at runtime:

```bash
_RST_PYTHON="${RST_PYTHON:-python3}"
```

**Lesson**: PyO3 extensions are ABI-specific. Build and run must use the same Python. Use `RST_PYTHON` to pin the interpreter.

### Bug 12: seqstats Return Type Mismatch

**Symptom**: The Rust `seqstats` function returns a tuple of integers, but the Python wrapper expects a formatted TSV string. The shell dispatch layer receives a Python tuple and crashes trying to write it to stdout.

**Root Cause**: The Rust function was ported to return native types (tuple of counts and averages) for efficiency, but the CLI dispatch layer expected a pre-formatted string matching the original Python tool's output format.

**Fix**: Have the Rust function return a formatted string directly, matching the original tool's output contract:

```rust
#[pyfunction]
pub fn seqstats(fasta_text: &str) -> PyResult<String> {
    // ... compute stats ...
    Ok(format!("{}\t{}\t{}\t{:.1}\t{}\t{}", n_seqs, total_len, min_len, avg_len, max_len, n50))
}
```

**Lesson**: When porting CLI tools, match the original output format exactly — including column order, delimiter, and header. The dispatch layer should be a thin pass-through, not a format translator.

### Bug Taxonomy: Where Things Break

After 12 bugs across 5 crates, a pattern is clear. Nearly every hard bug was a boundary issue, not a Rust issue. The following taxonomy generalizes to any Python-to-Rust port:

| Category | Bugs | Generalizable Lesson |
|----------|------|----------------------|
| Import / module system | #1 (numpy cascade), #9 (path coupling), #10 (`__pycache__`), #11 (conda ABI) | Python's import system is the most fragile layer. Every C extension is an ABI contract that can break silently on environment updates. Isolate every submodule, cache nothing in `__pycache__` that can go stale, and always build against the same interpreter you run against. |
| Shell compatibility | #2 (`BASH_SOURCE`), #3 (`export -f`) | Bash and zsh are not the same language despite sharing syntax. Test in both from day one. The `ZSH_VERSION` guard and the `${(%):-%x}` path idiom are not optional for dual-shell support — they are load-bearing. |
| Stdin / TTY | #4 (stdin hang), #5 (TTY under PyO3) | Python owns the file descriptors when Rust is invoked through the dispatch chain. Never let Rust read stdin directly; never trust `std::io::stdin().is_terminal()` inside PyO3. Do all TTY detection and stdin reading in Python, then pass data as function arguments. |
| Build system | #6 (cdylib `cargo test`), #11 (conda ABI mismatch) | `cargo test` and `maturin` have sharp edges with `cdylib` crates. `cargo test` cannot link cdylib + executable; `maturin develop` must be run inside the same conda environment that will run the code at runtime. Pytest is the correct test harness for PyO3 extensions. |
| Concurrency / initialization | #7 (OnceLock poisoning) | Static initialization must never panic. `OnceLock` poisons permanently on panic — there is no retry, no recovery. The empty-automaton fallback is not just defensive coding; it is the only mechanism that keeps the process functional after a transient initialization error. |
| Security / filesystem | #8 (TOCTOU) | Path-based checks are inherently racy. Any code path that checks a file's properties and then opens it has a TOCTOU window. For security-sensitive code, always open the file descriptor first and then call `fstat` on the open fd. |
| API contract | #12 (return type mismatch) | The exit-code contract and the output format contract are both APIs. Breaking either one breaks downstream scripts silently. Treat output format as a first-class API contract from the beginning. |

**The meta-lesson**: if you are debugging a Python-to-Rust port and the bug seems mysterious, check the category list above. Ask: is this an import isolation issue? A shell compatibility issue? A stdin/TTY issue? A build system issue? In the rotifer port, every bug fit one of these seven categories. The categories are not rotifer-specific — they are structural properties of the PyO3 embedding architecture.

---

## 6. The Shell Dispatch Layer

*2908 lines of bash+zsh-compatible dispatch that makes Rust acceleration invisible to users. The shell layer is the most user-facing part of the port — a bug here breaks every tool simultaneously, and the error messages are the hardest to interpret.*

### Architecture of `rust_functions.sh`

The file is exactly **2908 lines** (verified with `wc -l`). It is structured in five major zones:

1. **Setup** (lines 1-30): Double-source guard (`_RST_LOADED`), zsh/bash path detection, `_RST_ROOT`/`_RST_PYTHONPATH`/`_RST_PYTHON` initialization, `RST_DEBUG` and `ROTIFER_RUST` defaults
2. **Internal helpers** (lines 31-147): `_rst_debug`, `_rst_time_ms`, `_rst_exec`, `_rust_available`, `_rust_dispatch`, `_rust_has_tool`, `_rust_cache_clear`
3. **Named `rst_*` tool wrappers** (lines 149-2237): Individual functions grouped by category (see section breakdown below)
4. **Status/build/help utilities** (lines 2238-2417): `rst_status`, `rst_build`, `rst_help`
5. **Transparent dispatch wrappers** (lines 2419-2908): `export -f` blocks, transparent wrappers for original command names, load confirmation message

**Section breakdown with line ranges**:

| Section | Lines | Functions | Description |
|---------|-------|-----------|-------------|
| FASTA & SEQUENCE I/O | 149-390 | 7 | `rst_parse_fasta`, `rst_parse_fasta_mmap`, `rst_fasta_to_table`, `rst_table_to_fasta`, `rst_aln_to_fasta`, `rst_fasta_to_aln`, `rst_consensus` |
| ARCHITECTURE & DOMAINS | 391-535 | 4 | `rst_parse_architecture`, `rst_count_domains`, `rst_tokenize_architecture`, `rst_cluster_architectures` |
| ALIGNMENT & SIMILARITY | 536-676 | 4 | `rst_nw_align`, `rst_needleman_wunsch`, `rst_levenshtein`, `rst_compositional_similarity` |
| SIMILARITY MATRICES & REDUNDANCY | 677-947 | 6 | `rst_similarity_matrix`, `rst_pairwise_scores`, `rst_jaccard_matrix`, `rst_remove_redundancy`, `rst_cluster_identity`, `rst_coarse_cluster` |
| CONSERVATION & MSA | 948-1178 | 8 | `rst_entropy`, `rst_conservation`, `rst_pairwise_identity`, `rst_column_entropy`, `rst_relative_entropy`, `rst_jensen_shannon`, `rst_bit_score`, `rst_property_entropy` |
| NETWORK & NEIGHBORHOODS | 1179-1344 | 5 | `rst_build_network`, `rst_connected_components`, `rst_parse_neigh`, `rst_detect_swaps`, `rst_bfs_neighbors` |
| TREE & PHYLOGENETICS | 1345-1471 | 4 | `rst_parse_newick`, `rst_robinson_foulds`, `rst_tree_agreement`, `rst_pairwise_identity_matrix` |
| AV SCANNING & SECURITY | 1472-1840 | 13 | `rst_scan_entropy`, `rst_parse_macho`, `rst_scan_file`, `rst_scan_binary`, `rst_hash_file`, `rst_scan_api_keys`, `rst_quarantine`, `rst_verify_quarantine`, `rst_compile_definitions`, `rst_classify_ciphers`, `rst_is_weak_protocol`, `rst_tls_check`, `rst_fim_baseline` |
| PARSING | 1841-1924 | 3 | `rst_parse_tsv`, `rst_parse_tblout`, `rst_parse_domtblout` |
| TROUTER & UTILITIES | 1925-2116 | 6 | `rst_classify_tier`, `rst_map_model`, `rst_sanitize_prompt`, `rst_parse_config`, `rst_resolve_binary`, `rst_infer_health` |
| ENCODING | 2117-2173 | 2 | `rst_encode_sequence`, `rst_encode_batch` |
| CLUSTERING METRICS | 2174-2237 | 2 | `rst_ari_score`, `rst_nmi_score` |
| STATUS/BUILD/HELP | 2238-2417 | 3 | `rst_status`, `rst_build`, `rst_help` |
| TRANSPARENT DISPATCH | 2419-2908 | 63+ | Original command name wrappers + export blocks + load confirmation |

### Core Dispatch Functions

**`_rst_exec`** — the lowest-level executor. Sets `PYTHONPATH` and runs Python code:

```bash
_rst_exec() {
    local func_name="$1"; shift
    local code="$1"; shift
    PYTHONPATH="${_RST_PYTHONPATH}:${PYTHONPATH:-}" "$_RST_PYTHON" -c "$code" "$@"
}
```

When `RST_DEBUG=1`, it also measures and reports execution time in milliseconds using the `_rst_time_ms` helper.

**Why `python3 -c` instead of a Python script file?** Three reasons: First, it avoids `.pyc` cache issues — when a script file is run, Python writes a cached bytecode file to `__pycache__/` alongside the script. After renames or rebuilds, stale `.pyc` files can shadow the updated module (see Bug #10). A `-c` invocation has no associated file path and generates no `.pyc` cache. Second, it works regardless of `PYTHONPATH` state — the `_rst_exec` helper sets `PYTHONPATH` as part of the invocation, so the import resolves correctly even if the user's shell environment has a conflicting or missing `PYTHONPATH`. A script file run as `python3 script.py` would depend on the working directory and `PYTHONPATH` already being correct. Third, it is a single atomic invocation — the shell generates the entire dispatch call as a string and passes it to one Python process, with no intermediate files that could fail to be found or have wrong permissions.

### The `_rst_time_ms` Debug Helper

`_rst_time_ms` (lines 40-42) is a one-liner that returns the current time in milliseconds since epoch:

```bash
_rst_time_ms() {
    "$_RST_PYTHON" -c "import time; print(int(time.time()*1000))"
}
```

It is called twice in `_rst_exec` (lines 50 and 54) when `RST_DEBUG=1` — before and after the Python invocation — and the elapsed milliseconds are reported to stderr: `[RST_DEBUG] <func_name>: <elapsed>ms (rc=<exit_code>)`. This adds ~50ms overhead per call (due to launching Python twice) and is explicitly only enabled via `export RST_DEBUG=1`.

**`_rust_available`** — checks if a crate is importable, with per-session caching and a global kill switch (`ROTIFER_RUST=0`):

```bash
_rust_available() {
    local crate="$1"
    if [[ "${ROTIFER_RUST:-1}" == "0" ]]; then return 1; fi
    if [[ -n "${_RUST_CRATE_CACHE[$crate]+x}" ]]; then
        return "${_RUST_CRATE_CACHE[$crate]}"
    fi
    # ... probe and cache result ...
}
```

**`_rust_dispatch`** — routes a tool name through the Python `TOOL_REGISTRY`:

```bash
_rust_dispatch() {
    local tool="$1"; shift
    _rst_exec "$tool" "
import sys
from wrappers.cli_native import dispatch_tool
sys.exit(dispatch_tool('$tool', sys.argv[1:]))
" "$@"
}
```

### Transparent Wrapper Pattern

Each tool wrapper follows the same pattern:

```bash
seqstats() {
    if _rust_available "rotifer_cli_native"; then
        _rust_dispatch "seqstats" "$@"
        local rc=$?
        if [[ $rc -ne 2 ]]; then return $rc; fi
    fi
    # Original Python fallback
    "${ROTIFER_ROOT}/bin/seqstats" "$@"
}
```

Users type `seqstats foo.fa` and get Rust speed transparently. If the Rust crate isn't built, they get the Python version with zero change to their workflow.

### `export -f` With zsh Guards

Functions must be exported for subshells in bash. The zsh guard prevents the `export -f` dump problem (Bug #3):

```bash
if [[ -z "${ZSH_VERSION:-}" ]]; then
    export -f create_neighborhood_diagrams extract_operon_distances
    export -f condense_operon_families merge_superalignment
    # ...
fi
```

---

## 7. Testing Strategy

*Test through pytest, benchmark with synthetic data, and never trust `cargo test` for cdylib crates. The test suite has three jobs: prove the registry is complete, prove imports work, and prove Rust output matches Python output format exactly.*

### Why `cargo test` Doesn't Work for cdylib

PyO3 cdylib crates compile to shared libraries (`.so` / `.dylib`) that Python loads. The `cargo test` harness tries to build an executable, which conflicts with PyO3's linking requirements. See Bug #6 for details.

### pytest Approach

`rust/tests/test_transparent_dispatch.py` is 395 lines and contains exactly **7 test classes** and **28 test methods** (verified by regex count). There are **6 module-level fixtures** and **1 class-scoped fixture**.

**Fixtures** (lines 35-368):

| Fixture | Scope | What it provides |
|---------|-------|-----------------|
| `cli_native` | module | `from wrappers import cli_native` — the dispatch shim module |
| `bio_native` | module | `from wrappers import bio_native` — the bio wrapper module |
| `sample_fasta` | function | 3-sequence FASTA string with `seq1`, `seq2`, `seq3` |
| `sample_tsv` | function | 5-row neighborhood TSV with `pid`, `block_id`, `architecture`, `strand`, `start`, `end` columns |
| `sample_blast_tab` | function | 2-row BLAST tabular output (tab6 format) |
| `sample_compact` | function | 3-row compact architecture TSV with `pid`, `architecture` columns |
| `shell_defined_functions` | class (TestShellWrappers) | Runs `bash -c "source rust_functions.sh"` and returns dict of `{func_name: bool}` |

**Parametrize lists** — `TestBioNativeImports` uses 6 `@pytest.mark.parametrize` decorators, each with a list class variable:
- `GIAN_FUNCS` (6 items): `compact_to_df_parse`, `count_series_domains`, `split_domain_table`, `rpsblast_output_to_table`, `phobius_to_table`, `pid2tax_parse`
- `NEIGH_FUNCS` (5 items): `sort_neighborhoods_by_similarity`, `clean_neighborhoods`, `neighborhood_to_network_edges`, `detect_domain_swaps_table`, `neighborhood_summary`
- `SEQ_FUNCS` (9 items): `seqstats`, `rcomplement`, `nt2aa`, `protein2dna`, `seqsplit`, `fasta2description`, `fasta2id`, `fasta2slice`, `rconvertseq`
- `FORMAT_FUNCS` (9 items): `blast2table`, `hmmer2table`, `hhsuite2table`, `genbank2table`, `architecture2table`, `domain2architecture`, `blastclust2table`, `operon2table`, `aln2seqrows`
- `TAXONOMY_FUNCS` (8 items): `parse_taxonomy_lineage`, `parse_taxonomy_batch`, `taxids_to_rank_table`, `group_by_taxonomy`, `taxonomy_distance`, `taxonomy_distance_matrix`, `filter_by_taxonomy`, `lca`
- `TREE_FUNCS` (8 items): `parse_newick_tree`, `tree_dfs_order`, `render_ascii_tree`, `render_unicode_tree`, `tree_to_table`, `tree_layout`, `prune_tree`, `reroot_tree`

`TestShellWrappers` has a 7th parametrize list: `EXPECTED_WRAPPERS` (28 items) — the transparent dispatch wrappers verified to be defined after sourcing `rust_functions.sh`.

**Test class summary**:

| Class | Methods | Purpose |
|-------|---------|---------|
| `TestToolRegistry` | 2 | Registry completeness: all handlers exist, expected tools present |
| `TestBioNativeImports` | 6 | Import smoke: all wrapper functions are importable (6 parametrized groups) |
| `TestDispatchExitCodes` | 2 | Exit codes: valid tool returns 0/1/2, unknown tool returns 2 |
| `TestFallbackBehavior` | 1 | Shell env var `ROTIFER_RUST=0` disables dispatch |
| `TestFunctionalParity` | 15 | Rust functions return correct output with sample data |
| `TestShellWrappers` | 1 | Shell wrappers defined after sourcing (parametrized over 28 names) |
| `TestNativeAvailableFlag` | 2 | `NATIVE_AVAILABLE` is a bool and True when built |

Tests are in `rust/tests/` and organized into four categories:

**1. Registry completeness** — every `TOOL_REGISTRY` entry has a corresponding handler:

```python
def test_all_entries_have_handlers(self, cli_native):
    registry = cli_native.TOOL_REGISTRY
    assert len(registry) >= 43
    missing = []
    for tool_name, handler_name in registry.items():
        if not hasattr(cli_native, handler_name):
            missing.append(f"{tool_name} -> {handler_name}")
    assert not missing
```

**2. Import smoke tests** — every wrapper function is importable:

```python
@pytest.mark.parametrize("func_name", SEQ_FUNCS)
def test_seq_func_importable(self, bio_native, func_name):
    assert hasattr(bio_native, func_name)
```

**3. Functional parity** — Rust output matches expected format with sample data:

```python
def test_seqstats(self, bio_native, sample_fasta):
    if not bio_native.NATIVE_AVAILABLE:
        pytest.skip("Native crate not built")
    result = bio_native.seqstats(sample_fasta)
    assert isinstance(result, str)
    assert "seq1" in result
```

**4. Benchmarks** — `bench_dispatch.py` (157 lines, `rust/tests/bench_dispatch.py`) generates synthetic datasets and measures wall-clock time. It uses `time.perf_counter()` for sub-millisecond resolution and reports the median of 3 runs:

```python
def bench(name, func, *args, repeats=3):
    times = []
    for _ in range(repeats):
        t0 = time.perf_counter()
        result = func(*args)
        t1 = time.perf_counter()
        times.append(t1 - t0)
    median = sorted(times)[len(times) // 2]
    print(f"  {name:40s}  {median*1000:8.1f} ms  (median of {repeats})")
```

**Why median-of-3 instead of mean?** The median is robust to outliers that systematically inflate the mean in short benchmark runs: the first run often pays a JIT or page-fault cost (OS loads the `.so` from disk, fills TLB entries), GC pauses can add 5–50ms to any single run, and OS scheduler preemption can stall any individual run unpredictably. With only 3 repetitions, one bad run inflates the mean by 33%. The median of 3 is simply the middle value — it is unaffected by one outlier high or one outlier low. For more stable estimates, increase `repeats` to 7 or 11 (odd numbers so the median is unambiguous), but 3 is fast enough for CI and catches gross regressions reliably.

Synthetic data generators use `random.seed(42)` for reproducibility:
- `generate_fasta(n_seqs, avg_len=300)` — amino acid sequences drawn from `"ACDEFGHIKLMNPQRSTVWY"`, headers with format `>seq_000001 random protein N`
- `generate_tsv(n_rows, n_blocks=None)` — neighborhood table with `pid`, `block_id`, `architecture` (1-4 domains from pool of 10), `strand`, `start`, `end`
- `generate_compact(n_rows)` — architecture-only table with `pid`, `architecture` (1-5 domains from pool of 5)

**What the benchmark measures** (run with `python3 rust/tests/bench_dispatch.py`):

| Category | Functions benchmarked | Dataset sizes |
|----------|-----------------------|---------------|
| Sequence utilities | `seqstats`, `fasta_ids`, `fasta_descriptions`, `reverse_complement` | 1K and 10K sequences |
| Format converters | `blast_to_table` | 10K BLAST hits |
| Architecture | `compact_to_df_parse`, `count_series_domains`, `split_domain_table` | 1K and 10K rows |
| Neighborhood | `neighborhood_summary`, `clean_neighborhoods`, `sort_neighborhoods_by_similarity` | 1K and 10K rows |
| Taxonomy | `parse_taxonomy_lineage`, `parse_taxonomy_batch` | 1 lineage and 1K lineages |
| Tree | `parse_newick_tree`, `render_ascii_tree`, `tree_dfs_order` | 128-leaf balanced binary tree |

The benchmark generates a 128-leaf Newick tree using a recursive `build_newick(depth=7)` helper — depth 7 gives 2^7 = 128 leaves.

---

## 8. Migration Playbook: Step by Step

*Four phases from workspace setup to production deployment. Each phase has a concrete verification gate — do not proceed to the next phase until all gates pass. The gates catch the class of bugs most likely to surface at that phase.*

### Phase 0: Workspace Setup

**Expected time**: 1 day. **Bugs that typically surface**: Bug #2 (zsh `BASH_SOURCE[0]` incompatibility), Bug #3 (`export -f` dumps in zsh).

1. Create `rust/Cargo.toml` with `[workspace]` and shared dependencies
2. Create `rust/wrappers/__init__.py` with fault-tolerant imports
3. Add `rust/rust_functions.sh` with the dispatch skeleton (`_rst_exec`, `_rust_available`, `_rust_dispatch`)
4. Verify: `source rust_functions.sh` works in both bash and zsh

**Before moving to Phase 1**: Source the script in both `bash` and `zsh` and confirm no output is produced (no function body dumps, no errors). Run `type _rust_available` in both shells to confirm the function is defined. If you see any output during sourcing, debug Bug #2 and Bug #3 before proceeding.

### Phase 1: First Crate (Start Small)

**Expected time**: 1–2 weeks. **Bugs that typically surface**: Bug #6 (cdylib `cargo test` conflicts), Bug #11 (conda ABI mismatch), Bug #9 (import path coupling).

Pick 3-5 functions that are clearly CPU-bound and have simple signatures (strings in, strings out).

1. `cargo init --lib rust/my_crate`
2. Set `crate-type = ["cdylib"]` in `Cargo.toml`
3. Add `pyproject.toml` and `build.rs`
4. Implement 3-5 `#[pyfunction]`s in a single `.rs` file
5. Register them in `lib.rs`'s `#[pymodule]`
6. Build: `maturin develop --release`
7. Test: `python3 -c "import my_crate; print(dir(my_crate))"`

**Do not** try to port complex functions first. Start with something like "count sequences in FASTA" or "reverse complement a string." Get the build pipeline working before tackling anything algorithmic.

**Before moving to Phase 2**: Confirm `from wrappers import my_crate` works from the repository root (not just from `rust/`). Confirm `cargo test` works for pure-Rust functions and fails gracefully for PyO3-facing ones. Confirm the build used the correct conda environment (`python3 -c "import my_crate; print(my_crate.__file__)")` should show a path inside your conda env's `site-packages`).

### Phase 2: Shell Integration

**Expected time**: 2–3 days. **Bugs that typically surface**: Bug #1 (numpy cascade failure in `wrappers/__init__.py`), Bug #10 (stale `__pycache__` after renames).

1. Create `wrappers/my_crate.py` with the `NATIVE_AVAILABLE` pattern
2. Add the module to `wrappers/__init__.py`'s `_SUBMODULES` list
3. Add wrapper functions to `rust_functions.sh`
4. Test: `source rust_functions.sh && my_tool input.fa`

**Before moving to Phase 3**: Verify that `ROTIFER_RUST=0 source rust_functions.sh && my_tool input.fa` falls through to the Python implementation. Verify that breaking one submodule (e.g., temporarily renaming a `.so` file) does not prevent other submodules from loading. Run `from wrappers import status; print(status())` and confirm only the broken crate shows `False`.

### Phase 3: Transparent Dispatch

**Expected time**: 1 week. **Bugs that typically surface**: Bug #4 (stdin hang), Bug #5 (TTY detection under PyO3), Bug #12 (return type mismatch).

1. Create `TOOL_REGISTRY` entries in `wrappers/cli_native.py` (or your dispatch module)
2. Write `_dispatch_my_tool(args)` functions that parse args and call Rust
3. Add TTY guards for any tool that reads stdin
4. Test exit code contract: `0` = success, `1` = error, `2` = fallback
5. Write pytest parity tests comparing Rust output to Python output

**Cross-reference**: For any tool that reads stdin, see Bug #4 and Bug #5 before writing the dispatch handler. The TTY detection must happen in Python (`sys.stdin.isatty()`), not in Rust (`std::io::stdin().is_terminal()`).

**Before moving to Phase 4**: Run the full pytest suite. Every tool in `TOOL_REGISTRY` must have a handler that passes the registry completeness test. Run each tool with `--help` and verify the output looks correct. Test the fallback: `export ROTIFER_RUST=0` and run each tool to confirm it produces the same output via the Python path.

### Phase 4: Scaling Up

**Expected time**: Ongoing. **Bugs that typically surface**: Bug #7 (OnceLock poisoning in new singletons), Bug #8 (TOCTOU in new file-scanning functions).

1. Add feature flags to `Cargo.toml` for logical groupings
2. Gate modules with `#[cfg(feature = "...")]` in `lib.rs`
3. Split large crates if they exceed ~50 functions (compilation gets slow)
4. Add benchmark scripts with synthetic data generators
5. Document everything in a porting notes file

**Cross-reference**: For any new singleton (global state initialized once), follow the `OnceLock` pattern in Section 4.2 and review Bug #7 before writing the initialization closure. For any new file-reading function in security-sensitive code, follow the TOCTOU pattern in Section 4.7 and review Bug #8.

**Important**: At each phase, verify that the Python fallback still works when Rust is disabled:

```bash
export ROTIFER_RUST=0
source rust_functions.sh
seqstats input.fa  # Should use Python version
```

---

## 9. Performance Patterns

*Release the GIL, use rayon, set empirical thresholds — and never benchmark debug builds. Performance work done against a debug build is not just wasted time; it produces actively misleading numbers that will send you optimizing the wrong thing.*

### rayon for Batch Parallelism

Use `rayon::par_iter()` inside `py.allow_threads()` for batch operations:

```rust
let all_threats: Vec<ThreatInfo> = py.allow_threads(|| {
    paths
        .par_iter()
        .flat_map(|path| {
            let content = match read_file_contents(path) {
                Ok(c) => c,
                Err(_) => return vec![],
            };
            // ... scan each file ...
        })
        .collect()
});
```

The GIL must be released first (`py.allow_threads`), then rayon's thread pool handles parallelism. Without releasing the GIL, rayon threads will deadlock trying to acquire it.

### memchr for Fast Line Scanning

For parsing large text files (FASTA, TSV), `memchr::memchr` finds newlines 5-10x faster than `str::find('\n')` because it uses SIMD instructions:

```rust
use memchr::memchr;
let mut pos = 0;
while let Some(nl) = memchr(b'\n', &data[pos..]) {
    let line = &data[pos..pos + nl];
    // process line
    pos += nl + 1;
}
```

### Aho-Corasick for Multi-Pattern Matching

When scanning for multiple patterns simultaneously (malware signatures, domain keywords), build an Aho-Corasick automaton once and scan in a single pass:

```rust
let ac = AhoCorasick::new(&patterns)?;
for mat in ac.find_iter(&content) {
    // mat.pattern() gives which pattern matched
    // mat.start() gives the byte offset
}
```

This is O(n) in the input length regardless of the number of patterns, compared to O(n*m) for running each pattern separately.

### PARALLEL_THRESHOLD Constant

Not every batch benefits from parallelism. rayon's thread pool has overhead (~1-5 microseconds per task). The production code uses this threshold in `rotifer_cli_native/src/table_cli.rs:25`:

```rust
// rotifer_cli_native/src/table_cli.rs:25
const PARALLEL_THRESHOLD: usize = 10_000;
```

The value is **10,000 rows** (not 100 as in the conceptual example). Table operations on fewer than 10,000 rows run sequentially; at or above 10,000 rows they switch to `par_iter()`. This threshold was determined empirically for TSV row processing — the crossover point where rayon thread-pool overhead is recouped by parallel work. The pattern at `table_cli.rs:1034-1046`:

```rust
py.allow_threads(|| {
    if lines.len() >= PARALLEL_THRESHOLD {
        let updated: Vec<String> = lines[1..]
            .par_iter()
            .map(|line| { /* row transform */ })
            .collect();
        // ...
    } else {
        for line in &lines[1..] { /* sequential */ }
    }
});
```

The threshold may differ per function type — 10,000 rows for TSV table ops, a different value might be appropriate for entropy calculations (where each file is heavier work). Profile your specific workload before choosing a threshold.

**Trade-off**: Setting `PARALLEL_THRESHOLD` too low wastes CPU on rayon thread-pool overhead — each task dispatch and work-steal costs roughly 1–5 microseconds, so parallelizing 100 trivial row transforms may be slower than running them sequentially. Setting it too high misses parallelism opportunities — a threshold of 1,000,000 rows effectively disables parallelism for all but the largest datasets. The 10,000-row value in production was chosen by bisection benchmarking with representative TSV inputs. Profile with your actual workload distribution, not synthetic benchmarks of a single size — the crossover point shifts significantly based on the per-row work complexity.

---

## 10. Common Pitfalls

*Eight things that will consume your debugging time if you do not know about them in advance. All eight appeared in the rotifer port. None of them are obvious from reading the PyO3 documentation.*

**Don't read stdin in Rust** when called via PyO3. TTY detection (`is_terminal()`) may give wrong results because Python has already set up the file descriptor. Read stdin in the Python dispatch layer and pass data as a string argument.

**Don't use `cargo test` on cdylib crates** for PyO3-facing code. Use pytest instead. Reserve `cargo test` for pure-Rust unit tests that don't involve any `Python<'py>` types.

**Don't use `export -f` without a zsh guard**. zsh interprets `export -f` differently and dumps function bodies to the terminal. Always wrap in `if [[ -z "${ZSH_VERSION:-}" ]]; then ... fi`.

**Don't eagerly import all submodules**. Use lazy, fault-tolerant imports (try/except around each submodule). A broken numpy installation in one crate should not prevent other crates from loading.

**Don't use `.expect()` on `OnceLock`**. If initialization panics, the lock is permanently poisoned. Use `match` with a safe fallback.

**Don't build in debug mode and benchmark**. Debug builds disable optimizations and give 10-50x worse performance than release builds. Always use `maturin develop --release` for any performance measurement.

**Don't return custom PyO3 classes from ported functions**. Stick to Python-native types (str, int, float, list, dict, tuple, numpy arrays). This keeps the fallback path trivial and avoids serialization headaches.

**Don't forget `__pycache__` cleanup after renames**. Stale bytecode files can shadow renamed modules and cause confusing import failures.

---

## 11. Lessons Learned

This section is a retrospective on the full port: what went well, what did not, and what we would do differently. It is written for teams considering a similar migration.

### What Took Longer Than Expected

**Shell compatibility consumed the most debugging time.** We assumed bash and zsh were similar enough to handle with minor guards. They are not. The `BASH_SOURCE[0]` incompatibility (Bug #2) and the `export -f` dump problem (Bug #3) required rewriting the entire path detection and function export logic. The cost was not the fix — both fixes were one-liners once the root cause was understood — but the diagnosis. Shell bugs surface as cryptic empty-string errors or silent behavior changes that look like Python or Rust bugs until you bisect carefully.

**Import isolation required more thought than expected.** The fault-tolerant `try/except Exception` pattern (Section 4.3, Bug #1) sounds obvious in retrospect. It was not obvious at the time. The numpy cascade failure appeared to be a PyO3 bug because the error message mentioned numpy and we were importing a numpy-using Rust extension. The actual root cause — a different submodule failing and propagating upward — took hours to find. The lesson: in a multi-crate wrapper, the first thing you build should be the fault isolation layer, not the first function.

### What Was Easier Than Expected

**PyO3 itself.** The Rust/Python boundary was almost entirely frictionless once we understood the `'py` lifetime and the `Bound<'py, T>` return type pattern. The numpy bridge (`PyReadonlyArray2`, `into_pyarray_bound`) worked on the first attempt for the Needleman-Wunsch implementation. The GIL release pattern required some thought the first time, but it is mechanically consistent across every function — extract before `allow_threads`, convert after.

**The conda ABI problem (Bug #11) was easy to avoid once named.** "Always activate the target environment before running maturin" is a one-sentence rule. The hard part was naming the failure mode. Once you know the error message (`symbol not found in flat namespace '__PyArg_ParseTupleAndKeywords_SizeT'`) maps to "wrong Python", the fix is immediate.

**Rayon parallelism.** Replacing `.iter()` with `.par_iter()` inside `allow_threads` is genuinely as easy as the documentation suggests. The only complication was setting the `PARALLEL_THRESHOLD` empirically rather than theoretically — but that is a benchmarking task, not a correctness task.

### What We Would Do Differently

**Start with the shell layer first.** We started with the first Rust crate, then built the Python wrapper, then built the shell layer. This ordering meant that shell bugs surfaced late, when the codebase was complex. The correct ordering is: build the shell skeleton (Phase 0) and verify it in both bash and zsh before writing a single line of Rust. The shell layer is the most fragile part of the architecture and the hardest to debug. Front-load the risk.

**Establish the exit-code contract before writing any Rust.** The exit-code contract (`0` = success, `1` = error, `2` = fallback) is the core invariant of the entire architecture. Every dispatch handler, every shell wrapper, every test depends on it. We defined it informally and had to retrofit it onto early handlers. It should be the first thing documented and the first thing tested.

**Write parity tests earlier.** The functional parity tests (Section 7, `TestFunctionalParity`) were added late in the process. Adding them early would have caught Bug #12 (return type mismatch) immediately rather than after it reached production shell scripts.

### Advice for Teams Considering a Similar Port

**Do not port because Rust is fast. Port because your profiler says this specific function is slow.** Rust's performance advantage is real, but it only matters for CPU-bound code. If your bottleneck is NCBI API latency or pandas dataframe manipulation, porting to Rust will not help.

**Budget for the boundary work.** Every hour you spend writing Rust functions will be matched by at least one hour spent on the boundary layer — the shell wrapper, the Python dispatch, the import isolation. Plan for it.

**The fallback is not a crutch; it is an architectural guarantee.** The exit-code `2` fallback means the Rust layer can be incomplete forever — you can ship 10 ported functions and 200 Python fallbacks and users will never know the difference. This is a feature, not a compromise. Use it to reduce the scope of each release.

**Document every bug as you fix it.** This porting guide exists because we wrote down every bug immediately after fixing it. Without that discipline, the institutional knowledge would have evaporated. The guide has already saved debugging time for every subsequent crate. The cost of writing it was one paragraph per bug. The return is incalculable.

---

## Appendix A: File Inventory

| File | Lines | Purpose |
|------|-------|---------|
| `rust/Cargo.toml` | 70 | Workspace root: members, shared deps, release profile |
| `rust/wrappers/__init__.py` | — | Fault-tolerant package init, `status()`, `is_native_available()` |
| `rust/wrappers/cli_native.py` | 1658 | TOOL_REGISTRY (63 entries), dispatch handlers, argparse |
| `rust/wrappers/bio_native.py` | — | Wrapper shims for rotifer_bio_native functions |
| `rust/wrappers/av_native.py` | — | Wrapper shims for rotifer_av_native functions |
| `rust/wrappers/core.py` | — | Wrapper shims for rotifer_core functions |
| `rust/wrappers/trouter.py` | — | Wrapper shims for trouter_core functions |
| `rust/wrappers/_dispatch.py` | — | `_require_native()` factory |
| `rust/rust_functions.sh` | 2908 | Shell dispatch layer (bash+zsh compatible, 14 sections) |
| `rust/rotifer_bio_native/` | — | 150 functions: alignment, clustering, network, parsing, taxonomy, trees |
| `rust/rotifer_av_native/` | — | 24 functions: entropy, pattern matching, Mach-O analysis, API key scanning |
| `rust/rotifer_cli_native/` | — | 18 functions: table ops, alignment viewer, FASTA conversion |
| `rust/trouter_core/` | — | 12 functions: tier classification, model mapping |
| `rust/rotifer_core/` | — | 11 functions: FASTA parsing, similarity, architecture |
| `rust/tests/test_transparent_dispatch.py` | 395 | Registry, import, parity, shell wrapper tests (7 classes, 28 methods) |
| `rust/tests/bench_dispatch.py` | 157 | Benchmark suite with synthetic data generators |
| `rust/rotifer_bio_native/src/nw.rs` | 401 | Needleman-Wunsch DP with affine gaps (3 exported functions) |
| `rust/rotifer_av_native/src/scanner.rs` | 286 | File scanning: `scan_file`, `scan_files_batch`, TOCTOU-safe read |
| `rust/trouter_core/src/models.rs` | 180 | Tier classification and model mapping (7 tiers, 63 keywords) |
| `rust/rotifer_cli_native/src/table_cli.rs` | — | Table operations (`PARALLEL_THRESHOLD = 10_000` at line 25) |

**TOOL_REGISTRY correction**: The document header says "67 entries" but the actual count in the current `cli_native.py` is **63 entries** (verified: `grep -c '"_dispatch_' wrappers/cli_native.py` = 63). The test assertion uses `>= 43` to allow growth without requiring test updates.

---

## Appendix B: Quick Reference Card

### Build

```bash
# Build all crates
for crate in rotifer_core rotifer_bio_native rotifer_av_native trouter_core rotifer_cli_native; do
    (cd rust/$crate && maturin develop --release)
done

# Build one crate with specific features
cd rust/rotifer_bio_native && maturin develop --release --features core

# Verify build
python3 -c "from wrappers import status; print(status())"
```

### Test

```bash
# Full test suite
python3 -m pytest rust/tests/test_transparent_dispatch.py -v

# Benchmarks
python3 rust/tests/bench_dispatch.py

# Quick smoke test
python3 -c "from wrappers import bio_native; print(bio_native.NATIVE_AVAILABLE)"
```

### Disable / Debug

```bash
# Disable all Rust acceleration
export ROTIFER_RUST=0

# Enable debug timing
export RST_DEBUG=1

# Clear crate cache (after rebuild)
_rust_cache_clear
```

### Status

```bash
# Shell status
source rust/rust_functions.sh
rst_help

# Python status
python3 -c "from wrappers import status; import json; print(json.dumps(status(), indent=2))"

# Per-crate check
python3 -c "import rotifer_bio_native; print(len(dir(rotifer_bio_native)), 'functions')"
```

### Common Workflows

```bash
# Port a new function
# 1. Write Rust implementation in src/my_module.rs
# 2. Register in lib.rs: m.add_function(wrap_pyfunction!(my_module::my_func, m)?)?;
# 3. Build: maturin develop --release
# 4. Add wrapper in wrappers/bio_native.py
# 5. Add dispatch entry in wrappers/cli_native.py TOOL_REGISTRY
# 6. Add shell wrapper in rust_functions.sh
# 7. Add pytest in tests/test_transparent_dispatch.py
# 8. Run tests: python3 -m pytest rust/tests/ -v
```
