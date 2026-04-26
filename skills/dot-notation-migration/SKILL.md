---
name: motoko-dot-notation-migration
description: Use new dot-notation syntax in projects with mo:core dependency
---

# Motoko dot notation migration

## Purpose

Convert Motoko function calls from the old `Module.func(self, ...)` style to the new `self.func(...)` dot notation, for all functions in the `core` package whose first parameter is named `self`.

This skill was created from a successful migration of the `motoko-bitcoin` project. It contains the complete function catalog for core 2.2.0, a battle-tested Python conversion script, and all the critical pitfalls that must be handled.

## When to Use

- When migrating a Motoko project from `base` to `core`, or upgrading `core` versions
- When the user asks to convert function calls to dot notation
- When the user asks to modernize Motoko call syntax

## Background

In Motoko's `core` package (successor to `base`), many module functions declare their first parameter as `self`. These functions support **dot notation**: instead of writing `Module.func(self, arg2, arg3)`, you can write `self.func(arg2, arg3)`.

**Only functions whose first parameter is literally named `self` support this.** Factory functions like `Array.tabulate`, `Array.fromVarArray`, `Blob.fromArray`, etc. do NOT have `self` as first parameter and must NOT be converted.

## Conversion Rules

### Zero-extra-arg functions
```
Module.func(expr)         →  expr.func()
Module.func<T>(expr)      →  expr.func<T>()
```

### Multi-arg functions
```
Module.func(expr, a, b)   →  expr.func(a, b)
Module.func<T>(expr, a)   →  expr.func<T>(a)
```

### Nested calls chain naturally
```
Array.flatten(Array.map<T, U>(xs, f))  →  xs.map<T, U>(f).flatten()
List.toArray(listExpr)                 →  listExpr.toArray()
Blob.toArray(Text.encodeUtf8(t))       →  t.encodeUtf8().toArray()
```

## Critical Pitfalls

### 1. Operator Precedence — Dot Binds Tighter Than Infix

In Motoko, `.` binds tighter than all infix operators (`>>`, `<<`, `&`, `|`, `+`, `-`, `*`, `/`, `%`, `#`, `==`, `!=`, `and`, `or`).

```motoko
// WRONG: dot binds to 24, not the whole expression
(value >> 24).toNat()   // ← need parens
value >> 24.toNat()     // ← BROKEN: parses as value >> (24.toNat())

// WRONG: dot binds to b, not the whole expression  
(a & b).toNat()         // ← need parens
a & b.toNat()           // ← BROKEN

// OK: no operators at top level
myArray.flatten()
foo.bar.toNat()
someFunc(x, y).toText()
```

**Rule:** If the self-expression contains ANY top-level infix operator, wrap it in parentheses before appending `.func()`.

### 2. Type Parameters Contain Commas — Track Angle Bracket Depth

When splitting `Module.func(self, rest...)` at the first comma, you MUST track `<>` angle bracket depth. Type parameters like `<Nat64, [Nat8]>` contain commas that are NOT argument separators.

```motoko
// The comma between Nat64 and [Nat8] is inside <>, not an argument separator
Array.map<Nat64, [Nat8]>(amounts, func(x) { ... })
//                       ^ THIS is the first real comma (after amounts)

// If you split at the first comma ignoring <>, you get:
//   self = "Array.map<Nat64"    ← WRONG, corrupted
//   rest = "[Nat8]>(amounts..." ← WRONG, corrupted
```

**Rule:** In the comma-finder, track `depth_angle` for `<>` just like `depth_paren` for `()`. Only decrement on `>` when `depth_angle > 0` (to avoid confusing comparison operators with closing angle brackets).

### 3. Functions That Must NOT Be Converted

These functions do NOT have `self` as their first parameter. Do not convert them:

```
Array: tabulate, fromVarArray, fromIter, empty, repeat
Blob: fromArray, fromVarArray, empty
Text: fromArray, fromIter, fromVarArray
List: empty, repeat, tabulate, fromIter, fromArray, fromVarArray
VarArray: empty, repeat, fromArray, fromIter, tabulate
Nat: range, min, max, rangeByInclusive
Nat8/16/32/64: fromNat, fromIntWrap, fromNat16, fromNat32, fromNat64 (constructors)
Int: abs, fromNat, range, min, max
Char: fromNat32, fromText
Runtime: trap
```

