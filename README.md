# code-review-plugin

Structured code review for Claude Code. Reviews the current diff, a branch, or specific files and reports severity-tagged findings with concrete fixes.

## Contents

| Piece | Type | Purpose |
| --- | --- | --- |
| `/review` | command | Entry point. Resolves the diff, reviews it, reports findings. `--fix` applies critical/high ones. |
| `code-reviewer` | agent | Read-only reviewer. Auto-invoked for larger diffs, or callable directly. |
| `review-checklist` | skill | Stack-specific failure modes (PHP/Laravel, TS/React, SQL, shell). |

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
/review staged             # staged changes only
/review main               # everything on this branch since main
/review HEAD~3             # last three commits
/review src/auth.ts        # specific files
/review main --fix         # review, then apply critical + high findings
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
