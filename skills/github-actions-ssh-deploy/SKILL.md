---
name: github-actions-ssh-deploy
description: "Use when adding or debugging push-to-deploy over SSH in Actions."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [github-actions, deploy, ssh, docker-compose, ci, verification]
    category: software-development
    related_skills: [github-private-projects, docker-test-runner, git-repo-workspace-layout]
---

# Push-to-deploy over SSH from GitHub Actions

## When to Use

- "Push to `<branch>` must deploy to `<host>`", where the host is reachable by SSH from a
  GitHub runner but not (conveniently) from your machine.
- A deploy workflow fails and the log gives nothing useful.
- You need to know whether a deploy actually shipped, not whether the job went green.

The workflow runs on GitHub's Linux runners, so authoring it from Windows is fine; only the
local pre-flight checks need care.

## The pattern

`.github/workflows/deploy-<env>.yml`:

```yaml
name: Deploy to <env>
on:
  push:
    branches: [<branch>]
concurrency:
  group: deploy-<env>
  cancel-in-progress: false        # never cancel a deploy mid-flight
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: <env>             # gates on environment protection rules
    steps:
      - uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script_stop: true
          script: |
            set -euo pipefail
            cd <deploy-dir>
            git fetch origin <branch>
            git checkout <branch>
            git pull --ff-only origin <branch>
            ./scripts/deploy.sh <env>
```

Required repository secrets: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY` (the **private**
key, PEM, with the public half in the host's `~/.ssh/authorized_keys`). Set them with
`gh secret set DEPLOY_HOST --body "<host>"`, or paste in the repository settings UI.

Design rules that make the difference between a deploy that works and one that works most of
the time:

- `git fetch` + `checkout` + `pull --ff-only` happen **in the workflow**, before the script is
  invoked (see the bootstrap pitfall below). `--ff-only` fails loudly instead of creating a
  merge commit on a server.
- The deploy directory is per environment (`<base>/<env>`), and the branch checkout is
  asserted: `git rev-parse --abbrev-ref HEAD` must equal the expected branch.
- `scripts/deploy.sh` in the repo does the rest: `set -euo pipefail`, `--env-file`, an explicit
  compose project name, a health-check poll loop that **exits 1** on failure, and cleanup.
- One concurrency group per environment so two pushes cannot interleave.

Skeleton `scripts/deploy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
env="${1:?usage: deploy.sh <env>}"
cd "$(dirname "$0")/.."
docker compose -p "<service>-$env" --env-file ".env.$env" up -d --build
for i in $(seq 1 30); do
  if curl -fsS "http://127.0.0.1:${PORT:?}/health" >/dev/null; then
    echo "Deployed commit: $(git rev-parse HEAD)"; exit 0
  fi
  sleep 2
