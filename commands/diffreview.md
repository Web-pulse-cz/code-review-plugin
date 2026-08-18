---
description: Review the current diff (or given target) and report severity-tagged findings with fixes.
argument-hint: "[target: current | staged | HEAD~1 | <branch> | <file...>] [--fix]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git merge-base:*), Bash(git symbolic-ref:*), Edit, Agent, Skill
---

# Code review

Target: `$ARGUMENTS` (empty → uncommitted working-tree changes).

## Steps

1. **Load project instructions.** Already read `agents/**/*` earlier this session? Reuse that, don't re-read. Otherwise: check the repo root for an `agents/` folder. If it exists, walk it fully — subfolders can nest subfolders to any depth, so glob `agents/**/*` rather than listing just the top level — and read every file found. These are the project's own conventions and take priority over the generic dimensions in step 4 — a project rule beats a generic guideline when they conflict.

2. **Resolve the diff.** Use `--unified=1` on every `git diff` below — one line of context is enough to locate a hunk; step 3 reads the real surrounding code anyway, so a wider diff context is wasted tokens.
   - No args → `git status --short` then `git diff -U1` (plus `git diff -U1 --staged`).
   - `current` → the whole current branch, committed and uncommitted alike. Determine the base branch: run `git symbolic-ref -q --short refs/remotes/origin/HEAD` and strip the `origin/` prefix; a plain clone/fetch has no default set (this repo included), so when that returns nothing, check `git show-ref --verify -q refs/heads/main` — use `main` if it exists, else `master`. Then `git diff -U1 $(git merge-base HEAD <base>)` — a single ref against the working tree, no `..`/`...` — which is why it captures both commits since the merge-base and any uncommitted edits; do not use `<base>...HEAD`, which excludes uncommitted changes. Also run `git status --short` for untracked files.
   - `staged` → `git diff -U1 --staged`.
   - A branch name → `git diff -U1 $(git merge-base HEAD <branch>)...HEAD`.
   - A commit-ish (`HEAD~1`, sha) → `git diff -U1 <commit-ish>`.
   - File paths → `git diff -U1 -- <paths>`; if the files are untracked, read them in full.
   - Empty diff → say so and stop. Do not invent a target.

3. **Read enough context.** For each changed hunk, read the surrounding function and its callers — use `Read` with an offset/limit around the hunk, not the whole file. A finding based only on the diff text is a guess — verify before reporting.

4. **Review.** Delegate to the `code-reviewer` agent for anything over ~3 changed files — hand it the diff you already resolved in step 2, not just the raw target string, so it doesn't re-derive it (and potentially pick the wrong dot form for `current`). The agent owns the full dimension list (project rules, correctness, security, contract breaks, reuse, tests, lint) and the stack-specific `review-checklist` skill — relay its findings as-is rather than re-deriving your own.

   Reviewing inline instead (≤3 files, no delegation)? Invoke the `review-checklist` skill for the matching stack first, then check: project-rule violations (step 1) first, then correctness, security, contract breaks, reuse, test gaps, and lint drift against a named `agents/` rule only.

5. **Report.** Findings always in Czech — write every finding (summary, fix) in Czech regardless of the language the user wrote in. One line per finding, starting with the severity emoji, project-rule violations listed first:

   ```
   🔴 path/to/file.ts:42  [critical] Token expiruje s `<`, takže token expirující právě teď je přijat. Použij `<=`.
   ```

   Exactly four severity groups, every finding tagged with one, emoji first on the line:

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
- All finding text is Czech — always, even if `$ARGUMENTS` or the conversation is in another language.
