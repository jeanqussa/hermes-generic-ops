---
name: git-repo-workspace-layout
description: "Use when cloning repos or moving them between owners/hosts."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [git, clone, workspace, layout, mirrors, migration, secrets]
    category: software-development
    related_skills: [git-worktrees, github-private-projects, docker-test-runner]
---

# Repository workspace layout

## When to Use

- Cloning anything and deciding where it should live.
- A machine accumulates repos and nobody can tell which `api` is which.
- Moving a repo between owners, organisations or hosts.
- Before pushing a repository somewhere new — including the first push of a private one.

## The layout

```
<repos-root>/<host>/<owner>/<repo>
```

```
~/repos/github.com/<owner>/<repo>
~/repos/github.com/<org>/<repo>
~/repos/gitlab.com/<group>/<repo>
C:\src\github.com\<owner>\<repo>
```

The path mirrors the remote URL, so two different owners can both have a repository called
`api` without inventing suffixes. This is not cosmetic: name collisions across organisations
are normal, and a flat `~/projects/api` layout silently merges two histories the first time
someone clones the second one.

Rules that keep it working:

- Everything git lives under one root. Not `$HOME` itself, not scattered across `~/Desktop`.
- Toolchains, virtualenvs and downloaded binaries do **not** go in the repos root — they
  belong in a tool directory or a container.
- Throwaway experiments belong in a scratch directory outside the repos root, so they can
  never be committed to a real clone by accident.
- Case matters where the filesystem says it does. Windows and default macOS filesystems are
  case-insensitive, Linux is not: a repo owned by `Acme` and one owned by `acme` are the same
  directory on Windows. If you need both, one of them needs a different root.

## The `clone` helper

A shell function that derives the destination from the URL. Add to `~/.bashrc` / `~/.zshrc`:

```bash
clone() {
  local url path
  case "$1" in
    http*|git@*|ssh://*) url="$1" ;;
    */*) url="https://github.com/$1.git" ;;
    *) printf 'clone: give a URL or owner/repo\n' >&2; return 1 ;;
  esac
  path="${url%%.git}"; path="${path#*://}"; path="${path#*@}"; path="${path/://}"
  path="$HOME/repos/$path"
  if [ -d "$path/.git" ]; then
    printf 'clone: %s already exists\n' "$path"; cd "$path" && return 0
  fi
  mkdir -p "$(dirname "$path")" && git clone "$url" "$path" && cd "$path"
}
```

PowerShell equivalent (`$PROFILE`):

```powershell
function clone {
  param([Parameter(Mandatory)][string]$Ref)
  if ($Ref -notmatch '^(https?://|git@|ssh://)') { $Ref = "https://github.com/$Ref" }
  $path = $Ref -replace '\.git$','' -replace '^[a-z+]+://','' -replace '^git@','' -replace ':','/'
  $dest = Join-Path $HOME "repos/$path"
  if (Test-Path (Join-Path $dest '.git')) { Write-Host "clone: $dest already exists"; Set-Location $dest; return }
  New-Item -ItemType Directory -Force (Split-Path $dest) | Out-Null
  git clone $Ref $dest; if ($LASTEXITCODE -eq 0) { Set-Location $dest }
}
```

Handles a full URL or `owner/repo` shorthand, creates the owner directory, and re-running on
an existing clone just changes into it.

**Non-interactive shells do not load `.bashrc`** (it usually starts with a guard returning
early when `$PS1` is unset), so agent tool calls and cron jobs will not have `clone`. Write
the `git clone` out in full there, or invoke `bash -ic`, and expect terminal-control noise on
stderr from the latter.

## Moving a repo between owners

The path mirrors the URL, so an owner change moves the clone too — and the clone carries
state that breaks silently if it is only `mv`'d.

1. **On the target**, check for a name collision and record what the move may drop:
   ```bash
   gh api orgs/<org>/repos --paginate --jq '.[].name'
   gh api repos/<owner>/<repo>/actions/secrets --jq '.total_count'
   gh repo view <owner>/<repo> --json visibility,defaultBranchRef,hasIssuesEnabled
   ```
   Confirm your role: `gh api orgs/<org>/memberships/<you> --jq '.role'` → `admin`.
2. **Transfer** with the host's API (GitHub: `gh api -X POST repos/<owner>/<repo>/transfer -f new_owner=<org>`).
   The response echoes the pre-transfer owner; verify separately with
   `gh repo view <org>/<repo>`.