However, `Array.fromVarArray(x)` has a dot-notation equivalent: `x.toArray()` (via `VarArray.toArray`). This is NOT handled by the dot-notation conversion script — it's a separate refactor (see motoko-core-code-improvements skill, Section H).

### 4. Dot-Notation Requires the Type's Module to Be Imported

When calling a method via dot-notation on a value (e.g., `someBlob.toArray()`), the module for that value's type (`Blob`) must still be imported, even though the name `Blob` doesn't appear explicitly in the calling code.

Common examples where the import looks unused but is required:
- `Blob` — for `.toArray()`, `.size()` on Blob return values (e.g., `Sha256.fromArray(...).toArray()`)
- `Array` — for `.flatten()`, `.foldLeft()`, `.map()`, `.sliceToArray()` on `[T]` values
- `Nat` — for `.toText()` on Nat values from `.size()` calls
- `VarArray` — for `.toArray()` on `[var T]` values (after converting `Array.fromVarArray`)

If you remove these imports, compilation will fail with `field X does not exist in type` and a hint to import the module.

### 5. String Literals Must Be Skipped

When scanning for operators or commas, skip over string literals (`"..."`) including escaped quotes (`\"`), so operators inside strings don't trigger false positives.

### 6. Numeric Literals Are Safe Without Parens

`42.toText()` and `0xFF.toNat()` are valid Motoko — numeric literals can receive dot notation directly. No parens needed.

## Function Catalog (core 2.2.0)

### Zero-Extra-Arg Functions (self only)

These take exactly one argument named `self`. Convert `Module.func(expr)` → `expr.func()`.

```python
ZERO_EXTRA = [
    # Nat8
    ('Nat8', 'toNat'), ('Nat8', 'toText'), ('Nat8', 'toNat16'),
    ('Nat8', 'toNat32'), ('Nat8', 'toNat64'),
    # Nat16
    ('Nat16', 'toNat'), ('Nat16', 'toText'), ('Nat16', 'toNat8'),
    ('Nat16', 'toNat32'), ('Nat16', 'toNat64'),
    # Nat32
    ('Nat32', 'toNat'), ('Nat32', 'toText'), ('Nat32', 'toNat8'),
    ('Nat32', 'toNat16'), ('Nat32', 'toNat64'), ('Nat32', 'toChar'),
    # Nat64
    ('Nat64', 'toNat'), ('Nat64', 'toText'), ('Nat64', 'toNat8'),
    ('Nat64', 'toNat16'), ('Nat64', 'toNat32'),
    # Nat
    ('Nat', 'toText'), ('Nat', 'toInt'), ('Nat', 'toFloat'),
    ('Nat', 'toNat8'), ('Nat', 'toNat16'), ('Nat', 'toNat32'), ('Nat', 'toNat64'),
    # Int
    ('Int', 'toText'), ('Int', 'toNat'), ('Int', 'toFloat'),
    ('Int', 'toInt8'), ('Int', 'toInt16'), ('Int', 'toInt32'), ('Int', 'toInt64'),
    # Char
    ('Char', 'toNat32'), ('Char', 'toText'), ('Char', 'isDigit'),
    ('Char', 'isAlphabetic'), ('Char', 'isWhitespace'),
    ('Char', 'isLowercase'), ('Char', 'isUppercase'),
    # Blob
    ('Blob', 'toArray'), ('Blob', 'toVarArray'), ('Blob', 'isEmpty'),
    ('Blob', 'size'), ('Blob', 'hash'),
    # Text (zero-arg self functions)
    ('Text', 'toArray'), ('Text', 'toIter'), ('Text', 'toVarArray'),
    ('Text', 'encodeUtf8'), ('Text', 'decodeUtf8'), ('Text', 'size'),
    ('Text', 'isEmpty'), ('Text', 'reverse'), ('Text', 'toText'),
    # Array (zero-arg self functions)
    ('Array', 'toVarArray'), ('Array', 'size'), ('Array', 'isEmpty'),
    ('Array', 'keys'), ('Array', 'values'), ('Array', 'reverse'),
    ('Array', 'enumerate'),
    # VarArray (zero-arg self functions)
    ('VarArray', 'toArray'), ('VarArray', 'clone'), ('VarArray', 'size'),
    ('VarArray', 'isEmpty'), ('VarArray', 'keys'), ('VarArray', 'values'),
    ('VarArray', 'reverse'), ('VarArray', 'enumerate'),
    # List (zero-arg self functions)
    ('List', 'size'), ('List', 'toArray'), ('List', 'toVarArray'),
    ('List', 'isEmpty'), ('List', 'first'), ('List', 'last'),
    ('List', 'clone'), ('List', 'clear'), ('List', 'values'),
    ('List', 'keys'), ('List', 'enumerate'), ('List', 'reverse'),
    ('List', 'removeLast'),
]
```

