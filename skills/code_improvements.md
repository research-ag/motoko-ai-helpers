---
name: Motoko mo:core Code Improvements Skill
description: Optional, modular cleanups and style improvements to apply on new mo:core projects (or after mo:core migration). Covers import ordering, dot‑notation, unused import cleanup, and single‑expression return removal, with detection checks and automation recipes.
type: reference
---

# Skill: Motoko mo:core Code Improvements (Optional)

---

## Purpose & Scope

Use this skill after test pass status to raise readability and consistency without changing behavior.

This skill focuses on mechanical, semantics‑preserving improvements:
- Aggregate imports into sections (1) mo:core/... (2) other mo:*/... from mops or similar third‑party sources (3) local project modules; sort each section alphabetically per file
- Prefer dot‑notation where available in `mo:core`
- Clean up truly unused `import` lines while respecting implicit needs created by dot‑notation
- Remove redundant `return` in single‑expression functions

Safety first:
- Run each improvement category independently; commit after each to isolate diffs
- Prefer scripted, reviewable changes; use audit checks provided below
- Rebuild after every category; run tests if present

---

## AI Quick Checklist (Do Not Skip)

1) Preconditions
- Project compiles on `mo:core` (see Migration Skill). Keep `mo:base` around only if still referenced; otherwise remove base dependency already.
- Ensure consistent Motoko and dfx versions per migration skill (moc ≥ 1.3.0, dfx ≥ 0.31).

2) Order of improvements (recommended)
- A. Remove `return` in single‑expression functions
- B. Convert to dot‑notation where available
- C. Ensure necessary `mo:core` imports exist for newly used dot‑notation
- D. Clean up unused imports (be conservative re: dot‑notation)
- E. Aggregate imports into three sections and sort each section alphabetically per file (1) `mo:core/...` (2) other `mo:*/...` from mops/third‑party (3) local project modules)

3) Verify after each step
- Build all canisters or packages
- Grep/audit with provided commands
- Keep diffs minimal and readable

Acceptance Criteria
- No compiler errors or warnings introduced by the changes (esp. missing imports)
- No behavior changes; public interfaces unchanged unless stylistic
- Imports are aggregated into three sections — (1) `mo:core/...`, (2) other `mo:*/...` from mops/third‑party, (3) local project modules — and each section is alphabetized; no truly unused `import`s remain
- Dot‑notation is consistently used where directly supported

---

## A) Remove `return` in single‑expression functions

Pattern
```motoko
// Before
func f(x : T) : U { return <expr>; };

// After
func f(x : T) : U { <expr> };
```

Notes
- Only apply when the function body consists of a single `return <expr>;` statement.
- Do not transform multi‑statement bodies or bodies that include `try`, `label`, `switch`, or `await` leading to different control flow.

Automation (example)
- Grep candidates: `grep -rn "func \\w\\+(.*) *:.*{ *return .*; *};" . --include="*.mo" | grep -v \.mops`
- Review matches; then apply with your editor or a scripted replacement.

---

## B) Prefer dot‑notation where available

Rationale
- `mo:core` exposes many APIs as methods usable via dot‑notation on values. This improves readability and local discoverability.

Examples (illustrative, not exhaustive)
```motoko
// Arrays
Array.fromVarArray  → Array.fromVarArray           // factory stays on module
Array.length(a)     → a.length()
Array.find(a, f)    → a.find(f)
Array.concat(a, b)  → a.concat(b)
Array.fromVarArray(v) → v.toArray()
Array.toVarArray(a)   → a.toVarArray()
Array.sliceToArray(a, s, e) → a.sliceToArray(s, e)

// VarArray
VarArray.repeat(x, n)  → VarArray.repeat(x, n)     // factory stays on module
VarArray.fromArray     → VarArray.fromArray        // factory stays on module
VarArray.size(v)       → v.size()
VarArray.clear(v)      → v.clear()

// Map (mutable in core)
Map.empty<K,V>()          → Map.empty<K,V>()       // factory stays on module
Map.size(m)               → m.size()
m.add(cmp, k, v)          // already method style in core
m.get(cmp, k)             // already method style in core
m.entries()               // iterator style

// Set (mutable in core)
Set.empty<T>()            → Set.empty<T>()         // factory stays on module
Set.size(s)               → s.size()

// List (mutable in core)
List.empty<T>()           → List.empty<T>()        // factory stays on module
List.add(l, x)            → l.add(x)               // mutates; returns void
List.get(l, i)            → l.get(i)

// Text
Text.toLower(t)           → t.toLower()
Text.toUpper(t)           → t.toUpper()
Text.size(t)              → t.size()
Text.contains(t, sub)     → t.contains(sub)
```

