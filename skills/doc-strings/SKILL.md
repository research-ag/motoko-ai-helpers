---
name: motoko-doc-strings
description: Add `///` doc strings to public objects in Motoko modules so that `mo-doc` produces useful HTML/Markdown documentation. Covers placement rules, formatting, code examples, common pitfalls, and a verification workflow.
---

# Motoko doc strings

## Purpose

Add triple-slash (`///`) doc comments to every public declaration in Motoko
source files so that `mo-doc` renders meaningful documentation pages. This
skill captures the conventions used in `mo:core` (caffeinelabs/motoko-core)
and the lessons learned applying them across the `motoko-bitcoin` library.

## When to Use

- The user asks to "add doc strings", "document the public API", or
  "generate docs" for a Motoko package.
- Reviewing or polishing a library before release / publication on MOPS.
- After adding new public symbols and noticing missing entries in the
  generated `docs/`.

## What Counts as a "Public Object"

Document every declaration that ends up in the rendered docs. In a typical
`module { ... }` file these are:

- `public type ...`
- `public let ...`
- `public func ...`
- `public class ...`
- The module itself (`module { ... }` or `module Name { ... }`)
- For a `public class`, every `public let`, `public var`, and
  `public func` member inside the class body.
- For a nested `public module`, recurse and apply the same rules.

Private declarations (`func`, `let`, `class` without `public`) and helper
types should NOT receive `///` comments — they don't appear in `mo-doc`
output and the noise hurts readability.

## Comprehensiveness by Location

How thorough a doc string needs to be depends on where the file lives:

- **Inside `src/internal/`** — doc strings can be brief. These modules are
  implementation details that end users are not expected to call directly,
  so a short one-liner stating what a declaration does is usually enough.
  Trap/error notes can be omitted unless the behavior is surprising to
  another maintainer.
- **Anywhere else under `src/` (i.e. outside `src/internal/`)** — doc
  strings MUST be comprehensive. These are the public API surface that
  users will call, so they need every detail: argument units and formats,
  size/range constraints, return-value semantics, full failure behavior
  (`Traps` / `Errors` paragraphs), and runnable examples where helpful.
  Apply the full "User-Perspective Read-Through" checklist at the end of
  this skill to every declaration here.

## Doc String Format

`mo-doc` parses lines that start with `///` as Markdown documentation
attached to the next declaration.

```motoko
/// Brief one-line description ending in a period.
///
/// Optional longer paragraph(s) describing semantics, edge cases,
/// and return values. Refer to parameters by `name` in backticks.
///
/// Example:
/// ```motoko include=import
/// let result = Module.func(arg);
/// ```
public func func(arg : T) : U { ... };
```

Conventions used in `mo:core`:

- First line is a short imperative summary ("Returns ...", "Computes ...",
  "Decodes ...") ending with a period.
- One blank `///` line separates paragraphs.
- Wrap parameter names, types, and short code in single backticks.
- Code blocks use ```` ```motoko ```` fences. Use the
  ```` ```motoko include=import ```` annotation when the snippet relies on
  the module's import header (defined elsewhere with
  ```` ```motoko name=import ````).
- For module-level docs, define a named import block once at the top so all
  example snippets can reference it:
  ```motoko
  /// ```motoko name=import
  /// import Hash "mo:bitcoin/Hash";
  /// ```
  module { ... }
  ```
- The `name=import` / `include=import` annotations are NOT consumed by
  `mo-doc` itself — they are directives for an external doctest runner
  (used by `mo:core`'s CI and by Docusaurus-based doc sites) that
  prepends the named snippet before compiling each example. `mo-doc`
  passes them through verbatim into the rendered code-fence info string.
  If the project does not run doctests, you can omit them and just use
  plain ```` ```motoko ```` fences.

### Where to put the module-level doc string

The module-level `///` block must sit at the **very top of the file**,
before the `import` statements, with one blank line separating it from
the imports and one blank line between the imports and the `module { ... }`
line. This matches the layout used in `dfinity/motoko-core` and is the
only placement that `mo-doc` actually attaches to the module page —
a doc block placed between the imports and `module { ... }` is silently
ignored and the rendered module page will have no description.

```motoko
/// One-line module summary.
///
/// Longer description.
///
/// ```motoko name=import
/// import Foo "mo:pkg/Foo";
/// ```

import Bar "mo:core/Bar";
import Baz "mo:core/Baz";

module {
  // ...
}
```

For named modules (`module Name { ... }`, e.g. `src/bitcoin/Script.mo`)
the same rule applies: doc block at the top of the file, then a blank
line, then imports, then a blank line, then `module Name { ... }`.

### Where to put per-declaration comments

Place `///` lines **immediately above** the declaration with no blank line
between them. If there is an existing legacy `// ...` comment, put the
`///` block above the legacy comment (the legacy comment can stay as
implementation notes).

