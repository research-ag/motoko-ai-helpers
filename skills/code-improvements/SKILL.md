---
name: motoko-core-code-improvements
description: Optional, modular cleanups and style improvements to apply on new mo:core projects (or after mo:core migration). Covers import ordering, unused import cleanup, and single‑expression return removal, with detection checks and automation recipes.
---

## Purpose & Scope

Use this skill after test pass status to raise readability and consistency without changing behavior.

This skill focuses on mechanical, semantics‑preserving improvements:
- Aggregate imports into sections (1) mo:core/... (2) other mo:*/... from mops or similar third‑party sources (3) local project modules; sort each section alphabetically per file
- Prefer dot‑notation where available in `mo:core`
- Clean up truly unused `import` lines while respecting implicit needs created by dot‑notation
- Remove redundant `return` in single‑expression functions
- Use direct string‑to‑Blob assignment for constant ASCII strings where appropriate

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
- B. Convert to dot‑notation where available — see Motoko Dot‑Notation Migration Skill (`skills/dot-notation-migration/SKILL.md`)
- C. Ensure necessary `mo:core` imports for dot‑notation — see Motoko Dot‑Notation Migration Skill (import mapping)
- D. Clean up unused imports (be conservative re: dot‑notation)
- E. Aggregate imports into three sections and sort each section alphabetically per file (1) `mo:core/...` (2) other `mo:*/...` from mops/third‑party (3) local project modules)
- F. Use direct string-to-Blob assignment for constant ASCII strings where appropriate

3) Verify after each step
- Build all canisters or packages
- Grep/audit with provided commands
- Keep diffs minimal and readable

Acceptance Criteria
- No compiler errors or warnings introduced by the changes (esp. missing imports)
- No behavior changes; public interfaces unchanged unless stylistic
- Imports are aggregated into three sections — (1) `mo:core/...`, (2) other `mo:*/...` from mops/third‑party, (3) local project modules — and each section is alphabetized; no truly unused `import`s remain
- Dot‑notation is consistently used where directly supported
- Constant Text strings are assigned directly to `Blob` without redundant `Text.encodeUtf8` calls

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
- A function with multiple `return` statements (e.g., early returns in `switch` cases like `return null`) must NOT have any returns removed.
- **`return switch (...) { ... }` at the end of a function is OK.** Each case block ends with the expression being returned (no `return` keyword inside the cases). This is the preferred style when all cases produce values normally.
  ```motoko
  // OK — all cases produce values, no traps or throws
  return switch (decode(data)) {
    case (#ok key) { (key.x, key.y) };
    case (#err msg) { #err(msg) };
  };
  ```
- **Exception: if any case has a return-equivalent statement (`Runtime.trap` or `throw`) that represents a genuine function exit, pull `return` inside the case blocks.** Remove `return` before `switch`, then add `return` only in the case blocks that produce values. Cases with `Runtime.trap` or `throw` do not need `return`. This makes it clear which branches return and which abort:
  ```motoko
  // Avoid — trap is hidden inside return switch
  return switch (decode(data)) {
    case (#ok key) { (key.x, key.y) };
    case (#err msg) { Runtime.trap(msg) };
  };

  // Prefer — return only in value-producing cases
  switch (decode(data)) {
    case (#ok key) { return (key.x, key.y) };
    case (#err msg) { Runtime.trap(msg) };
  };
  ```
- **Unreachable traps are NOT return-equivalent.** If `Runtime.trap("unreachable")` is used, or a comment or message indicates the branch is only reachable through a bug, that trap is not a normal function exit — it's a defensive assertion. In that case, `return switch` is fine and `return` should NOT be pulled inside:
  ```motoko
  // OK — the trap just guards an impossible case
  return switch (Jacobi.fromNat(x, y, 1, curve)) {
    case (null) Runtime.trap("unreachable");
    case (?point) point;
  };
  ```

Automation (example)
- Grep candidates: `grep -rn "func \\w\\+(.*) *:.*{ *return .*; *};" . --include="*.mo" | grep -v \.mops`
- Review matches; then apply with your editor or a scripted replacement.