Cautions
- Some factory/constructor functions remain on the module (e.g., `Array.fromIter`, `Array.fromVarArray`, `VarArray.fromArray`, `Map.empty`). Do not force dot‑notation when it does not exist.
- Blob conversions have no dot‑notation: keep `Blob.fromArray` and `Blob.fromVarArray` as module calls; there is no `blob.toArray()` or similar. Ensure `import Blob "mo:core/Blob";` when using them.
- Ensure the backing module for a dot call is imported (see Section C and D below). Without an `import Array "mo:core/Array";` in scope, `a.length()` will fail, even if `Array.*` names are not otherwise referenced.

Audit helpers
- Grep for likely pre‑dot calls to convert:
  - `.values()` usage from migration: `grep -rn "\\.values()" . --include="*.mo" | grep -v \.mops`
  - Arrays: `grep -rn "Array\\.(length|find|concat|fromVarArray|toVarArray|sliceToArray)" . --include="*.mo" | grep -v \.mops`
  - Text: `grep -rn "Text\\.(toLower|toUpper|size|contains)" . --include="*.mo" | grep -v \.mops`

---

## C) Ensure necessary imports for dot‑notation

Why this matters
- Dot‑notation in Motoko requires the corresponding module be in scope. After converting `Array.length(a)` → `a.length()`, the `Array` module may appear unused by grep, but it is still required.

Minimal import mapping (keep if any such dot‑use appears)
- `a.length(), a.find(...), a.concat(...), a.sliceToArray(...)` → keep `import Array "mo:core/Array";`
- `v.size(), v.clear(), v.toArray()` on var arrays → keep `import VarArray "mo:core/VarArray";`
- `m.size(), m.entries(), m.add/get/remove/...` on core map → keep `import Map "mo:core/Map";`
- `s.size(), s.add/delete/contains/...` on core set → keep `import Set "mo:core/Set";`
- `l.add/get/...` on core list → keep `import List "mo:core/List";`
- `t.toLower(), t.toUpper(), t.size(), t.contains(...)` → keep `import Text "mo:core/Text";`

Heuristic script idea
- When cleaning imports (Section D), scan each file for these dot‑patterns. If present, mark the corresponding module as required even if no `Module.` references remain.

---

## D) Clean up unused imports (safely)

Goal
- Remove imports that are truly unused after prior refactors, but do not remove modules implicitly required by dot‑notation.

Reality check
- Editor tooling (VSCode Motoko extension) correctly marks unused imports, including dot‑notation awareness. CLI detection can be trickier.

Approaches
1) Editor‑guided
   - Open the workspace in VSCode. For each `*.mo` file, accept quick‑fix to remove imports marked as unused. Review diffs.

2) Compiler/LSP‑assisted batch
   - Use the Motoko language server via the VSCode extension to surface all diagnostics; apply code actions in batches where supported.

3) Script‑assisted conservative removal
   - Write a simple script that:
     - Parses each `import ... "mo:core/XYZ";`
     - Searches file for either `XYZ.` or any of the known dot‑patterns mapped to `XYZ` (Section C)
     - If neither is found, flag the line as removable
   - Manually review flagged lines before deletion

Audit helpers
- After cleanup, search for "import" lines whose module name never appears and no mapped dot‑pattern is present.
- Build the project. If a required module was removed, dot‑calls will fail at compile time — restore import and refine rules.

---

## E) Aggregate and alphabetize imports by section

Why
- Consistent ordering reduces merge conflicts and speeds reviews. Clear grouping improves scanning and avoids mixing external modules with local ones.

Sections (in this order, each separated by a single blank line)
1) mo:core imports
   - All imports whose path starts with "mo:core/..." (including `mo:core/Types`).
2) Other mo:* third‑party imports (mops or similar)
   - Any `mo:...` imports that are not `mo:core/...` (e.g., `mo:uuid/UUID`, `mo:sha2/SHA256`, etc.).
3) Local project modules
   - Relative path imports such as "../..." and "./...".

Sorting rules (apply within each section independently)
- Sort alphabetically by the quoted path string.
- Preserve import style (module vs. named type imports).
- Keep multiple named‑type imports from the same path on a single line as‑is.
- Optionally keep a comment header above each section (Core, Third‑party, Local) if your repo style prefers.

Example
```motoko
// Before (mixed)
import Runtime "mo:core/Runtime";
import { type Result } "mo:core/Types";
import SHA256 "mo:sha2/SHA256";
import Map "mo:core/Map";
import BitVec "mo:bitvec/BitVec";
import Utils "../lib/Utils";
import Logger "./Logger";

// After (aggregated and sorted per section)
//// Core
import Map "mo:core/Map";
import Runtime "mo:core/Runtime";
import { type Result } "mo:core/Types";

//// Third‑party (mops)
import BitVec "mo:bitvec/BitVec";
import SHA256 "mo:sha2/SHA256";

//// Local
import Logger "./Logger";
import Utils "../lib/Utils";
```