3. **Move the clone and re-point it**, leaving the old owner directory if other clones still
   use it:
   ```bash
   mkdir -p <repos-root>/<host>/<new-owner>
   mv <repos-root>/<host>/<old-owner>/<repo> <repos-root>/<host>/<new-owner>/<repo>
   cd <repos-root>/<host>/<new-owner>/<repo>
   git remote set-url origin <new-url>
   git fetch -q origin && git status -sb
   ```
   ```powershell
   Move-Item <repos-root>\<host>\<old-owner>\<repo> <repos-root>\<host>\<new-owner>\<repo>
   ```
4. **Rebuild anything with an absolute path baked in.** A POSIX `.venv` hardcodes the
   interpreter path in every `bin/*` shebang and in editable-install metadata; Windows
   `.venv\Scripts\*.exe` launchers embed it too. After a move, `uv run`/`pytest`/`pip -e`
   fail with a path that no longer exists, or worse, quietly import the old tree. Delete and
   recreate (`rm -rf .venv && uv sync`), then run the suite from the new path. The same
   applies to `node_modules` with native builds, compiled test fixtures, and lockfile tools
   that record a prefix.
5. **Old URLs keep working** — GitHub redirects after a transfer, so `git ls-remote` on the
   old URL still returns branches. Update references anyway, but anything left behind
   (editor caches, shell history, agent session dirs) is a record, not a live reference.
6. Sweep the repo for stale references to the old owner: `git grep -nI 'github\.com/<old-owner>'`,
   plus the repo's own description and homepage, which no move updates.

## Mirroring between hosts

A one-shot copy that keeps every branch, tag and ref:

```bash
git clone --mirror https://<host>/<owner>/<repo>.git <repos-root>/<host>/<owner>/<repo>.git
cd <repos-root>/<host>/<owner>/<repo>.git
git remote add <target> https://<target-host>/<owner>/<repo>.git
GIT_TERMINAL_PROMPT=0 git push --mirror <target>
```

- `--mirror` pushes refs exactly, deleting remote refs absent locally, so a second run
  re-syncs instead of accumulating. Point it at the **target** remote, never `origin`.
- Prefer `--mirror` even for a single-branch source: tags and non-`heads` refs are dropped by
  a plain clone.
- Verify by content, not by "push succeeded": clone the target elsewhere and compare
  `git rev-parse HEAD^{tree}`, `git rev-list --count HEAD` and the file count with the source.

## Secret hygiene before pushing

- A file can be added despite `.gitignore` with `git add -f`. Do **not** add a `!<path>`
  negation instead — that un-ignores every future file of that shape. Once tracked, the file
  is no longer subject to the ignore rule at all, so any comment claiming it is untracked is
  now a lie: fix the comment.
- Before staging anything env-shaped, scan the **staged** blob, not the working file:
  ```bash
  git show :<path> | grep -E '^[A-Z_]+=.+$'
  ```
  Compare the header names against real credentials; a placeholder is fine, a live key is not.
- To scan history before publishing a repository:
  ```bash
  git rev-list --all --objects | awk '{print $1}' | git cat-file --batch \
    | grep -aiE '(ghp_|gho_|github_pat_|sk-|AKIA|BEGIN [A-Z ]*PRIVATE KEY)'
  ```
  `gitleaks` or `trufflehog` in Docker is stronger if available.
- Committing a secret and deleting it in the next commit does not remove it — rotate the
  secret, then rewrite history if it truly must go. Rotation is the part that matters.

## Pitfalls

- `ssh://git@<host>:<port>/<owner>/<repo>` loses the port when naively splitting on `:`
  (it ends up in the path). Use the https URL for non-standard ports, or parse with git itself.
- Tools and dependencies do not belong in the repos root. On a machine without admin rights,
  install user-locally (a tool directory plus a `PATH` entry) or use a container — but keep
  the *result* out of the repository.
- Windows: `core.autocrlf` rewrites line endings on checkout; a repo with `LF`-only scripts
  breaks when they arrive as `CRLF` (`bad interpreter: /bin/bash^M`). Set
  `core.autocrlf=input` and keep a `.gitattributes` with `* text=auto`.
- Windows: filesystem is case-insensitive, so a repo tracking both `README.md` and `readme.md`
  cannot be checked out there. `git config core.ignorecase true` is set by git already; the
  problem is real, the fix is not.
- Long paths: enable `core.longpaths` and the OS policy before cloning a deep monorepo.
- A repo that was cloned with `--depth 1` cannot be pushed from usefully, and `git worktree`
  from it sees only one commit. Re-clone full for anything you will develop.
