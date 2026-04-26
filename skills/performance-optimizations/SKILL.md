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

---

## Block-Cipher / Hash-Function Hot Paths

Techniques in this section are specifically for **MD-style block hash functions and similar block-structured primitives** (RIPEMD-160, SHA-1/2, MD5, Blake, block ciphers, etc.). They are *not* general advice — most code shouldn't look like this.

### When These Techniques Apply

ALL of the following must be true to justify this style of optimization:

1. **Fixed-size input blocks processed by a transform function.** The algorithm specifies an N-byte block (e.g., 64 bytes for SHA-256/RIPEMD-160) that is repeatedly fed to a compression function.
2. **Many rounds per block, each updating a small fixed set of state words.** Typically 64–160 rounds operating on 4–8 `Nat32`/`Nat64` chaining variables. Tuple/record allocations on the per-round path dominate cost.
3. **Public streaming API** (`write`/`update` + `sum`/`finalize`) where partial blocks must be buffered between calls.
4. **Throughput matters.** The function is in a measurable hot path (e.g., signing, address derivation, large-message hashing).

If your code is one-shot, processes variable-size frames, or runs only a handful of times per request — **stop**. Use plain, readable code instead. Reference: `src/Ripemd160.mo` was rewritten for ~2× instructions and ~3.6× less GC at 1 KiB inputs; the techniques cost ~300 lines of inlined rounds and are only worth it for primitives that consumers depend on heavily.

### The Reference Pattern (research-ag/sha2 style)

The combined patterns below are the same shape used by `research-ag/sha2`'s SHA-256 and were applied successfully to `src/Ripemd160.mo`. Apply them together — partial application leaves most of the gains on the table.

#### Pattern 1 — Mutable chaining state in `[var Nat32]` (or `[var Nat64]`)

```motoko
// Persistent chaining state, single allocation reused across all blocks.
private let s : [var Nat32] = VarArray.repeat<Nat32>(0, 5);
```

Don't use individual `var` fields for the chaining variables when there are many of them — the array slot stores fixed-width words inline. Don't split into `Nat16` halves unless you measure a benefit; for ≤8 state words it's not worth it.

#### Pattern 2 — Pre-decoded message schedule in `[var Nat32]` (NOT `[var Nat8]`)

```motoko
// 16 little-endian (or big-endian) words for the current block.
// Bytes are folded in at write time; transform() reads words directly.
private let msg : [var Nat32] = VarArray.repeat<Nat32>(0, 16);
```

This replaces the common (and slow) pattern:

```motoko
// AVOID for hot-path block hashing
private let buf : [var Nat8] = ...;          // 64-byte buffer
transform(buf.toArray(), 0);                  // allocates a 64-byte [Nat8] PER BLOCK
// transform then re-decodes 4 bytes → Nat32 word internally
```

The `[var Nat32]` design eliminates both the per-block `toArray()` copy and the read-side decoding inside `transform()`.

#### Pattern 3 — Unboxed `Nat16` byte counter for partial-block position

```motoko
// 0..63 byte position within the current block.
// Nat16 is unboxed in mutable storage; Nat is heap-allocated per increment.
private var i_msg : Nat16 = 0;
```

Use `Nat16` (or `Nat8` if the range fits) for any small mutable counter on a hot path — `var x : Nat = 0` allocates a boxed bignum on every assignment. `Nat64` is also unboxed but wastes register width when the value is provably small.

#### Pattern 4 — `writeByte` folds bytes into the word schedule

```motoko
private func writeByte(b : Nat8) {
  let pos = i_msg;
  let wi = Nat16.toNat(pos >> 2);
  let lane = pos & 0x3;
  let v : Nat32 = Nat32.fromNat16(b.toNat16()) << Nat32.fromNat16(lane << 3);
  if (lane == 0) { msg[wi] := v }            // first byte: overwrite stale word
  else { msg[wi] := msg[wi] | v };           // subsequent bytes: OR in
  let next = pos +% 1;
  if (next == 64) { transform(); n_blocks +%= 1; i_msg := 0 }
  else { i_msg := next };
};
```

Adjust the lane shift formula for big-endian schedules (e.g., SHA-2): `(3 - lane) << 3`.

#### Pattern 5 — Fast-path block decoder in `write`

```motoko
public func write(data : [Nat8]) {
  let n = data.size();
  if (n == 0) return;
  var i = 0;

  // (1) Finish any partial block one byte at a time.
  while (i_msg != 0 and i < n) { writeByte(data[i]); i += 1 };

  // (2) Fast path: decode 16 LE words inline directly from input → msg.
  while (i + 64 <= n) {
    msg[0] := data[i].toNat16().toNat32()
            | (data[i+1].toNat16().toNat32() <<  8)
            | (data[i+2].toNat16().toNat32() << 16)
            | (data[i+3].toNat16().toNat32() << 24);
    // ... msg[1] .. msg[15] (15 more identical lines)
    transform();
    n_blocks +%= 1;
    i += 64;
  };

  // (3) Tail: remaining < 64 bytes go into the partial block.
  while (i < n) { writeByte(data[i]); i += 1 };
};
```

Three loops, not one. The fast path is what makes large inputs cheap.

#### Pattern 6 — Inline ALL rounds inside `transform()`

The single biggest win for multi-round primitives. The original RIPEMD-160 code looked like:

