---
name: motoko-class-to-static-module-migration
description: Convert class-based mops packages into static module-based ones whose values are plain records, storable directly in stable variables and usable with dot-notation. Load when migrating an OOP-style Motoko package (with `class`, `share`/`unshare`) to the modern EOP (Explicit-Object-Passing) style used by `mo:core` (e.g. `List`, `Map`, `Set`).
---

# Motoko class-to-static-module migration

## Purpose

Convert a Motoko package written in the **class style** (`class Foo<T>(...)` with private state and methods, plus `share` / `unshare` helpers) into the **static module style** used by `mo:core` data structures (`List`, `Map`, `Set`, ...).

In the static style:

- The package is a `module`, not a `class`.
- The data type is a **plain record** exposed as a `type` alias (no methods, no private state). This makes it directly **stable** — no `share`/`unshare` and no `preupgrade`/`postupgrade` plumbing needed.
- Operations are top-level `func`s whose first parameter is named `self`. Callers use them via **dot-notation** (`xs.push(e)`), so the call sites look identical to the old class style.

This is the same pattern as `mo:core`'s `List`: see [`motoko-dot-notation-migration`](../dot-notation-migration/SKILL.md) for how dot-notation works, and `motoko-general-style-guidelines` (sections "Classes" and "Modules") for the underlying rationale ("Use modules for 'static' classes").

## When to Use

- Migrating a mops package whose public API is a `class` so it can be stored in a `stable` variable without `share`/`unshare`.
- Replacing custom `share` / `unshare` boilerplate with EOP-style modules.
- The user asks to "make the package stable", "remove share/unshare", "convert class to module", or "follow the `List`/`Map`/`Set` style".

## Background: why static modules

A Motoko `class` produces an object with methods. Objects with methods are **not stable** — they cannot be stored in `stable var` directly. Authors usually work around this with:

```motoko
public class ArrayWrapper<T>() {
  var arr : [T] = [];
  public func push(e : T) { arr := Array.append(arr, [e]) };
  public func share() : SharedData<T> = { arr };
  public func unshare(d : SharedData<T>) { arr := d.arr };
};
```

…and on the actor side:

```motoko
actor {
  stable var data : ArrayWrapper.SharedData<Nat> = { arr = [] };
  let wrapper = ArrayWrapper.ArrayWrapper<Nat>();
  system func preupgrade()  { data := wrapper.share() };
  system func postupgrade() { wrapper.unshare(data) };
};
```

In the static module style the type *itself* is a stable record, so all that ceremony disappears:

```motoko
module ArrayWrapper {
  public type ArrayWrapper<T> = { var arr : [T] };
  public func new<T>() : ArrayWrapper<T> = { var arr = [] };
  public func push<T>(self : ArrayWrapper<T>, e : T) {
    self.arr := Array.append(self.arr, [e]);
  };
};
```

```motoko
actor {
  stable var arr : ArrayWrapper.ArrayWrapper<Nat> = ArrayWrapper.new();
  // call sites look exactly like the class version thanks to dot-notation:
  arr.push(1);
};
```

## Conversion Rules

For each `class Foo<T>(initArgs)` in the package:

1. **Replace `class` with `module`** (or move it into an existing module). Keep the file/module name (`Foo`).

2. **Lift the private state to a public record type** named after the class:

   ```motoko
   public type Foo<T> = { var field1 : A; var field2 : B; ... };
   ```

   - Use `var` on fields that the methods mutate; leave the rest immutable.
   - All field types must themselves be stable (no functions, no objects with methods). If a field currently holds another class instance, migrate that class first.

3. **Replace the constructor with a `new` (or `empty` / `fromX`) function** that returns the record:

   ```motoko
   public func new<T>(initArgs) : Foo<T> = { var field1 = ...; var field2 = ... };
   ```

   Mirror `mo:core` naming: `empty()`, `singleton(x)`, `fromIter(it)`, `fromArray(xs)`, etc. — see `motoko-general-style-guidelines` ("Conventions").

4. **Turn each public method into a top-level `func` whose first parameter is `self`**:

   ```motoko
   // Before (inside the class):
   public func push(e : T) { arr := Array.append(arr, [e]) };

   // After (at module top level):
   public func push<T>(self : Foo<T>, e : T) {
     self.arr := Array.append(self.arr, [e]);
   };
   ```

   Notes:
   - The parameter **must literally be named `self`** for dot-notation to work (see `motoko-dot-notation-migration`).
   - Re-introduce type parameters (`<T>`, `<K, V>`, ...) on each function, because the module is no longer generic over them.
   - Replace every reference to a former field `foo` with `self.foo`.
   - Replace calls to other methods (`bar(x)`) with `self.bar(x)` or `Foo.bar(self, x)` — both compile, prefer dot-notation for readability.

