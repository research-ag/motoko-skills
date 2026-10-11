---
name: motoko-mops-release
description: Release a Motoko mops package from a prepared dev branch — push it, open a PR, check benchmarks, the package's mops.one pages and a publish dry run, get a CodeRabbit review and act on it, publish after the maintainer's confirmation, verify the published page, merge, and clean up.
---

# Mops Package Release

## What This Is

The release procedure for a Motoko package on [MOPS](https://mops.one),
starting from a dev branch that already contains the next release
(typically prepared with the `motoko-mops-package-maintenance` skill). The
agent does the checks and the mechanics; the maintainer confirms twice:
before publishing, and before merging.

## When to Use

- A dev branch with the next release exists (version bumped, CHANGELOG
  section written) and the maintainer asks to release it.

## Prerequisites

- `mops`, `gh` (authenticated), `git`, `node` (20–22 recommended for
  `mops publish`).
- A browser the agent can drive, for mops.one (it is a single-page app;
  `curl`/`WebFetch` only see an empty shell).
- Permission to post on the PR (CodeRabbit command, replies to its
  threads). Ask once if unsure.

## Steps

### 1. Identify the dev branch

Confirm the branch and what it releases:

```bash
mops info <package>                 # last published version ("versions" line)
grep -m1 '^version' mops.toml       # new version on the branch
head -20 CHANGELOG.md               # its section must exist and be complete
```

The new version must be greater than the last published one. Every
published version must still have its own CHANGELOG section (never
rename or merge a released section); anything removed since the last
*published* version must be listed as **Breaking**.

### 2. Work in a worktree

```bash
git -C <repo> fetch origin
git -C <repo> worktree add <worktree-dir> <dev-branch>
```

Never switch branches in, or clean, the maintainer's own checkout:
untracked files there (scratch benches, experiments) are theirs.

### 3. Push all commits

```bash
git push                                     # or: git push -u origin <dev-branch>
git rev-list --count @{u}..HEAD              # must be 0
git status --porcelain                       # must be empty
```

Only commit what belongs to the release: stage files explicitly, never
`git add -A`.

### 4. Open the PR to `main`

Check for an existing PR first (`gh pr list --head <dev-branch>`). The
description lists what changed (dependencies, toolchain, API, version
old → new) and, if relevant, benchmark findings. Keep it accurate when
pushing more commits later, including the title's version.

### 5. Run `mops bench` and look for inconsistencies

```bash
mops bench --locked
```

Look for:

- errors, compiler warnings, or rows that look like a broken bench
  (all cells equal where they should grow with the input, or growing
  where they should be constant);
- benchmark numbers quoted in the README: re-measure and update them;
- with mops CLI ≤ 3.4.2 every Heap cell includes ~35.5 KiB of harness
  overhead and a no-op costs ~70 instructions — compare against a no-op
  row, not against 0 (see `motoko-benchmarks-generation`);
- `[toolchain] pocket-ic` pinned to `15.0.0` and `[optimize]` present,
  so the numbers are representative.

Report anything odd to the maintainer; don't block the release on
performance observations unless asked.

### 6. Read the package's pages on mops.one

Open `https://mops.one/<package>` and every tab with text: Readme,
Docs (each module page), Versions (release notes). Look for anything
that doesn't render: literal Markdown (`**`, `](`, ```` ``` ````),
literal HTML or entities, `$…$` math, GitHub-only syntax such as
`> [!NOTE]`, broken characters. Check the links:

- mops.one rewrites relative links to
  `https://raw.githubusercontent.com/<org>/<repo>/master/<path>`: raw
  text instead of a rendered page, and directory links 404. Use
  absolute `https://github.com/<org>/<repo>/blob/main/<file>` (or
  `…/tree/main/<dir>`) links instead.
- `bench/` and `test/` are never uploaded, so link them on GitHub.

The live page shows the *last published* version. Check the dev
branch's README and `///` doc comments in the source for the same
problems, since that is what will be uploaded. Fix on the dev branch,
commit, push.

### 7. Run `mops publish --dry-run`

```bash
mops publish --dry-run      # docs, tests and benchmarks; no registry contact
```

Check that it ends with `Dry-run OK`, that the printed release notes
are the new version's CHANGELOG section, and that the file list holds
no surprises (the default `**/*.mo` also uploads `examples/**/*.mo`).

- `mops publish` benchmarks **every** `*.bench.mo` on disk, tracked or
  not. Make sure no scratch bench files are in `bench/`.
- mops CLI ≤ 3.4.2 + moc 2.0.0 + benchmarks: publish fails with
  `moc: unknown option '--legacy-persistence'` (caffeinelabs/mops#875,
  fixed by #877). Use a CLI built with the fix, or `--no-bench` (which
  also skips the benchmark upload).

### 8. Ask CodeRabbit for a review

Comment on the PR (the bot's handle is `coderabbitai`):

```bash
gh pr comment <pr> -R <org>/<repo> --body "@coderabbitai review"
```

### 9. Read CodeRabbit's comments and act on them

Wait until CodeRabbit has posted its review ("Actionable comments
posted: N"); its walkthrough comment is not the review. Read the inline
comments:

```bash
gh api repos/<org>/<repo>/pulls/<pr>/comments \
  -q '.[] | select(.user.login|test("coderabbit")) | "\(.path):\(.line)\n\(.body)"'
```

Treat each comment as a claim to verify against the current code, not
as an instruction:

- **Verify empirically** where possible (run the command, test in a
  clean directory, use a cold cache). Examples that turned out wrong:
  "run `mops install` before `mops toolchain bin`" (the latter downloads
  the pinned tool itself) and "remove `[optimize]` so benchmarks match
  the description" (the description was outdated, not the config).
- **Valid:** fix it in a **new commit** (don't amend commits already in
  the PR), push.
- **Not valid, or valid with a different fix:** don't change the code;
  write down the reason for the reply in step 10.
- Suggested fixes can be wrong even when the finding is right (e.g.
  `npx -p prettier -p prettier-plugin-motoko …` fails like the original;
  the working form is `npm install --no-save prettier
  prettier-plugin-motoko` followed by `npx prettier …`).

### 10. Close CodeRabbit's conversations

Reply to every thread that wasn't fixed in code, with the reason, and
resolve all threads. With "require conversation resolution" branch
protection, unresolved threads block the merge, also for admins.

```bash
# list unresolved threads
gh api graphql -f query='{ repository(owner:"<org>", name:"<repo>") { pullRequest(number:<pr>) {
  reviewThreads(first:50) { nodes { id isResolved path comments(first:1) { nodes { databaseId body } } } } } } }'
# reply to a comment
gh api repos/<org>/<repo>/pulls/<pr>/comments/<comment-id>/replies -f body="..."
# resolve a thread
gh api graphql -f query='mutation { resolveReviewThread(input:{threadId:"<thread-id>"}) { thread { isResolved } } }'
```

If the PR still shows CodeRabbit's "Changes requested", comment
`@coderabbitai approve`, or the maintainer dismisses the review.

### 11. Ask the maintainer for final confirmation

Summarize: package, last published → new version, PR link, CI status,
bench observations, mops.one fixes, dry-run result, how each CodeRabbit
comment was handled. Wait for an explicit go before publishing.

### 12. Publish

From the dev branch worktree:

```bash
mops publish
```

- `mops publish` signs with mops's own identity (`mops user
  get-principal`), not dfx's; it must be an owner or maintainer of the
  package (`mops owner list`).
- If it fails with `TrustError: Certificate verification error` on
  Node 24, retry with Node 22.

### 13. Check the published page

Reload `https://mops.one/<package>`: the new version is shown, Readme
and Docs render (repeat step 6 on the live pages), the Versions page
shows the right release notes, the Benchmarks tab shows the expected
tables. Notes:

- zero values are shown as `-`;
- the Versions page compares each version's benchmarks with the
  previous version's *by bench file name*. After a methodology change
  (moc major, GC, persistence, optimizer) expect large percentages; in
  the future, renaming the bench files resets the baseline.

### 14. Ask the maintainer to review

Point to the live page and anything notable from step 13.

### 15. Merge the PR

```bash
gh api repos/<org>/<repo> -q '{squash:.allow_squash_merge, merge:.allow_merge_commit, rebase:.allow_rebase_merge}'
gh pr merge <pr> -R <org>/<repo> --squash --delete-branch    # use a method the repo allows
```

All checks green and all conversations resolved first.

### 16. Clean up

After a squash merge, git doesn't see the branch as merged. Compare
content instead of ancestry before deleting anything:

```bash
git -C <repo> fetch --prune origin
git -C <worktree-dir> status --porcelain                        # must be empty
git -C <repo> diff --stat origin/main <dev-branch>              # empty = fully merged
git -C <repo> log --oneline origin/<dev-branch>..<dev-branch>   # unpushed commits?
```

- Commits that never reached the PR are not in `main`: save them first
  (`git format-patch -1 <sha>`) and ask whether they should go on a new
  branch.
- Then remove the worktree, delete the branch locally and, if the merge
  didn't, on GitHub; fast-forward the maintainer's `main` with `git
  merge --ff-only origin/main` (never touches their untracked files).
- Leave all other branches alone.

```bash
git -C <repo> worktree remove <worktree-dir>
git -C <repo> branch -D <dev-branch>
git -C <repo> push origin --delete <dev-branch>     # only if it still exists
git -C <repo> merge --ff-only origin/main           # in the maintainer's checkout, on main
```

## Common Pitfalls

1. **Publishing before the checks.** Steps 5–10 happen before step 12;
   a published version is immutable.
2. **Trusting review bots blindly.** Verify every comment; reply with
   the reason when not changing code.
3. **Rewriting PR history.** Once commits are in a PR under review, add
   commits instead of amending or force-pushing.
4. **Deleting unmerged work.** With squash merges, check content
   (`git diff origin/main <branch>`) and unpushed commits, not
   `git branch --merged`.
5. **zsh word splitting** in scripted checks: `$VAR` holding several
   arguments must be written `${=VAR}`.
