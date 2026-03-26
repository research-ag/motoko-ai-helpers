---
name: motoko-performance-optimizations
description: General performance optimization techniques for Motoko. Reducing allocations, efficient Text building, fixed-width arithmetic, block processing, async patterns, and more. Load when you need to improve hot paths or reduce overhead without changing behavior.
---

# Motoko Performance Optimizations

## What This Is

An extensible guide for speeding up Motoko code safely and predictably. It focuses on mechanical, behavior-preserving improvements: allocation reduction, fixed-width arithmetic, block processing, efficient `Text` building, and clear loop shapes. Use this skill when you want to improve throughput/latency without changing semantics.

- Benchmarking details and harnesses live in: skills/benchmarks-generation/SKILL.md
- Style and safe refactors that often precede perf work: skills/code-improvements/SKILL.md
- Dot-notation improvements that reduce verbosity/overhead: skills/dot-notation-migration/SKILL.md

## Quick Wins (General)

- Minimize allocations in hot paths
  - Avoid materializing entire buffers just to iterate (e.g., prefer indexing `Blob` directly).
  - Reuse lengths and capacities; cache `size()` calls to a local variable.

- Prefer fixed-width arithmetic in tight loops
  - Keep shifts/masks on `Nat32`/`Nat64` intermediates; avoid widening to `Nat` mid-loop.

- Build `Text` in larger chunks
  - Avoid per-character `Text.fromChar` + many small `#` appends; emit ASCII/UTF‑8 blocks and append once per block.

- Shape loops for the steady state
  - Process uniform blocks in the main loop; handle a tiny tail separately.
  - Hoist invariants (sizes, constants) and keep the inner loop straight-line.

- Verify after each change
  - Rebuild, run unit tests/property checks, then benchmark representative inputs.

## Technique Areas Overview

- Byte/Bit Hot‑Path Techniques (detailed below)
- Allocation & Memory Management (coming soon)
- Text/String Building Patterns (coming soon)
- Numeric/Arithmetic & Fixed‑Width Ops (coming soon)
- Data Structures & Algorithms (coming soon)
- Async/Await & Inter‑canister Patterns (coming soon)
- Candid/Serialization Efficiency (coming soon)
- Caching & Memoization (coming soon)

## Related Skills

- Benchmarks: skills/benchmarks-generation/SKILL.md
- Code Quality Cleanup: skills/code-improvements/SKILL.md
- Dot‑notation Migration: skills/dot-notation-migration/SKILL.md
- General Style: skills/motoko-general-style-guidelines/SKILL.md

---

## Byte/Bit Hot‑Path Techniques

These techniques target code that iterates over `Blob` or `[Nat8]`, performs bit packing/unpacking, or emits `Text` (e.g., encoders/decoders, checksums, binary parsers). They reduce allocations, avoid expensive integer widening, and reshape loops for better throughput while preserving correctness.

### When To Use

- You see hot loops over bytes and bit operations
- Code constructs `Text` character-by-character or via many small concatenations
- `Blob` is converted to `[Nat8]` only to iterate or index

### Prerequisites

- Motoko compiler (moc) at a reasonably recent version (1.3.0+ recommended)
- Familiarity with fixed-width integer types (`Nat8`, `Nat16`, `Nat32`, `Nat64`) and wrapping arithmetic (`+%`)
- For benchmarking guidance, see: skills/benchmarks-generation/SKILL.md

### Quick Wins Checklist (Heuristics)

Scan candidate code for these patterns and replace accordingly:

- Blob handling
  - If you see `let bytes = Blob.toArray(data)` just to iterate or index → Prefer indexing `Blob` directly: `data[i]`.
  - Cache `data.size()` to a local; choose an appropriate width (often `Nat64`).

