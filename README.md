# hermes-generic-ops

A small, self-contained skill bundle for [Hermes Agent](https://github.com/NousResearch/hermes-agent).
Seven skills covering the day-to-day operational work that is easy to get subtly wrong:
GitHub private projects, Cloudflare Tunnel + DNS, git worktrees, repo layout, containerised
test runs, Actions-driven deploys, and the "should we run/buy this at all?" decision.

Everything here is **generic**: placeholders instead of accounts, env-var names instead of
tokens, commands instead of tables of machine-specific facts. No credentials, no hostnames,
no customer or organisation names, no internal paths. The skills are written to be *run and
verified* by the instance that loads them — they tell you which command proves the claim
rather than asserting that it holds.

Windows, macOS and Linux are all supported. Where a command is shell-specific, both a POSIX
and a PowerShell form are given, and anything that only exists on one platform is marked.

## Contents

| Skill | What it is for |
|---|---|
| [`github-private-projects`](skills/github-private-projects/SKILL.md) | Authenticate `git`/`gh` on a fresh machine (token, device flow, SSH), then create, configure, grant access to, and retire **private** repositories — personal and organisation. |
| [`cloudflare-tunnel-subdomains`](skills/cloudflare-tunnel-subdomains/SKILL.md) | Put a local service on `<name>.<your-domain>` through a named Cloudflare Tunnel: token scopes, tunnel + ingress config, proxied CNAME, running the daemon on Linux/macOS/Windows/Docker, verification, teardown. |
| [`git-worktrees`](skills/git-worktrees/SKILL.md) | Several checkouts of one repository for parallel work or hotfixes without dirtying the main clone: add/list/move/repair/remove/prune, per-worktree ports and env files, the failure modes. |
| [`git-repo-workspace-layout`](skills/git-repo-workspace-layout/SKILL.md) | A `<repos-root>/<host>/<owner>/<repo>` layout that survives same-named repos across orgs, plus move/transfer procedure (including the virtualenv that breaks silently), mirror migration, and pre-push secret hygiene. |
| [`docker-test-runner`](skills/docker-test-runner/SKILL.md) | Run a repository's test suite in its own image with a read-only bind mount, keep the worktree clean, and avoid port/project-name collisions between stacks. |
| [`github-actions-ssh-deploy`](skills/github-actions-ssh-deploy/SKILL.md) | Push-to-deploy over SSH from GitHub Actions: the bootstrap ordering trap, compose project-name collisions, secret masking in logs, remote diagnostics without an SSH session, and proving a deploy actually shipped. |
| [`service-fit-and-selection`](skills/service-fit-and-selection/SKILL.md) | Decide whether to self-host a service or buy one: resource fit, backup/restore, auth, exposure, maintenance and exit cost, plus a vendor comparison procedure. |

## Install

Skills are plain directories under the Hermes skills root. `$HERMES_HOME` defaults to
`~/.hermes` (`%USERPROFILE%\.hermes` on Windows); a profile uses
`$HERMES_HOME/profiles/<profile>/skills/`.

```bash
git clone https://github.com/jeanqussa/hermes-generic-ops.git
cp -r hermes-generic-ops/skills/* "${HERMES_HOME:-$HOME/.hermes}/skills/"
```

```powershell
git clone https://github.com/jeanqussa/hermes-generic-ops.git
Copy-Item -Recurse -Force hermes-generic-ops\skills\* "$env:USERPROFILE\.hermes\skills\"
```

Start a session afterwards: the skills are discovered on startup and listed by name with
their one-line descriptions. Load one with `skill_view(name="git-worktrees")` (or just let
the agent load it when the task matches the description).

To install into a single profile instead of the default, replace the destination with
`$HERMES_HOME/profiles/<profile>/skills/`.

## Using it

Each `SKILL.md` opens with **When to Use** and then a sequence of steps. Steps state a
command and what its output must look like; where a platform differs, a short table follows.
Read the *Pitfalls* section before running a procedure for the first time — it is the part
that came from failures, not from documentation.

Placeholders used throughout: `<owner>`, `<org>`, `<repo>`, `<branch>`, `<domain>`,
`<name>`, `<port>`, and environment variables such as `$GH_TOKEN`, `$CF_API_TOKEN`,
`$CF_ACCOUNT_ID`, `$CF_ZONE_ID`. None of them are real values.

## Verification status

These skills were written on a Linux workstation and reviewed for portability, but they are
**not pre-tested for you**: commands are meant to be exercised on the target machine, and
every procedure includes the command that proves it worked. If a step disagrees with what
you observe, fix the skill (`skill_manage(action="patch")`) rather than working around it —
that is the point of shipping procedures instead of prose.

## Licence

MIT — see [LICENSE](LICENSE).
