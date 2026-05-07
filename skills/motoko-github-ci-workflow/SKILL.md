---
name: motoko-github-ci-workflow
description: "Add a GitHub CI workflow for PRs to a Motoko repository, including mops tests, benchmarks, formatting checks, and building canisters or examples."
compatibility: "mops >= 0.45.0, node >= 20"
metadata:
   title: "Motoko GitHub CI Workflow"
   category: DevOps
   internal: false
---

# Motoko GitHub CI Workflow

## What This Is

This skill provides a comprehensive playbook for adding or updating a GitHub Actions CI workflow for Motoko projects. It covers automated testing, benchmarking, code formatting, and building (canisters or examples) using `mops`, `prettier`, `icp-cli`, and `dfx`. It is a perfect companion to the `mops-package-maintenance` skill.

For general information on GitHub Actions, refer to the [GitHub Actions Documentation](https://docs.github.com/en/actions). For Motoko-specific examples, you can explore public repositories in the [research-ag](https://github.com/research-ag) organization.

## Prerequisites

- `mops` CLI (local: `npm i -g ic-mops`, CI: `caffeinelabs/setup-mops@v1`)
- `node` (latest LTS recommended)
- `pocket-ic` version `9.0.3` specified in `mops.toml` `[toolchain]` to avoid `dfx` dependency for benchmarks.

## How It Works

1. **Analyze the Repository**:
   - Determine if it's a Motoko package (has `[package]` in `mops.toml`) or a canister project (has `[canister]` in `mops.toml` or `dfx.json`).
   - **Check for tests**: Look for a `test/` directory and `*.test.mo` files. If none are found, do NOT include the `mops test` step.
   - **Check for benchmarks**: Look for a `bench/` directory and `*.bench.mo` files. If none are found, do NOT include the `mops bench` step.
2. **Optimize `mops.toml`**: Ensure `pocket-ic = "9.0.3"` is in the `[toolchain]` section.
3. **Create Workflow File**: Add `.github/workflows/ci.yml` with parallel jobs for efficiency.
4. **Configure Parallel Jobs**:
   - **`test` job**: Handles dependency installation, `mops test`, and `mops bench`.
   - **`fmt` job**: Handles `prettier` formatting checks.
5. **Add Build Steps**: If examples or canisters are present, add steps to build them using `icp-cli` (preferred) or `dfx`.

## Implementation

### Step 1 — Optimize `mops.toml`

Before adding the CI, ensure `mops.toml` is configured for a fast CI run. Check the `[toolchain]` section. If `pocket-ic` is missing or not version `9.0.3`, update it:

```toml
[toolchain]
pocket-ic = "9.0.3"
```

*Rationale: This allows `mops bench` to run without installing the full `dfx` SDK, significantly speeding up the workflow.*

### Step 2 — Create the GitHub CI Workflow

Create or update `.github/workflows/ci.yml`. Use the latest versions of actions (e.g., `actions/checkout@v6`, `actions/setup-node@v4`) and parallelize the tasks.

*Note: Using the latest major versions of actions (nested actions) ensures you get the latest features and security updates.*

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
        uses: actions/setup-node@v4
        with:
          node-version: latest

      - name: Install mops
        uses: caffeinelabs/setup-mops@v1

      - name: Install dependencies
        run: mops install

      - name: Run tests
        run: mops test  # Omit this step if the package has no tests

      - name: Run benchmarks
        run: mops bench --replica pocket-ic  # Omit this step if the package has no benchmarks

  fmt:
    name: Formatting Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: latest

      - name: Prettier Check
        run: |
          npm install prettier prettier-plugin-motoko --no-save
          npx -y prettier --plugin prettier-plugin-motoko --check '**/*.{mo,json,md}'
```

### Step 3 — Handle Examples and Canisters

#### If the repo has examples:
Add a step to the `test` job (or a new job) to build examples. Prefer `icp-cli` if available, as it is generally faster and lighter than `dfx`.

```yaml
      - name: Install icp-cli
        run: npm i -g icp-cli

      - name: Build examples
        run: |
          # Detect and build examples
          if [ -d "examples" ]; then
            cd examples
            if [ -f "mops.toml" ]; then
              mops install
            fi
            # Preferably build with icp
            if command -v icp >/dev/null; then
              icp build --all
            elif [ -f "dfx.json" ]; then
              dfx build
            fi
            cd ..
          fi
```

#### If the repo is a Canister (not a package):
If the project is a canister, you should build the canisters and optionally compare `.did` files to ensure the interface hasn't changed unexpectedly.

**Install tools:**
```yaml
      - name: Install build tools
        run: |
          npm i -g icp-cli
          # Only install dfx if needed for .did comparison or E2E tests
          sh -ci "$(curl -fsSL https://internetcomputer.org/install.sh)"
```

**Build and Compare DID:**
Comparing `.did` files often requires `dfx` to generate the latest interface from the source code.

```yaml
      - name: Check Candid interface
        run: |
          # 1. Build canisters (e.g. with icp or dfx)
          icp build
          
          # 2. Compare generated .did with tracked .did
          # This often involves comparing files in the 'did/' folder 
          # with artifacts in '.dfx/local/canisters/<canister_name>/<canister_name>.did'
          # diff -u did/my_canister.did .dfx/local/canisters/my_canister/my_canister.did
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

1. **Outdated Node version.** Prettier and some Motoko tools require recent Node.js versions. Always use `latest` or at least `v20` in CI.
2. **Missing `pocket-ic` in toolchain.** If `pocket-ic` is not in `mops.toml`, `mops bench` will attempt to use `dfx`, which might not be installed, causing the CI to fail or be very slow.
3. **Slow `dfx` installation.** Avoid installing `dfx` unless absolutely necessary (e.g., for E2E tests or Candid comparisons that `icp-cli` cannot handle yet).
4. **Nested Workflow Files.** Prefer a single `ci.yml` with multiple jobs, or separate focused files like `prettier.yml` and `test.yml`.
5. **Not using parallel jobs.** Running formatting and tests in the same job is slower. Use separate jobs so GitHub runs them in parallel.
6. **Including `mops test` when no tests exist.** Always check if the package has a `test/` directory with `*.test.mo` files. If not, omit the `mops test` step to avoid CI failures (e.g., "No test files found").
7. **Including `mops bench` when no benchmarks exist.** Always check if the package has a `bench/` directory with `*.bench.mo` files. If not, omit the `mops bench` step to avoid CI failures (e.g., "No *.bench.mo files found").

## Verify It Works

1. **Check GitHub Actions tab.** Ensure both the `test` and `fmt` jobs are present and running.
2. **Review logs.** Verify that `mops install` correctly pulls dependencies and `mops bench` uses `pocket-ic`.
3. **Check formatting.** Intentionally misformat a file and open a PR to ensure the `fmt` job fails as expected.