### Multi-Extra-Arg Functions (self + more args)

These take `self` plus additional arguments. Convert `Module.func(expr, a, b)` → `expr.func(a, b)`.

Note: Some of these CAN be called with just self (e.g. `Array.flatten(xs)` has no extra args). The multi-arg handler must handle both cases: if no comma is found after self, emit `self.func()`; if comma is found, emit `self.func(rest...)`.

```python
MULTI_EXTRA = [
    # Array
    ('Array', 'flatten'), ('Array', 'map'), ('Array', 'filter'),
    ('Array', 'filterMap'), ('Array', 'flatMap'),
    ('Array', 'foldLeft'), ('Array', 'foldRight'),
    ('Array', 'concat'), ('Array', 'sliceToArray'),
    ('Array', 'forEach'), ('Array', 'mapEntries'),
    ('Array', 'sort'), ('Array', 'find'), ('Array', 'findIndex'),
    ('Array', 'equal'), ('Array', 'compare'),
    ('Array', 'all'), ('Array', 'any'),
    ('Array', 'indexOf'), ('Array', 'contains'),
    ('Array', 'toText'), ('Array', 'binarySearch'),
    ('Array', 'isSorted'), ('Array', 'mapResult'), ('Array', 'range'),
    # VarArray
    ('VarArray', 'sortInPlace'), ('VarArray', 'map'), ('VarArray', 'filter'),
    ('VarArray', 'filterMap'), ('VarArray', 'flatMap'),
    ('VarArray', 'foldLeft'), ('VarArray', 'foldRight'),
    ('VarArray', 'concat'), ('VarArray', 'forEach'),
    ('VarArray', 'sort'), ('VarArray', 'find'), ('VarArray', 'findIndex'),
    ('VarArray', 'equal'), ('VarArray', 'compare'),
    ('VarArray', 'all'), ('VarArray', 'any'),
    ('VarArray', 'indexOf'), ('VarArray', 'contains'),
    ('VarArray', 'toText'), ('VarArray', 'mapResult'),
    # Text
    ('Text', 'replace'), ('Text', 'tokens'), ('Text', 'contains'),
    ('Text', 'split'), ('Text', 'startsWith'), ('Text', 'endsWith'),
    ('Text', 'concat'), ('Text', 'map'), ('Text', 'flatMap'),
    ('Text', 'foldLeft'), ('Text', 'join'),
    ('Text', 'trimStart'), ('Text', 'trimEnd'), ('Text', 'trim'),
    ('Text', 'stripStart'), ('Text', 'stripEnd'),
    # List
    ('List', 'add'), ('List', 'at'), ('List', 'get'), ('List', 'put'),
    ('List', 'map'), ('List', 'filter'), ('List', 'forEach'),
    ('List', 'sort'), ('List', 'find'), ('List', 'findIndex'),
    # Blob
    ('Blob', 'compare'), ('Blob', 'equal'),
]
```

## How to Update the Catalog for a New Core Version

When the `core` package is updated, new functions may have been added or signatures changed. Run this shell snippet to regenerate the catalog from the actual source:

