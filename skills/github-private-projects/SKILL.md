---
name: github-private-projects
description: "Use when connecting a machine to GitHub and creating or managing private repos."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [github, gh, git, auth, private-repos, organisations, access-control, lifecycle]
    category: software-development
    related_skills: [git-repo-workspace-layout, git-worktrees, github-actions-ssh-deploy]
---

# GitHub private projects

## When to Use

- A machine, container or agent account has never talked to GitHub.
- Creating a **private** repository, personally or in an organisation.
- Granting or revoking access, changing visibility, transferring, archiving or deleting one.
- `gh` works but `git push` still prompts, or the reverse.

Public-repo work (issues, PR review, releases) is covered by the bundled `github` skill; this
one is about standing a private project up correctly and keeping it private.

## 0. Preflight — find out what already exists

```bash
git --version
gh --version 2>/dev/null || echo "gh not installed"
gh auth status 2>/dev/null || echo "gh not authenticated"
git config --global credential.helper 2>/dev/null || echo "no git credential helper"
```

Three independent stores are in play and **they are not the same thing**:

| Store | Used by | Where it lives |
|---|---|---|
| `gh` token | `gh` commands and `gh api` | `~/.config/gh/hosts.yml` (macOS/Linux), `%APPDATA%\GitHub CLI\hosts.yml` (Windows) |
| git credential helper | `git clone/push` over HTTPS | OS keyring, `~/.git-credentials`, or Windows Credential Manager |
| SSH key | `git` over `git@github.com:` | `~/.ssh/`, `%USERPROFILE%\.ssh` |

Authenticating `gh` does **not** by itself fix `git push`. Bridge them with
`gh auth setup-git`, which registers `gh` as git's credential helper for github.com.

## 1. Authenticate

Pick one path; all three end at the same verification step.

### A. Device flow (interactive machine)

```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
```

### B. Personal access token (headless, containers, CI-like setup)

Create a **classic** token at `https://github.com/settings/tokens` with:

| Scope | Needed for |
|---|---|
| `repo` | private repo read/write, creating personal private repos |
| `workflow` | adding or editing `.github/workflows/*` |
| `read:org` | listing org repos, team membership |
| `admin:org` | creating repos in an org you administer, team management |
| `delete_repo` | only if you will delete repositories |

Then, reading the token from a file or stdin — never as an argument:

```bash
gh auth login --with-token < /path/to/token.txt
# or: printf '%s' "$GH_TOKEN" | gh auth login --with-token
```

No OS keyring available (containers, some headless Linux, locked keychain)? Add
`--insecure-storage`; the token then lands in `hosts.yml` (mode `600`) instead of failing.
Fine-grained tokens work too, but must include **Administration: read and write** (account or
org level) or repo creation returns 403; an org must also permit fine-grained tokens.

### C. SSH

```bash
ssh-keygen -t ed25519 -C "<label-for-this-machine>" -f ~/.ssh/id_ed25519
gh ssh-key add ~/.ssh/id_ed25519.pub --title "<label-for-this-machine>"
ssh -T git@github.com    # exits 1, prints "successfully authenticated" — the exit code is not the test
```

The text is the test; `ssh -T` always exits non-zero on GitHub.

### Verify (all paths)

```bash
gh auth status
gh api user --jq '.login'
```

`gh auth status` prints the scopes it holds. A fine-grained token shows no scope line at all —
that is normal, not a failure. Then prove git transport itself, with prompting disabled:

```bash
GIT_TERMINAL_PROMPT=0 git ls-remote https://github.com/<owner>/<repo>.git
```

```powershell
$env:GIT_TERMINAL_PROMPT="0"; git ls-remote https://github.com/<owner>/<repo>.git
```

Read access passing but push failing is almost always a scope/role problem, not a credential
problem: check `gh api repos/<owner>/<repo> --jq '.permissions'`.

## 2. Create a private repository

```bash
# personal, empty, with a README, cloned next to you
gh repo create <name> --private --add-readme --clone

# organisation
gh repo create <org>/<name> --private --description "<one line>"

# from a directory that already has commits
gh repo create <owner>/<name> --private --source . --remote origin --push
```

Useful flags: `--gitignore <template>`, `--license mit`, `--template <owner>/<repo>`.
Names are case-insensitive on GitHub even where your filesystem is not, may not contain
spaces, and must be unique within the owner.

`--push` with no commits fails — commit first (`git add -A && git commit -m "init"`).

### Verify creation

```bash
gh repo view <owner>/<name> --json nameWithOwner,visibility,isPrivate,defaultBranchRef,url
```

Assert `"visibility":"PRIVATE"` **and** `"isPrivate":true`. Visibility is the one property
where a successful command does not imply the requested state: org policy can override what
you asked for, so read it back.

## 3. Configure it

```bash
gh repo edit <owner>/<name> \
  --description "<one line>" --homepage "<url>" \
  --add-topic <topic> --add-topic <topic> \
  --enable-issues --enable-wiki --enable-projects \
  --delete-branch-on-merge \
  --allow-squash-merge --allow-merge-commit false --allow-rebase-merge
```

