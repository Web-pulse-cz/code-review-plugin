---
description: Review the current diff (or given target) and report severity-tagged findings with fixes.
argument-hint: "[target: staged | HEAD~1 | <branch> | <file...>] [--fix]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*), Bash(git merge-base:*), Edit, Agent
---

# Code review

Target: `$ARGUMENTS` (empty → uncommitted working-tree changes).

## Steps

1. **Resolve the diff.**
   - No args → `git status --short` then `git diff` (plus `git diff --staged`).
   - `staged` → `git diff --staged`.
   - A branch name → `git diff $(git merge-base HEAD <branch>)...HEAD`.
   - A commit-ish (`HEAD~1`, sha) → `git diff <commit-ish>`.
   - File paths → `git diff -- <paths>`; if the files are untracked, read them in full.
   - Empty diff → say so and stop. Do not invent a target.

2. **Read enough context.** For each changed hunk, read the surrounding function and its callers. A finding based only on the diff text is a guess — verify before reporting.

3. **Review.** Delegate to the `code-reviewer` agent for anything over ~3 changed files, then relay its findings. Review dimensions:
   - **Correctness** — logic errors, off-by-one, wrong operator, null/undefined, unhandled error paths, race conditions.
   - **Security** — injection, missing authz check, secrets in code, unsafe deserialization, path traversal.
   - **Contract breaks** — changed signature or return shape with un-updated callers, removed public API.
   - **Reuse** — reimplements something that already exists in the repo.
   - **Tests** — new behavior with no test, or a test that cannot fail.

4. **Report.** One line per finding:

   ```
   path/to/file.ts:42  [high] Token expiry uses `<` so a token expiring this second is accepted. Use `<=`.
   ```

   Severities: `critical` (data loss, auth bypass, prod breakage) · `high` (wrong behavior on a realistic input) · `medium` (edge case, missing test) · `low` (clarity, naming).

   End with a one-line verdict. If nothing is wrong, say that plainly — do not pad the list.

5. **`--fix`** — after reporting, apply only the `critical` and `high` findings, then show the diff. Leave the rest for the user to decide.

## Rules

- Report problems, not preferences. Skip formatting nits a linter would catch.
- No praise, no summary of what the code does.
- Never claim a bug without naming the input or sequence that triggers it.