Script Safety Requirements (learned from real migration)
- A simple line-based return counter is NOT sufficient. You must track function boundaries using brace-depth parsing at the character level.
- **Nested functions**: When a function body contains nested `func` declarations, skip the nested function's body entirely — only count returns at the direct (outermost) function scope.
- **Accurate function boundary detection**: Use character-level scanning that handles string literals (skip `"..."` including `\"` escapes), comments (`//` line comments and `/* ... */` block comments), and tracks brace depth to find the true closing `}` of each function.
- **Counting rule**: Count every `return` anywhere inside the outer function body, including inside nested control-flow blocks such as `switch`, `if`, `for`, and `while`, but excluding any `return` inside nested `func` bodies. A function is safe to rewrite only if this total count is exactly 1; if it has more than 1, skip the entire function.

Battle-tested Python script

Save as `remove_returns.py` in the project root, run with `python3 remove_returns.py`, then delete the script.

```python
#!/usr/bin/env python3
"""
Remove terminal `return` from Motoko functions that have exactly one return
statement in their direct body (excluding nested functions).

Usage:
  1. Set src_dirs to match your project layout.
  2. Run: python3 remove_returns.py
  3. Run tests: npx mops test
  4. Delete this script after confirming all tests pass.

Safety:
  - Tracks function boundaries via character-level brace-depth parsing.
  - Skips string literals ("..." with \" escapes) and comments (// and /* */).
  - Detects nested `func` declarations and skips their bodies entirely.
  - Only removes the `return` keyword (+ trailing space) from functions with
    exactly 1 return at any depth within the direct body.
"""

import re
import glob
import os

# ── Configuration ──────────────────────────────────────────────────────
# Directories to process (relative to script location or cwd).
SRC_DIRS = ["src", "test", "bench"]
# ──────────────────────────────────────────────────────────────────────


def find_func_bodies(text):
    """
    Yield (body_start, body_end) for every top-level and nested function
    found in `text`. body_start is the index of the opening '{' of the
    function body; body_end is the index of the matching closing '}'.

    Handles:
      - String literals (skips content inside "...")
      - Line comments (// ...)
      - Block comments (/* ... */)
      - Nested braces
    """
    i = 0
    n = len(text)
    while i < n:
        # Skip string literals
        if text[i] == '"':
            i += 1
            while i < n and text[i] != '"':
                if text[i] == '\\':
                    i += 1  # skip escaped char
                i += 1
            i += 1  # skip closing "
            continue

        # Skip line comments
        if text[i] == '/' and i + 1 < n and text[i + 1] == '/':
            i += 2
            while i < n and text[i] != '\n':
                i += 1
            continue

        # Skip block comments
        if text[i] == '/' and i + 1 < n and text[i + 1] == '*':
            i += 2
            while i < n and not (text[i] == '*' and i + 1 < n and text[i + 1] == '/'):
                i += 1
            i += 2  # skip */
            continue

        # Look for 'func' keyword at a word boundary
        if text[i:i+4] == 'func' and (i == 0 or not text[i-1].isalnum() and text[i-1] != '_'):
            after = text[i+4:i+5] if i + 4 < n else ''
            if after == '' or not (after.isalnum() or after == '_'):
                # Found a func keyword. Scan forward to find the opening '{'.
                j = i + 4
                while j < n:
                    if text[j] == '"':
                        j += 1
                        while j < n and text[j] != '"':
                            if text[j] == '\\':
                                j += 1
                            j += 1
                        j += 1
                        continue
                    if text[j] == '{':
                        # Found the opening brace of the function body.
                        brace_start = j
                        depth = 1
                        j += 1
                        while j < n and depth > 0:
                            if text[j] == '"':
                                j += 1
                                while j < n and text[j] != '"':
                                    if text[j] == '\\':
                                        j += 1
                                    j += 1
                                j += 1
                                continue
                            if text[j] == '/' and j + 1 < n and text[j + 1] == '/':
                                j += 2
                                while j < n and text[j] != '\n':
                                    j += 1
                                continue
                            if text[j] == '/' and j + 1 < n and text[j + 1] == '*':
                                j += 2
                                while j < n and not (text[j] == '*' and j + 1 < n and text[j + 1] == '/'):
                                    j += 1
                                j += 2
                                continue
                            if text[j] == '{':
                                depth += 1
                            elif text[j] == '}':
                                depth -= 1
                            j += 1
                        brace_end = j - 1  # index of closing '}'
                        yield (brace_start, brace_end)
                        i = j
                        break
                    if text[j] == '=' or text[j] == ';':
                        # func ... = expr; (no body) or forward decl
                        i = j + 1
                        break
                    j += 1
                else:
                    i = j
                continue
        i += 1


def count_returns_in_direct_body(text, body_start, body_end):
    """
    Count `return` statements that are directly inside this function body
    (not inside nested functions). Returns list of (return_keyword_start,
    return_keyword_end) positions.
    """
    body = text[body_start + 1 : body_end]  # content between { and }
    offset = body_start + 1

    # First, find all nested func bodies within this body so we can skip them.
    nested_ranges = []
    for ns, ne in find_func_bodies(body):
        # Adjust to absolute positions
        nested_ranges.append((ns + offset, ne + offset))

    def is_inside_nested(pos):
        for ns, ne in nested_ranges:
            if ns <= pos <= ne:
                return True
        return False

    # Now scan for `return` keywords in the body, skipping nested funcs.
    returns = []
    i = 0
    while i < len(body):
        # Skip strings
        if body[i] == '"':
            i += 1
            while i < len(body) and body[i] != '"':
                if body[i] == '\\':
                    i += 1
                i += 1
            i += 1
            continue

        # Skip line comments
        if body[i] == '/' and i + 1 < len(body) and body[i + 1] == '/':
            i += 2
            while i < len(body) and body[i] != '\n':
                i += 1
            continue

        # Skip block comments
        if body[i] == '/' and i + 1 < len(body) and body[i + 1] == '*':
            i += 2
            while i < len(body) and not (body[i] == '*' and i + 1 < len(body) and body[i + 1] == '/'):
                i += 1
            i += 2
            continue

        # Check for 'return' keyword
        if body[i:i+6] == 'return' and (i == 0 or not body[i-1].isalnum() and body[i-1] != '_'):
            after = body[i+6:i+7] if i + 6 < len(body) else ''
            if after == '' or not (after.isalnum() or after == '_'):
                abs_pos = i + offset
                if not is_inside_nested(abs_pos):
                    returns.append((abs_pos, abs_pos + 6))
                i += 6
                continue
        i += 1

    return returns


def process_file(filepath):
    with open(filepath, 'r') as f:
        text = f.read()

    original = text
    removals = 0

    # Collect all function bodies
    func_bodies = list(find_func_bodies(text))

    # For each function, check if it has exactly 1 return in its direct body
    # Process in reverse order to preserve indices when editing
    edits = []  # list of (start, end) of "return " to remove

    for body_start, body_end in func_bodies:
        returns = count_returns_in_direct_body(text, body_start, body_end)
        if len(returns) == 1:
            ret_start, ret_end = returns[0]
            # Remove "return " (keyword + trailing space)
            if ret_end < len(text) and text[ret_end] == ' ':
                edits.append((ret_start, ret_end + 1))
            else:
                edits.append((ret_start, ret_end))

    # Apply edits in reverse order to preserve positions
    edits.sort(key=lambda x: x[0], reverse=True)
    for start, end in edits:
        text = text[:start] + text[end:]
        removals += 1

    if text != original:
        with open(filepath, 'w') as f:
            f.write(text)

    return removals


def main():
    total = 0
    files_modified = 0
    for src_dir in SRC_DIRS:
        for filepath in sorted(glob.glob(os.path.join(src_dir, "**/*.mo"), recursive=True)):
            r = process_file(filepath)
            if r > 0:
                print(f"  {filepath}: {r} returns removed")
                total += r
                files_modified += 1
    print(f"\nTotal: {total} returns removed in {files_modified} files")


if __name__ == '__main__':
    main()
```

