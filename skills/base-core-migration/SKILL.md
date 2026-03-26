---
name: motoko-base-to-core-migration
description: Complete, AI-ready playbook to migrate Motoko projects from mo:base to mo:core — phases, renames, data structure changes, agent strategy, verification scripts, upgrade tests, and production rollout.
---

# Skill: Motoko mo:base → mo:core Migration

## AI Quick Checklist (Do Not Skip)

1) Versions
- Ensure dfx 0.31+
- Confirm repo compiles before starting
- In mops.toml add core = "2.2.0"
- update moc in mops.toml to "1.3.0" in [toolchain] and [requirements] sections.
- if wasmtime is present in mops.toml, upgrade it to at least "42.0.1"
- Do NOT remove base dependency until Phase 5 is completed

2) dfx.json Flags (per canister)
- wasm_memory_persistence: keep

3) Mechanical Renames (Phases 1–2)
- Imports: mo:base/* → mo:core/* (with noted exceptions)
- Types-only: Prefer mo:core/Types for type-only imports (e.g., Iter.Iter<T> → Types.Iter<T>, Result.Result<T,E> → Types.Result<T,E>)
- Style: If importing ≤2 types from mo:core/Types, prefer named type imports (e.g., `import { type Result; type Iter } "mo:core/Types";`) and then use `Result<T>` / `Iter<T>` in code; if importing ≥3 types, import the whole module (`import Types "mo:core/Types";`).
- Methods: .vals() → .values(), Array.*, Option.*, Debug.trap → Runtime.trap, etc.

4) Data Structure Migrations (Phase 3)
- Buffer → List (MUTABLE, add() returns void)
- HashMap/TrieMap → Map (MUTABLE, add() returns void, Map.size(map))
- TrieSet → Set (MUTABLE, add() returns void)
- Eliminate all “:= Map.add/List.add/Set.add” — they now mutate in place

5) Persistent Actor (Phase 4)
- actor → persistent actor (including actor class)

6) Remove mo:base (Phase 5)
- Only after zero references remain and all canisters build

7) Stable Layout & Upgrades (Phase 6)
- Keep stable serialization arrays compatible or use with migration = syntax
- Test local upgrade with wasm_memory_persistence keep

8) Verification & Sign-off
- Build each canister individually
- Run regex audits to ensure critical mistakes are gone
- Compare DID interfaces where applicable
- Execute local upgrade test

Acceptance Criteria:
- grep for mo:base, Buffer., HashMap., TrieSet., Debug.trap, .vals(), and “:= Map.add/List.add/Set.add” all return zero (excluding .mops where appropriate)
- All canisters build
- Upgrade test keeps state (heap) with wasm_memory_persistence keep
- API compatibility checked for public methods

---

## Prerequisites

mops.toml (dependency staging)
- Add: core = "2.2.0"
- Keep base dependency present until Phase 5 is complete

dfx.json (per canister)
- type: "motoko"
- wasm_memory_persistence: "keep"

Requires:
- dfx 0.31+
- moc 1.3.0

---

## Architecture Shift: Enhanced Orthogonal Persistence (EOP)

Old (mo:base)
- stable var + preupgrade/postupgrade + manual serialization to stable memory

New (mo:core)
- persistent actor + transient var
- Heap is persisted; most structures live across upgrades without serialization

Rules inside persistent actor:
- let x = ... → stable by default
- var x = ... → stable by default
- transient var x = ... → NOT stable (resets on upgrade)
- stable keyword is implicit
- preupgrade/postupgrade still available when needed

---

## The 6 Phases

### Phase 1: Import Renames (mechanical)

Bulk rename:
- mo:base/X → mo:core/X (for most modules)
- Special cases:
    - mo:base/ExperimentalCycles → mo:core/Cycles
    - mo:base/ExperimentalInternetComputer → mo:core/InternetComputer
    - mo:base/List → mo:core/pure/List (immutable list; not mo:core/List)
    - mo:base/OrderedMap → mo:core/pure/Map
    - mo:base/OrderedSet → mo:core/pure/Set
    - mo:base/Deque → mo:core/pure/Queue

#### Types-only imports (mo:core/Types)

- When you only need a type and not the functions from its module, import `Types` instead of that module.
- Most common cases to fix explicitly:
    - Replace `import Iter "mo:core/Iter"` used only for the type `Iter.Iter<T>` with `import Types "mo:core/Types"` and update `Iter.Iter<T>` to `Types.Iter<T>`.
    - Replace `import Result "mo:core/Result"` used only for the type `Result.Result<T,E>` with `import Types "mo:core/Types"` and update `Result.Result<T,E>` to `Types.Result<T,E>`.
- Keep it simple: only fix these two patterns unless you clearly see another types-only import.
- Audit (to find likely spots):
    - `grep -rn "Iter\.Iter<" . --include="*.mo" | grep -v \.mops`
    - `grep -rn "Result\.Result<" . --include="*.mo" | grep -v \.mops`
- Minimal fix pattern:
    - If no `Iter.*` functions are called, change `import Iter "mo:core/Iter"` to `import Types "mo:core/Types"` and rewrite the type to `Types.Iter<...>`.
    - Similarly for `Result`, when only the `Result.Result<...>` type is used.

- Preferred import style for small sets of types:
    - If you need no more than 2 types from `mo:core/Types`, import them by name as types and then use them directly in code without a prefix.
    - Example:
      ```motoko
      // Before (module import or prefixed usage)
      import Types "mo:core/Types";
      type R = Types.Result<Nat, Text>;
      type I = Types.Iter<Nat>;
  
      // After (≤2 types: direct named type import)
      import { type Result; type Iter } "mo:core/Types";
      type R = Result<Nat, Text>;
      type I = Iter<Nat>;
      ```
    - If you import 3 or more types from `Types`, import the whole module: `import Types "mo:core/Types";` and use `Types.Result<...>`, `Types.Iter<...>`, etc.

Use sed commands to replace all occurrences at once

Removed without direct replacement (manual fix required):
- AssocList, Buffer, Hash, HashMap, Heap, IterType, None, Prelude, RBTree, Trie, TrieMap, TrieSet

Audit:
- grep -rl 'mo:base' . --include="*.mo" | grep -v \.mops

### Phase 2: Function Renames (mechanical)

Common mappings:
- Debug.trap(msg) → Runtime.trap(msg) (import Runtime "mo:core/Runtime")
- Prelude.unreachable() → Runtime.unreachable() (import Runtime "mo:core/Runtime")
- .vals() → .values()
- Array.append(a, b) → Array.concat(a, b)
- Array.init<T>(size, val) → VarArray.repeat<T>(val, size)  (args reversed)
- Array.freeze(arr) → Array.fromVarArray(arr)
- Array.thaw(arr) → Array.toVarArray(arr)
- Array.slice(a, s, e) → Array.range(a, s, e)
- Array.subArray(a, s, l) → Array.sliceToArray(a, s, s + l)
- Array.make(x) → Array.singleton(x)
- Array.mapFilter → Array.filterMap
- Array.chain → Array.flatMap
- Iter.range(a, b) → Nat.range(a, b + 1) (exclusive upper bound now)
- Iter.revRange(a, b) → Nat.rangeByInclusive(a, b, -1)
- Text.toLowercase → Text.toLower
- Text.toUppercase → Text.toUpper
- Text.translate → Text.flatMap
- Option.make → Option.some
- Option.iterate → Option.forEach
- Float.equal(a,b) → Float.equal(a,b,epsilon)
- Cycles.add(n); await call(args) → use “await (with cycles = n) call(args)”

Audit:
- grep -rn "\.vals()" . --include="*.mo" | grep -v \.mops

### Phase 3a: Buffer → List (MUTABLE, biggest gotcha)

- List in mo:core is MUTABLE; List.add(list, x) returns void and mutates in place
- Never write: list := List.add(list, x)
- Accessors:
    - List.at(list, i) → T (traps on OOB)
    - List.get(list, i) → ?T (safe)
- Conversions:
    - List.toArray(list), List.fromArray(arr)
- Prefer stable let list = List.empty<T>() for persistent state

### Phase 3b: HashMap/TrieMap → Map (MUTABLE)

- Map is MUTABLE; Map.add(map, compare, k, v) returns void
- Never write: map := Map.add(map, ...)
- API patterns with dot notation:
    - Map.empty<K,V>()
    - map.add(cmp, k, v)
    - map.get(cmp, k) → ?V
    - map.remove(cmp, k) → void
    - map.delete(cmp, k) → Bool
    - map.take(cmp, k) → ?V
    - map.entries(), map.values(), map.size()
- "cmp" is a key compare function, e.g., Text.compare, Nat.compare

### Phase 3c: TrieSet → Set (MUTABLE)

- Set.add(set, compare, x) returns void
- Set.contains(set, compare, x) → Bool
- Set.delete returns Bool; Set.remove is void
- Prefer stable let set = Set.empty<T>()

### Phase 3d: Dot notation

For dot‑notation details and safe patterns, see the Skill “Motoko mo:core Code Improvements” — Section B (Prefer dot‑notation) and Section C (Ensure necessary imports for dot‑notation).

Short rule: Prefer dot‑notation where supported by `mo:core` APIs; keep module‑level factory/constructor calls when no method exists (e.g., `Blob.fromArray` / `Blob.fromVarArray`). Run this step after the basic renames so behavior stays unchanged.

### Phase 4: persistent actor Keyword

- actor → persistent actor
- actor class → persistent actor class

Audit:
- grep -rn "^actor\b\|^  actor\b" . --include="*.mo" | grep -v persistent

### Phase 5: Remove mo:base From Dependencies

Only after:
- All files compile with mo:core
- No references to mo:base in your own code

Audit:
- grep -r 'mo:base' . --include="*.mo" | grep -v \.mops
- If zero, remove base = "..." from mops.toml

Note:
- Third-party .mops deps may still use mo:base; skip them

### Phase 6: Stable Variable Migration (Production-Critical)

If changing serialization strategy or types:
- Keep stable arrays of simple types compatible across versions; reuse same var names/types
- Use transient var for in-memory Map/List/Set that you reconstruct on upgrade
- Use preupgrade/postupgrade to serialize/deserialize when necessary
- For incompatible changes, consider the with migration = syntax to translate old state to new

---

## The 7 Critical Errors (and Their Fixes)

1) := Map.add / := List.add / := Set.add
- Symptom: type error [M0098], cannot implicitly instantiate function of type
- Fix: Remove :=; these mutate in place and return void

2) Array.init → VarArray.repeat (args reversed)
- Fix: VarArray.repeat<T>(val, size)

3) Cycles.add removed
- Fix: await (with cycles = n) call(args)

4) Garbled automated edits
- Symptom: stray constructor or type fragments near Map.empty/List.empty
- Fix: Clean declarations to simple Map.empty<K,V>() / List.empty<T>()

5) .values() as module functions
- Fix: List.values(list), Map.values(map), not l.values() or m.values()

6) Nat8 inference on hex literals
- Fix: Explicit type annotations, e.g., 0xB5 : Nat8

7) Error.message vs Runtime.message
- Fix: Keep Error.message(e) from mo:core/Error; only Debug.trap → Runtime.trap

---

## Agent Strategy (Scale to 100+ Files)

Safe for automated pass:
- Phase 1 (import renames)
- Phase 2 (function renames)
- Simple Buffer→List in utilities

Needs careful/manual review:
- Phase 3 Map migrations in actor mains
- preupgrade/postupgrade files (stable layout critical)
- Any Cycles.add usage
- Third-party .mops packages

Agent Prompt Template:
- Migrate [FILE] from mo:base to mo:core. CRITICAL RULES:
    1. Map.add/List.add/Set.add return VOID — remove all := assignments
    2. VarArray.repeat(val, size) — args REVERSED from Array.init(size, val)
    3. List/Map/Set mutate in-place — prefer stable let for persistent data
    4. Error.message stays as Error.message — NOT Runtime.message
    5. Add explicit type params when compiler gives M0098
- Show me the diff before writing.

---

## Build and Verification Workflow

Per-canister build (fast iteration):
- dfx build <canister> 2>&1 | head -30

Migration completeness (must be zero before Phase 5):
- grep -rn "mo:base" . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn "HashMap\." . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn "Buffer\." . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn "TrieSet\." . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn "Debug\.trap" . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn "\.vals()" . --include="*.mo" | grep -v \.mops | wc -l
- grep -rn ":= Map\.add\|:= List\.add\|:= Set\.add" . --include="*.mo" | wc -l

API compatibility:
- Compare generated .did files with baseline and/or run targeted calls

---

## Local Upgrade Test (dfx)

Build both WASMs:
- Old (baseline) and new (migrated)

Install and upgrade locally:
- Ensure the .most file is in place for local EOP testing
- Use --wasm-memory-persistence keep on upgrade
- Verify module hash changes
- Populate state in old, upgrade to new, verify state persists and APIs work

---

## Mainnet Deployment Order (Canary Strategy)

- Deploy low-risk canisters first; verify each before proceeding
- Representative canisters early to surface systemic issues
- High-value/complex canisters last

After each deploy:
- API compatibility checks
- Domain-specific verification (crypto/math where applicable)

---

## Quick Reference: mo:core APIs

Map
- Map.empty<K,V>() → Map<K,V>
- Map.add(m, cmp, k, v) → void
- Map.get(m, cmp, k) → ?V
- Map.remove(m, cmp, k) → void
- Map.delete(m, cmp, k) → Bool
- Map.take(m, cmp, k) → ?V
- Map.swap(m, cmp, k, v) → ?V
- Map.insert(m, cmp, k, v) → Bool
- Map.size(m) → Nat
- Map.entries(m) → Iter<(K,V)>
- Map.containsKey(m, cmp, k) → Bool
- Map.clear(m) → void
- Map.fromIter(iter, cmp) → Map<K,V>

List
- List.empty<T>() → List<T>
- List.add(l, x) → void
- List.addAll(l, iter) → void
- List.size(l) → Nat
- List.get(l, i) → ?T
- List.at(l, i) → T
- List.put(l, i, v) → void
- List.toArray(l) → [T]
- List.fromArray(arr) → List<T>
- List.values(l) → Iter<T>
- List.removeLast(l) → ?T
- List.clear(l) → void
- List.sort(l, compare) → void

Set
- Set.empty<T>() → Set<T>
- Set.add(s, cmp, x) → void
- Set.contains(s, cmp, x) → Bool
- Set.remove(s, cmp, x) → void
- Set.delete(s, cmp, x) → Bool
- Set.insert(s, cmp, x) → Bool
- Set.size(s) → Nat
- Set.values(s) → Iter<T>
- Set.fromIter(iter, cmp) → Set<T>

VarArray (mutable arrays)
- VarArray.repeat<T>(val, size) → [var T]
- VarArray.tabulate<T>(n, f) → [var T]
- Use Array.fromVarArray/Array.toVarArray for conversions

---

## Lessons Learned

1) Read mo:core sources directly when in doubt; public docs may lag.
2) Build one canister at a time for faster feedback.
3) “:= void” is the #1 migration error; search for it explicitly.
4) Third-party packages often require hands-on patches in .mops.
5) EOP + persistent actor typically removes the need for serialization; still use stable arrays when upgrading across versions with changed structures.
6) Local upgrades require the .most file and --wasm-memory-persistence keep.
7) Use -y/--yes for non-interactive upgrades and verify module hash after.

---

## AI Runbook (Step-by-Step)

1) Prep
- Branch, confirm versions, add mo:core, set dfx flags

2) Phase 1
- Systematic import renames
- Exceptions for pure modules (immutable)

3) Phase 2
- Method/function renames per mapping list
- Fix Iter ranges and Text casing changes

4) Phase 3
- Buffer → List; remove all := with List.add
- HashMap/TrieMap → Map; add compare at every call; remove :=; switch to Map.size(map)
- TrieSet → Set; remove :=; update membership to Set.contains

5) Phase 4
- Convert actor/actor class to persistent actor

6) Build + Audit
- Per-canister build
- Grep audits for mo:base, Debug.trap, .vals(), and := add calls

7) Fix Third-Party
- Patch .mops packages as needed; re-run builds

8) Phase 5
- Remove mo:base from mops.toml once audits are clean

9) Phase 6
- Ensure state layout continuity or implement with migration
- Local upgrade test with wasm_memory_persistence keep

10) Sign-off
- All builds passing, audits zero, upgrade verified, API checked