5. **Delete `share` / `unshare`.** The record *is* the shareable representation. If the old `SharedData<T>` type was re-exported, keep it as `public type SharedData<T> = Foo<T>` for one release to ease migration, then remove it.

6. **Update the actor**:
   - Remove `system func preupgrade` / `postupgrade` (when they only did `share` / `unshare`).
   - Declare the value directly as a stable variable:
     ```motoko
     stable var arr : Foo.Foo<Nat> = Foo.new();
     ```
   - Keep call sites unchanged: `arr.push(e)` still works via dot-notation.

7. **Run `mops test`, then `mops bench` (if a `bench/` folder exists), then build every project under `example/` / `examples/` (if such a directory exists)**, to make sure behavior is preserved. Fix all failures before moving on to the next class — see the "Step-by-Step Procedure" section for the full one-class-at-a-time, bottom-up workflow.

## Multiple classes per file: export one named module per class inside the file's outer `module { ... }`, do NOT rename functions

**Whenever a single `.mo` file declares two or more classes, the migration target is one named `module` per original class, all declared inside the file's single outer `module { ... }` — regardless of whether their method names overlap.** Do not flatten multiple classes into one module, and do not rename / prefix functions to work around the lack of overloading.

**Why:** Motoko does not support function overloading. If you flatten `Foo` and `Bar` into one module and they both have `push` / `size` / `clear`, you'd be tempted to rename them (`pushFoo`, `pushBar`, ...). **Do not do this.** It breaks the core goal of this migration — that call sites stay identical thanks to dot-notation — because callers would have to switch from `x.push(e)` to `Foo.pushFoo(x, e)`. It also diverges from the `mo:core` convention where each data structure is its own module with its own `push` / `size` / `clear` / ... . Even when the method names *don't* overlap, keeping one module per original class preserves the one-class-↔-one-module mental model and keeps the per-class migration / per-class test gating from the "Step-by-Step Procedure" meaningful.

**Do this instead.** Keep the function names exactly as they were on the original classes, and declare each class's functions inside its own named module. A Motoko `.mo` file is itself a `module { ... }`, so when you need to expose more than one named module from one file, wrap them all in the file's single outer `module { ... }` — every `.mo` file has exactly one top-level `module` declaration:

```motoko
// src/Wrappers.mo  -- one file, two original classes Foo and Bar
import Array "mo:base/Array";

module {
  public module Foo {
    public type Foo<T> = { var arr : [T] };
    public func new<T>() : Foo<T> = { var arr = [] };
    public func push<T>(self : Foo<T>, e : T) {
      self.arr := Array.append(self.arr, [e]);
    };
    public func size<T>(self : Foo<T>) : Nat = self.arr.size();
  };

  public module Bar {
    public type Bar<T> = { var arr : [T]; var tag : Text };
    public func new<T>(tag : Text) : Bar<T> = { var arr = []; var tag };
    public func push<T>(self : Bar<T>, e : T) {
      self.arr := Array.append(self.arr, [e]);
    };
    public func size<T>(self : Bar<T>) : Nat = self.arr.size();
  };
};
```

Call sites stay identical to the original class-style API:

```motoko
import { Foo; Bar } "Wrappers";   // or however the project imports the named module

stable var f : Foo.Foo<Nat> = Foo.new();
stable var b : Bar.Bar<Nat> = Bar.new("xs");

f.push(1);   // resolves to Foo.push(f, 1)
b.push(2);   // resolves to Bar.push(b, 2)
```

Rules:

- **One nested `public module <ClassName> { ... }` per original class**, all inside the file's single outer `module { ... }`. The inner module name matches the original class name (`Foo`, `Bar`). The outer `module { ... }` is the file's anonymous module declaration that every `.mo` file has — it is required, not optional, when exporting more than one named module from the same file.
- **Apply this whenever the original file has more than one class**, regardless of whether the classes share method names. (For a file with a single class, keep using a plain single-module file as in the main example.)
- Inside each named inner module, follow the same conversion rules as for a single-class file (type alias named after the class, `new` / factory functions, `func name<T>(self : Foo<T>, ...)`, no `share` / `unshare`).
- **Never** prefix function names with the class/module name as a workaround for missing overloading. `push`, `size`, `clear`, etc. stay as-is.
- Keep the original file layout: if the original codebase put both classes in one file, keep both modules in that one file; do not split them into separate files just because they're now separate modules, and do not merge files that were originally separate.

## Common Pitfalls

1. **The `self` parameter must be named `self`.** Any other name (`this`, `s`, `xs`) disables dot-notation. The function still works as `Foo.push(xs, e)`, but the migration goal (identical call sites) is lost.