---

## B) Dot‑notation conversion

For all Motoko dot‑notation rules, automation scripts, and pitfalls, see the dedicated skill:
- skills/dot-notation-migration/SKILL.md

This file intentionally does not duplicate those instructions. Apply dot‑notation changes using the dedicated skill, then continue here with import cleanup (Section D) and import ordering (Section F).

---

## C) Dot‑notation import requirements

For import mapping and rules related to dot‑notation, use the dedicated skill:
- skills/dot-notation-migration/SKILL.md

This file intentionally does not duplicate the import mapping. After applying dot‑notation changes per that skill, proceed with Section D (unused import cleanup) and Section F (import ordering).

---

## D) Clean up unused imports (safely)

Goal
- Remove imports that are truly unused after prior refactors, but do not remove modules implicitly required by dot‑notation.

Reality check
- Editor tooling (VSCode Motoko extension) correctly marks unused imports, including dot‑notation awareness. CLI detection can be trickier.

Common false positives (imports that LOOK unused but are REQUIRED)
- `Blob` — needed when `.toArray()`, `.size()`, `.isEmpty()`, `.hash()` are called on `Blob` values (e.g., `Sha256.fromArray(...).toArray()`, `hmac.sum().toArray()`)
- `Array` — needed when `.flatten()`, `.foldLeft()`, `.sliceToArray()`, `.map()`, `.filter()` etc. are called on `[T]` values (e.g., `[arr1, arr2].flatten()`)
- `Nat` — needed when `.toText()` is called on `Nat` values from `.size()` (e.g., `arr.size().toText()`)
- `VarArray` — needed when `.toArray()` is called on `[var T]` values
- **Rule**: If ANY dot-notation method is called on a value of that module's type, the import is required even though the module name never appears explicitly in the code.

