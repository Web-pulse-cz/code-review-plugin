---
description: Review the current diff (or given target) and report severity-tagged findings with fixes.
argument-hint: "[target: current | staged | HEAD~1 | <branch> | <file...>] [--fix]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git merge-base:*), Bash(git symbolic-ref:*), Edit, Agent
---

# Code review

Target: `$ARGUMENTS` (empty → uncommitted working-tree changes).

## Steps

1. **Load project instructions.** Check the repo root for an `agents/` folder. If it exists, walk it fully — subfolders can nest subfolders to any depth, so glob `agents/**/*` rather than listing just the top level — and read every file found. These are the project's own conventions and take priority over the generic dimensions in step 3 — a project rule beats a generic guideline when they conflict.

2. **Resolve the diff.**
   - No args → `git status --short` then `git diff` (plus `git diff --staged`).
   - `current` → the whole current branch, committed and uncommitted alike. Determine the base branch: run `git symbolic-ref -q --short refs/remotes/origin/HEAD` and strip the `origin/` prefix; a plain clone/fetch has no default set (this repo included), so when that returns nothing, check `git show-ref --verify -q refs/heads/main` — use `main` if it exists, else `master`. Then `git diff $(git merge-base HEAD <base>)` — a single ref against the working tree, no `..`/`...` — which is why it captures both commits since the merge-base and any uncommitted edits; do not use `<base>...HEAD`, which excludes uncommitted changes. Also run `git status --short` for untracked files.
   - `staged` → `git diff --staged`.
   - A branch name → `git diff $(git merge-base HEAD <branch>)...HEAD`.
   - A commit-ish (`HEAD~1`, sha) → `git diff <commit-ish>`.
   - File paths → `git diff -- <paths>`; if the files are untracked, read them in full.
   - Empty diff → say so and stop. Do not invent a target.

3. **Read enough context.** For each changed hunk, read the surrounding function and its callers. A finding based only on the diff text is a guess — verify before reporting.

4. **Review.** Delegate to the `code-reviewer` agent for anything over ~3 changed files — hand it the diff you already resolved in step 2, not just the raw target string, so it doesn't re-derive it (and potentially pick the wrong dot form for `current`). Then relay its findings. Review dimensions, project rules from step 1 first:
   - **Project rules** — anything the change contradicts in the `agents/` instructions, if present.
   - **Correctness** — logic errors, off-by-one, wrong operator, null/undefined, unhandled error paths, race conditions.
   - **Security** — injection, missing authz check, secrets in code, unsafe deserialization, path traversal.
   - **Contract breaks** — changed signature or return shape with un-updated callers, removed public API.
   - **Reuse** — reimplements something that already exists in the repo.
   - **Tests** — new behavior with no test, or a test that cannot fail.
   - **Lint vs. project rules** — style/format the diff breaks that an explicit `agents/` rule calls out. No `agents/` rule on it → not a finding.

5. **Report.** One line per finding, project-rule violations listed first:

   ```
   path/to/file.ts:42  [🔴 critical] Token expiry uses `<` so a token expiring this second is accepted. Use `<=`.
   ```

   Exactly four severity groups, every finding tagged with one:

   - 🔴 **critical** — wrong on a realistic input right now: data loss, auth bypass, prod breakage.
   - 🟡 **medium** — works today, but a real problem: edge case, missing test, perf/tech-debt risk.
   - 🟢 **minor** — no functional risk: clarity, naming — informational only.
   - 🔵 **lint** — style/lint violation against an explicit `agents/` rule. Requires a named rule; no `agents/` folder → this tier never fires.

   End with a one-line verdict. If nothing is wrong, say that plainly — do not pad the list.

6. **`--fix`** — after reporting, apply only the 🔴 `critical` findings, then show the diff. Leave 🟡, 🟢, and 🔵 for the user to decide.

## Rules

- Report problems, not preferences. Skip formatting nits a linter would catch.
- No praise, no summary of what the code does.
- Never claim a bug without naming the input or sequence that triggers it.
