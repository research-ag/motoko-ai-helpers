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

## B) Dot‑notation conversion

For all Motoko dot‑notation rules, automation scripts, and pitfalls, see the dedicated skill:
- skills/dot-notation-migration/SKILL.md

This file intentionally does not duplicate those instructions. Apply dot‑notation changes using the dedicated skill, then continue here with import cleanup (Section D) and import ordering (Section E).

---

## C) Dot‑notation import requirements

For import mapping and rules related to dot‑notation, use the dedicated skill:
- skills/dot-notation-migration/SKILL.md

This file intentionally does not duplicate the import mapping. After applying dot‑notation changes per that skill, proceed with Section D (unused import cleanup) and Section E (import ordering).

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
        - Searches file for either `XYZ.` or any of the known dot‑patterns mapped to `XYZ` (see Dot‑Notation Migration Skill import mapping)
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
    - E. Aggregate imports into the three sections and sort each section alphabetically (Core → Third‑party mo:* → Local)
3) After each file: compile; if failure due to missing import, restore and mark mapping
4) After each category across repo: run a full build and optionally tests
5) Produce a short report of changes and any edge cases deferred for manual review

---

## Edge Cases & Gotchas

- For all dot‑notation behavior, method availability, factories vs methods, and mutability notes, see:
    - skills/dot-notation-migration/SKILL.md
- When aggregating imports, keep named type imports from `mo:core/Types` within the `mo:core` group; see Section E for ordering rules.

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
