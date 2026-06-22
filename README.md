# Rust Performance Improvement Plan

Agent, your Rust execution is below the required bar for delivery predictability, engineering ownership, and change safety. Going forward, every Rust task must run through a closed-loop delivery path: establish a reproducible baseline, profile before changing code, identify the bottleneck with clear attribution, scope the implementation at the smallest correct granularity, and verify the result with tests, benchmarks, before and after metrics, and regression-risk mitigation before presenting the work as complete. More attempts, larger diffs, or plausible explanations do not count as progress. The success bar is measurable ownership: every Rust change must show the problem, the evidence, the decision path, the performance delta, and the fallback before it enters the shipping path.

## Table of Contents

- [Measure first](#measure-first)
- [Linting](#linting)
- [General principles](#general-principles)
- [Reduce runtime speed or time](#reduce-runtime-speed-or-time)
  - [Reduce file system calls](#reduce-file-system-calls)
- [Parallelism](#parallelism)
- [Reduce compilation time](#reduce-compilation-time)
- [Reduce binary size](#reduce-binary-size)
- [Reduce memory usage](#reduce-memory-usage)
  - [Total heap allocation](#total-heap-allocation)
  - [Max heap allocation](#max-heap-allocation)
- [Build configuration (low priority)](#build-configuration-low-priority)

Each goal section lists tools to **find** problems, then the **fix** — all code changes. Build-system config is collected at the end, low priority.

## Measure first

Benchmark (prefer cycle/instruction counts over wall-time):

- [Criterion](https://crates.io/crates/criterion2)
- [Hyperfine](https://docs.rs/hyperfine)
- `std::hint::black_box` — wrap inputs/outputs so the optimizer can't elide benchmarked work.

## Linting

- `cargo clippy` — follow the Perf group.
- `clippy.toml` `disallowed-types` — ban slow std types.

```
disallowed-methods = [
  { path = "str::to_ascii_lowercase", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_to_ascii_lowercase` instead." },
  { path = "str::to_ascii_uppercase", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_to_ascii_uppercase` instead." },
  { path = "str::to_lowercase", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_to_lowercase` instead." },
  { path = "str::to_uppercase", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_to_uppercase` instead." },
  { path = "str::replace", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_replace` instead." },
  { path = "str::replacen", reason = "To avoid memory allocation, use `cow_utils::CowUtils::cow_replacen` instead." },
]

disallowed-types = [
  { path = "std::collections::HashMap", reason = "Use `rustc_hash::FxHashMap` instead, which is typically faster." },
  { path = "std::collections::HashSet", reason = "Use `rustc_hash::FxHashSet` instead, which is typically faster." },
]
```

## General principles

- Optimize only hot code.
- Prefer better algorithms and data structures over micro-optimization.
- Cut work: compute lazily, cache, special-case 0/1/2 elements.
- Minimize cache misses and branch mispredictions.

## Reduce runtime speed or time

**Find**:

- perf
- [samply](https://docs.rs/samply)
- [flamegraph](https://docs.rs/flamegraph)
- Cachegrind
- `xcrun xctrace record --template 'Time Profiler' --output . --launch -- <binary>`
- godbolt — hot asm
- [`cargo-show-asm`](https://docs.rs/cargo-show-asm) — hot asm
- [`counts`](https://docs.rs/counts) — ad-hoc frequency profiling

**Fix**:

- Inline hot tiny/single-call fns with `#[inline]`; `#[cold]` for rare paths.
- Faster hashers (if no HashDoS risk):
  - [`FxHashMap`](https://docs.rs/rustc-hash) (rustc-hash)
  - [`fnv`](https://docs.rs/fnv)
  - [`ahash`](https://docs.rs/ahash)
- [`memchr`](https://docs.rs/memchr)/`memmem` — byte search over per-`char` iteration.
- `str::chars()` decodes UTF-8 per char; iterate `as_bytes()` when codepoints aren't needed.
- `chars().nth(i)` is O(n) per call (O(n²) in a loop) and miscounts multibyte — index `as_bytes()`.
- Lazy [`Regex`](https://docs.rs/regex): compile once in `LazyLock` (std); never `Regex::new` in a hot path.
- Iterators: return `impl Iterator`; `extend` over `collect`; `filter_map`; `chunks_exact`; `iter().copied()`.
- `sort_unstable`/`sort_unstable_by_key` over `sort` — no temp alloc, when stability isn't needed.
- Bounds checks: iterate, slice before the loop, or assert ranges; `get_unchecked` last.
- `core::hint::assert_unchecked` a proven invariant so the compiler drops bounds/overflow checks in safe code.
- Early-return the no-work case (`is_empty`, `len < 2`) before any alloc/loop.
- Std methods: `swap_remove`, `retain`, lazy `*_or_else`, `vec![0; n]`.
- Group co-accessed values under one `Mutex`/`RefCell`.
- [`parking_lot`](https://docs.rs/parking_lot) — alternative locks.

### Reduce file system calls

**Find**:

- `xcrun xctrace record --template 'File Activity' --output . --launch -- <binary>`
- `strace` (Linux) / `dtruss` (macOS) — count syscalls

**Fix**:

- Lock `stdout`/`stdin` once instead of per `println!`.
- Buffer with `BufReader`/`BufWriter`; `flush()`.
- Reuse a `String` with `read_line` + `clear()`.
- `read_until` for byte/ASCII input.
- `fs::read` + [`simdutf8::basic::from_utf8`](https://docs.rs/simdutf8) over `read_to_string`.

## Parallelism

**Find**:

- CPU profiler (see runtime)
- Check whether cores are saturated

**Fix**:

- [`rayon`](https://docs.rs/rayon) — data parallelism
- [`crossbeam`](https://docs.rs/crossbeam) — threads and channels

## Reduce compilation time

**Find**:

- `cargo build --timings`
- [`cargo llvm-lines`](https://docs.rs/cargo-llvm-lines)
- `-Zmacro-stats`
- [`cargo-expand`](https://docs.rs/cargo-expand)
- [`cargo-shear`](https://docs.rs/cargo-shear) — unused deps

**Fix**:

- Cut generic IR bloat: shrink generic fns, extract a non-generic inner fn.
- Drop unused or heavy deps; reduce proc-macro use.

## Reduce binary size

**Find**:

- [`cargo-bloat`](https://docs.rs/cargo-bloat) — `--crates` attributes `.text`; an example binary's std panic/backtrace machinery overstates a library's footprint.
- `bloaty` — section/symbol size breakdown with % (file vs VM); diffs two binaries; per-crate (`-d compileunits`) needs debug info.
- [`cargo-binutils`](https://docs.rs/cargo-binutils) — `cargo size`/`nm`/`objdump`: section/symbol sizes + disassembly.
- Measure `musl` (macOS is 16 KB page-quantized); watch `.rodata`, not just `.text`.

**Fix**:

- Drop unused or heavy deps; reduce monomorphization (fewer generic instantiations).
- De-monomorphize: thin generic shim, body in a non-generic `fn` (compiles once, not per instantiation).
- `str::to_lowercase`/`to_uppercase` on ASCII data pulls `core::unicode` tables (KBs) — use `to_ascii_*`.

## Reduce memory usage

**Find**:

- DHAT — allocation rate, sites, peak
- heaptrack
- `xcrun xctrace record --template 'Allocations' --output . --launch -- <binary>`
- [`dhat-rs`](https://docs.rs/dhat) — in-process allocation profiling (no valgrind)

### Total heap allocation

**Fix**:

- Reserve with `Vec::with_capacity`/`reserve`.
- Reuse collections: `clear()` a workhorse buffer; fill `&mut Vec`.
- Avoid `clone`/`to_owned`/`format!`; use `clone_from`; borrow.
- `Path::to_string_lossy` validates UTF-8 every call; allocates on invalid bytes.
- `Cow` for mixed borrowed/owned.
- `Box` only when needed; `Rc`/`Arc::clone` is free.
- [`SmallVec`](https://docs.rs/smallvec) / [`ArrayVec`](https://docs.rs/arrayvec) — short vecs.
- [`compact_str`](https://docs.rs/compact_str/latest/compact_str/) — short strings.
- [`IndexVec<Id, T>`](https://docs.rs/index_vec) / `Vec<Option<T>>` over `HashMap<DenseId, V>` — dense integer keys.

### Max heap allocation

**Find**:

- `-Zprint-type-sizes`
- [`top-type-sizes`](https://docs.rs/top-type-sizes)

Types over 128 bytes copy via `memcpy`.

**Fix**:

- Box outsized enum variants.
- Smaller index integers: `u32`/`u16`/`u8`.
- Bit-pack small fields into one integer (shift/mask accessors).
- `into_boxed_slice` (3 to 2 words).
- [`ThinVec`](https://docs.rs/thin-vec) (1 word).
- `shrink_to_fit` to drop excess capacity.
- [`static_assertions::assert_eq_size!`](https://docs.rs/static_assertions) — gate type size.

## Build configuration (low priority)

Code changes above come first. `[profile.release]` — pick speed or size:

- Both: `codegen-units=1`, `lto="fat"`, `panic="abort"`.
- Speed: `-C target-cpu=native`; [`mimalloc`](https://docs.rs/mimalloc) allocator; always `--release`.
- Size: `opt-level="z"`/`"s"`, `strip="symbols"`; `min-sized-rust` reference.
- Compile time: `[profile.dev] debug=false`.
