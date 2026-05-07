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
4. **Optimize `mops.toml`**: If `pocket-ic` is missing from the `[toolchain]` section, add `pocket-ic = "9.0.3"`. If it is already present, do NOT change the version. This ensures a fast CI run while respecting project-specific versioning.
5. **Create or Update Workflow File**: Identify existing workflow files in `.github/workflows/` (e.g., `pull_request_build.yml`, `test.yml`, `ci.yml`). If none exist, create `.github/workflows/ci.yml`. Ensure parallel jobs are used for efficiency.
6. **Configure Parallel Jobs**:
   - **`test` job**: Handles dependency installation, `mops test`, and `mops bench`.
   - **`fmt` job**: Handles `prettier` formatting checks.
7. **Add Build Steps**: If examples or canisters are present, add steps to build them using the selected canister build tool.

## Implementation

### Step 1 — Optimize `mops.toml`

Before adding the CI, ensure `mops.toml` is configured for a fast CI run. Check the `[toolchain]` section. If `pocket-ic` is missing, add it with version `9.0.3`. If it is already present, keep the existing version:

```toml
[toolchain]
pocket-ic = "9.0.3"
```

*Rationale: This allows `mops bench` to run without installing the full `dfx` SDK, significantly speeding up the workflow. Version 9.0.3 is our recommended stable version when introducing the tool; however, if the project already specifies a version, we respect that to avoid breaking changes.*

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
        run: mops toolchain bin moc || (mops toolchain use moc latest && mops toolchain bin moc)

      - name: Show versions
        run: |
          mops --version
          $(mops toolchain bin moc) --version

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
        uses: actions/setup-node@v6
        with:
          node-version: latest

      - name: Prettier Check
        run: |
          npm install prettier prettier-plugin-motoko --no-save
          npx -y prettier --plugin prettier-plugin-motoko --check '**/*.{mo,json,md}'
```

### Step 3 — Handle Examples and Canisters (Optional)

#### If the repo has examples that are canisters:
Add a step to the `test` job (or a new job) to build examples. Use the tool already present in the project or `icp-cli` for new ones. Omit this section if no examples/canisters need building.

```yaml
      - name: Build examples
        run: |
          # Detect and build examples
          if [ -d "examples" ]; then
            cd examples
            if [ -f "mops.toml" ]; then
              mops install
            fi
            
            # Use icp if it exists, otherwise fallback to dfx if dfx.json is present
            if command -v icp >/dev/null; then
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
          if command -v icp >/dev/null; then
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
2. **Missing `pocket-ic` version in toolchain.** If `pocket-ic` is not in `mops.toml`, `mops bench` will attempt to use `dfx`, which might not be installed, causing the CI to fail or be very slow. If missing, always add version `9.0.3`.
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
