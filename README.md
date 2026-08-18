# code-review-plugin

Structured code review for Claude Code. Reviews the current diff, a branch, or specific files and reports severity-tagged findings with concrete fixes.

## Contents

| Piece | Type | Purpose |
| --- | --- | --- |
| `/cr` | command | Entry point. Loads project `agents/` rules if present, resolves the diff, reviews it, reports findings. `--fix` applies 🔴 critical ones. |
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

Published at [Web-pulse-cz/code-review-plugin](https://github.com/Web-pulse-cz/code-review-plugin). Install from anywhere with:

```bash
claude plugin marketplace add Web-pulse-cz/code-review-plugin
claude plugin install code-review-plugin@code-review-plugin
```

Confirm it loaded:

```bash
claude plugin list
```

### Local dev install

Working on this repo locally? Point the marketplace at your working copy instead — the plugin directory doubles as its own marketplace:

```bash
claude plugin marketplace add /path/to/code-review-plugin
claude plugin install code-review-plugin@code-review-plugin
claude plugin validate /path/to/code-review-plugin
```

Edits to the files take effect on the next Claude Code session — no reinstall needed while the marketplace points at this directory.

## Usage

```
/cr                    # uncommitted working-tree changes
/cr current            # current branch vs. its base, committed + uncommitted
/cr staged             # staged changes only
/cr main               # everything on this branch since main
/cr HEAD~3             # last three commits
/cr src/auth.ts        # specific files
/cr main --fix         # review, then apply only 🔴 critical findings
```

## Layout

```
code-review-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # makes this dir installable as a marketplace
├── commands/cr.md
├── agents/code-reviewer.md
├── skills/review-checklist/SKILL.md
└── README.md
```

## Publishing

Push changes to `main` on [Web-pulse-cz/code-review-plugin](https://github.com/Web-pulse-cz/code-review-plugin). New installs use the `Install` command above; existing installs pick up the update with:

```bash
claude plugin update code-review-plugin
```