Default branch (for a repo created empty or from a template):

```bash
gh repo edit <owner>/<name> --default-branch <branch>
```

### Branch protection / rulesets

```bash
gh api repos/<owner>/<repo>/branches/<branch>/protection
gh api repos/<owner>/<repo>/rulesets
```

Read the response before assuming what your plan allows. Branch protection on **private**
repositories has historically been plan-dependent (Pro/Team/Enterprise on personal accounts,
org plans vary). A 403 that names a plan limitation means "not included", not "misconfigured":
either upgrade, rely on rulesets if they are available to you, or enforce the rule in CI.
Do not claim protection is on because a command exited 0 — re-read the endpoint.

### Actions

```bash
gh api repos/<owner>/<repo>/actions/permissions
gh api -X PUT repos/<owner>/<repo>/actions/permissions -F enabled=true -f allowed_actions=all
```

`allowed_actions` is `all`, `local_only` or `selected`. If workflows must be restricted, set
`selected` and then configure `actions/permissions/selected-actions`; setting the value
without that call leaves an empty allowlist and every action fails to resolve.

### Secrets and variables

```bash
gh secret set <NAME> --body "$VALUE"
gh secret set <NAME> --env-file .env.production      # many at once
gh secret list
gh variable set <NAME> --body "<value>"
```

Secret values are write-only: `gh secret list` shows names and timestamps, never values.
Verification is "the name appears", plus a workflow run that consumes it.

## 4. Access

```bash
gh api repos/<owner>/<repo>/collaborators --jq '.[].login'
gh api -X PUT repos/<owner>/<repo>/collaborators/<user> -f permission=push
gh api -X DELETE repos/<owner>/<repo>/collaborators/<user>

# team (organisations)
gh api -X PUT orgs/<org>/teams/<team>/repos/<org>/<repo> -f permission=push
```

Permissions: `pull`, `triage`, `push`, `maintain`, `admin`. For an org repo, confirm your own
standing first — `gh api orgs/<org>/memberships/<you> --jq '.role'` must be `admin` to create
repos or change team access.

## 5. Retire, move, or expose

```bash
gh repo rename <new-name>                 # old URLs redirect, clones keep working
gh repo archive <owner>/<name> --yes      # read-only, keeps history and issues
gh repo archive <owner>/<name> --yes --unarchive
gh api -X POST repos/<owner>/<repo>/transfer -f new_owner=<org>
gh repo delete <owner>/<name> --yes       # requires the delete_repo scope
```

Before a transfer, check on the target: name collision
(`gh api orgs/<org>/repos --paginate --jq '.[].name'`), and what the transfer may drop —
Actions secrets, environments, branch protection, webhooks, packages and Pages settings do
not all travel. Record them first. The transfer API's response echoes the **pre-transfer**
owner; that is normal, so verify with `gh repo view <new-owner>/<repo>`.

The local clone's path and remote must be updated separately — see
`git-repo-workspace-layout`, "Moving a repo between owners".

### Before switching a private repo to public

Irreversible in practice: forks cannot be recalled and the history is world-readable.

1. Scan **all history**, not HEAD. Either run `gitleaks`/`trufflehog` in Docker over the
   clone, or list every blob and grep it:
   ```bash
   git rev-list --all --objects | awk '{print $1}' | git cat-file --batch \
     | grep -aiE '(ghp_|gho_|github_pat_|sk-|AKIA|BEGIN [A-Z ]*PRIVATE KEY|password[=:])'
   ```
2. Check the files a private repo can afford to be sloppy about: `.env*`, deploy scripts,
   CI logs, test fixtures with real data, screenshots.
3. Check commit author emails and any internal URLs in code, docs and the repo description.
4. Check the licence actually permits publication.
5. `gh repo edit <owner>/<repo> --visibility public --accept-visibility-change-consequences`

If anything is found, rotate the credential **first**. Removing it from history afterwards
does not un-leak it, and the repo may already be cached or forked.

## Pitfalls

- `gh auth login --with-token` reads the token from **stdin**. `--with-token "$TOKEN"` treats
  the value as a filename and fails with a confusing parse error.
- Never put a token in a command line, a remote URL (`https://user:token@...`) or a commit —
  it lands in shell history, `.git/config`, or the log. Prefer the credential helper or SSH.
- `gh auth setup-git` is a separate step. Skip it and `git push` prompts while `gh` works.
- On Windows the default credential helper (`manager`) stores in Windows Credential Manager,
  which is preferable to `store` (`~/.git-credentials`, plaintext). Check with
  `git config --global credential.helper` before changing anything.
- Org repos behind SAML SSO return 403 "Resource protected by organization SAML enforcement"
  until the token is authorised for that org in GitHub's settings — no re-login fixes it.
- `gh repo create` run inside an existing repository without `--source` creates an unrelated
  empty repo; with `--source .` it pushes what is there, including anything staged by accident.
- Private-repo Actions minutes are metered against your plan's included quota; public-repo
  minutes are not. A workflow added to a private repo can therefore start costing money.
- Deleting a repo does not delete a local clone, CI caches, or forks made while it was public.