2. **Fields must stay stable.** Do not put functions, classes, or objects with methods inside the record. If you need behavior, expose it as a module-level `func` taking `self`. Stable compatibility is checked by the compiler on upgrade — see `motoko-general-style-guidelines` ("Objects and records": *"Only records can be sent as message parameters or results and can be stored in stable variables."*).

3. **Mutability lives on the fields, not on the binding.** Use `stable var arr : Foo.Foo<Nat> = Foo.new();` (the outer `var` lets you reassign; the inner `var` fields let methods mutate). A `let` binding still works because the fields themselves are `var`, but `var` is the idiomatic choice when the value can be replaced wholesale (e.g. `arr := Foo.new()`).

4. **Re-introduce generics on every function.** A class `Foo<T>` introduces `T` once; the module functions each need their own `<T>` (or `<K, V>`, etc.). Forgetting this is the most common compile error during migration.

5. **Internal helper methods.** Private `func`s inside the class become private (non-`public`) module functions. If they accessed `self`, add `self : Foo<T>` as their first parameter too.

6. **Don't expose factory functions as `self`-methods.** `new`, `empty`, `fromArray`, ... do NOT take `self` and must NOT be called with dot-notation. This matches `List.empty()`, `List.fromArray(xs)` in `mo:core`.

7. **Equality / comparison helpers.** If the class implemented structural equality via a method, expose it as `equal<T>(self : Foo<T>, other : Foo<T>, eq : (T, T) -> Bool) : Bool` so callers write `a.equal(b, Nat.equal)`.

8. **Iteration.** Provide `values`, `keys`, `entries`, ... as `func values<T>(self : Foo<T>) : Iter.Iter<T>` so `for (x in xs.values()) { ... }` works the same way as on `mo:core` collections.

## Minimal Example (end-to-end)

### Before — class style

```motoko
// src/ArrayWrapper.mo
import Array "mo:base/Array";

module {
  public type SharedData<T> = { arr : [T] };

  public class ArrayWrapper<T>() {
    var arr : [T] = [];

    public func push(e : T) {
      arr := Array.append(arr, [e]);
    };

    public func size() : Nat = arr.size();

    public func share() : SharedData<T> = { arr };
    public func unshare(d : SharedData<T>) { arr := d.arr };
  };
};
```

```motoko
// main.mo
import ArrayWrapper "ArrayWrapper";

actor {
  stable var data : ArrayWrapper.SharedData<Nat> = { arr = [] };
  let wrapper = ArrayWrapper.ArrayWrapper<Nat>();

  system func preupgrade()  { data := wrapper.share() };
  system func postupgrade() { wrapper.unshare(data) };

  public func add(n : Nat) : async () { wrapper.push(n) };
  public query func count() : async Nat { wrapper.size() };
};
```

### After — static module style

```motoko
// src/ArrayWrapper.mo
import Array "mo:base/Array";

module {
  public type ArrayWrapper<T> = { var arr : [T] };

  public func new<T>() : ArrayWrapper<T> = { var arr = [] };

  public func push<T>(self : ArrayWrapper<T>, e : T) {
    self.arr := Array.append(self.arr, [e]);
  };

  public func size<T>(self : ArrayWrapper<T>) : Nat = self.arr.size();
};
```

```motoko
// main.mo
import ArrayWrapper "ArrayWrapper";

actor {
  stable var wrapper : ArrayWrapper.ArrayWrapper<Nat> = ArrayWrapper.new();

  // Call sites are identical to the class version, thanks to dot-notation:
  public func add(n : Nat) : async () { wrapper.push(n) };
  public query func count() : async Nat { wrapper.size() };
};
```

What disappeared: `class`, `share`, `unshare`, the `SharedData` type, the auxiliary `let wrapper = ...`, `preupgrade`, and `postupgrade`.

## Step-by-Step Procedure

**Migrate classes one at a time, bottom-up along the dependency tree.** Do NOT attempt to convert every class in the package in a single sweep — that produces a long stretch of broken code with no way to tell which change caused which failure.

### Pre-flight: verify the package is healthy *before* changing anything

Before touching any source file, confirm that the package currently builds and passes its own checks. Run, from the package root:

1. `mops test` — must pass.
2. `mops bench` — must pass, *if* a `bench/` folder exists in the package. If there is no `bench/` folder, skip this step.
3. Build every example project — *if* an `example/` or `examples/` directory exists at the package root. Treat each immediate subdirectory of `example*/` that contains a `mops.toml` (or otherwise looks like a buildable project) as one example, and run its build (typically `mops build` or `dfx build`, whichever the example uses). If no `example/` / `examples/` directory exists, skip this step.

