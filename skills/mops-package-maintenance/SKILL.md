---
name: motoko-mops-package-maintenance
description: Automate maintaining a Motoko mops package — upgrade dependencies, run tests and benchmarks, fix breakages, update CHANGELOG, review doc strings and docs, run the formatter, and prepare a patch release on a new git branch.
---

# Mops Package Maintenance

## What This Is

A step-by-step playbook for an AI agent to fully maintain a Motoko package
published on [MOPS](https://mops.one). The workflow upgrades all
dependencies, validates the package still works, polishes documentation,
formats the code, and prepares a versioned release branch — all in one
automated pass.

## When to Use

- Periodic dependency-bump maintenance runs.
- Before cutting a new release of a MOPS package.
- When the user asks to "update deps", "maintain the package", or
  "prepare a release".

## Prerequisites

The following tools must be available on the machine:

- `mops` CLI (install: `npm i -g ic-mops`)
- `moc` (Motoko compiler, ships with `dfx`)
- `dfx` (DFINITY SDK)
- `node` / `npm`
- `git`
- `prettier` and `prettier-plugin-motoko` (installed locally per step below)

## Workflow

Work through every step in order. Do NOT skip steps. If a step fails,
fix the issue before moving on.

### Step 0 — Create a maintenance branch

```bash
git checkout -b chore/dependency-bump-$(date +%Y-%m-%d)
```

All subsequent changes are committed on this branch.

### Step 1 — Discover outdated dependencies

Open `mops.toml` and list every dependency under `[dependencies]` and
`[dev-dependencies]`.

For each dependency, check the latest available version:

```bash
mops search <package-name>
```

Compare the installed version with the latest version. Record which
packages need upgrading.

### Step 2 — Upgrade dependencies in `mops.toml`

For each outdated dependency, update the version string in `mops.toml`
to the latest version. Then install:

```bash
mops install
```

Verify the lock file updated cleanly and there are no resolution errors.

#### Step 2a — Sync nested `mops.toml` files (examples, sub-projects)

Search the repository for any additional `mops.toml` files outside the
root (commonly under `examples/`, `bench/`, or sub-canister folders):

```bash
find . -name mops.toml -not -path "./node_modules/*" -not -path "./.mops/*"
```

For **every** nested `mops.toml`, ensure:

- Third-party `[dependencies]` versions match the root.
- `[toolchain] moc` matches the root, and the version exists in the
  fork used by the CI `setup-mops` action.
- The package being maintained references itself by a **relative local
  path** to the directory containing the root `mops.toml` (e.g. `"../"`
  for `examples/mops.toml`, `"../../"` for deeper nesting), never by a
  version string — the new version is not yet published to the registry.

Example `examples/mops.toml`:

```toml
[dependencies]
core = "2.5.0"
self-package-name = "../"   # local path to self

[toolchain]
moc = "1.6.0"
```

#### Step 2b — Audit repo hygiene files

- `.gitignore`: do NOT ignore `package-lock.json` — committing it
  ensures deterministic, reproducible installs across CI and dev.
  Only `node_modules/` and build artifacts should be ignored.
  **IMPORTANT**: Explicitly exclude IDE directories (like `.idea/`, `.vscode/`) and agent-specific directories (like `.agents/`, `.junie/`, `.claude/`, `.copilot/`)
  from the repository. These are local configuration and agent-specific files. Also exclude `skills-lock.json`.
- `package.json` `license` MUST match `mops.toml` `[package] license`.
  Mismatched license metadata produces incorrect data for npm consumers
  and is a compliance risk.

### Step 3 — Run tests

```bash
mops test
```

If tests fail:

1. Read the error output carefully.
2. Identify whether the failure is caused by an API change in an upgraded
   dependency.
3. Fix the source code to accommodate the new API.
4. Re-run `mops test` until all tests pass.

If a fix requires non-trivial changes (renamed functions, changed
signatures, new semantics), note each change — you will need it for the
CHANGELOG.

### Step 4 — Run benchmarks

```bash
mops bench
```

If benchmarks fail to compile or run:

1. Apply the same kind of fixes as in Step 3.
2. Re-run until benchmarks complete.

If benchmarks show a significant regression, note it for the CHANGELOG
but do not block the release unless the user explicitly asks.

### Step 5 — Update the CHANGELOG

Open (or create) `CHANGELOG.md`. Add a new section at the top for the
upcoming version (you will finalize the version number in Step 9).

Required bullet points:

- **Dependencies bumped** — list every dependency that was upgraded and
  its old → new version.
- **Breaking / notable changes** — if any source code had to change to
  accommodate new APIs, describe what changed and why.
- **Bug fixes** — if any bugs were discovered and fixed during the
  process.

Use the existing CHANGELOG style if one exists. If no CHANGELOG exists,
create one with this format:

```markdown
# Changelog

## [Unreleased]

### Changed

- Updated `core` from `2.0.0` to `2.5.0`.
- Bumped `bench` from `1.0.0` to `2.0.1`.

### Fixed

- Adapted `<function>` to new `<dep>` API (renamed `old` → `new`).
```

### Step 6 — Review and improve doc strings

Scan every `.mo` file under `src/` for public declarations. For each
public `type`, `func`, `class`, `let`, and `module`:

1. Check that a `///` doc string exists directly above the declaration.
2. Read the doc string from a first-time user's perspective:
    - Is the purpose clear?
    - Are argument types/units/constraints documented?
    - Is trap / error behavior described?
    - Are examples present for non-trivial functions?
3. Improve or add doc strings where they are missing or unclear.

If a `motoko-doc-strings` skill is installed, follow its full checklist.

### Step 7 — Review README and other Markdown files

Read `README.md` and every other `.md` file in the repository. For each:

1. Check factual accuracy (version numbers, API names, examples).
2. Fix outdated information.
3. Improve clarity, grammar, and completeness where possible.
4. Add any new sections that would help users (e.g., new API surface
   from upgraded deps).

### Step 8 — Format the code with Prettier

#### 8a — Ensure Prettier and the Motoko plugin are available

Check if a `.prettierrc` file exists at the repo root. If not, create one:

```json
{
  "plugins": ["prettier-plugin-motoko"]
}
```

Install Prettier and the Motoko plugin locally:

```bash
npm install --save-dev prettier prettier-plugin-motoko
```

If a `package.json` does not exist, initialize one first and remove the default "scripts" section to avoid failing CI jobs:

```bash
npm init -y
# Remove the default scripts section
node -e "const pkg=require('./package.json'); delete pkg.scripts; require('fs').writeFileSync('./package.json', JSON.stringify(pkg, null, 2))"
npm install --save-dev prettier prettier-plugin-motoko
```

#### 8b — Run the formatter

```bash
npx prettier --write '**/*.mo'
```

If the project also has Markdown or JSON files you want formatted:

```bash
npx prettier --write '**/*.md' '**/*.json'
```

After formatting, re-run tests to make sure formatting did not break
anything (it shouldn't, but verify):

```bash
mops test
```

### Step 9 — Bump the version

Open `mops.toml` and increment the **patch** version. For example, if the
current version is `1.2.3`, change it to `1.2.4`.

```toml
[package]
name = "my-package"
version = "1.2.4"   # was 1.2.3
```

Update the CHANGELOG `[Unreleased]` header to the new version number:

```markdown
## [1.2.4]
```

### Step 10 — Commit and push

**IMPORTANT**: Before committing, verify with `git status` that ONLY the changes you intended to make are staged.
Specifically, ensure that IDE directories (like `.idea/`, `.vscode/`) and agent directories (like `.agents/`, `.junie/`, `.claude/`, `.copilot/`) are NOT staged. If they are, unstage them:
`git restore --staged .idea .vscode .agents .junie .claude .copilot skills-lock.json`

Stage all changes and commit:

```bash
git add -A
git commit -m "chore: bump dependencies and prepare v<NEW_VERSION>"
```

Inform the user that the branch is ready for review and/or pushing:

```
Branch: chore/dependency-bump-YYYY-MM-DD
Ready for review. Run `git push -u origin HEAD` to push.
```

## Common Pitfalls

1. **Don't blindly bump major versions.** A dependency going from `1.x`
   to `2.x` likely has breaking changes. Read the dependency's CHANGELOG
   or release notes before upgrading. If the migration is non-trivial,
   flag it to the user.

2. **Lock file drift.** Always run `mops install` after editing
   `mops.toml` so the lock file stays in sync. Never commit a hand-edited
   lock file.

3. **Formatter changes tests.** `prettier-plugin-motoko` may reformat
   test files. This is fine — the formatting is canonical. But always
   re-run tests afterward to confirm nothing broke.

4. **Missing `node_modules` in `.gitignore`.** After running `npm install`,
   make sure `node_modules/` is listed in `.gitignore`. Add it if missing.

5. **CHANGELOG ordering.** Newest version goes at the top. Don't append
   to the bottom.

6. **Don't skip the doc-string review.** Even if no code changed, the AI
   may have improved since the last pass and can now write better docs.
   Always do a full scan.

## Verify It Works

After completing all steps, run the full validation suite one final time:

```bash
mops install
mops test
mops bench
npx prettier --check '**/*.mo'
```

All four commands must succeed with no errors. If `prettier --check`
reports unformatted files, run `npx prettier --write` again and re-commit.
