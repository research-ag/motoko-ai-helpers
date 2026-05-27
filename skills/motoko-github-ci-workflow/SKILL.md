---
name: motoko-github-ci-workflow
description: "Add a GitHub CI workflow for PRs to a Motoko repository, including mops tests, benchmarks, formatting checks, and building canisters or examples."
compatibility: "mops >= 0.45.0, node >= 22"
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
- `dfx` (installed in CI via `dfinity/setup-dfx@main`) for running `mops bench`. `dfx` is the **preferred** runtime for benchmarks — it produces more accurate and stable numbers than `pocket-ic` and has proven significantly more reliable across our repositories. `pocket-ic` is only used in CI when it is *already* present in `mops.toml` `[toolchain]` (see Step 1).

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
4. **Inspect `mops.toml` (do NOT modify `[toolchain]`)**: Read the existing `[toolchain]` section as-is. Do **NOT** add `pocket-ic` (or any other entry) automatically — treat `[toolchain]` like the `files` field. The CI default for benchmarks is to install `dfx` (see Step 1 and Pitfall #2). The only reason to use `pocket-ic` in CI is if it is *already* present in `[toolchain]`; even then, `dfx` is the preferred choice when accurate, stable numbers matter.
5. **Create or Update Workflow File**: Identify existing workflow files in `.github/workflows/` (e.g., `pull_request_build.yml`, `test.yml`, `ci.yml`). If none exist, create `.github/workflows/ci.yml`. Ensure parallel jobs are used for efficiency.
6. **Configure Parallel Jobs**:
   - **`test` job**: Handles dependency installation, `mops test`, and `mops bench`.
   - **`fmt` job**: Handles `prettier` formatting checks.
7. **Add Build Steps**: If examples or canisters are present, add steps to build them using the selected canister build tool.

## Implementation

### Step 1 — Inspect `mops.toml` (do NOT modify `[toolchain]`)

Before adding the CI, **read** `mops.toml` but do **NOT** modify the `[toolchain]` section. Use whatever is already configured.

**CRITICAL — Never auto-add `pocket-ic` (policy):** Do NOT add `pocket-ic` (or change its version) to `mops.toml` automatically — ever. Treat `[toolchain]` like the `files` field — leave it to the user to add manually if they explicitly want it. We favour `dfx` over `pocket-ic` because `dfx` produces more accurate benchmark numbers (in some projects the difference is significant) and has been markedly more reliable. Note that `mops test` does **not** use `pocket-ic` (it runs via the Motoko interpreter or WASI), so `pocket-ic` is only relevant for `mops bench`.

Determine which CI strategy to use based on the current `[toolchain]` content:

- **`pocket-ic` is absent AND the package has benchmarks (the default and recommended case)**: Install `dfx` in CI and run `mops bench` against a local replica — this is the preferred path. See Step 2 template and Pitfall #2 for the exact snippet. Do **NOT** add `pocket-ic` to `mops.toml` to "fix" the missing runtime.
- **`pocket-ic` is already present in `[toolchain]`**: The user has explicitly opted in. You may keep using `pocket-ic` by installing it in CI via `mops toolchain bin pocket-ic` and invoking `mops bench --replica pocket-ic` (the explicit flag makes the choice obvious in the workflow). Only prefer this path over `dfx` if the user has specifically asked for it or if CI time savings clearly outweigh the loss of accuracy and reliability; otherwise still install `dfx` and run plain `mops bench`.
- **`pocket-ic` is absent AND there are no benchmarks**: Nothing to do — `mops test` does not need `pocket-ic` and no benchmark runtime is required.

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
        run: mops toolchain init && mops toolchain bin moc

      - name: Show versions
        run: |
          mops --version
          $(mops toolchain bin moc) --version

      - name: Install dependencies
        run: mops install

      # Bench runtime — preferred path: install dfx and run `mops bench`.
      # `mops bench` starts (and stops) the local replica itself, so there
      # is no need for a separate `dfx start` step. Omit both the install
      # dfx step AND the `mops bench` step if the package has no benchmarks.
      - name: Install dfx
        uses: dfinity/setup-dfx@main
      # Alternative (ONLY if `pocket-ic` is already in `mops.toml` [toolchain]
      # AND the user explicitly prefers it, e.g. for CI speed): drop the
      # `dfx` step above and install pocket-ic instead, then invoke
      # `mops bench --replica pocket-ic` below.
      # - name: Make sure pocket-ic is installed
      #   run: mops toolchain init && mops toolchain bin pocket-ic

      - name: Run tests
        run: mops test  # Omit this step if the package has no tests. Note: `mops test` does not use pocket-ic.

      - name: Run benchmarks
        run: mops bench  # Omit this step if the package has no benchmarks. Use `mops bench --replica pocket-ic` only when using the pocket-ic alternative above.

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
2. **Benchmarks default to `dfx`; never auto-add `pocket-ic` to `mops.toml`.** We favour `dfx` over `pocket-ic`: `dfx` produces more accurate, stable benchmark numbers (the difference can be substantial) and has been significantly more reliable. Never add `pocket-ic` to `mops.toml` automatically (see Step 1). If the package has **benchmarks**, add a `dfx` installation step to the CI workflow **before** the `mops bench` step (only in the job that runs `mops bench`). There is no need for a separate `dfx start` step — `mops bench` starts (and stops) the local replica itself:

    ```yaml
          - name: Install dfx
            uses: dfinity/setup-dfx@main
          - run: mops bench
    ```

    The only exception is when `pocket-ic` is **already** in `mops.toml` `[toolchain]` (user opt-in). In that case you *may* skip installing `dfx` and instead install pocket-ic and invoke `mops bench --replica pocket-ic` — but only do so when the user explicitly prefers it (e.g. for CI runtime). Otherwise still prefer `dfx` even when pocket-ic is configured.

    Note: `mops test` does not use `pocket-ic` (it runs via the Motoko interpreter or WASI), so the choice of benchmark runtime only affects `mops bench`.

    **Remove any redundant `dfx start` step.** If you are updating an existing workflow that already contains a `dfx start --background --clean` (or similar) step before `mops bench`, delete it. `mops bench` starts and stops its own local replica, so a manual `dfx start` step is unnecessary and should be removed:

    ```yaml
          # REMOVE this step if present — mops bench manages the replica itself:
          - name: Start dfx
            run: dfx start --background --clean
    ```
3. **`dfx` installation cost.** Installing `dfx` adds a noticeable amount of CI time, but it is the preferred runtime for `mops bench` (more accurate, more reliable than `pocket-ic`). Do not avoid `dfx` for benchmarks just because of install time. Always use `dfinity/setup-dfx@main` (the official action) rather than shell scripts. Only skip the `dfx` install when the job does not run `mops bench`, has no canisters/E2E/Candid needs, or when `pocket-ic` is explicitly configured in `[toolchain]` and the user has opted into the `mops bench --replica pocket-ic` path.
4. **Not showing versions.** Always include a step to show `mops` and `moc` versions. This helps in debugging CI issues.
5. **Not using parallel jobs.** Running formatting and tests in the same job is slower. Use separate jobs so GitHub runs them in parallel.
6. **Including `mops test` when no tests exist.** Always check if the package has a `test/` directory with `*.test.mo` files. If not, omit the `mops test` step to avoid CI failures.
7. **Including `mops bench` when no benchmarks exist.** Always check if the package has a `bench/` directory with `*.bench.mo` files. If not, omit the `mops bench` step.
8. **Old mops installation.** Avoid `npm i -g ic-mops` or `ZenVoich/setup-mops@v1`. Use `caffeinelabs/setup-mops@v1` instead.
9. **Simultaneous tool installation.** Do not install both `dfx` and `icp-cli`. Stick to one.
10. **Skipping existing tests.** If you are updating an existing CI, ensure it runs all tests and benchmarks found in the repo.
11. **Unnecessary tool installation.** Do not install `dfx` or `icp-cli` for library packages that only run `mops test` with no benchmarks. If the package has benchmarks, `dfx` IS required (it is the preferred `mops bench` runtime) — install it in the bench job only. Do not install `icp-cli` just to run `mops bench`.

## Verify It Works

1. **Check GitHub Actions tab.** Ensure both the `test` and `fmt` jobs are present and running.
2. **Review logs.** Verify that `mops install` correctly pulls dependencies and `mops bench` runs against the expected runtime (the local `dfx` replica by default, or `pocket-ic` only when the user explicitly opted in via `[toolchain]` and `--replica pocket-ic`).
3. **Check formatting.** Intentionally misformat a file and open a PR to ensure the `fmt` job fails as expected.
