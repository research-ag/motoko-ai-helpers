---
name: motoko-mops-package-maintenance
description: Automate maintaining a Motoko mops package — upgrade dependencies, fix compiler warnings, run tests and benchmarks, fix breakages, update CHANGELOG, review doc strings and docs, run the formatter, and prepare a patch release on a new git branch.
---

# Mops Package Maintenance

## What This Is

A step-by-step playbook for an AI agent to fully maintain a Motoko package
published on [MOPS](https://mops.one): upgrade dependencies, validate the
package, polish docs, format the code, ensure CI, and prepare a versioned
release branch — all in one automated pass.

## When to Use

- Periodic dependency-bump maintenance runs.
- Before cutting a new release.
- When the user asks to "update deps", "maintain the package", or
  "prepare a release".

## Key Conventions (apply throughout)

These rules are referenced from individual steps. Read them once.

- **`pocket-ic` is the only bench runtime — pin it like `moc`.** mops
  CLI v3.0.0 removed dfx-replica support entirely (the `--replica`
  flag on `mops test`/`mops bench` is gone too). If the package has
  benchmarks, replica tests, or uses `--check-deploy`, `[toolchain]
  pocket-ic` **must** be pinned or `mops bench` fails outright. Unlike
  the `files` field, this is not hands-off: if it's missing, add it
  with `mops toolchain use pocket-ic <version>` (mops's own error
  names a concrete version, e.g. `15.0.0`) the same way `[toolchain]
  moc` is actively managed.
- **Hands-off fields in `mops.toml`:** never auto-edit `[package] files`
  or auto-add entries to `[toolchain]`. Report and warn instead.
  **Exception: `[toolchain] pocket-ic`** — see the bullet above; add it
  when benchmarks require it, same as `moc`.
- **CHANGELOG only lists consumer-visible bumps:** `[dependencies]`
  and `[requirements]`. Never list `[toolchain]` or
  `[dev-dependencies]`.
- **Documentation gate:** if MOPS reports Documentation = 100% (Step 1),
  skip Steps 7 and 8 entirely. Otherwise aim for 100% — fill in every
  missing `///` doc string on a public declaration at minimum.
- **No `moc --check` in CI.** It's a local-only check the agent runs
  during maintenance; different CI compiler versions could cause
  spurious failures for consumers.
- **`mops add` rewrites `mops.toml`** and can silently reset
  `[requirements] moc` to `"1.0.0"`. Diff after every `mops add` and
  restore the intended value.

## Prerequisites

- `mops` CLI (`npm i -g ic-mops`)
- `moc` (Motoko compiler; pinned via `[toolchain] moc` in `mops.toml`
  and fetched with `mops toolchain bin moc` — as of mops CLI v3.0.0
  there's no dfx-bundled fallback, the pin is mandatory)
- `pocket-ic` (bench runtime; pinned via `[toolchain] pocket-ic` when
  the package has benchmarks — see Key Conventions)
- `dfx` (DFINITY SDK; optional — only needed for building canisters
  or examples, not for `mops bench`)
- `node` >= 22 / `npm`. Verify with `node --version`.
- `git`
- `prettier` and `prettier-plugin-motoko` — install **locally** before
  running prettier; `npx -y` alone does not auto-resolve the plugin in
  a clean checkout.

## Workflow

Work through every step in order. If a step fails, fix it before moving on.

### Step 0 — Create a maintenance branch

```bash
git checkout -b chore/dependency-bump-$(date +%Y-%m-%d)
```

Also snapshot the untracked-file list now so you can tell scratch files
from your own additions in Step 11:

```bash
git status --short > /tmp/baseline.txt
```

### Step 1 — Verify Package Quality on MOPS

1. Open `mops.toml`. If it has no `[package]` section (only `[canister]`,
   or neither), it's not a published package — skip to Step 2.
2. Otherwise visit `https://mops.one/<package-name>` and review the
   **Package Quality** section (Documentation, License, Repository,
   Tests, Benchmarks).
   - `mops.one` is a SPA — `curl`/`WebFetch` see only an empty shell.
     If you can't render it, derive signals locally: run `mo-doc` and
     grep for `(no description)`; verify `[package] license` and
     `[package] repository` in `mops.toml`; confirm `mops test` and
     `mops bench` succeed. Ask the user to confirm anything you can't
     verify programmatically.
3. For any metric that isn't **"yes"** / **"100%"**:
   - **Documentation < 100%**: run the `motoko-doc-strings` skill in
     Steps 7–8. If already 100%, skip Steps 7–8.
   - **Missing License/Repository**: set them in `mops.toml` `[package]`.
   - **Tests/Benchmarks**: handled in Steps 4–5; ensure CI in Step 9c.
4. Raise a warning to the user for anything that can't be auto-fixed.

### Step 2 — Discover outdated dependencies

```bash
mops outdated
# Fallback: mops search <package-name>
```

Record packages needing upgrades.

**`moc` policy:** upgrade `[toolchain] moc` to the latest; set
`[requirements] moc` to the **absolute minimum** that still works
(determined iteratively in Step 3) — do not blindly align it with
the toolchain or with what dependencies declare.

### Step 3 — Upgrade dependencies in `mops.toml`

Update versions for every outdated entry in `[dependencies]` and
`[dev-dependencies]`.

**Determining `[requirements] moc`:**

1. **Initial version** — the max `moc` required by your regular
   `[dependencies]`, **excluding `core`**:
   1. List `[dependencies]` from your root `mops.toml` (skip `core`).
   2. For each, read `.mops/<name>@<version>/mops.toml` and extract
      `[requirements] moc` (skip if absent).
   3. Take the max. If lower than your current `[requirements] moc`,
      keep the current value instead.
2. **Check `core`** — after `mops install`, read
   `.mops/core@<version>/mops.toml` `[requirements] moc`.
3. **Iterative validation** (only if `core`'s requirement > initial):
   You MUST test EVERY intermediate `moc` version sequentially — do
   not skip (e.g., from 1.0.0 → 1.4.0; test 1.1.0, 1.2.0, 1.3.0…).
   For each candidate `X`:
   1. Set `[toolchain] moc = "X"`.
   2. Run `mops test` (if tests exist).
   3. Run `mops bench` (if benchmarks exist).
   4. Build examples (Motoko canisters only — ignore asset/Rust):
      ```bash
      EXAMPLES_DIR=""
      if [ -d "examples" ]; then EXAMPLES_DIR="examples"
      elif [ -d "example" ]; then EXAMPLES_DIR="example"; fi
      if [ -n "$EXAMPLES_DIR" ]; then
        cd "$EXAMPLES_DIR"
        mops install
        if [ -f "icp.yaml" ] && command -v icp >/dev/null; then icp build --all
        elif [ -f "dfx.json" ]; then dfx build; fi
        cd ..
      fi
      ```
   5. If all pass, set `[requirements] moc = "X"` and stop.
4. **Failure:** if even `core`'s required version fails, revert
   `[requirements] moc` to the initial value and warn the user.
5. **Final sync:** restore `[toolchain] moc` to the latest version.

Then install:

```bash
mops install
```

Verify `mops install` succeeded without resolution errors and that
`mops.lock` updated. Never hand-edit the lock file.

#### Step 3a — Sync nested `mops.toml` files

Find every nested manifest (examples, sub-canisters, etc.):

```bash
find . -name mops.toml -not -path "./node_modules/*" -not -path "./.mops/*"
```

For each nested `mops.toml`:

- Third-party `[dependencies]` versions should generally match the root
  (examples may add extras, but shared deps should align).
- `[toolchain] moc` must match the root and exist in the fork used by
  the CI `setup-mops` action.
- The package being maintained must reference itself by its **real
  package name** via a **relative local path**
  (e.g. `self-package-name = "../"`), with a comment showing the latest
  published version for easy manual swap by consumers.
- **Replace relative source imports** in example `.mo` files:
  `import "../../src/Main"` → `import "mo:self-package-name/Main"`.
  This ensures examples are copy-paste ready.

Example `examples/mops.toml`:

```toml
[dependencies]
core = "2.5.0"
self-package-name = "../"
# Replace with:
# self-package-name = "0.0.3"

[toolchain]
moc = "1.6.0"
```

#### Step 3b — Audit repo hygiene files

- **`.gitignore`** — create if missing; ensure it contains:
  ```text
  .mops/
  .icp/
  mops.lock        # for libraries only — mops install reminds you
  node_modules/
  package.json
  package-lock.json
  .dfx/
  build/
  skills-lock.json
  .agents
  ```
  Personal/editor files (`.idea`, `.vscode`, `.claude`, `.junie`,
  `.copilot`, `.tmp`, `*.swp`, `.DS_Store`) belong in the developer's
  global ignore, not here.
- **`package-lock.json`** — if absent, don't introduce it.
- **`package.json`** — if present, ensure its `license` matches
  `mops.toml`. If absent, don't introduce it (and don't commit one
  that `npm` may create).
- **`mops.toml` `[package] files`** — hands off (see Key Conventions).
  Report its contents to the user and warn if something looks off
  (e.g., `examples/` exists but isn't listed). Only `.mo`, `.did`,
  `.md`, `.toml` are supported by `mops publish` — warn if other
  extensions appear.

#### Step 3c — Fix compiler warnings

Run the `fix-compiler-warnings` skill. Only act on warnings/errors
the compiler actually emits — do not invent "improvements".

Capture warnings for **Motoko canisters only** (ignore asset/Rust):

```bash
# DFX projects:
dfx build --check 2>&1 | tee /tmp/dfx_build_output.txt

# MOPS packages:
find src -type f -name "*.mo" -print0 | xargs -0 -n1 $(mops toolchain bin moc) --check $(mops sources) 2>&1 | tee /tmp/moc_check_output.txt

# ICP-CLI projects:
icp build 2>&1 | tee /tmp/icp_build_output.txt
```

Fix one warning class at a time, re-run the check, and run
`mops test` / `mops bench` if relevant to catch regressions.

### Step 4 — Run tests

```bash
mops test
```

If tests fail, identify whether an upgraded dependency changed its
API, adapt the source, and re-run. Note any non-trivial changes
(renames, signature changes) for the CHANGELOG.

### Step 5 — Run benchmarks

```bash
mops bench
```

Apply Step 4-style fixes for compile/run failures.

**Environmental failures vs code failures.** If `mops bench` errors
**after** the `Deploying canisters...` line — typically:

- `TrustError: Certificate verification error: "Invalid signature"`
- `UND_ERR_SOCKET` / `fetch failed`
- `Could not find the PocketIC binary`

— benchmarks compiled fine; this is a `pocket-ic`/`mops` agent
compatibility issue, not your bump. In order: kill stale `pocket-ic`
processes; confirm `[toolchain] pocket-ic` is pinned and re-fetch the
binary (`mops toolchain bin pocket-ic`), re-pinning to a newer
version with `mops toolchain use pocket-ic <version>` if it looks
stale or missing (see Key Conventions — there is no dfx-replica
fallback to drop back to as of mops CLI v3.0.0); as a last resort,
drop the `bench` job from CI and flag the user. Don't burn the
release chasing this.

Significant regressions: note for the CHANGELOG but don't block the
release unless the user asks.

### Step 6 — Update the CHANGELOG

Open or create `CHANGELOG.md` and add a section at the top for the
upcoming version (number finalized in Step 10).

Include (only `[dependencies]` and `[requirements]` — see Key
Conventions):

- **Dependencies bumped** — list each upgraded `[dependencies]` entry,
  old → new.
- **Breaking / notable changes** — source-code adaptations.
- **Bug fixes** — anything found and fixed during the run.

If no CHANGELOG exists, create one:

```markdown
# Changelog

## Unreleased

### Changed

- Updated `core` from `2.0.0` to `2.5.0`.
- Bumped `bench` from `1.0.0` to `2.0.1`.
- Updated `[requirements] moc` from `1.2.0` to `1.3.0` (only if changed).

### Fixed

- Adapted `<function>` to new `<dep>` API (renamed `old` → `new`).
```

Newest version on top. Match an existing CHANGELOG style if present.

### Step 7 — Review and improve doc strings

Skip if MOPS Documentation = 100% (see Key Conventions). Otherwise
aim for 100%.

Scan every `.mo` file under `src/` for public declarations
(`type`, `func`, `actor`, `actor class`, `let`, `module`). For each:

1. Ensure a `///` doc string exists directly above.
2. From a first-time user's perspective: is the purpose clear? Are
   argument types/units/constraints documented? Trap/error behavior?
   Examples for non-trivial functions?
3. Improve or add as needed.

If the `motoko-doc-strings` skill is installed, follow its checklist.

### Step 8 — Review README and other Markdown

Skip if MOPS Documentation = 100%.

For `README.md` and every other `.md`:

1. Check factual accuracy (versions, API names, examples).
2. Fix outdated information; improve clarity, grammar, completeness.
3. Add sections for new API surface from upgraded deps.
4. Ensure `README.md` documents the formatter command, e.g.
   `npx -y prettier --plugin prettier-plugin-motoko --write '**/*.{mo,json,md}'`
   (add a "Development" or "Formatting" section if missing).

### Step 9 — Formatting

#### 9a — Prettier configuration

If `.prettierrc` is missing, or exists but is trivial (e.g. only
`tabWidth`), write:

```json
{
  "plugins": ["prettier-plugin-motoko"],
  "bracketSpacing": true,
  "printWidth": 80,
  "semi": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "useTabs": false
}
```

#### 9b — Format the code

`prettier`'s `--plugin` flag does not auto-resolve packages. Install
the plugin locally first (a bare `npx -y prettier --plugin
prettier-plugin-motoko ...` fails with `Cannot find package
'prettier-plugin-motoko'` in a clean checkout):

```bash
npm install --no-save prettier prettier-plugin-motoko
npx prettier --plugin prettier-plugin-motoko --write '**/*.{mo,json,md}'
```

#### 9c — Verify or add CI workflow

If the `motoko-github-ci-workflow` skill is installed, follow its
instructions instead of this section.

Otherwise create/update `.github/workflows/ci.yml` with tests +
formatting. If the package has **benchmarks**, `[toolchain]
pocket-ic` must be pinned (per Key Conventions) and installed in CI
with `mops toolchain bin pocket-ic` before `mops bench` — PocketIC is
the only bench runtime as of mops CLI v3.0.0. `mops bench` starts and
stops its own PocketIC instance — never add a manual `dfx start
--background` step; remove any you find.

```yaml
name: CI

on:
  push:
    branches: [main, master]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  test:
    name: Tests and Benchmarks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: latest
      - uses: caffeinelabs/setup-mops@v1
      - run: mops install
      - run: mops test   # Omit if no tests
      # Benchmarks: pocket-ic is the only bench runtime (mops CLI
      # v3.0.0 removed dfx-replica support). Requires [toolchain]
      # pocket-ic to be pinned in mops.toml. Omit both steps if there
      # are no benchmarks.
      - name: Make sure pocket-ic is installed
        run: mops toolchain bin pocket-ic
      - run: mops bench

  fmt:
    name: Formatting Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: latest
      - name: Prettier Check
        run: |
          npm install prettier prettier-plugin-motoko --no-save
          npx -y prettier --plugin prettier-plugin-motoko --check '**/*.{mo,json,md}'
```

Per Key Conventions, do **not** add a `moc --check` step to CI.

### Step 10 — Bump the version

1. In root `mops.toml`, increment the version. Default = **patch**
   (`1.2.3` → `1.2.4`). Bump **minor** (and reset patch) if any of:
   - `[requirements] moc` was raised (consumers need a newer compiler);
   - a `[dependencies]` package crossed a major and this package
     re-exports types/functions from it;
   - source signatures or runtime semantics consumers might rely on
     changed.

   For `0.x.y`, shift the rule one segment right.

   ```toml
   [package]
   name = "my-package"
   version = "1.2.4"   # was 1.2.3
   ```

2. Update the CHANGELOG `Unreleased` header to the new number:
   `## 1.2.4`.

3. Update self-dependency comments in every nested `mops.toml`
   (from Step 3a) to the new version:

   ```toml
   self-package-name = "../"
   # Replace with:
   # self-package-name = "1.2.4"
   ```

### Step 11 — Commit and push

**Audit untracked files first.** Compare `git status --short` against
`/tmp/baseline.txt` (Step 0). Files that predate your work
(`*.debug.bench.mo`, `roxy.txt`, ad-hoc experiments) are scratch —
leave unstaged, `.gitignore` them, or delete. Never `git add -A`
blindly. Ask the user if unsure.

Stage changes. If `package.json` / `package-lock.json` didn't exist
at the start but `npm install` created them, do **not** commit them.

```bash
git add -u
git add .gitignore CHANGELOG.md .prettierrc .github/workflows/*.yml 2>/dev/null || true
git status
git commit -m "chore: bump dependencies and prepare v<NEW_VERSION>"
```

Inform the user:

```text
Branch: chore/dependency-bump-YYYY-MM-DD
Ready for review. Run `git push -u origin HEAD` to push.
```

## Common Pitfalls

1. **Major-version dep bumps.** If a MOPS dep jumps a major
   (`1.x` → `2.x`), read its CHANGELOG — the API likely changed. Flag
   non-trivial migrations to the user.
2. **`[requirements] moc` exhaustive search.** Test EVERY intermediate
   version (Step 3); never skip. Prefer the lowest version that
   passes all checks, even below what `core` declares, to maximize
   consumer compatibility.
3. **Lock file drift.** Always run `mops install` after editing
   `mops.toml`. Never hand-edit `mops.lock`.
4. **CHANGELOG ordering.** Newest at the top.

## Verify It Works

```bash
mops install
mops test
mops bench
```

All must succeed with no errors.