Approaches
1) Editor‑guided
    - Open the workspace in VSCode. For each `*.mo` file, accept quick‑fix to remove imports marked as unused. Review diffs.

2) Compiler/LSP‑assisted batch
    - Use the Motoko language server via the VSCode extension to surface all diagnostics; apply code actions in batches where supported.

3) Script‑assisted conservative removal
    - Write a simple script that:
        - Parses each `import ... "mo:core/XYZ";`
        - Searches file for either `XYZ.` or any of the known dot‑patterns mapped to `XYZ` (see Dot‑Notation Migration Skill import mapping)
        - If neither is found, flag the line as removable
    - Manually review flagged lines before deletion

Audit helpers
- After cleanup, search for "import" lines whose module name never appears and no mapped dot‑pattern is present.
- Build the project. If a required module was removed, dot‑calls will fail at compile time — restore import and refine rules.

---

## E) Shorten local (sibling) import paths

-   Use bare module names for local imports when possible: `"Bech32"` instead of `"./Bech32"`. Both resolve correctly, but bare names are more concise and idiomatic.

    ``` motoko no-repl
    // Good
    import Bech32 "Bech32";
    import Script "Script";
    import Types "Types";

    // Acceptable (cross-directory)
    import ByteUtils "../ByteUtils";
    import Curves "../ec/Curves";

    // Avoid (unnecessary ./ prefix for siblings)
    import Bech32 "./Bech32";
    ```

-   For cross-directory imports, relative paths with `../` are required and acceptable.

---

## F) Aggregate and alphabetize imports by section

Why
- Consistent ordering reduces merge conflicts and speeds reviews. Clear grouping improves scanning and avoids mixing external modules with local ones.

Sections (in this order, each separated by a single blank line)
1) mo:core imports
    - All imports whose path starts with "mo:core/..." (including `mo:core/Types`).
2) Other mo:* third‑party imports (mops or similar)
    - Any `mo:...` imports that are not `mo:core/...` (e.g., `mo:uuid/UUID`, `mo:sha2/SHA256`, etc.).
3) Local project modules
    - Bare module name imports like `"Bech32"`, `"Common"` (preferred), or relative path imports like `"../ByteUtils"`, `"./Script"`.
    - Prefer bare module names without `./` prefix for sibling imports (e.g., `"Bech32"` instead of `"./Bech32"`). Both work, but bare names are cleaner.
    - Sort local project module imports by the local name they are imported as, not by the module name in the imported path.

Sorting rules (apply within each section independently)
- Sort alphabetically by the local name modules are imported as
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
import Logger "Logger";
import Utils "../lib/Utils";
```

Lightweight automation idea (per file)
- Collect all import lines at the file top.
- Partition into the three sections by path prefix.
- Sort each partition by the quoted path.
- Re‑emit sections in the order Core → Third‑party → Local, with a blank line between sections.
- Keep any non‑import comments at their relative positions unless they clearly belong to a section header.

---

## G) Direct string‑to‑Blob assignment for constants

Pattern
```motoko
// Before
let b : Blob = Text.encodeUtf8("hello");

// After
let b : Blob = "hello";
```

Why
- For constant Text strings, the Motoko compiler allows direct assignment to the `Blob` type.
- The result is identical to `Text.encodeUtf8`, but the code is cleaner and avoids an explicit function call.

Examples
```motoko
// Good: direct assignment
let blobs = [
  "strategy",
  Text.encodeUtf8(Nat.toText(slot)),
];