- Integer conversions & bit ops
  - Avoid widening `Nat8 -> Nat` (arbitrary precision) for bit operations.
  - Use staged widening on fixed-width types: `b.toNat16().toNat32()` before shifts and masks.
  - Perform `<<`, `>>`, `&`, `|` on `Nat32`/`Nat64` intermediates, then narrow at the end if needed.
  - Use wrapping arithmetic `+%` for loop indices when overflow cannot occur by invariant; prefer a wide counter (e.g., `Nat64`).

- Alphabets / lookup tables
  - Avoid `Char`/`Text` tables in hot paths. Store alphabets as `[Nat8]` (UTF-8/ASCII codes) for direct byte emission.

- Text construction
  - Avoid per-character `Text.fromChar` plus repeated `#` concatenations.
  - Emit text in blocks: build a small ASCII `Blob` (e.g., 4–8 bytes), `Text.decodeUtf8` once, then append once per block.
  - Minimize `#` operations; 1 append per 4–8 output chars is far cheaper than many appends.

- Loop shape
  - Process input in uniform blocks (e.g., 3 bytes → 4 chars), then handle a small tail (0–2 bytes) separately.
  - Hoist invariants (sizes, constants); keep inner loops straight-line.

- Correctness guards
  - For theoretically unreachable states (e.g., ASCII decode failure), insert a `Prim.trap("…")` with a clear message.
  - Double-check off-by-one errors at block boundaries (e.g., `next_i <= sz`).

### Before → After Patterns

1) Avoid Blob→Array conversion

```motoko
// Before
let bytes = Blob.toArray(data);
var i = 0;
let b1 = bytes[i];

// After
let sz = Nat64.fromIntWrap(data.size());
var i : Nat64 = 0;
let b1 = data[i.toNat()];
```

2) Prefer fixed-width ints and staged widening

```motoko
// Before (goes through arbitrary-precision Nat)
let n = (Nat32.fromNat(Nat8.toNat(b1)) << 16)
      | (Nat32.fromNat(Nat8.toNat(b2)) << 8)
      |  Nat32.fromNat(Nat8.toNat(b3));

// After (fixed-width path)
let n = (b1.toNat16().toNat32() << 16)
      | (b2.toNat16().toNat32() << 8)
      |  b3.toNat16().toNat32();
```

3) Replace `Char`/`Text` alphabet + per‑char concat with `[Nat8]` + block decode

```motoko
// Before
private let alphabet : [Char] = ['A', 'B', /*…*/, '/'];
let c1 = Text.fromChar(alphabet[idx1]);
let c2 = Text.fromChar(alphabet[idx2]);
result #= c1 # c2 # c3 # c4; // many small concats

// After
private let alphabet : [Nat8] = [65, 66, /*…*/, 47]; // ASCII bytes
let bytes = Blob.fromArray([
  alphabet[idx1], alphabet[idx2], alphabet[idx3], alphabet[idx4]
]);
switch (Text.decodeUtf8(bytes)) {
  case (?t) { result := result # t } // one append per block
  case (_)  { Prim.trap("Cannot happen: Utf8 decode error …") }
};
```

4) Blocked processing with tail

```motoko
// Example: main loop over full blocks, then a small tail path
var i : Nat64 = 0;
var next_i : Nat64 = block; // e.g., 3, 6, etc.
while (next_i <= sz) {
  // read <block> bytes, produce <k> output chars
  i := next_i; next_i +%= block;
};
while (i < sz) {
  // read remaining bytes (tail), produce padded output as needed
  i +%= tailStep;
};
```

### Step‑by‑Step Procedure (Agent Playbook)

1. Scoping
   - Identify hot byte/bit paths: encoders/decoders, hashing, binary parsers.
   - If needed, prepare a small benchmark harness (see skills/benchmarks-generation/SKILL.md).

2. Baseline
   - Run a benchmark on realistic inputs (sequential bytes, random/mixed); record results.

