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

## Directory & File Conventions

- Put benches under `bench/` at the repo root.
- Name files with the suffix `.bench.mo`, one benchmark per file; for example: `bench/base64.bench.mo`.
- Each bench file is a Motoko `module { ... }` that exposes a single `public func init() : Bench.V1` function.
- Inside `init`, construct a `Bench.Schema` and return `Bench.V1(schema, run)` where
  `run : (rowIndex : Nat, colIndex : Nat) -> ()` performs the measured operation.

Minimal skeleton

```motoko
import Array "mo:core/Array";
import Text  "mo:core/Text";
import Bench "mo:bench-helper";

module {
  public func init() : Bench.V1 {
    let schema : Bench.Schema = {
      name = "My bench";
      description = "What this bench measures";
      rows = ["size 16", "size 64", "size 256"]; // your row labels
      columns = ["operation A", "operation B"];  // your column labels
    };

    // Prepare inputs outside of `run` so they are not re-created on every iteration
    let inputs : [[Nat8]] = [
      Array.init<Nat8>(16, 0),
      Array.init<Nat8>(64, 0),
      Array.init<Nat8>(256, 0),
    ];

    // Build a table of routines to measure: routines[row][col] : () -> ()
    let routines : [[() -> ()]] = Array.tabulate(
      rows.size(),
      func(ri) {
        let input = inputs[ri]; // capture precomputed input
        [
          func() { ignore input.size() },    // operation A @ inputs[ri]
          func() { ignore input.toArray() }, // operation B @ inputs[ri]
        ]
      },
    );

    // The runner calls this many times; keep it tiny and branch-free.
    Bench.V1(schema, func(ri : Nat, ci : Nat) = routines[ri][ci]());
  };
};
```

Note: if you're not using "core" dependency, replace "mo:core" imports with "mo:base"

## How It Works

- Schema
    - `name` and `description` describe the bench.
    - `rows` enumerate different operations or variants you measure (e.g., "encode", "decode").
    - `cols` enumerate different input categories (e.g., message sizes).
- Runner contract
    - You return a versioned record `Bench.V1(schema, run)`; the runner calls `run(rowIndex, colIndex)` many times to
      record timings.
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