// Avoid: redundant encoding for constant string
let blobs = [
  Text.encodeUtf8("strategy"),
  Text.encodeUtf8(Nat.toText(slot)),
];
```
---

## Practical Automation Recipes (opt‑in)

These are optional starting points. Prefer editor‑integrated refactors when available. Always review diffs.

1) Find one‑line return functions
```bash
rg -n --glob '!**/.mops/**' --glob '**/*.mo' "func [A-Za-z_][A-Za-z0-9_]*\(.*\) *:.*\{ *return .*; *};"
```

2) Dot‑notation conversion & candidate detection
- See the dedicated skill for full automation and grep recipes:
    - skills/dot-notation-migration/SKILL.md

3) Flag possibly unused core imports (conservative)
```bash
# Rough heuristic: list imports, then search for name or dot‑patterns
rg -n --glob '!**/.mops/**' --glob '**/*.mo' '^import .*"mo:core/([A-Za-z/]+)";' -o -r '$1'
# For each file, ensure presence of module references OR mapped dot‑patterns before removal
```

4) Aggregate + sort imports into sections (editor macro)
- Select all `import` lines at the top of the file → group into three sections (Core, Third‑party mo:*, Local) → sort each group alphabetically by path → insert blank lines between sections → keep import styles as‑is.

---

## Agent Strategy (for AI assistants)

1) Confirm the project builds on `mo:core` before starting improvements.
2) Work file‑by‑file. For each file:
    - A. Remove single‑expression `return` forms
    - B. Apply dot‑notation per skills/dot-notation-migration/SKILL.md
    - C. Ensure required imports for any introduced dot‑notation (see import mapping in skills/dot-notation-migration/SKILL.md)
    - D. Remove truly unused imports (respect the dot‑notation import mapping from the dedicated skill)
    - E. Shorten local (sibling) import paths (remove `./` prefix where applicable)
    - F. Aggregate imports into the three sections and sort each section alphabetically (Core → Third‑party mo:* → Local)
    - G. Replace `Text.encodeUtf8("<literal>")` with `"<literal>"` where the target type is `Blob`.
3) After each file: compile; if failure due to missing import, restore and mark mapping
4) After each category across repo: run a full build and optionally tests
5) Produce a short report of changes and any edge cases deferred for manual review

---

## H) Convert `Array.fromVarArray(x)` to `x.toArray()`

Pattern
```motoko
// Before
Array.fromVarArray(buf)
Array.fromVarArray<Nat8>(buf)

// After
buf.toArray()
```

Notes
- `Array.fromVarArray` is a factory function (first param is NOT `self`), so the dot-notation migration script does NOT convert it automatically. This is a separate conversion.
- `VarArray.toArray()` is the dot-notation equivalent — it's defined on `[var T]` values.
- After conversion, check if `Array` import can be removed (it may still be needed for `Array.tabulate`, `Array.flatten` dot-notation on `[T]`, etc.)
- Strip optional type params: `Array.fromVarArray<Nat8>(buf)` → `buf.toArray()` (the type is inferred from the var array).

## I) `Array.tabulate` type annotations are usually required

- Do NOT remove type annotations from `Array.tabulate<T>(...)` calls.
- The Motoko compiler often cannot infer the element type, especially when the callback uses `fromNat`, arithmetic, or other expressions that could return multiple numeric types.
- Removing annotations caused 13 of 24 test files to fail in a real project with errors like `expression of type [Any] cannot produce expected type [Nat8]`.
- Keep them: `Array.tabulate<Nat8>(n, func i { ... })`.

---

## Edge Cases & Gotchas

- For all dot‑notation behavior, method availability, factories vs methods, and mutability notes, see:
    - skills/dot-notation-migration/SKILL.md
- When aggregating imports, keep named type imports from `mo:core/Types` within the `mo:core` group; see Section F for ordering rules.
- Local imports: prefer bare module names (`"Bech32"`) over relative paths (`"./Bech32"`) for sibling files. Both resolve correctly but bare names are more concise.
- Import paths like `"../src/Bech32"` from within `src/` are incorrect — use `"Bech32"` for siblings or `"../SubDir/Module"` for cross-directory references.

---

## Verification & Sign‑off

- Build all canisters successfully after changes.
- Run static audits:
    - No `import` lines flagged unused by editor or heuristic scripts (after accounting for dot‑notation needs).
    - Spot check: arrays, maps, sets, text operations use dot‑notation where natural.
- Diffs remain mechanical; no public API or behavioral changes.

---

## Appendix: Dot‑notation reference

For the complete, maintained dot‑notation catalog, automation scripts, and import mapping, see:
- skills/dot-notation-migration/SKILL.md