```motoko
/// Public API description goes here.
// Legacy implementation notes can stay below the doc string.
public func encode(input : [Nat8]) : Text { ... };
```

## Common Pitfalls

### 1. `apply_patch` and literal `\n`

When inserting multi-line content, write actual newlines in the patch body
(one `+` per real line). NEVER use the escape sequence `\n` inside
inserted text — it will be written as the literal two characters and break
the file.

### 2. Don't accidentally insert into the middle of a function

`apply_patch` matches on context. When the surrounding context is too
short or appears multiple times, the inserted block may land inside a
function body. After inserting docs, run a quick syntax check (e.g.
`mo-doc` or `moc --check`) to catch this. A telltale sign is a
`syntax error [M0001], unexpected token 'import'` message — that means a
new module-level block was placed inside a `loop`/`func` body.

### 3. Class member documentation order

`mo-doc` renders class fields and methods in source order, but the class's
own description block is shown after the member list when the class doc
appears _above_ the constructor. To keep the class summary at the top of
the rendered class page, place the `///` block on the line directly
preceding `public class Name(...)`.

### 4. Re-exported types

Re-exported types like
`public type Signature = Types.Signature;` still need a one-line `///`
description so they are not rendered as "(no description)".

### 5. Skip private helpers and constants

Don't add `///` to `let`, `func`, or `type` declarations that lack
`public`. They never appear in the output and the comments add visual
noise.

### 6. `module Name { ... }` named modules

Files like `src/bitcoin/Script.mo` use the form `module Script { ... }`
instead of the bare `module { ... }`. The same top-of-file placement rule
applies — the module-level `///` block goes at the very start of the
file (before the imports), not on the line directly preceding
`module Script {`.

### 7. Module doc must be at the top of the file

`mo-doc` only treats a `///` block as the module description when it
appears at the very beginning of the file, ahead of the `import`
statements. A doc block placed between the imports and `module { ... }`
compiles fine but produces an empty module description in the rendered
HTML. If you find an existing project with module docs adjacent to
`module { ... }`, relocate them to the top of the file (a small Python
script that finds the trailing `///` block before `module ` and prepends
it to the file works well for batch migration).

## Workflow

1. Inventory public declarations:
   ```bash
   grep -RInE "public (type|func|class|let)" src
   ```
2. For each `.mo` file:
   - Add a module-level `///` block (with a `name=import` example) at the beginning of the file, even before the block of import statements.
   - Add `///` blocks above every public declaration inside the top-level modules.
   - Recurse into nested public declarations. For instance public members of public classes need doc strings. Public members of public modules need doc strings. And so on.
3. Re-scan to catch anything missed:
   ```bash
   awk '
     FNR==1{prev=""}
     {
       if ($0 ~ /^[[:space:]]*public[[:space:]]+(type|func|class|let)\b/) {
         p=prev; gsub(/^[[:space:]]+|[[:space:]]+$/, "", p);
         if (p !~ /^\/{3}/) printf "%s:%d:%s\n", FILENAME, FNR, $0;
       }
       if ($0 !~ /^[[:space:]]*$/) prev=$0;
     }
   ' src/*.mo src/**/*.mo
   ```
   Empty output = all public declarations are preceded by a `///` line.
4. Generate docs:
   ```bash
   mo-doc --source src --output docs --format html
   ```
   No output = success. Any "Skipping ..." line indicates a syntax error
   that must be fixed (often a stray `apply_patch` corruption).
5. Spot-check the rendered output:
   - `docs/index.html` — every module should appear in the listing.
   - Each `docs/<Module>.html` — module description, types, functions,
     and class members should all show their text.

## Tips for Writing Useful Descriptions

- Describe **what** the function does and what it **returns**, not how it
  is implemented.
- For low-level helpers (`readBE32`, `writeLE64`, etc.) a one-liner stating
  the byte order, width, and offset semantics is sufficient.
- For domain types (`SighashType`, `WitnessProgram`, `OutPoint`) name the
  spec or BIP that defines the format and link to it.
- Keep examples short and self-contained; prefer literal byte arrays over
  reading from external sources.

## Documenting Error and Trap Behavior (REQUIRED)

Every public function doc string MUST describe its full failure behavior.
Readers cannot tell from a type signature alone whether a function traps,
returns `null`, returns `#err`, or simply produces a wrong-but-defined
result on bad input — the doc string is the only place this contract is
recorded.

### What to look for in the implementation

Scan the function body (and every helper it calls) for:

- `Runtime.trap(...)` / `Debug.trap(...)` / `Prim.trap(...)` calls.
- `assert ...;` statements (a failed assert traps).
- Pattern matches that are non-exhaustive in practice (e.g. a `switch` on
  `?T` whose `null` branch traps, or a `case (#err _) Runtime.trap ...`).
- Implicit traps from the standard library: out-of-bounds array indexing
  (`a[i]` when `i >= a.size()`), `Nat` subtraction underflow, division by
  zero, `Option.unwrap` on `null`.