```bash
#!/bin/bash
# Usage: ./scan_self_functions.sh /path/to/.mops/core@X.Y.Z/src
CORE_SRC="${1:-.mops/core@2.2.0/src}"

echo "=== Functions WITH self as first parameter (convert to dot notation) ==="
for f in "$CORE_SRC"/*.mo; do
    module=$(basename "$f" .mo)
    # Match: public func name(self  OR  public let name = func(self
    grep -nE '^\s*public\s+(func|let)\s+\w+' "$f" | while read -r line; do
        # Extract function name
        fname=$(echo "$line" | sed -E 's/.*public\s+(func|let)\s+([a-zA-Z0-9_]+).*/\2/')
        # Check if first param is named 'self'
        if echo "$line" | grep -qE '\(\s*self\s*[:\)]'; then
            echo "  ($module, $fname)  -- has self"
        elif echo "$line" | grep -qE '=\s*func\s*\(\s*self\s*[:\)]'; then
            echo "  ($module, $fname)  -- has self (let)"
        fi
    done
done

echo ""
echo "=== Functions WITHOUT self (do NOT convert) ==="
for f in "$CORE_SRC"/*.mo; do
    module=$(basename "$f" .mo)
    grep -nE '^\s*public\s+(func|let)\s+\w+' "$f" | while read -r line; do
        fname=$(echo "$line" | sed -E 's/.*public\s+(func|let)\s+([a-zA-Z0-9_]+).*/\2/')
        if ! echo "$line" | grep -qE '\(\s*self\s*[:\)]' && \
           ! echo "$line" | grep -qE '=\s*func\s*\(\s*self\s*[:\)]'; then
            echo "  $module.$fname"
        fi
    done
done
```

**Important:** Some `public let` declarations use inline lambda syntax:
```motoko
public let toNat = Prim.nat8ToNat;  // No parens visible — check Prim binding
public let encodeUtf8 : Text -> Blob = Prim.encodeUtf8;  // self is implicit
```
For `public let` bindings that reference Prim functions taking a single value, check the type signature: if it's `T -> U`, the single argument is `self`. These support dot notation.

After scanning, classify each function:
1. **ZERO_EXTRA**: Takes only `self` (no other params). Goes in the zero-extra list.
2. **MULTI_EXTRA**: Takes `self` plus other params. Goes in the multi-extra list.
3. **NO_SELF**: First param is NOT named `self`. Do NOT convert.

Then update the `ZERO_EXTRA` and `MULTI_EXTRA` lists in the conversion script below, limiting to only the functions actually used in your project's source files (to avoid false matches on identically-named local functions).

## Conversion Script

Save this as `convert_dot_notation.py` and run with `python3 convert_dot_notation.py`. Edit the `ZERO_EXTRA`, `MULTI_EXTRA` lists and `src_dir` path as needed for your project.

