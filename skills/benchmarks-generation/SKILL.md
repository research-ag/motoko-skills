---
name: motoko-benchmarks-generation
description: How to write benchmarks in Motoko using bench‑helper. Covers project setup (mops.toml), bench file layout in bench/*.bench.mo, the Bench.Schema rows/cols model, and safe patterns for encode/decode, hashing, crypto, and allocation benches.
---

# Motoko Benchmarks with bench‑helper

## What This Is

`bench-helper` is a tiny Motoko library that standardizes how to write benchmarks.
You describe a benchmark using a small schema (name, rows, cols), provide a `run(row, col)`
function, and return a versioned bench record. Each file under `bench/*.bench.mo` defines one
benchmark module. A runner can then discover and execute all benches consistently.

## Prerequisites

mops.toml (add dependencies and toolchain)

If you already have a `mops.toml`, just add `bench-helper` under `[dev-dependencies]`.
If your project still uses `mo:base` instead of `mo:core`, you can keep it — benches themselves can be written with
`mo:base` without affecting your runtime canisters.

**Make sure `[optimize]` is configured.** `mops bench` only runs Binaryen's `wasm-opt` if `mops.toml` has an
`[optimize]` section — even an empty one activates it (default `level = "O3"`). Without it, `mops bench` measures
unoptimized Wasm, which doesn't reflect what a production canister actually runs. `[optimize]` requires a pinned
`[toolchain] wasm-opt` version, e.g.:

```toml
[optimize]

[toolchain]
wasm-opt = "132"   # mops toolchain use wasm-opt <version>
```

The `motoko-mops-package-maintenance` skill adds this automatically for packages with benchmarks during a
maintenance pass — if it hasn't run yet, add it yourself before trusting bench numbers.

## Directory & File Conventions

- Put benches under `bench/` at the repo root.
- Name files with the suffix `.bench.mo`, one benchmark per file; for example: `bench/base64.bench.mo`.
- Each bench file is a Motoko `module { ... }` that exposes a single `public func init() : Bench.V1` function.
- Inside `init`, construct a `Bench.Schema` and return `Bench.V1(schema, run)` where
  `run : (rowIndex : Nat, colIndex : Nat) -> ()` performs the measured operation.

Minimal skeleton

```motoko
import Array "mo:core/Array";
import Bench "mo:bench-helper";

module {
  /// One benchmark per file; `mops bench` calls `init()` to obtain the
  /// schema and the run function.
  public func init() : Bench.V1 {
    let schema : Bench.Schema = {
      name = "My bench";
      description = "What this bench measures";
      rows = ["size 16", "size 64", "size 256"]; // input categories
      cols = ["operation A", "operation B"];     // operations to measure
    };

    // Prepare inputs once in `init` so they are not re-created on every call.
    // `Array.tabulate` returns an immutable `[Nat8]`; use `VarArray.repeat`
    // (or `VarArray.tabulate`) if you need a `[var Nat8]` instead.
    let inputs : [[Nat8]] = [
      Array.tabulate<Nat8>(16, func _ = 0),
      Array.tabulate<Nat8>(64, func _ = 0),
      Array.tabulate<Nat8>(256, func _ = 0),
    ];

    // Build a table of routines: routines[ri][ci] : () -> ()
    let routines : [[() -> ()]] = Array.tabulate<[() -> ()]>(
      schema.rows.size(),
      func(ri) {
        let input = inputs[ri]; // capture precomputed input
        [
          func() { ignore input.size() }, // operation A @ inputs[ri]
          func() { ignore input[0] },     // operation B @ inputs[ri]
        ];
      },
    );

    // `Bench.V1` is a *class* (instance type), not a record alias.
    // The runner calls the second argument many times; keep it tiny
    // and branch-free.
    Bench.V1(schema, func(ri : Nat, ci : Nat) = routines[ri][ci]());
  };
};
```

Notes:
- The `Schema` field is `cols`, not `columns`. Same for `rows.size()` — you
  must say `schema.rows.size()` (the bare `rows` identifier is not in scope).
- `mo:core/Array` has no `init` function. To build an immutable `[T]`, use
  `Array.tabulate<T>(size, gen)` or `Array.repeat<T>(val, size)`. For a
  mutable `[var T]`, import `VarArray "mo:core/VarArray"` and use
  `VarArray.tabulate<T>(size, gen)` or `VarArray.repeat<T>(val, size)`;
  annotate such inputs as `[var T]` (e.g. `[[var Nat8]]` for a table of them).
- If you're not using `mo:core`, replace `mo:core` imports with `mo:base`.
  Note that `mo:base/Array` covers both immutable and mutable arrays — there
  is no separate `VarArray` module. The interface is not identical to
  `mo:core`; for the Array API the relevant equivalences are:
    - `mo:core` `Array.tabulate(size, gen)` → `mo:base` `Array.tabulate(size, gen)`
    - `mo:core` `Array.repeat(val, size)` → `mo:base` `Array.freeze(Array.init(size, val))` (no direct immutable `repeat`)
    - `mo:core` `VarArray.tabulate(size, gen)` → `mo:base` `Array.tabulateVar(size, gen)`
    - `mo:core` `VarArray.repeat(val, size)` → `mo:base` `Array.init(size, val)` (args reversed)
  See the `base-core-migration` skill for the full mapping.

## How It Works

- Schema
    - `name` and `description` describe the bench.
    - `rows` enumerate different operations or variants you measure (e.g., "encode", "decode").
    - `cols` enumerate different input categories (e.g., message sizes).
- Runner contract
    - You return a versioned record `Bench.V1(schema, run)`. `mops bench` calls `run(rowIndex, colIndex)` exactly
      once per cell, row by row, all cells of a bench file in the same canister; `init()` runs once before.
    - Side effects/results inside `run` should be consumed (e.g., `ignore ...`) to prevent dead‑code elimination.

## Common Pitfalls

1. Rows/cols mismatch
    - Ensure your `routines` table has dimensions `rows.size() x cols.size()`. If you add or remove a row/col label, update how you build `routines`; otherwise some cells will be no‑ops.
2. Doing expensive setup inside `run`
    - Generate inputs once in `init` and capture them in closures. Only do the core operation in `run`.
3. Forgetting to consume results
    - Use `ignore` to consume return values; otherwise the compiler might drop the call as dead code.
4. Non‑determinism and timing noise
    - Keep `run` free of logging/printing and random allocation; keep GC pressure comparable across rows.
5. Mutating operations and shared state
    - Each cell runs once, but all cells share the canister's state. If
      the measured operation mutates a captured input (an in-place sort,
      a hasher's state), give every cell its own input in `init`, or
      later cells measure on the state an earlier cell left behind. A
      retaining operation (appending to a shared list) can also make
      cells differ when the structure grows across a block boundary.

## Reading the numbers (moc 2.0.0, incremental GC)

- **Instructions** include the harness's call into your runner: ~70 for
  a no-op cell with mops CLI ≤ 3.4.2, ~34 after caffeinelabs/mops#878.
  A table of routines (`routines[ri][ci]()`) adds ~85 for the 2-D lookup
  and the inner call; a "no-op function" column is the right baseline.
- **Heap** is everything the cell allocated, temporaries and retained
  memory combined. With mops CLI ≤ 3.4.2 every cell also shows a
  constant ~35.5 KiB of harness overhead (mops#876): subtract a no-op
  row. With #878 a no-op shows 0 B.
- **Garbage Collection** is always 0 under the incremental GC (it never
  collects inside a measured call), so it can't separate garbage from
  retained memory. Measuring what a structure *retains* needs a
  different approach (below).
- Pin `pocket-ic` 15.0.0: 16.1.0 adds ~20,000 instructions to every call
  that writes to the heap.

## Asserting allocations in tests

To guarantee that a function stays allocation-free, test it with
`mops test` in wasi mode, where the GC never runs:

```motoko
// @testmode wasi
import Prim "mo:prim";

func allocDelta(f : () -> ()) : Nat {
  f(); // warm-up: exclude one-time allocations
  let before = Prim.rts_total_allocation();
  f();
  Prim.rts_total_allocation() - before;
};

assert allocDelta(func() { ignore myList.get(5) }) == 0;
```

Also assert that the helper detects an allocation (e.g.
`Prim.Array_init<Nat>(8, 0)` gives `> 0`), so a wrong test mode can't
pass silently. Capture-free lambdas are static; a lambda capturing
test-local variables is allocated where it is created. Destructuring a
tuple with a pattern (`let (a, b) = (x, y)`) allocates nothing.

The *retained* size of a structure can't be read from the counters (no
GC runs in wasi, and Motoko code can't trigger one). Compute it from
the structure's layout instead: on the 64-bit heap an array is
24 B + 8 B per slot, a `Blob` 24 B + its bytes. Check the formula
against `rts_total_allocation` in a case that creates no garbage (e.g.
an empty structure).

## More examples

bench-helper reference benches: https://github.com/research-ag/bench-helper/tree/main/bench

## Verify It Works

The exact runner/command may vary depending on your environment. After adding `bench-helper` to `mops.toml` and writing
`.bench.mo` files:

- Ensure your project resolves dependencies:
  ```bash
  mops install
  ```
- Run benchmarks:
  ```bash
  mops bench
  ```
- Consult the `bench-helper` package README for the latest recommended runner command for your toolchain/version.