- Explicit error returns: `?T` returning `null`, `Result<T, E>` returning
  `#err`, variant returns like `{ #ok; #err }`.

### What to write

For each failure mode, state in the doc:

1. **The condition** — what input or state triggers it, in user-facing
   terms ("when `input` contains a character outside the Base58
   alphabet", not "when `mapBase58[c] == 255`").
2. **The outcome** — `traps`, `returns null`, `returns #err(...)`, etc.
3. **For `Result`/option returns**, list every distinct error case
   separately when the variants carry meaning.

Use a dedicated `Traps` and/or `Errors` paragraph (or both) at the end of
the doc, after the example and before the runtime/space notes:

```motoko
/// Decodes a Base58-encoded string into the original byte array.
///
/// ```motoko include=import
/// let bytes = Base58.decode("StV1DL6CwTryKyV");
/// ```
///
/// Traps if `encoded` contains any character that is not in the Base58
/// alphabet (i.e. not in
/// `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`).
public func decode(encoded : Text) : [Nat8] { ... };
```

For graceful errors:

```motoko
/// Parses a WIF-encoded private key.
///
/// Returns `#err(msg)` when:
/// - the input is not valid Base58Check (bad checksum or alphabet),
/// - the decoded payload has an unexpected length (must be 33 or 34 bytes),
/// - the version byte does not match `network`.
public func decode(wif : Text, network : Network) : Result<PrivateKey, Text> { ... };
```

Use the wording **"Traps"** (capitalised, present tense) for unrecoverable
failures so it stands out and is greppable across the codebase.

### Pure / total functions

If a function genuinely cannot fail (e.g. `Array.size`, a pure arithmetic
helper that uses fixed-width types and total operators), say so explicitly
with a one-liner like *"Never traps."* or omit the failure section
entirely — but only after auditing the body to confirm.

### Worked examples from `motoko-bitcoin`

- `Base58.decode` — traps on any character outside the alphabet (not a
  `Result`, no graceful fallback).
- `Base58Check.decode` — returns `?[Nat8]`; `null` on bad alphabet, bad
  length, or checksum mismatch. Document each.
- `Bech32.decode` — returns `Result<_, Text>` with distinct messages for
  invalid characters, mixed case, bad checksum, length out of range,
  invalid HRP. Mention each error category, not just "returns `#err`".
- `Bip32.derivePath` — traps on hardened derivation from a public key.
- `Fp.inverse` (and the `/` operator) — traps when the value is zero.
- `Affine.add` / `Jacobi.add` — document behaviour at the point at
  infinity and for equal/opposite inputs.
- Transaction serialization (`Transaction.toBytes`, `TxInput.toBytes`,
  etc.) — note any size limits that would cause `Nat32`/`Nat64`
  conversion traps.

### Audit workflow for an existing file

1. List every `public func` and `public class` member.
2. For each, read the body and follow the call graph one level into
   private helpers.
3. Note every `trap`, `assert`, `Result`/`Option` return, and any implicit
   trap source (subtraction, indexing, division).
4. Update the doc string to enumerate the conditions.
5. Re-run `mo-doc` and visually scan the rendered HTML for sections that
   still lack a "Traps" / "Errors" paragraph on a non-trivial function.

## Final Step: User-Perspective Read-Through (REQUIRED)

After every public declaration has a doc string, do one more pass. Read
each doc string from the perspective of a first-time user of the API who
has not seen the implementation. For every doc, ask:

- What is the unit / format of each argument and the return value?
  (bytes vs. bits, big- vs. little-endian, satoshis vs. BTC, raw vs.
  DER-encoded, compressed vs. uncompressed, 0-based vs. 1-based, …)
- What are the size or range constraints on each input?
- Which BIP / RFC / spec defines the format, and is it linked?
- For mutating methods, what state changes? Is the receiver still usable
  afterward?
- For functions that take a callback or proxy (e.g. an ECDSA signer), what
  is the expected input/output shape of the callback?
- For variant returns, what does each variant mean semantically (not just
  what tag it carries)?
- Are domain-specific terms ("witness program", "tap leaf", "sighash",
  "scriptPubKey") used without a one-line explanation or link?
- For constants (`EMPTY_WITNESS`, `dustThreshold`), what is the value and
  why does it have the value it has?
- For re-exported types, where is the actual definition (and is the link
  there)?
- If you removed the function name, would the description still be
  unambiguous? If two near-identical functions exist (e.g. `toBytes` vs.
  `toBytesIgnoringWitness`, `encode` vs. `encodeUncompressed`), is the
  difference between them spelled out?

If any question remains unanswered, extend the doc string to address it.
Prefer one extra sentence in the doc over forcing the user to read the
source. Do this pass file by file; it usually surfaces 2–5 missing facts
per non-trivial module.