```python
#!/usr/bin/env python3
"""
Convert Module.func(self, ...) calls to self.func(...) dot notation.

Usage:
  1. Update ZERO_EXTRA and MULTI_EXTRA lists for your core version and project.
  2. Set src_dir to your project's source directory.
  3. Run: python3 convert_dot_notation.py
  4. Run tests: npx mops test
  5. If all pass, delete this script.
"""

import re
import os

# ──────────────────────────────────────────────────────────────────────
# CONFIGURATION: Update these lists per your core version and project.
# Only include (Module, func) pairs that are actually used in your code.
# ──────────────────────────────────────────────────────────────────────

# Functions that take ONLY self (zero extra args):
#   Module.func(self) -> self.func()
ZERO_EXTRA = [
    ('Nat8', 'toNat'), ('Nat8', 'toText'), ('Nat8', 'toNat32'), ('Nat8', 'toNat64'),
    ('Nat16', 'toNat'), ('Nat16', 'toText'), ('Nat16', 'toNat64'),
    ('Nat32', 'toNat'), ('Nat32', 'toText'), ('Nat32', 'toNat8'), ('Nat32', 'toNat64'),
    ('Nat64', 'toNat'), ('Nat64', 'toText'), ('Nat64', 'toNat8'), ('Nat64', 'toNat16'),
    ('Nat', 'toText'),
    ('Int', 'toText'), ('Int', 'toNat'),
    ('Char', 'toNat32'),
    ('Blob', 'toArray'),
    ('Text', 'toArray'), ('Text', 'encodeUtf8'), ('Text', 'decodeUtf8'),
]

# Functions that take self + possibly more args:
#   Module.func(self, rest...) -> self.func(rest...)
#   Module.func(self) -> self.func()  (when no extra args)
MULTI_EXTRA = [
    ('List', 'size'),
    ('List', 'toArray'),
    ('Array', 'flatten'),
    ('Array', 'map'),
    ('Array', 'filter'),
    ('Array', 'foldLeft'),
    ('Array', 'concat'),
    ('Array', 'sliceToArray'),
    ('Text', 'replace'),
    ('Text', 'tokens'),
    ('Text', 'contains'),
]

# ──────────────────────────────────────────────────────────────────────
# ENGINE: Do not modify below unless fixing bugs.
# ──────────────────────────────────────────────────────────────────────

def needs_parens(s):
    """Check if an expression needs wrapping in parens for safe dot notation.

    Returns True if the expression contains top-level infix operators that
    would cause `.func()` to bind incorrectly (dot binds tighter).
    """
    s = s.strip()
    if not s:
        return False

    # Simple identifier: no parens
    if re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', s):
        return False

    # Entire expression already wrapped in matching parens
    if s[0] == '(' and s[-1] == ')':
        depth = 0
        for i, c in enumerate(s):
            if c == '(':
                depth += 1
            elif c == ')':
                depth -= 1
            if depth == 0 and i < len(s) - 1:
                break
        else:
            return False

    # Entire expression is an array literal
    if s[0] == '[' and s[-1] == ']':
        depth = 0
        for i, c in enumerate(s):
            if c == '[':
                depth += 1
            elif c == ']':
                depth -= 1
            if depth == 0 and i < len(s) - 1:
                break
        else:
            return False

    # Numeric literal
    if re.match(r'^[0-9][0-9a-fA-FxX_]*$', s):
        return False

    # Scan for top-level infix operators
    depth_paren = 0
    depth_bracket = 0
    depth_brace = 0
    i = 0
    while i < len(s):
        c = s[i]
        if c == '(':
            depth_paren += 1
        elif c == ')':
            depth_paren -= 1
        elif c == '[':
            depth_bracket += 1
        elif c == ']':
            depth_bracket -= 1
        elif c == '{':
            depth_brace += 1
        elif c == '}':
            depth_brace -= 1
        elif c == '"':
            # Skip string literals
            i += 1
            while i < len(s) and s[i] != '"':
                if s[i] == '\\':
                    i += 1
                i += 1
        elif depth_paren == 0 and depth_bracket == 0 and depth_brace == 0:
            rest = s[i:]
            # Multi-char operators
            if (rest.startswith('>>') or rest.startswith('<<') or
                rest.startswith('==') or rest.startswith('!=') or
                rest.startswith('and ') or rest.startswith('or ') or
                rest.startswith('#')):
                return True
            # Single-char binary operators in infix position
            if c in '&|^%' and i > 0:
                return True
            if c in '+-' and i > 0 and s[i-1] not in '(,;:=[<>':
                return True
            if c in '*/' and i > 0:
                return True
        i += 1

    return False


def wrap_if_needed(s):
    """Wrap expression in parens if needed for dot notation."""
    if needs_parens(s):
        return '(' + s + ')'
    return s


def find_matching_paren(s, start):
    """Find matching closing paren for opening paren at position start.
    Returns the index of the closing paren, or -1 if not found.
    """
    depth = 1
    i = start + 1
    while i < len(s) and depth > 0:
        if s[i] == '(':
            depth += 1
        elif s[i] == ')':
            depth -= 1
        elif s[i] == '"':
            # Skip string literal
            i += 1
            while i < len(s) and s[i] != '"':
                if s[i] == '\\':
                    i += 1
                i += 1
        i += 1
    return i - 1 if depth == 0 else -1


def find_first_comma_at_depth0(s):
    """Find first comma at depth 0 (outside parens, brackets, braces, AND angle brackets).

    CRITICAL: Must track <> depth so commas inside type parameters like
    <Nat64, [Nat8]> are not treated as argument separators.
    """
    depth_paren = 0
    depth_bracket = 0
    depth_brace = 0
    depth_angle = 0
    i = 0
    while i < len(s):
        c = s[i]
        if c == '(':
            depth_paren += 1
        elif c == ')':
            depth_paren -= 1
        elif c == '[':
            depth_bracket += 1
        elif c == ']':
            depth_bracket -= 1
        elif c == '{':
            depth_brace += 1
        elif c == '}':
            depth_brace -= 1
        elif c == '<':
            depth_angle += 1
        elif c == '>' and depth_angle > 0:
            # Only decrement if we're inside angle brackets.
            # Bare > (comparison) should not affect depth.
            depth_angle -= 1
        elif c == '"':
            i += 1
            while i < len(s) and s[i] != '"':
                if s[i] == '\\':
                    i += 1
                i += 1
        elif (c == ',' and depth_paren == 0 and depth_bracket == 0
              and depth_brace == 0 and depth_angle == 0):
            return i
        i += 1
    return -1


def find_all_args(s):
    """Split content inside parens into (self_arg, rest_text).
    rest_text includes everything after the first depth-0 comma (preserving whitespace).
    """
    comma_pos = find_first_comma_at_depth0(s)
    if comma_pos == -1:
        return s.strip(), ""
    else:
        return s[:comma_pos].strip(), s[comma_pos + 1:]


def process_zero_extra(text, module, func):
    """Convert Module.func(arg) -> arg.func() for zero-extra-arg functions."""
    pattern = re.compile(
        r'\b' + re.escape(module) + r'\.' + re.escape(func) + r'(?=[\s<(])')

    result = text
    offset = 0
    changes = 0

    while True:
        m = pattern.search(result, offset)
        if not m:
            break

        pos = m.end()
        while pos < len(result) and result[pos] in ' \t\n':
            pos += 1

        # Optional type params <...>
        type_params = ""
        if pos < len(result) and result[pos] == '<':
            depth = 1
            tp_start = pos
            pos += 1
            while pos < len(result) and depth > 0:
                if result[pos] == '<':
                    depth += 1
                elif result[pos] == '>':
                    depth -= 1
                pos += 1
            type_params = result[tp_start:pos]
            while pos < len(result) and result[pos] in ' \t\n':
                pos += 1

        if pos >= len(result) or result[pos] != '(':
            offset = m.end()
            continue

        close = find_matching_paren(result, pos)
        if close == -1:
            offset = m.end()
            continue

        inner = result[pos + 1:close]
        self_arg = inner.strip()
        if not self_arg:
            offset = m.end()
            continue

        wrapped = wrap_if_needed(self_arg)
        replacement = f"{wrapped}.{func}{type_params}()"
        result = result[:m.start()] + replacement + result[close + 1:]
        offset = m.start() + len(replacement)
        changes += 1

    return result, changes


def process_multi_extra(text, module, func):
    """Convert Module.func(self, rest...) -> self.func(rest...) for multi-arg functions."""
    pattern = re.compile(
        r'\b' + re.escape(module) + r'\.' + re.escape(func) + r'(?=[\s<(])')

    result = text
    offset = 0
    changes = 0

    while True:
        m = pattern.search(result, offset)
        if not m:
            break

        pos = m.end()
        while pos < len(result) and result[pos] in ' \t\n':
            pos += 1

        type_params = ""
        if pos < len(result) and result[pos] == '<':
            depth = 1
            tp_start = pos
            pos += 1
            while pos < len(result) and depth > 0:
                if result[pos] == '<':
                    depth += 1
                elif result[pos] == '>':
                    depth -= 1
                pos += 1
            type_params = result[tp_start:pos]
            while pos < len(result) and result[pos] in ' \t\n':
                pos += 1

        if pos >= len(result) or result[pos] != '(':
            offset = m.end()
            continue

        close = find_matching_paren(result, pos)
        if close == -1:
            offset = m.end()
            continue

        inner = result[pos + 1:close]
        self_arg, rest = find_all_args(inner)
        if not self_arg:
            offset = m.end()
            continue

        wrapped = wrap_if_needed(self_arg)
        if rest == "":
            replacement = f"{wrapped}.{func}{type_params}()"
        else:
            replacement = f"{wrapped}.{func}{type_params}({rest})"

        result = result[:m.start()] + replacement + result[close + 1:]
        offset = m.start() + len(replacement)
        changes += 1

    return result, changes


def process_file(filepath):
    with open(filepath, 'r') as f:
        text = f.read()

    total_changes = 0

    for module, func in ZERO_EXTRA:
        text, changes = process_zero_extra(text, module, func)
        total_changes += changes

    for module, func in MULTI_EXTRA:
        text, changes = process_multi_extra(text, module, func)
        total_changes += changes

    if total_changes > 0:
        with open(filepath, 'w') as f:
            f.write(text)
        print(f"  {filepath}: {total_changes} changes")

    return total_changes


def main():
    # ── Change this to your project's source directory ──
    src_dir = os.path.join(os.path.dirname(__file__), 'src')
    total = 0
    for root, dirs, files in os.walk(src_dir):
        for f in sorted(files):
            if f.endswith('.mo'):
                filepath = os.path.join(root, f)
                total += process_file(filepath)
    print(f"\nTotal changes: {total}")


if __name__ == '__main__':
    main()
```