Lightweight automation idea (per file)
- Collect all import lines at the file top.
- Partition into the three sections by path prefix.
- Sort each partition by the quoted path.
- Re‑emit sections in the order Core → Third‑party → Local, with a blank line between sections.
- Keep any non‑import comments at their relative positions unless they clearly belong to a section header.

---

## Practical Automation Recipes (opt‑in)

These are optional starting points. Prefer editor‑integrated refactors when available. Always review diffs.

1) Find one‑line return functions
```bash
rg -n --glob '!**/.mops/**' --glob '**/*.mo' "func [A-Za-z_][A-Za-z0-9_]*\(.*\) *:.*\{ *return .*; *};"
```

2) Spot pre‑dot candidates
```bash
rg -n --glob '!**/.mops/**' --glob '**/*.mo' 'Array\.(length|find|concat|fromVarArray|toVarArray|sliceToArray)' 
rg -n --glob '!**/.mops/**' --glob '**/*.mo' 'Text\.(toLower|toUpper|size|contains)'
```

3) Flag possibly unused core imports (conservative)
```bash
# Rough heuristic: list imports, then search for name or dot‑patterns
rg -n --glob '!**/.mops/**' --glob '**/*.mo' '^import .*"mo:core/([A-Za-z/]+)";' -or '$1' -r ''
# For each file, ensure presence of module references OR mapped dot‑patterns before removal
```

4) Aggregate + sort imports into sections (editor macro)
- Select all `import` lines at the top of the file → group into three sections (Core, Third‑party mo:*, Local) → sort each group alphabetically by path → insert blank lines between sections → keep import styles as‑is.

---

## Agent Strategy (for AI assistants)

1) Confirm the project builds on `mo:core` before starting improvements.
2) Work file‑by‑file. For each file:
   - A. Remove single‑expression `return` forms
   - B. Convert obvious `Module.func(value, ...)` to `value.func(...)` where supported
   - C. Ensure required imports for any introduced dot‑notation remain
   - D. Remove truly unused imports (respect Section C mapping)
   - E. Aggregate imports into the three sections and sort each section alphabetically (Core → Third‑party mo:* → Local)
3) After each file: compile; if failure due to missing import, restore and mark mapping
4) After each category across repo: run a full build and optionally tests
5) Produce a short report of changes and any edge cases deferred for manual review

---

## Edge Cases & Gotchas

- Map/List/Set in `mo:core` are mutable. Dot‑notation like `list.add(x)` or `map.add(cmp, k, v)` mutates in place and returns `()`; do not write `:=` around such calls.
- Do not force dot‑notation on functions that are only on the module (constructors, converters). Keep module calls for those.
- Named type imports from `mo:core/Types` do not contribute to dot‑notation. Sorting rules should not intermix `type`‑only named imports with module imports unless you keep them within the same `mo:core` group ordering.
- Some symbols may be brought in under different aliases. Always map dot‑patterns to the actual path/module, not just the alias.

---

## Verification & Sign‑off

- Build all canisters successfully after changes.
- Run static audits:
  - No `import` lines flagged unused by editor or heuristic scripts (after accounting for dot‑notation needs).
  - Spot check: arrays, maps, sets, text operations use dot‑notation where natural.
- Diffs remain mechanical; no public API or behavioral changes.

---

## Appendix: Minimal Dot‑Notation Map (starter)

This list is intentionally short and safe. Extend as you confirm more methods in your codebase version of `mo:core`.

- Array (mo:core/Array)
  - `a.length()`
  - `a.find(f)`
  - `a.concat(b)`
  - `a.sliceToArray(s, e)`
  - `v.toArray()` / `a.toVarArray()` when `v` is VarArray / `a` is Array

- VarArray (mo:core/VarArray)
  - `v.size()`, `v.clear()`, `v.toArray()`, `v.push(x)` (if present in your version)

- Map (mo:core/Map)
  - `m.size()`, `m.entries()`, `m.values()`, `m.add(...)`, `m.get(...)`, `m.delete(...)`, `m.remove(...)`, `m.take(...)`

- Set (mo:core/Set)
  - `s.size()`, `s.add(...)`, `s.contains(...)`, `s.delete(...)`, `s.remove(...)`

- List (mo:core/List)
  - `l.add(x)`, `l.get(i)`

- Text (mo:core/Text)
  - `t.toLower()`, `t.toUpper()`, `t.size()`, `t.contains(sub)`

Update this list as your project confirms additional methods that support dot‑notation in your locked `mo:core` version.
