---
description: Review the current diff (or given target) and report severity-tagged findings with fixes.
argument-hint: "[target: current | commits | commits <n> | commit <ish> | <a>..<b> | staged | <branch> | <file...>] [--fix]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git show:*), Bash(git show-ref:*), Bash(git merge-base:*), Bash(git symbolic-ref:*), Edit, Agent, Skill
---

# Code review

Target: `$ARGUMENTS` (empty → uncommitted working-tree changes).

## Steps

1. **Load project instructions.** Already read `agents/**/*` earlier this session? Reuse that, don't re-read. Otherwise: check the repo root for an `agents/` folder. If it exists, walk it fully — subfolders can nest subfolders to any depth, so glob `agents/**/*` rather than listing just the top level — and read every file found. These are the project's own conventions and take priority over the generic dimensions in step 5 — a project rule beats a generic guideline when they conflict.

2. **Resolve the diff.** Use `--unified=1` on every `git diff` below — one line of context is enough to locate a hunk; step 4 reads the real surrounding code anyway, so a wider diff context is wasted tokens.
   - No args → `git status --short` then `git diff -U1` (plus `git diff -U1 --staged`).
   - `current` → the whole current branch, committed and uncommitted alike. Determine the base branch: run `git symbolic-ref -q --short refs/remotes/origin/HEAD` and strip the `origin/` prefix; a plain clone/fetch has no default set (this repo included), so when that returns nothing, check `git show-ref --verify -q refs/heads/main` — use `main` if it exists, else `master`. Then `git diff -U1 $(git merge-base HEAD <base>)` — a single ref against the working tree, no `..`/`...` — which is why it captures both commits since the merge-base and any uncommitted edits; do not use `<base>...HEAD`, which excludes uncommitted changes. Also run `git status --short` for untracked files. Caveat worth telling the user about: this form is a net tree diff, so anything merged into the branch from a *third* branch (a colleague's feature branch) shows up as part of the review — merging `main`/`master` in is harmless (the merge-base moves forward with it), a foreign feature branch is not. When the diff turns out to contain files the user plainly did not touch, say so and offer `commits`.
   - `staged` → `git diff -U1 --staged`.
   - `commits` → **only the commits made on this branch, with everything merged in from elsewhere excluded.** Resolve `<base>` exactly as for `current`, then `git log --first-parent --no-merges -p -U1 <base>..HEAD`. `--first-parent` walks only the branch's own side of every merge, `--no-merges` drops the merge commits themselves — what is left is this branch's own work, even if a colleague's branch or `main` was merged in halfway through. Uncommitted changes are deliberately *not* included; use `current` for those.
   - `commits <n>` (a number, e.g. `commits 3`) → the last `<n>` of this branch's own commits: `git log --first-parent --no-merges -p -U1 -n <n> HEAD`. No base resolution needed.
   - `commit <ish>` (e.g. `commit HEAD~2`, `commit abc1234`) → that single commit and nothing else: `git show -U1 -m --first-parent <ish>`. On a *merge* commit this yields everything the merge brought in from the other branch (the diff against the first parent), which is usually not what the user wants — say so and offer `git show -U1 --cc <ish>`, which shows only the lines the merge resolved differently from both parents.
   - `<a>..<b>` (e.g. `origin/main..HEAD`, `abc1234..def5678`) → that range, again the branch's own commits only: `git log --first-parent --no-merges -p -U1 <a>..<b>`.
   - A branch name → `git diff -U1 $(git merge-base HEAD <branch>)...HEAD`.
   - A bare commit-ish (`HEAD~1`, sha) → `git diff -U1 <commit-ish>` — that commit's tree against the working tree, i.e. everything since it. To review that one commit *alone*, the user must say `commit HEAD~1`; do not silently switch between the two forms.
   - File paths → `git diff -U1 -- <paths>`; if the files are untracked, read them in full.
   - Empty diff → say so and stop. Do not invent a target.

3. **Commit-stream targets: review the final state, not the history.** The `commits` / `commits <n>` / `<a>..<b>` forms produce a *sequence* of per-commit patches, so the same line can appear several times. Before reporting anything from such a stream, check the current file content (step 4 below reads it anyway): if a later commit in the range already fixed it, it is not a finding. Report each issue once, attributed to the commit that left it standing, and prefix the finding with that commit's short sha (see step 6).

4. **Read enough context.** For each changed hunk, read the surrounding function and its callers — use `Read` with an offset/limit around the hunk, not the whole file. A finding based only on the diff text is a guess — verify before reporting.

5. **Review.** Delegate to the `code-reviewer` agent for anything over ~3 changed files — hand it the diff you already resolved in step 2, not just the raw target string, so it doesn't re-derive it (and potentially pick the wrong dot form for `current`). The agent owns the full dimension list (project rules, correctness, security, contract breaks, reuse, tests, lint) and the stack-specific `review-checklist` skill — relay its findings as-is rather than re-deriving your own.

   Reviewing inline instead (≤3 files, no delegation)? Invoke the `review-checklist` skill for the matching stack first, then check: project-rule violations (step 1) first, then correctness, security, contract breaks, reuse, test gaps, and lint drift against a named `agents/` rule only.

6. **Report.** Findings always in Czech — write every finding (summary, fix) in Czech regardless of the language the user wrote in. One line per finding, starting with the severity emoji, project-rule violations listed first:

   ```
   🔴 path/to/file.ts:42  [critical] Token expiruje s `<`, takže token expirující právě teď je přijat. Použij `<=`.
   ```

   Reviewing a commit stream (`commits`, `commits <n>`, `<a>..<b>`) → put the short sha right after the emoji so the user knows which commit to fix:

   ```
   🔴 a1b2c3d path/to/file.ts:42  [critical] ...
   ```

   Exactly four severity groups, every finding tagged with one, emoji first on the line:

   - 🔴 **critical** — wrong on a realistic input right now: data loss, auth bypass, prod breakage.
   - 🟡 **medium** — works today, but a real problem: edge case, missing test, perf/tech-debt risk.
   - 🟢 **minor** — no functional risk: clarity, naming — informational only.
   - 🔵 **lint** — style/lint violation against an explicit `agents/` rule. Requires a named rule; no `agents/` folder → this tier never fires.

   End with a one-line verdict. If nothing is wrong, say that plainly — do not pad the list.

7. **`--fix`** — after reporting, apply only the 🔴 `critical` findings, then show the diff. Leave 🟡, 🟢, and 🔵 for the user to decide.

## Rules

- Report problems, not preferences. Skip formatting nits a linter would catch.
- No praise, no summary of what the code does.
- Never claim a bug without naming the input or sequence that triggers it.
- All finding text is Czech — always, even if `$ARGUMENTS` or the conversation is in another language.
