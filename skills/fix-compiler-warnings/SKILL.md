---
name: motoko-compiler-warnings-fixes 
description: "Guidelines for fixing Motoko compiler warnings (moc). Use when asked to fix, suppress, or clean up Motoko compiler warnings from `dfx build --check`."
---

# Fixing Motoko Compiler Warnings

## How to Run the Build Check

```bash
dfx build --check 2>&1 | tee /tmp/dfx_build_output.txt
```

This type-checks all canisters without deploying. Redirect stderr to capture warnings. The build takes several minutes for large projects.

To count warnings by type:
```bash
grep -o 'M0[0-9]*' /tmp/dfx_build_output.txt | sort | uniq -c | sort -rn
```

---

## Warning Code Reference

### M0194 — Unused Identifier

**What it means:** A variable, parameter, import, or binding is declared but never used.

**How to fix:** Prefix the identifier with `_` (e.g., `caller` → `_caller`). For catch bindings, use plain `_` (e.g., `catch (e)` → `catch (_)`), never `_e`.

**CRITICAL PITFALLS:**

1. **Unused record fields in function parameter destructuring: remove the field entirely.** In Motoko, record types are structural and support width subtyping — a function that expects `{ response : ... }` will accept `{ context : Blob; response : ... }` because the caller's record has *more* fields than required. So the correct fix is to drop the unused field from the parameter record:

   ```motoko
   // ❌ WRONG — renaming the field changes the record type, causes M0096 type errors
   public query func transform({
     _context : Blob;
     response : { ... };
   }) : async { ... } { ... }

   // ❌ ALSO WRONG — leaving it triggers M0194 warning
   public query func transform({
     context : Blob;
     response : { ... };
   }) : async { ... } { ... }

   // ✅ CORRECT — remove the unused field; subtyping accepts the extra field from callers
   public query func transform({
     response : { ... };
   }) : async { ... } { ... }
   ```

   The IC runtime passes `{ context = <blob>; response = <http_response> }`. Because Motoko uses structural subtyping, a function expecting just `{ response : ... }` will accept records that also have a `context` field — the extra field is simply ignored.

   **Rule:** If M0194 fires on a record field in a function parameter, remove the field from the destructuring pattern. Do NOT rename it with `_` prefix.

2. **Word-boundary regex can rename method calls.** If a variable `liquidity` is unused on the same line as `poolActor.liquidity()`, a naive `\b` regex will rename both. Use a negative lookbehind/lookahead that excludes dots:
   ```python
   # ❌ WRONG — also renames poolActor.liquidity()
   pattern = rf'\b{identifier}\b'

   # ✅ CORRECT — won't match after a dot
   pattern = rf'(?<![._a-zA-Z0-9]){re.escape(identifier)}(?![_a-zA-Z0-9])'
   ```

3. **Unused imports** are safe to prefix (e.g., `import Time "mo:base/Time"` → `import _Time "mo:base/Time"`), but consider just deleting the import instead if it's truly not needed.

**Safe categories to rename:**
- `catch (e)` → `catch (_)` (catch bindings — always use plain `_`, never `_e` or `_err`)
- `let foo = ...` → `let _foo = ...` (let bindings, only if truly unused)
- `func bar(caller : Principal)` → `func bar(_caller : Principal)` (function params that are simple names, NOT record fields)
- Unused record fields in function parameter destructuring → remove the field entirely (subtyping handles it)
- `import Foo "..."` → `import _Foo "..."` (unused imports, or just delete them)

**Catch block convention:** Always use `catch (_)` with a plain wildcard, not `catch (_e)` or `catch (_err)`. Since the error value is intentionally discarded, `_` communicates that clearly and doesn't create a named identifier.

**Bulk fix approach:** Extract all M0194 warnings from build output, parse file:line:identifier, apply fixes via Python script. Example:

```python
import re

def fix_m0194(filepath, line_num, identifier):
    """Prefix an unused identifier with _ on a specific line."""
    with open(filepath, 'r') as f:
        lines = f.readlines()

    line = lines[line_num - 1]  # 1-indexed
    # Careful regex: don't match after dots (method calls)
    pattern = rf'(?<![._a-zA-Z0-9]){re.escape(identifier)}(?![_a-zA-Z0-9])'
    new_line = re.sub(pattern, f'_{identifier}', line, count=1)
    lines[line_num - 1] = new_line

    with open(filepath, 'w') as f:
        f.writelines(lines)
```

---

### M0244 — Variable Could Be `let`

**What it means:** A `var` binding is never reassigned, so it could be `let` (immutable).

**How to fix:** Change `var` to `let`:
```motoko
// Before
var foo = computeSomething();
// After
let foo = computeSomething();
```

Also applies to `transient var` → `transient let` and `stable var` → `stable let` (though `stable let` has restrictions — only use if the value is a constant or initializer that doesn't need upgradeability).

**Bulk fix approach:** Parse warnings from build output (file:line:varname), then for each line replace `\bvar\b` with `let` (count=1). This is safe because the compiler has already verified the variable is never reassigned.

**Safe to bulk fix.** These are always safe to change unless the variable is intentionally left mutable for future use.

---

## General Strategy

1. Run `dfx build --check` and capture output
2. Count warnings by code to prioritize
3. Fix one warning type at a time
4. After bulk fixes, always rebuild to verify no new errors were introduced
5. Watch for **type errors (M0096)** after renaming — they indicate you broke a type signature
6. Record field renames in IC interface functions will cause cascading type errors

## Build Notes for This Project

- Build time: ~5 minutes for full `dfx build --check`
- The `users_management` canister may fail with a missing `.did` file — this is a pre-existing issue unrelated to warnings
- Compiler version: moc 1.4.1
