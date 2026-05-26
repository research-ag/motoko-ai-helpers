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
- `pocket-ic` version `9.0.3` specified in `mops.toml` `[toolchain]` (if introduced by the agent; otherwise keep existing version) to avoid `dfx` dependency for benchmarks.

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
4. **Inspect `mops.toml` (do NOT modify `[toolchain]`)**: Read the existing `[toolchain]` section as-is. Do **NOT** add `pocket-ic` (or any other entry) automatically — treat `[toolchain]` like the `files` field. If benchmarks exist AND `pocket-ic` is missing, the CI MUST install `dfx` instead (see Step 1 and Pitfall #2).
5. **Create or Update Workflow File**: Identify existing workflow files in `.github/workflows/` (e.g., `pull_request_build.yml`, `test.yml`, `ci.yml`). If none exist, create `.github/workflows/ci.yml`. Ensure parallel jobs are used for efficiency.
6. **Configure Parallel Jobs**:
   - **`test` job**: Handles dependency installation, `mops test`, and `mops bench`.
   - **`fmt` job**: Handles `prettier` formatting checks.
7. **Add Build Steps**: If examples or canisters are present, add steps to build them using the selected canister build tool.

## Implementation

### Step 1 — Inspect `mops.toml` (do NOT modify `[toolchain]`)

Before adding the CI, **read** `mops.toml` but do **NOT** modify the `[toolchain]` section. Use whatever is already configured.

**CRITICAL — Never auto-add `pocket-ic`:** Do NOT add `pocket-ic` (or change its version) automatically. Treat `[toolchain]` like the `files` field — leave it to the user to add manually if they want it. Note that `mops test` does **not** use `pocket-ic` (it runs via the Motoko interpreter or WASI), so `pocket-ic` is only relevant for `mops bench`.

Determine which CI strategy to use based on the current `[toolchain]` content:

- **`pocket-ic` is present in `[toolchain]`**: `mops bench` will use it automatically. No extra runtime is needed in CI — just install it via `mops toolchain bin pocket-ic` (see Step 2 template).
- **`pocket-ic` is absent AND the package has benchmarks**: `mops bench` will not have a runtime. In practice `mops bench` with `pocket-ic` has proven unreliable on several repositories, so the recommended fallback is to install `dfx` and start a local replica in the CI job that runs `mops bench` (see Pitfall #2 for the exact snippet). Do **NOT** add `pocket-ic` to `mops.toml` to "fix" this.
- **`pocket-ic` is absent AND there are no benchmarks**: Nothing to do — `mops test` does not need `pocket-ic`.

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

      # Bench runtime — pick ONE of the following based on `mops.toml` `[toolchain]`:
      # (A) If `pocket-ic` IS present in [toolchain] AND the package has benchmarks:
      - name: Make sure pocket-ic is installed
        run: mops toolchain init && mops toolchain bin pocket-ic
      # (B) If `pocket-ic` is NOT in [toolchain] AND the package has benchmarks,
      # install dfx instead (see Pitfall #2). Omit both (A) and (B) if there are no benchmarks.
      # - name: Install dfx
      #   uses: dfinity/setup-dfx@main
      # - name: Start dfx
      #   run: dfx start --background --clean

      - name: Run tests
        run: mops test  # Omit this step if the package has no tests. Note: `mops test` does not use pocket-ic.

      - name: Run benchmarks
        run: mops bench  # Omit this step if the package has no benchmarks

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
2. **Missing `pocket-ic` in `[toolchain]` — install `dfx` in CI, do NOT edit `mops.toml`.** Never add `pocket-ic` to `mops.toml` automatically (see Step 1). If the package has **benchmarks** AND `pocket-ic` is **not** in `mops.toml`'s `[toolchain]` section, `mops bench` will fail in CI unless `dfx` is installed. In practice, `mops bench` with `pocket-ic` has proven unreliable on several repositories, so `dfx` is often the only working option. In that case, add a `dfx` installation step to the CI workflow **before** the `mops bench` step (only in the job that runs `mops bench`, and only when both conditions above are met):

    ```yaml
          - name: Install dfx
            uses: dfinity/setup-dfx@main
          - name: Start dfx
            run: dfx start --background --clean
          - run: mops bench
    ```

    Note: `mops test` does not use `pocket-ic` (it runs via the Motoko interpreter or WASI), so the absence of `pocket-ic` only affects `mops bench`.
3. **Slow `dfx` installation.** Avoid installing `dfx` unless absolutely necessary (e.g., for E2E tests or Candid comparisons). Use `dfinity/setup-dfx@main` instead of shell scripts.
4. **Not showing versions.** Always include a step to show `mops` and `moc` versions. This helps in debugging CI issues.
5. **Not using parallel jobs.** Running formatting and tests in the same job is slower. Use separate jobs so GitHub runs them in parallel.
6. **Including `mops test` when no tests exist.** Always check if the package has a `test/` directory with `*.test.mo` files. If not, omit the `mops test` step to avoid CI failures.
7. **Including `mops bench` when no benchmarks exist.** Always check if the package has a `bench/` directory with `*.bench.mo` files. If not, omit the `mops bench` step.
8. **Old mops installation.** Avoid `npm i -g ic-mops` or `ZenVoich/setup-mops@v1`. Use `caffeinelabs/setup-mops@v1` instead.
9. **Simultaneous tool installation.** Do not install both `dfx` and `icp-cli`. Stick to one.
10. **Skipping existing tests.** If you are updating an existing CI, ensure it runs all tests and benchmarks found in the repo.
11. **Unnecessary tool installation.** Do not install `dfx` or `icp-cli` for library packages that only use `mops test` and `mops bench`.

## Verify It Works

1. **Check GitHub Actions tab.** Ensure both the `test` and `fmt` jobs are present and running.
2. **Review logs.** Verify that `mops install` correctly pulls dependencies and `mops bench` uses `pocket-ic`.
3. **Check formatting.** Intentionally misformat a file and open a PR to ensure the `fmt` job fails as expected.
