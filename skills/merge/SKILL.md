---
name: merge
description: Confirm PRs, watch required CI, fix failures, merge. Only runs when the user types /merge.
argument-hint: "[pr refs: backend#123 | owner/repo#123 | PR URL]..."
disable-model-invocation: true
---

# /merge

Takes confirmed PRs (at most one per repo, all in one feature) to merged. Works in any session.

## Paths
- `<skill>` = this skill's base dir. Watcher: `<skill>/scripts/merge-watch`.
- Config: `~/.config/builder/config.json` (`repos` slugs/paths, `merge` section; shape `<skill>/config.example.json`). `~/.config/forge/config.json` and `~/.config/forge/active` (`<folder>/state.json` `prs[]`) are optional extras.
- Run state: `~/.config/builder/merge/<run>.json`.

## Main thread
- Product Q&A only. Tool calls: Skill, Agent, and one `Bash run_in_background` to start or re-arm the watcher. Everything else (lookups, git, fixes, ClickUp) runs in subagents with full briefs: goal, paths, PR, worktree, rules below, output wanted (≤5 lines).
- Relay short, in product language.

## 1. Pick PRs
A survey subagent returns candidates, at most one per repo:
1. Args given → those PRs.
2. Active forge feature → the lowest unmerged PR per repo from `prs[]`.
3. Else the open PR for the current branch of each configured repo path.
4. Else ask the user.

For each: repo, PR #, base, blockers (required checks, approval, label, conflict). Use `gh pr checks --required`, `gh pr view --json` (read only).

Suggest them and wait for an explicit yes. Never take up a PR, including the next one after a merge, without it.

## 2. Authority (confirmed PRs only)
Pre-approved: add the configured label, commit and plain-push CI fixes (no cap, incl. behaviour fixes required to pass checks), merge the base into the PR branch and push, retarget the base, final merge.
Never: force-push, rebase, close PRs, delete branches, bypass hooks, auto-merge.

## 3. Watch
Start: `Bash run_in_background`: `"<skill>/scripts/merge-watch" <repo>#<n> ...`. Re-arm: `"<skill>/scripts/merge-watch" --run <run>` (run id is in the first output line and the EXIT line).

The watcher labels, retargets, merges the base (clean only), notifies once on missing approval, and merges (squash into the default branch/main/master, merge commit otherwise) with `--match-head-commit`. It exits on its last line `EXIT <reason> <repo>#<n> <detail> state=<path>`:

- `check_failed` → fix subagent (below), then re-arm.
- `conflict` → a subagent lists conflicted files and both sides (read only: `git diff`, `git merge-tree --write-tree`, `git show origin/<base>:<file>`). Lay them out in chat. The user resolves. Never resolve yourself; apply only a resolution the user dictates, commit, plain push, re-arm.
- `dirty` → tell the user which worktree; re-arm after they clean it.
- `merged` → ClickUp step, then back to step 1 for the next PR in that repo/stack.
- `error` → relay the detail; ask.

## Fix subagent brief
- Worktree from the state file (`prs[].worktree`); else one with the branch checked out (`git worktree list`); else `<repo path>-merge-<n>` from `origin/<branch>`. Never touch a dirty worktree.
- Read failures from GitHub only: `gh run view <run-id> -R <slug> --log-failed`, check-run annotations (`gh api repos/<slug>/check-runs/<id>/annotations`).
- Never run tests, lint, type-check or pre-commit locally. GitHub CI is the only gate.
- Edit, commit (no trailers, one-line comments), `git push` plain. Report sha and what changed.

## ClickUp after merge
If the head branch has `CU-<id>`, a subagent moves that task to its list's done status (read the list's statuses; pick type `done`, else `closed`). No id → skip.
