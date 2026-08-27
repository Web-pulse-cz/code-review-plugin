# code-review-plugin

Structured code review for Claude Code. Reviews the current diff, a branch, individual commits, or specific files and reports severity-tagged findings with concrete fixes.

## Contents

| Piece | Type | Purpose |
| --- | --- | --- |
| `/diffreview` | command | Entry point. Loads project `agents/` rules if present, resolves the diff, reviews it, reports findings. Targets the working tree, a branch, individual commits, or files. `--fix` applies 🔴 critical ones. |
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
/diffreview                      # uncommitted working-tree changes
/diffreview current              # current branch vs. its base, committed + uncommitted
/diffreview staged               # staged changes only
/diffreview main                 # everything on this branch since main
/diffreview HEAD~3               # from that commit up to the working tree
/diffreview src/auth.ts          # specific files
/diffreview main --fix           # review, then apply only 🔴 critical findings
```

### Commits only — skipping merged-in branches

```
/diffreview commits              # only this branch's own commits, merges excluded
/diffreview commits 3            # only the last 3 of them
/diffreview commit abc1234       # that one commit and nothing else
/diffreview origin/main..HEAD    # a range, again own commits only
```

These resolve to `git log --first-parent --no-merges -p`, which follows only the branch's own side of every merge and drops the merge commits themselves. Use them when a PR branch has other branches merged into it.

Why it matters — `current` and `main` are net tree diffs from the merge-base:

| Situation | `current` / `main` | `commits` |
| --- | --- | --- |
| `main` merged into the branch, before or after your commits | fine — the merge-base moves forward with it, so only your work is diffed | fine |
| A colleague's feature branch merged into yours | **their changes are reviewed too** — the merge-base against `main` is unchanged | only your commits |
| A line broken in one commit and fixed in a later one | not reported (only the end state is diffed) | not reported either — the reviewer checks the current file content and drops anything already fixed |

`commits` covers committed work only. For uncommitted edits use `current` or the bare `/diffreview`.

## Layout

```
code-review-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # makes this dir installable as a marketplace
├── commands/diffreview.md
├── agents/code-reviewer.md
├── skills/review-checklist/SKILL.md
└── README.md
```

## Publishing

Push changes to `main` on [Web-pulse-cz/code-review-plugin](https://github.com/Web-pulse-cz/code-review-plugin) and bump `version` in `.claude-plugin/plugin.json`. New installs use the `Install` command above.

### Updating an existing install

The marketplace metadata is cached locally, so refresh it first, then update the plugin, then restart Claude Code:

```bash
claude plugin marketplace update code-review-plugin
claude plugin update code-review-plugin
```

`claude plugin list` shows the installed version — after this it should read `0.2.0`. The restart is what actually loads the new command and agent definitions.