done
echo "health check failed after 60s" >&2
docker compose -p "<service>-$env" logs --tail=100 >&2
exit 1
```

## Pitfalls that actually bite

- **Bootstrap chicken-and-egg.** If `scripts/deploy.sh` lives on the branch being deployed and
  the server clone sits on an older branch, the first run dies with
  `./scripts/deploy.sh: No such file or directory` (exit 127). Fix it by checking out the branch
  in the workflow first — never by copying the script to the server.
- **Directory-derived compose project names collide.** Running `docker compose up -d` from
  `<base>/<service-a>/dev` and `<base>/<service-b>/dev` gives both the project name `dev`.
  `docker compose -p dev down --remove-orphans` from one then destroys the other's containers.
  Always pass `-p <service>-<env>`.
- **Volume identity follows the project name** (`<project>_<volume>`). Renaming the project
  gives the new stack an empty database; either reseed or copy the old volume on purpose.
- **Secret masking mangles log lines.** GitHub replaces every occurrence of a secret value with
  `***`, so a secret whose value is `deploy` makes the word "deploy" disappear from logs —
  paths become `./***.sh`, and a config line can vanish entirely. Confusing, not broken: read
  the raw log (`gh run view <id> --log`) and expect the substitution.
- **`script_stop` defaults to false** in older `ssh-action` versions, so a failing command does
  not fail the job. Set it explicitly, and also `set -euo pipefail` inside the script block.
- **The runner's public IP is not stable**, so IP allowlists on the host break intermittently.
  Prefer key-based auth with a dedicated deploy user over source-IP rules.
- **`appleboy/ssh-action` needs the key in its raw form**, including newlines. A key pasted
  into a GitHub secret usually survives fine; a key stored base64-encoded does not — decode
  it in the workflow if that is how it is stored.
- **Permissions drift**: `chmod +x` applied on the server but not committed means the file
  arrives as `100644`. `git ls-files -s scripts/deploy.sh` must show mode `100755`.

## Diagnosing a host you cannot SSH into

Commit a **read-only** workflow on a scratch branch — triggered only by pushes to that branch,
never by the deploy branch — run it, read the log, then delete the branch. It executes as the
deploy user and answers every question a shell session would:

```yaml
on: { push: { branches: [chore/<service>-diag] } }
# appleboy/ssh-action with script_stop: false and:
#   set +e
#   ls -la <deploy-dir>; ls -la .env*
#   sed -E 's/=.*/=<redacted>/' .env.*        # keys without values
#   grep -E '^(PORT|NODE_ENV|PUBLIC_URL)=' .env.*
#   git -C <deploy-dir> rev-parse --abbrev-ref HEAD; git -C <deploy-dir> rev-parse HEAD
#   git -C <deploy-dir> remote -v
#   docker ps -a --format '{{.Names}} | {{.Image}} | {{.Ports}} | {{.Status}}'
#   docker inspect --format '{{.Name}} project={{index .Config.Labels "com.docker.compose.project"}} wd={{index .Config.Labels "com.docker.compose.project.working_dir"}}' $(docker ps -aq)
#   docker compose ls
```

This is what turns "deploy failed" into "the clone is on the wrong branch, `.env.dev` does not
exist, and two services share a compose project name". Delete the branch afterwards — the file
must not stay in the repository.

On Windows, `sed -E 's/=.*/=<redacted>/'` runs *on the remote host* (Linux), not locally, so it
is fine; do not try to reproduce it in PowerShell.

## Verifying a deploy really worked

Four checks, in increasing strength:

1. `gh run watch <id> --exit-status` — the job's conclusion, then **read the log**, because
   `script_stop: false` and swallowed failures both produce green runs.
2. The deploy script prints `Deployed commit: <sha>`; assert it equals the pushed head:
   ```bash
   git ls-remote origin <branch>
   ```
3. Public smoke test: `curl -sS https://<host>/health` and the app root.
4. Prove the change actually shipped — capture before/after response headers:
   ```bash
   curl -sS -D - -o /dev/null https://<host>/ | grep -iE '^(etag|last-modified)'
   ```
   A new `last-modified` inside the deploy window is evidence; a `200` alone is not, because
   the previous version also returned 200.

Then confirm neighbours survived: `docker ps --filter label=com.docker.compose.project=<other>`
for any stack that shared the old project name.

## Local pre-flight

- `bash -n scripts/deploy.sh` (WSL or Git Bash on Windows).
- Executable bit committed: `git ls-files -s scripts/deploy.sh` → `100755`.
- Lint the workflow without installing anything:
  ```bash
  docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest
  ```
  ```powershell
  docker run --rm -v "${PWD}:/repo" -w /repo rhysd/actionlint:latest
  ```
- Diff against a workflow known to work in the same repository: identical except for the
  service path is the strongest evidence available without running it.
- Confirm Actions is enabled and permitted before debugging the YAML:
  `gh api repos/<owner>/<repo>/actions/permissions`.

## Rollback

Keep the previous release reachable before deploying: tag the commit, or run the stack with an
explicit image tag rather than `latest`. A rollback that depends on rebuilding the old code
after the new one failed is a rollback that happens during an incident.

```bash
git -C <deploy-dir> checkout <previous-sha>
./scripts/deploy.sh <env>
```

Rehearse it once while nothing is broken.
