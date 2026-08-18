# code-review-plugin

Structured code review for Claude Code. Reviews the current diff, a branch, or specific files and reports severity-tagged findings with concrete fixes.

## Contents

| Piece | Type | Purpose |
| --- | --- | --- |
| `/review` | command | Entry point. Loads project `agents/` rules if present, resolves the diff, reviews it, reports findings. `--fix` applies 🔴 critical ones. |
| `code-reviewer` | agent | Read-only reviewer. Auto-invoked for larger diffs, or callable directly. |
| `review-checklist` | skill | Stack-specific failure modes (PHP, Symfony, Laravel, TS/React, SQL, shell). |

### Severity

Every finding gets exactly one tag:

| Tag | Meaning |
| --- | --- |
| 🔴 critical | Wrong on a realistic input right now — data loss, auth bypass, prod breakage. |
| 🟡 medium | Works today, but a real problem — edge case, missing test, perf/tech-debt risk. |
| 🟢 minor | No functional risk — clarity, naming, informational only. |
| 🔵 lint | Style/lint violation against an explicit rule in the project's own `agents/` instructions. Only fires when that folder actually states the rule. |

### Project rules (`agents/`)

If the target repo has an `agents/` folder — including nested subfolders to any depth — the reviewer reads all of it first and treats those conventions as higher priority than the generic checklist. Violations of a stated project rule are reported first, ahead of the generic categories.

## Install

The plugin directory doubles as its own marketplace, so a local install is two steps:

```bash
claude plugin marketplace add /Users/martinhanzl/PhpstormProjects/code-review-plugin
claude plugin install code-review-plugin@code-review-plugin
```

Check the manifest before installing, and confirm it loaded after:

```bash
claude plugin validate /Users/martinhanzl/PhpstormProjects/code-review-plugin
claude plugin list
```

Edits to the files take effect on the next Claude Code session — no reinstall needed while the marketplace points at this directory.

## Usage

```
/review                    # uncommitted working-tree changes
/review current            # current branch vs. its base, committed + uncommitted
/review staged             # staged changes only
/review main               # everything on this branch since main
/review HEAD~3             # last three commits
/review src/auth.ts        # specific files
/review main --fix         # review, then apply only 🔴 critical findings
```

## Layout

```
code-review-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # makes this dir installable as a marketplace
├── commands/review.md
├── agents/code-reviewer.md
├── skills/review-checklist/SKILL.md
└── README.md
```

## Publishing

Push this directory as a git repo, then anyone can install it with:

```bash
claude plugin marketplace add <owner>/<repo>
claude plugin install code-review-plugin@code-review-plugin
```
