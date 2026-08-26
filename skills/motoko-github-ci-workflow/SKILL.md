---
name: motoko-github-ci-workflow
description: "Add a GitHub CI workflow for PRs to a Motoko repository, including mops tests, benchmarks, formatting checks, and building canisters or examples."
compatibility: "mops >= 3.0.0, node >= 22"
metadata:
   title: "Motoko GitHub CI Workflow"
   category: DevOps
   internal: false
---

# Motoko GitHub CI Workflow

## What This Is

This skill provides a comprehensive playbook for adding or updating a GitHub Actions CI workflow for Motoko projects. It covers automated testing, benchmarking, code formatting, and building (canisters or examples) using `mops`, `moc`, `prettier`, `icp-cli`, and `dfx`. It is a perfect companion to the `mops-package-maintenance` skill.

For general information on GitHub Actions, refer to the [GitHub Actions Documentation](https://docs.github.com/en/actions). For Motoko-specific examples, you can explore public repositories in the [research-ag](https://github.com/research-ag) organization.

## Prerequisites

- `mops` CLI (local: `npm i -g ic-mops`, CI: `caffeinelabs/setup-mops@v1`)
- `node` (latest LTS recommended, at least v22 for Prettier)
- `pocket-ic` for running `mops bench`. As of mops CLI v3.0.0, dfx-replica support is gone — PocketIC is the only runtime `mops bench` (and replica tests / `--check-deploy`) can use, and it must be **explicitly pinned** via `[toolchain] pocket-ic = "<version>"` in `mops.toml` (e.g. `mops toolchain use pocket-ic 15.0.0`). Without that pin, `mops bench` errors outright. CI installs the pinned binary via `mops toolchain bin pocket-ic` (see Step 1).
- `wasm-opt` for benchmarks that should reflect optimized (production-representative) Wasm. `mops bench`/`mops build` only run Binaryen's `wasm-opt` when `mops.toml` has an `[optimize]` section — and that section requires an explicit `[toolchain] wasm-opt = "<version>"` pin, same as `pocket-ic` (see Step 1).

## How It Works

1. **Analyze the Repository**:
   - Determine if it's a Motoko package (has `[package]` in `mops.toml`) or a canister project (has `[canister]` in `mops.toml` or `dfx.json`).
   - **Check for tests**: Look for a `test/` directory and `*.test.mo` files. If none are found, do NOT include the `mops test` step.
   - **Check for benchmarks**: Look for a `bench/` directory and `*.bench.mo` files. If none are found, do NOT include the `mops bench` step.
2. **Analyze Existing CI**:
   - If `.github/workflows/` already contains CI files, analyze them to see which tools are used (`dfx` vs `icp-cli` or none).
   - If tests/benchmarks exist in the repo but are missing from the CI, add them.
3. **Select Canister Build Tool**:
   - If the project is a library package (no canisters, and no examples requiring a canister build tool), skip `dfx` or `icp-cli` installation entirely. Note that `moc` (the Motoko compiler) is still required for tests and benchmarks, and is usually managed by `mops`.
   - For new projects with canisters/examples, prefer `icp-cli` as it is newer and lighter.
   - For existing projects, maintain the existing toolset (`dfx` or `icp-cli`).
   - **CRITICAL**: Do not install `dfx` and `icp-cli` simultaneously.
4. **Inspect `mops.toml`**: Read the existing `[toolchain]` section. Unlike `moc`, `pocket-ic` is not optional when the package has benchmarks (or replica tests / `--check-deploy`) — `mops bench` requires it to be pinned. If `[toolchain] pocket-ic` is missing and the package has benchmarks, flag it to the user and add the pin (`mops toolchain use pocket-ic <version>`, e.g. `15.0.0`) rather than letting CI fail later. Also check for `[optimize]` — if the package has benchmarks and it's missing, add it (with a `[toolchain] wasm-opt` pin) so bench numbers reflect optimized Wasm; if the `mops-package-maintenance` skill already ran, this is usually already done. Other `[toolchain]` entries and the `files` field stay hands-off (see Step 1).
5. **Create or Update Workflow File**: Identify existing workflow files in `.github/workflows/` (e.g., `pull_request_build.yml`, `test.yml`, `ci.yml`). If none exist, create `.github/workflows/ci.yml`. Ensure parallel jobs are used for efficiency.
6. **Configure Parallel Jobs**:
   - **`test` job**: Handles dependency installation, `mops test`, and `mops bench`.
   - **`fmt` job**: Handles `prettier` formatting checks.
7. **Add Build Steps**: If examples or canisters are present, add steps to build them using the selected canister build tool.

## Implementation

### Step 1 — Inspect `mops.toml` and pin `pocket-ic`/`wasm-opt` if needed

Before adding the CI, **read** `mops.toml`. Leave `[toolchain] moc` and other entries as-is.

**`pocket-ic` is mandatory for benchmarks (mops CLI v3.0.0+):** dfx-replica support was removed from `mops bench` entirely — PocketIC is now the only runtime, and it must be pinned in `[toolchain]` or `mops bench` fails immediately with an error naming the fix (`mops toolchain use pocket-ic <version>`). `mops test` still does **not** need `pocket-ic` (it runs via the Motoko interpreter or WASI) — the pin only matters when the package has benchmarks, replica tests, or uses `--check-deploy`.

Determine which CI strategy to use based on the current `[toolchain]` content:

- **Package has benchmarks and `pocket-ic` is already pinned in `[toolchain]`**: install it in CI via `mops toolchain bin pocket-ic` and run `mops bench`. See Step 2 template and Pitfall #2 for the exact snippet.
- **Package has benchmarks and `pocket-ic` is absent from `[toolchain]`**: tell the user `mops bench` will fail in CI without a pin, and add one (`mops toolchain use pocket-ic <version>` — mops's own error message names a concrete version, e.g. `15.0.0`). Unlike the `files` field, this is not something to leave hands-off — an unpinned package cannot run `mops bench` at all.
- **No benchmarks**: Nothing to do — `mops test` does not need `pocket-ic` and no benchmark runtime is required.

**`[optimize]` + `wasm-opt` — benchmarks should reflect optimized Wasm.** `mops bench` only runs Binaryen's `wasm-opt` if `mops.toml` has an `[optimize]` section (even an empty one, defaulting to `level = "O3"`); without it, benchmarks silently measure unoptimized code. Like `pocket-ic`, once `[optimize]` is present it needs a `[toolchain] wasm-opt` pin or the build fails before compiling.

- **Package has benchmarks and `[optimize]` is already configured**: install `wasm-opt` in CI via `mops toolchain bin wasm-opt` alongside `pocket-ic`, before `mops bench`.
- **Package has benchmarks and `[optimize]` is missing**: add it (with a `[toolchain] wasm-opt` pin, e.g. `mops toolchain use wasm-opt <version>`) the same way as the `pocket-ic` pin above — this is normally handled by the `mops-package-maintenance` skill during a maintenance pass, but add it here too if that hasn't happened yet, so CI doesn't ship benchmarks that measure unoptimized code.
- **No benchmarks**: Nothing to do.

Also, check if `mops.toml` contains a `files` field under `[package]`. If it does, do NOT modify it. Trying to be smart about the `files` field can lead to unintended side effects. Instead, report its current configuration to the user and warn them if it seems to be missing important directories (like `examples/`) or if it includes unsupported extensions (only `.mo`, `.did`, `.md`, and `.toml` are allowed for `mops publish`).

### Step 2 — Create or Update the GitHub CI Workflow

Identify existing workflow files in `.github/workflows/`. If one exists, update it. Otherwise, create `.github/workflows/ci.yml`. Use the latest versions of actions (e.g., `actions/checkout@v6`, `actions/setup-node@v6`) and parallelize the tasks.

*Note: Using the latest major versions of actions ensures you get the latest features and security updates.*

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
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: latest

      - name: Install mops
        uses: caffeinelabs/setup-mops@v1

      - name: Make sure moc is installed
        run: mops toolchain bin moc

      - name: Show versions
        run: |
          mops --version
          $(mops toolchain bin moc) --version

      - name: Install dependencies
        run: mops install

      # Bench runtime: PocketIC is the only runtime `mops bench` supports
      # (dfx-replica support was removed in mops CLI v3.0.0). `[toolchain]
      # pocket-ic` must already be pinned in mops.toml (see Step 1) —
      # `mops toolchain bin pocket-ic` downloads that pinned version.
      # `mops bench` starts (and stops) PocketIC itself, so there is no
      # separate "start" step. Omit both steps if the package has no
      # benchmarks.
      - name: Make sure pocket-ic is installed
        run: mops toolchain bin pocket-ic

      # Only if mops.toml has an [optimize] section (see Step 1) — it
      # requires a [toolchain] wasm-opt pin, downloaded the same way.
      # Omit this step if the package has no [optimize] section.
      - name: Make sure wasm-opt is installed
        run: mops toolchain bin wasm-opt

      - name: Run tests
        run: mops test  # Omit this step if the package has no tests. Note: `mops test` does not use pocket-ic.

      - name: Run benchmarks
        run: mops bench  # Omit this step if the package has no benchmarks.

  fmt:
    name: Formatting Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: latest

      - name: Prettier Check
        run: |
          npm install prettier prettier-plugin-motoko --no-save
          npx -y prettier --plugin prettier-plugin-motoko --check '**/*.{mo,json,md}'
```

### Step 3 — Handle Examples and Canisters (Optional)

#### If the repo has examples (or example) that are canisters:
Add a step to the `test` job (or a new job) to build examples. Use the tool already present in the project or `icp-cli` for new ones. Omit this section if no examples/canisters need building.

```yaml
      - name: Build examples
        run: |
          # Detect and build examples/example
          EXAMPLES_DIR=""
          if [ -d "examples" ]; then
            EXAMPLES_DIR="examples"
          elif [ -d "example" ]; then
            EXAMPLES_DIR="example"
          fi

          if [ -n "$EXAMPLES_DIR" ]; then
            cd "$EXAMPLES_DIR"
            if [ -f "mops.toml" ]; then
              mops install
            fi
            
            # Use icp if it exists, otherwise fallback to dfx if dfx.json is present
            if [ -f "icp.yaml" ] && command -v icp >/dev/null; then
              icp build --all
            elif [ -f "dfx.json" ]; then
              # Only run dfx build if dfx is already installed in this job
              if command -v dfx >/dev/null; then
                dfx build
              fi
            fi
            cd ..
          fi
```

#### If the repo is a Canister (not a package):
If the project is a canister, you should build the canisters and optionally compare `.did` files.

**Install tools (Choose ONE):**
```yaml
      # Option A: For new projects or those already using icp
      - name: Install icp-cli
        run: npm i -g icp-cli

      # Option B: For existing projects already using dfx
      - name: Install dfx
        uses: dfinity/setup-dfx@main
```

**Build and Compare DID:**
Comparing `.did` files often requires `didc` to check the latest interface.

```yaml
      - name: Build canisters
        run: |
          if [ -f "icp.yaml" ] && command -v icp >/dev/null; then
            icp build
          else
            dfx build
          fi

      - name: Get didc
        run: |
          mkdir -p /home/runner/bin
          release=$(curl --silent "https://api.github.com/repos/dfinity/candid/releases/latest" | awk -F\" '/tag_name/ { print $4 }')
          curl -fsSL https://github.com/dfinity/candid/releases/download/$release/didc-linux64 > /home/runner/bin/didc
          chmod +x /home/runner/bin/didc
          echo "/home/runner/bin" >> $GITHUB_PATH

      - name: Check Candid interface
        run: |
          # Compare generated .did with tracked .did
          # e.g., didc check -s did/my_canister.did .dfx/local/canisters/my_canister/my_canister.did
          didc check -s did/backend.did .dfx/local/canisters/backend/backend.did
```

### Step 4 — Run End-to-End Tests
If the repository contains end-to-end tests (e.g., in `test/e2e` or similar), add a step to run them. This usually requires `dfx` and a running local replica or `pocket-ic`.

```yaml
      - name: Run E2E tests
        run: |
          # Your E2E test command here
          # e.g., npm run test:e2e
```

## Common Pitfalls

1. **Outdated Node version.** Prettier and some Motoko tools require recent Node.js versions. Always use `latest` or at least `v22` in CI.
2. **Benchmarks require a `pocket-ic` toolchain pin — there is no `dfx` fallback.** mops CLI v3.0.0 removed dfx-replica support entirely (the `--replica` flag on `mops test`/`mops bench` is also gone — it had been deprecated since 2.14). PocketIC is now the only runtime. If the package has **benchmarks**, make sure `[toolchain] pocket-ic` is pinned in `mops.toml` (add it with `mops toolchain use pocket-ic <version>` if missing — an unpinned project errors out naming the exact fix), then install the pinned binary and run bench (only in the job that runs `mops bench`). There is no need for a separate "start" step — `mops bench` starts (and stops) PocketIC itself:

    ```yaml
          - name: Make sure pocket-ic is installed
            run: mops toolchain bin pocket-ic
          - run: mops bench
    ```

    Note: `mops test` does not use `pocket-ic` (it runs via the Motoko interpreter or WASI), so the pin only matters for `mops bench` / replica tests / `--check-deploy`.

    **Remove any leftover `dfx` bench-runtime steps.** If you are updating an existing workflow that still installs `dfx` (`dfinity/setup-dfx@main`) or runs `dfx start --background --clean` purely to back `mops bench`, delete those steps — they no longer do anything for `mops bench` and cannot substitute for the `pocket-ic` pin:

    ```yaml
          # REMOVE these if present purely for `mops bench` — dfx-replica
          # support was removed in mops CLI v3.0.0:
          - name: Install dfx
            uses: dfinity/setup-dfx@main
          - name: Start dfx
            run: dfx start --background --clean
    ```

    (A `dfx` install is still legitimate for other jobs — e.g. building canisters/examples or E2E tests, see Step 3/4. This pitfall is specifically about the `mops bench` runtime.)
3. **Missing `pocket-ic` pin.** If `mops bench` (or `mops toolchain bin pocket-ic`) fails immediately with an error about an unpinned toolchain, `[toolchain] pocket-ic` is missing from `mops.toml` — pin it with `mops toolchain use pocket-ic <version>` (the error message names a concrete version) rather than trying to work around it with `dfx`.
4. **Benchmarking without `[optimize]`.** `mops bench` only runs `wasm-opt` if `mops.toml` has an `[optimize]` section — omit it and CI happily reports numbers for unoptimized Wasm, which don't represent what ships in production. If the package has benchmarks and `[optimize]` is missing, add it (with a `[toolchain] wasm-opt` pin) rather than treating "CI passes" as "CI measures the right thing" — see Step 1.
5. **Not showing versions.** Always include a step to show `mops` and `moc` versions. This helps in debugging CI issues.
6. **Not using parallel jobs.** Running formatting and tests in the same job is slower. Use separate jobs so GitHub runs them in parallel.
7. **Including `mops test` when no tests exist.** Always check if the package has a `test/` directory with `*.test.mo` files. If not, omit the `mops test` step to avoid CI failures.
8. **Including `mops bench` when no benchmarks exist.** Always check if the package has a `bench/` directory with `*.bench.mo` files. If not, omit the `mops bench` step.
9. **Old mops installation.** Avoid `npm i -g ic-mops` or `ZenVoich/setup-mops@v1`. Use `caffeinelabs/setup-mops@v1` instead.
10. **Simultaneous tool installation.** Do not install both `dfx` and `icp-cli`. Stick to one.
11. **Skipping existing tests.** If you are updating an existing CI, ensure it runs all tests and benchmarks found in the repo.
12. **Unnecessary tool installation.** Do not install `dfx` or `icp-cli` for library packages that only run `mops test` with no benchmarks. If the package has benchmarks, `pocket-ic` IS required (it is the only `mops bench` runtime) — install it via `mops toolchain bin pocket-ic` in the bench job only (plus `wasm-opt` if `[optimize]` is set). Do not install `dfx` or `icp-cli` just to run `mops bench`.

## Verify It Works

1. **Check GitHub Actions tab.** Ensure both the `test` and `fmt` jobs are present and running.
2. **Review logs.** Verify that `mops install` correctly pulls dependencies and `mops bench` runs against PocketIC, the version pinned in `[toolchain] pocket-ic` (the `--replica` flag no longer exists as of mops CLI v3.0.0). If `mops.toml` has an `[optimize]` section, also confirm the `wasm-opt` step ran — its output should appear before the benchmark results.
3. **Check formatting.** Intentionally misformat a file and open a PR to ensure the `fmt` job fails as expected.