3. Transformations (apply incrementally; keep each commit focused)
   - Remove `Blob.toArray` used solely for iteration or random access; index `Blob` directly.
   - Cache sizes; switch to `Nat64` index; use `+%` where safe.
   - Replace `Char`/`Text` alphabets with `[Nat8]` tables.
   - Convert per‑char concatenations into block emission using `Blob.fromArray` + `Text.decodeUtf8`.
   - Use `Nat16/Nat32/Nat64` intermediates for shifts and masks; avoid `Nat` widening in hot loops.
   - Reshape loops into large uniform blocks plus a small tail path.
   - Eliminate dead temporaries; prefer straight‑line code.
   - Add explicit `Prim.trap` for logically unreachable decode errors.

4. Validation
   - Unit tests: small ASCII examples, padding/edge cases, long sequential bytes; assert output length and alphabet membership where relevant.
   - Optional property checks: random inputs vs a reference implementation.
   - Benchmarks: verify improvements or parity across patterns and sizes (use skills/benchmarks-generation/SKILL.md).

### Common Pitfalls

1. Unnecessary widening to `Nat` in tight loops
   - Why it hurts: arbitrary-precision arithmetic is slower and allocates.
   - Fix: keep operations on `Nat32`/`Nat64` intermediates; stage `Nat8 -> Nat16 -> Nat32`.

2. Per-character `Text` operations
   - Why it hurts: `Text.fromChar` + repeated `#` creates many small allocations.
   - Fix: emit ASCII bytes into a `Blob` block and `Text.decodeUtf8` once per block, then append once.

3. Materializing `Blob` as `[Nat8]` to iterate
   - Why it hurts: full-buffer allocation and copy on the hot path.
   - Fix: index `Blob` directly and cache `size()`.

4. Overflow checks on loop counters
   - Why it hurts: extra checks per iteration when they are provably unnecessary.
   - Fix: widen counter to `Nat64` and use `+%` under a clear no-overflow invariant.

5. Off-by-one at block boundaries
   - Symptom: traps or incorrect output near the end of input.
   - Fix: use `next_i <= sz` for the main loop; handle the remaining tail explicitly.

### Safety & Edge Cases

- Wrapping arithmetic: Use `+%` only when an invariant guarantees no overflow in the chosen width; prefer widening to `Nat64`.
- `Text.decodeUtf8`: Safe for ASCII bytes (e.g., codec alphabets and `=` padding). Keep a defensive `Prim.trap` for auditability.
- Bounds: Cleanly separate the fast block loop and the tail; test edge sizes (e.g., 0, 1, 2, 3, 5, 6, 7 depending on block size).
- Allocation profile: Ensure you removed buffer materialization (`Blob.toArray`) and minimized concatenations.

### Validation Checklist (copy/paste)

- [ ] Tests cover: empty, minimal inputs, pad edges, long sequential bytes, multi-sentence ASCII where applicable.
- [ ] Output length formula holds (as defined by the codec or algorithm).
- [ ] Alphabet membership check passes; no stray chars (for encoders/decoders).
- [ ] Bench shows improvement or parity; no regressions for mixed vs zero patterns (see skills/benchmarks-generation/SKILL.md).
- [ ] No traps at block boundaries; indices don’t overflow.

### Commit Message Templates (optional)

Keep one change per commit when possible. Examples:

- Skip conversion from Blob to Array in encoder.
- Avoid unnecessary Char→Text conversion; use `[Nat8]` alphabet.
- Reuse `data.size()` and use `Nat64` indices.
- Convert `Nat8` to `Nat32` via `Nat16` for bit ops.
- Process input in uniform blocks and handle tail separately.
- Use `+%` for loop index under explicit invariant; widen index width.
- Replace many small `#` concatenations with a single append of a decoded block.
- Remove temporary `c1..c4` variables in tight loops.
- Trap in unreachable `decodeUtf8` error path.

### Outcome

Applying these techniques typically yields:
- Fewer allocations by avoiding full-buffer materialization and per-char `Text` ops
- Faster bit-packing/unpacking via fixed-width arithmetic
- Reduced concatenation overhead by emitting larger `Text` chunks
- Clearer separation of fast-path block processing and tail handling
- Safer, more auditable code via explicit invariants and traps
