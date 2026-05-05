---
name: motoko-mops-package-maintenance
description: Automate maintaining a Motoko mops package — upgrade dependencies, fix compiler warnings, run tests and benchmarks, fix breakages, update CHANGELOG, review doc strings and docs, run the formatter, and prepare a patch release on a new git branch.
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
- `moc` (Motoko compiler; usually ships with `dfx`, but can be standalone)
- `dfx` (DFINITY SDK; optional if `moc` and `mops` are otherwise available)
- `node` / `npm`
- `git`
- `prettier` and `prettier-plugin-motoko` (run via `npx -y`)

## Workflow

Work through every step in order. Do NOT skip steps. If a step fails,
fix the issue before moving on.

### Step 0 — Create a maintenance branch

```bash
git checkout -b chore/dependency-bump-$(date +%Y-%m-%d)
```

All subsequent changes are committed on this branch.

### Step 1 — Verify Package Quality on MOPS

1. Open `mops.toml` and check if it has a `[package]` section. If it only has `[canister]` sections or neither, this is not a published package; skip to Step 2.
2. If it is a package, find the `name` field under `[package]`.
3. Go to `https://mops.one/<name>` (replace `<name>` with the package name).
4. Locate the **Package Quality** section on the page.
5. Review the status of each quality metric (e.g., Documentation, License, Repository, Tests, Benchmarks).
6. If any metric is not **"yes"** or **"100%"**, attempt to fix it:
    - **Documentation < 100%**: You MUST run the `motoko-doc-strings` skill to improve doc-string coverage.
    - **Missing License/Repository**: Ensure these fields are correctly set in `mops.toml` `[package]` section.
    - **Tests/Benchmarks failing**: These will be addressed in Step 4 and Step 5.
7. If a quality issue cannot be fixed automatically, raise a warning to the user.

### Step 2 — Discover outdated dependencies

Open `mops.toml` and list every dependency under `[dependencies]` and
`[dev-dependencies]`.

**Important:** `moc` version in `[toolchain]` should usually be upgraded to the latest version to ensure a modern build environment. However, `moc` in `[requirements]` MUST remain stable and should not be automatically listed for upgrade unless a specific breaking change in a dependency requires it.

For each dependency in `[dependencies]` and `[dev-dependencies]`, check the latest available version:

```bash
mops search <package-name>
```

Compare the installed version with the latest version. Record which
packages need upgrading.

### Step 3 — Upgrade dependencies in `mops.toml`

For each outdated dependency in `[dependencies]` and `[dev-dependencies]`, update the version string in `mops.toml`
to the latest version.

**Upgrading `moc`:**
- `[toolchain] moc`: Upgrade this to the latest version.
- `[requirements] moc`: Do NOT upgrade this unless:
  1. It is absolutely necessary to support an upgraded dependency.
  2. You have verified the `moc` changelog for breaking changes and are prepared to fix any resulting issues.

Then install:

```bash
mops install
```

Verify the lock file updated cleanly and there are no resolution errors.

#### Step 3a — Sync nested `mops.toml` files (examples, sub-projects)

Search the repository for any additional `mops.toml` files outside the
root (commonly under `examples/`, `bench/`, or sub-canister folders):

```bash
find . -name mops.toml -not -path "./node_modules/*" -not -path "./.mops/*"
```

For **every** nested `mops.toml`, ensure:

- Third-party `[dependencies]` versions **usually** match the root. Examples may occasionally have extra dependencies, but common ones should be in sync.
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

#### Step 3b — Audit repo hygiene files

- `.gitignore`: Ignore `node_modules/`, build artifacts, IDE directories (e.g. `.idea/`, `.vscode/`),
  agent-specific directories (e.g. `.agents/`, `.junie/`, `.claude/`, `.copilot/`),
  and `skills-lock.json`.
- `package-lock.json`: If it does NOT exist, do NOT introduce it.
- `package.json`: If it exists, ensure the `license` field matches `mops.toml` `[package] license`. If it does NOT exist, do NOT introduce it (it may be created by `npm`, if so, do NOT commit it).

### Step 3c — Fix compiler warnings

Run the `fix-compiler-warnings` skill as a sub-task.

**CRITICAL RULE:** Do NOT modify any code unless an explicit warning or error was produced by `moc --check`. Never apply "improvements" or "fixes" for perceived issues that the compiler does not actually complain about.

1. Run `find src -type f -name "*.mo" -print0 | xargs -0 -n1 $(mops toolchain bin moc) --check $(mops sources)`. This runs the check on each file individually.
2. If warnings/errors are found:
    - Fix one type of error at a time.
    - Re-run the check to verify the fix.
    - Run `mops test` or `mops bench` if relevant to ensure no regressions.
    - Repeat until all warnings are resolved.

### Step 4 — Run tests

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

### Step 5 — Run benchmarks

```bash
mops bench
```

If benchmarks fail to compile or run:

1. Apply the same kind of fixes as in Step 4.
2. Re-run until benchmarks complete.

If benchmarks show a significant regression, note it for the CHANGELOG
but do not block the release unless the user explicitly asks.

### Step 6 — Update the CHANGELOG