## Step-by-Step Procedure

1. **Identify the core version** from `mops.toml`:
   ```bash
   grep 'core' mops.toml
   ```

2. **Scan the core source** to build/verify the function catalog:
   ```bash
   # List all public functions with self as first param
   grep -rnE 'public\s+(func|let)\s+\w+' .mops/core@*/src/*.mo | grep -E '\(\s*self\s*[:\)]'
   ```

3. **Determine which functions your project actually uses**:
   ```bash
   # Find all Module.func( patterns in your source
   grep -rnoE '\b(Array|Blob|Text|List|VarArray|Nat8?|Nat16|Nat32|Nat64|Int|Char)\.[a-zA-Z]+\(' src/ | \
     sed 's/.*://' | sort | uniq -c | sort -rn
   ```
   Only include used functions in the script to avoid false matches.

4. **Classify** each used function as ZERO_EXTRA or MULTI_EXTRA by checking its signature in the core source.

5. **Run the conversion**:
   ```bash
   python3 convert_dot_notation.py
   ```

6. **Run tests**:
   ```bash
   npx mops test
   ```

7. **If tests fail**, check for:
    - Operator precedence issues (missing parens around infix expressions)
    - Type parameter comma splitting (the `<>` depth tracking bug)
    - Functions that shouldn't have been converted (not in the `self` list)

8. **Clean up**: Delete `convert_dot_notation.py` after all tests pass.

## Verification Checklist

- [ ] All `Module.func(self, ...)` calls converted to `self.func(...)`
- [ ] No factory functions converted (`tabulate`, `fromArray`, `fromVarArray`, `repeat`, `empty`, `range`, `fromNat`, `fromIntWrap`, etc.)
- [ ] `Array.fromVarArray(x)` converted separately to `x.toArray()` (not part of dot-notation script)
- [ ] Infix expressions wrapped in parens: `(a >> b).toNat()`, `(x & y).toText()`
- [ ] Type parameters preserved: `xs.map<Nat64, [Nat8]>(f)` not corrupted
- [ ] `Array.tabulate<T>` type annotations kept (compiler cannot infer them)
- [ ] Imports for dot-notation types still present (`Blob`, `Array`, `Nat`, `VarArray`, etc.) even when module name doesn't appear explicitly in code
- [ ] Nested chains correct: `xs.map<T, U>(f).flatten()` not `xs.map<T.flatten( U>(`
- [ ] All tests pass with `npx mops test`