```motoko
// AVOID: each call allocates a (Nat32, Nat32) tuple — 320 tuples per block
let (a, c) = r11(a, b, c, d, e, msg, 0, 11);
let (e, b) = r11(e, a, b, c, d, msg, 1, 14);
// ... 158 more
```

Inline every round into a flat sequence of statement updates on local mutable vars:

```motoko
var a1 : Nat32 = s[0]; var b1 : Nat32 = s[1]; /* ... */
// Left line round 1: f1(b,c,d) = b ^ c ^ d, K = 0
a1 := rol(a1 +% (b1 ^ c1 ^ d1) +% w0, 11) +% e1; c1 := rol(c1, 10);
e1 := rol(e1 +% (a1 ^ b1 ^ c1) +% w1, 14) +% d1; b1 := rol(b1, 10);
// ... 158 more, with K and rotation amounts varying per round
// Combine back to s without allocating temporaries
let t = s[0];
s[0] := s[1] +% c1 +% d2;
// ...
```

Yes, this is hundreds of lines. Yes, it's mechanically transcribed from the spec or the original tuple-returning helpers. Comment each round group with its `f` and `K`. The compiler does not currently inline these helpers automatically, and the tuple allocation per call is real.

#### Pattern 7 — Boxing-free `Nat64` → `Nat8` for length encoding

Padding writes the bit-length as 8 little-endian bytes. The naive `Nat8.fromNat(Nat64.toNat(x & 0xff))` round-trips through arbitrary-precision `Nat`, allocating per byte:

```motoko
// Stage all narrowing on fixed-width types — no Nat allocation.
private func lowByte64(v : Nat64) : Nat8 {
  Nat8.fromNat16(Nat16.fromNat32(Nat32.fromNat64(v & 0xff)));
};
```

#### Pattern 8 — Padding via `writeByte`, not allocated `[var Nat8]`

```motoko
public func sum() : [Nat8] {
  let bitlen : Nat64 = ((n_blocks << 6) +% Nat64.fromNat(Nat16.toNat(i_msg))) << 3;
  writeByte(0x80);
  while (i_msg != 56) { writeByte(0) };
  writeByte(lowByte64(bitlen));
  writeByte(lowByte64(bitlen >> 8));
  // ... 6 more length bytes; the 8th triggers transform()
  // serialize state to 20/32 output bytes
};
```

This eliminates the typical `pad : [var Nat8]` and `sizedesc : [var Nat8]` allocations plus their `.toArray()` copies.

### Anti-Patterns To Remove When Adopting This Style

- `var counter : Nat = 0` on a per-byte path → use `Nat16` / `Nat64`.
- Returning multi-value tuples `(Nat32, Nat32)` from per-round helpers.
- Per-block `buf.toArray()` to hand bytes to `transform`.
- Allocating `pad` and `sizedesc` arrays in `sum`/`finalize`.
- Wrapping each round's bit operations with `Common.readLE32(arr, i)` calls when you can keep the words pre-decoded in `[var Nat32]`.
- Going through `Nat` for any byte-level conversion in the hot path.

### Operator-Precedence Caution

Motoko's `^` (XOR), `&`, `|`, `+%`, `<<`, `>>` precedences may not match C/Rust intuition. **Aggressively parenthesize** f-functions and round expressions:

```motoko
a1 := rol(a1 +% ((b1 & c1) | (^b1 & d1)) +% w0 +% 0x5A827999, 11) +% e1;
```

Don't trust the compiler to group `^b1 & d1` as `(^b1) & d1` if you haven't checked. Adding parens costs nothing at runtime.

### Validation Workflow (Required)

These optimizations are easy to mis-transcribe (160 round lines for RIPEMD-160 alone). Before claiming success:

1. **Full test suite must pass.** Hash test vectors are non-negotiable — a single wrong rotation amount, K constant, or message-word index will corrupt all outputs but may still produce stable-looking bytes.
2. **Benchmark against the previous version.** If instructions don't drop ≥1.5× and GC doesn't drop ≥2× at large inputs, something is wrong with the buffering or rounds are still allocating.
3. **Benchmark against an external reference** if one exists (e.g., another mops package). Confirms you're not just shuffling cost around.

### When NOT To Apply These Patterns

- **One-shot or low-frequency code.** A 5-line hash call invoked once per request: leave the readable version alone.
- **Algorithms with variable-length blocks** (e.g., compression, parsing). The `msg : [var Nat32]` schedule assumes fixed-width words at fixed positions.
- **Algorithms with few rounds** (e.g., `< 16`). Inlining is overkill; the tuple-allocation cost is small relative to other work.
- **Code that needs to remain easy to audit against a spec.** Inlined rounds are harder to read than a `for r in rounds` loop. For new primitives, write the clear version first, ship it, and only inline if benchmarks justify it.
- **Generic/parametric implementations.** This style hard-codes the algorithm; you can't easily share `transform()` across hash variants.

### Outcome (Reference: RIPEMD-160 in this repo)

Measured at 1024-byte input:

| Metric        | Before    | After   | Speedup |
| :------------ | --------: | ------: | ------: |
| Instructions  | 1,470,125 | 734,849 |   2.0×  |
| GC traffic    | 160.28 KiB|  43.88 KiB | 3.65×  |
| Heap (steady) |     272 B |   272 B |   1.0×  |

Public API and observable behavior unchanged; all 24 test files pass.