If **any** of the above fails, **stop immediately** and report to the user with a message like:

> Pre-flight check failed: `<which step>` is broken on the current `main` (before any migration). Fix the package to a known-good state first, then re-run the migration.

Do not start the migration until every applicable pre-flight check is green. The whole point of the per-class gating below is to attribute failures to the class you just migrated — that only works if the starting point is clean.

### 0. Build the dependency order (do this once, before touching code)

1. List every `class` in the package.
2. For each class, note which *other classes from the same package* appear in its fields, constructor, or method signatures/bodies.
3. Sort the list so that **leaf / utility classes come first** and the **top-level / "main" class comes last**. A class may only be migrated after every class it depends on has already been migrated.
4. If there is a dependency cycle between classes, break it first (extract a shared type, or merge the classes) — cycles cannot be migrated in this scheme.

### 1. For each class, in that order:

1. **Inventory** the public surface of the class: state fields, constructor params, public methods, private helpers, `share`/`unshare` shapes.
2. **Design the record type.** Decide which fields are `var` vs `let`, and confirm every field type is itself stable. Fields that hold *already-migrated* classes from step 0 are now plain records and are fine; fields that hold *not-yet-migrated* classes mean you picked the wrong order — go back to step 0.
3. **Write `new` / factory functions** matching `mo:core` naming (`empty`, `singleton`, `fromIter`, ...).
4. **Convert methods one by one** to `func name<T>(self : Foo<T>, ...)`, replacing `field` → `self.field` and method calls → `self.method(...)`.
5. **Delete `share` / `unshare`** and the `SharedData` type (or keep a one-release alias `type SharedData<T> = Foo<T>`).
6. **Update in-package call sites** of *this* class (other modules in the package, tests, benchmarks, and any projects under `example/` / `examples/`): remove `share`/`unshare`/`preupgrade`/`postupgrade`, change stable var types, keep call sites as-is thanks to dot-notation.
7. **Run `mops test`.** Fix every failure before going further — missed `<T>`, missed `self.` prefix, leftover `share`/`unshare` calls, etc.
8. **Run `mops bench`** *if a `bench/` folder exists in the package*. Fix any compile or runtime errors the same way.
9. **Build every example project** under `example/` / `examples/` *if such a directory exists*. Each example must compile; update its sources too if the migrated class's API surface changed in a way that affects them (it usually doesn't, thanks to dot-notation, but the stable-var type and any leftover `share`/`unshare`/`preupgrade`/`postupgrade` plumbing in the example actor will need to be updated).
10. **Only when `mops test`, (if present) `mops bench`, and (if present) every example build pass cleanly, move on to the next class in the dependency order.** Never start migrating class N+1 while class N still has failing tests, benchmarks, or example builds.

### 2. After the last (top-level) class is migrated

1. Update the consumer actor: remove `system func preupgrade` / `postupgrade` (when they only did `share` / `unshare`), and declare the value directly as `stable var x : Foo.Foo<...> = Foo.new()`.
2. Run `mops test`, `mops bench` (if present), and rebuild every project under `example/` / `examples/` (if present) one final time across the whole package.

> Scope note: this skill only modifies the package's own Motoko sources (and its tests/benchmarks). Do **not** touch `mops.toml`, `package-set.dhall`, lockfiles, or any other dependency / release metadata — versioning and release decisions are out of scope and are handled separately by the maintainer.

## Verification Checklist

- [ ] The package exports a `module`, not a `class`.
- [ ] The main type is a record with only stable field types.
- [ ] Every operation function's first parameter is literally named `self`.
- [ ] Factory functions (`new`, `empty`, `fromX`) do NOT take `self`.
- [ ] All generic parameters are re-introduced on each function.
- [ ] `share` / `unshare` / `SharedData` are gone (or temporarily aliased).
- [ ] Consumer actors store the value directly in `stable var` with no `preupgrade` / `postupgrade`.
- [ ] All call sites use dot-notation (`xs.push(e)`) and read identically to the old class API.
- [ ] Classes were migrated one at a time, from leaves up to the top-level class.
- [ ] Before starting the migration, a pre-flight check (`mops test`, `mops bench` if present, and building every project under `example/` / `examples/` if present) was run and all of it passed.
- [ ] After every single class migration, `mops test` was run and passed.
- [ ] After every single class migration, `mops bench` was run and passed (when a `bench/` folder exists).
- [ ] After every single class migration, every project under `example/` / `examples/` was rebuilt and built successfully (when such a directory exists).
- [ ] `mops test`, `mops bench`, and every example build pass on the final, fully-migrated package.
- [ ] No changes were made to `mops.toml` or any other dependency / release metadata — only the package's own Motoko sources, tests, and benchmarks were edited.
