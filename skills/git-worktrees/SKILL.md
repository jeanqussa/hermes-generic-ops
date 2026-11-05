---
name: git-worktrees
description: "Use when parallel work needs separate checkouts of one repo."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [git, worktree, parallel-work, branches, checkout, agents]
    category: software-development
    related_skills: [git-repo-workspace-layout, github-private-projects]
---

# Git worktrees

## When to Use

- Two branches must be worked on at once, or a hotfix must not disturb a dirty checkout.
- Several agents (or a human and an agent) work in the same repository simultaneously.
- A review needs the PR branch built and run while `main` stays checked out.
- A long build or test run occupies one tree and you still need to commit elsewhere.

A worktree is an extra working directory attached to the **same** repository: one `.git`
object store and ref database, many checkouts. Unlike a second `git clone`, branches, stashes
(config-dependent) and objects are shared, so a commit made in one tree is immediately visible
in the other without a fetch.

## 1. Layout

Put worktrees in a sibling directory, never inside the repository:

```
<repos-root>/<host>/<owner>/<repo>/                     main clone
<repos-root>/<host>/<owner>/<repo>.worktrees/<branch>/  one tree per branch
```

The rule matters: a worktree inside the repo would be picked up as untracked files by the
parent, and `git status` in the parent becomes noise. Keep the branch name in the last path
segment, and use the branch's full name with `/` replaced by `-` when it would otherwise nest
(`feature/x` → `feature-x`).

```bash
cd <repos-root>/<host>/<owner>
git -C <repo> worktree add ../<repo>.worktrees/<branch> -b <branch>
```

```powershell
cd <repos-root>\<host>\<owner>
git -C <repo> worktree add ..\<repo>.worktrees\<branch> -b <branch>
```

Relative paths resolve against the **current directory**, not the repository, so prefer an
absolute path when scripting — a worktree created from a subdirectory lands somewhere
unexpected otherwise.

## 2. Daily commands

```bash
git worktree add <path> <branch>              # existing branch
git worktree add <path> -b <branch> [<start>] # new branch from <start> (default HEAD)
git worktree add <path> -B <branch> <start>   # create or reset <branch> to <start>
git worktree add --detach <path> <sha-or-tag> # read-only inspection, no branch
git worktree list                             # path, commit, branch
git worktree list --porcelain                 # same, machine-readable
```

For reviewing a pull request without a local branch:

```bash
git fetch origin pull/<n>/head:pr-<n>
git worktree add --detach ../<repo>.worktrees/pr-<n> pr-<n>
```

Lifecycle:

```bash
git worktree lock   <path> --reason "long-running build"   # prevents prune
git worktree unlock <path>
git worktree move   <path> <new-path>
git worktree remove <path>            # refuses if dirty or has untracked files
git worktree remove --force <path>    # discard those changes
git worktree remove -f -f <path>      # also bypasses a lock (and submodule checks)
git worktree prune                    # drop admin entries for trees deleted by hand
git worktree prune --dry-run -v
git worktree repair [<path>]          # re-link after a directory was moved
```

`git worktree move` and `repair` need git 2.17 and 2.30 respectively —
`git --version` first if the machine is old.

## 3. What is NOT shared

This is where worktrees surprise people:

| Item | In a new worktree |
|---|---|
| Tracked files | present at the branch's commit |
| `.git` | a **file** containing `gitdir: <main>/.git/worktrees/<name>` |
| Ignored/untracked files (`.env`, keys, local config) | **absent** |
| `node_modules`, `.venv`, build caches | **absent** |
| Stashes | shared (one stash list per repository) |
| Hooks, config | shared |
| Index | separate per worktree |

So a worktree is not a runnable copy until its untracked prerequisites exist. Either copy
them deliberately or, for a handful of env files on POSIX, link them:

```bash
ln -s ../../<repo>/.env .env
```

On Windows a symlink needs Developer Mode or Administrator; copying is usually the honest
choice. Never commit an env file to make a worktree work.

Anything with per-checkout state needs a per-worktree identity: dev-server ports, database
names, Docker compose project names (`docker compose -p <repo>-<branch>`), test databases,
and browser profiles. Two worktrees that both claim port 3000 or the same compose project
name will silently fight.

## 4. Parallel agents

One worktree per agent, one branch per worktree. Two agents in one checkout corrupt each
other's index and staging area; their sessions are isolated but the checkout is not.

```bash
for b in agent-a agent-b; do
  git worktree add "../<repo>.worktrees/$b" -b "$b"
done
```

Give each its own port range, its own compose project name, and its own scratch directory for
logs and artefacts. Merge or open PRs from each branch as usual — the shared object store
means no fetching back and forth.

Hermes' own `-w` / worktree mode does exactly this when spawning agent processes; use it
rather than hand-rolling the same thing.

## 5. Verify

```bash
git worktree list --porcelain
```

Each entry has `worktree <path>`, `HEAD <sha>` and either `branch refs/heads/<b>` or
`detached`. Check that the paths are where you intended and that no entry is `prunable`.

## Pitfalls

- **A branch can only be checked out in one worktree.** `git worktree add <path> <branch>`
  fails with "already checked out at" — that is the feature, not an error to work around.
  Use `--detach` when you only need to look.
- **`git branch -d <branch>` fails while a worktree has it checked out.** Remove the worktree
  first, or delete with `-D` after removing.
- **Moving directories breaks the link.** Both `.git` and `.git/worktrees/<name>/gitdir`
  store absolute paths. Moving the main clone, or a worktree, leaves a tree that looks fine
  but cannot run git until `git worktree repair` (run from the main clone; pass paths when
  several moved).
- **`rm -rf` a worktree instead of `git worktree remove`** leaves an admin entry, and later
  `git worktree add` at the same path says "already registered". `git worktree prune` fixes it.
- **`git worktree remove` refuses a dirty tree**, including untracked files created by a test
  run. Commit, stash, or `--force` — read the diff before forcing.
- **Submodules are not initialised in a new worktree**:
  `git submodule update --init --recursive` there, after adding.
- **A `.git` that is a file breaks naive tooling** — scripts and GUIs that assume `.git` is a
  directory (some build scripts, some IDE watchers) fail in linked worktrees only.
- **Windows path length.** The path is stored absolutely and nested `node_modules`/`.venv`
  trees can exceed 260 characters. Keep the repos root shallow (`C:\src`), and enable long
  paths (`git config --global core.longpaths true`, plus the OS policy) or expect odd errors.
- **Disk use grows per checkout** with generated files, though objects stay shared. Prune
  stale worktrees rather than letting them pile up.
- `git worktree` does not exist before git 2.5, and `add --orphan`/`move` came later; an old
  CI runner is a real place to meet this.