Open (or create) `CHANGELOG.md`. Add a new section at the top for the
upcoming version (you will finalize the version number in Step 10).

Required bullet points:

- **Dependencies bumped** — list every dependency that was upgraded and
  its old → new version. (Note: Do NOT include `[toolchain] moc` bumps here; users only care about `[requirements]`).
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

### Step 7 — Review and improve doc strings

Scan every `.mo` file under `src/` for public declarations. For each
public `type`, `func`, `actor`, `actor class`, `let`, and `module`:
1. Check that a `///` doc string exists directly above the declaration.
2. Read the doc string from a first-time user's perspective:
    - Is the purpose clear?
    - Are argument types/units/constraints documented?
    - Is trap / error behavior described?
    - Are examples present for non-trivial functions?
3. Improve or add doc strings where they are missing or unclear.

If a `motoko-doc-strings` skill is installed, follow its full checklist.

### Step 8 — Review README and other Markdown files

Read `README.md` and every other `.md` file in the repository. For each:

1. Check factual accuracy (version numbers, API names, examples).
2. Fix outdated information.
3. Improve clarity, grammar, and completeness where possible.
4. Add any new sections that would help users (e.g., new API surface
   from upgraded deps).

### Step 9 — Ensure CI Formatting

Maintenance PRs should be automatically formatted or checked by CI. Your job is to ensure the repository has this capability without introducing unnecessary `package.json` files.

#### 9a — Check for Prettier Configuration

Check if a `.prettierrc` file exists at the repo root.

1. If it does NOT exist, create one with the following content.
2. If it DOES exist but is "simple" (e.g., only contains `tabWidth`), replace it with the following content.

Recommended `.prettierrc`:

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

#### 9b — Verify or Add Prettier Check and Format

Search for GitHub Action workflows (e.g., `.github/workflows/*.yml`).

If no formatting check exists, suggest adding one that automatically formats and commits changes:

```yaml
- name: Prettier Check and Format
  run: |
    npm install prettier prettier-plugin-motoko --no-save
    npx -y prettier --plugin prettier-plugin-motoko --write '**/*.{mo,json,md}'
- name: Commit formatted files
  run: |
    git config --local user.email "action@github.com"
    git config --local user.name "GitHub Action"
    git add -u
    git diff-index --quiet HEAD || git commit -m "chore: format code with prettier"
    git push
```

**CRITICAL:** Do NOT add a "Compiler Check" or `moc --check` step to the CI. While the agent MUST run this check locally during maintenance (Step 3c), it should NOT be part of the automated CI suite.

**Note:** Do not run `prettier --write` as part of this maintenance workflow. Formatting is now handled separately in CI.

### Step 10 — Bump the version

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

### Step 11 — Commit and push

Stage all changes. **CRITICAL:** If `package.json` or `package-lock.json` did not exist at the start of the task, do NOT commit them if they were created during the process (e.g., by `npm install`).

```bash
git add -A
git restore --staged .idea .vscode .agents .junie .claude .copilot skills-lock.json
# Do not commit package.json if it was just created
git status --porcelain | grep "^A  package.json" && git restore --staged package.json || true
git status --porcelain | grep "^A  package-lock.json" && git restore --staged package-lock.json || true
git status
```

Verify `git status` shows only intended staged files, then commit:

```bash
git commit -m "chore: bump dependencies and prepare v<NEW_VERSION>"
```

Inform the user that the branch is ready for review and/or pushing:

```text
Branch: chore/dependency-bump-YYYY-MM-DD
Ready for review. Run `git push -u origin HEAD` to push.
```

## Common Pitfalls

1. **Don't blindly bump major versions.** If a dependency from `mops.one` has a new major version number (e.g., `1.x` to `2.x`), you **MUST** read the CHANGELOG of that package. The API likely changed. If the migration is non-trivial, flag it to the user.

2. **Don't blindly bump `[requirements] moc` version.** While the `[toolchain] moc` version should be kept up to date, the `moc` version in `[requirements]` should NOT be changed to the latest version (e.g., from `1.3.0` to `1.6.0`) unless absolutely necessary. Check the `moc` changelog for breaking changes before considering an upgrade to the requirements.

3. **Lock file drift.** Always run `mops install` after editing
   `mops.toml` so the lock file stays in sync. Never commit a hand-edited
   lock file.

4. **Missing `node_modules` in `.gitignore`.** After running `npm install`,
   make sure `node_modules/` is listed in `.gitignore`. Add it if missing.

5. **CHANGELOG ordering.** Newest version goes at the top. Don't append
   to the bottom.

6. **Don't skip the doc-string review.** Even if no code changed, the AI
   may have improved since the last pass and can now write better docs.
   Always do a full scan.

7. **Adding Compiler Checks to CI.** Do not include `moc --check` in the CI configuration. This check should only be performed by the agent during the maintenance process to fix warnings, as different CI environments might have different compiler versions that could cause unexpected failures for the end user.

## Verify It Works

After completing all steps, run the validation suite:

```bash
mops install
mops test
mops bench
```

All commands must succeed with no errors. (Note: Prettier formatting is verified by CI after the PR is created).
