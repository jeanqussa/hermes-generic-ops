---
name: docker-test-runner
description: "Use when running a repo's tests or stack in Docker containers."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [docker, docker-compose, tests, containers, ports, healthcheck, wsl]
    category: software-development
    related_skills: [git-worktrees, git-repo-workspace-layout, cloudflare-tunnel-subdomains]
---

# Running a repository's tests and stack in Docker

## When to Use

- A repository's runtime is defined by its `Dockerfile` (system packages, services, a specific
  interpreter) rather than by a lockfile you can install locally.
- You do not want host toolchains installed at all.
- Two stacks must run side by side without fighting over ports, networks or volumes.
- A container reports unhealthy and you need to know whether it is the app or the probe.

If the repo is pure-language with a lockfile (`uv`, `poetry`, `npm ci`), run it directly — a
container adds minutes per iteration for no isolation you need.

## 1. Run the suite in the repo's own image

Build the *runtime* image, then mount the working tree read-only and install test
dependencies inside the container:

```bash
docker build -t <name>:dev ./<context>
docker run --rm \
  -v "$PWD/<context>:/app:ro" -w /app \
  <name>:dev \
  sh -c "pip install -q -r requirements-dev.txt && python -m pytest -p no:cacheprovider -q"
```

```powershell
docker run --rm -v "${PWD}\<context>:/app:ro" -w /app <name>:dev `
  sh -c "pip install -q -r requirements-dev.txt && python -m pytest -p no:cacheprovider -q"
```

Node equivalent:

```bash
docker run --rm -v "$PWD:/app:ro" -w /app node:22-alpine \
  sh -c "npm ci --no-audit --no-fund && npm test -- --run"
```

Why each part:

| Part | Reason |
|---|---|
| `-v <code>:/app:ro` | the image has no test files (`.dockerignore` usually excludes them); the mount supplies them, and `:ro` keeps the git tree clean |
| `-w /app` | the image's `WORKDIR` is often a subdirectory of the mount |
| install test deps in the container | runtime images ship production requirements only |
| `-p no:cacheprovider` | pytest otherwise writes `.pytest_cache` into the mounted tree |
| `--rm` | a stale exited container per run is pure litter |

Check cleanliness afterwards — this is the assertion that the recipe worked:

```bash
git status --porcelain      # must print nothing
```

Run `docker run` with `--user "$(id -u):$(id -g)"` on Linux when the image runs as root and
writes into the mount; otherwise the working tree ends up owned by root and the next `git`
command fails with permission errors. On Docker Desktop (macOS/Windows) file ownership is
translated by the VM, so this is usually unnecessary there.

### Read-only mounts break more than caches

Any tool that writes a cache at import time fails against `:ro`, and the error reads like a
tool failure:

- `ruff check` → `Failed to initialize cache at /app/.ruff_cache: Read-only file system`,
  exit 1. Use `ruff check --no-cache`, or `RUFF_CACHE_DIR=/tmp`.
- `mypy` with `cache_dir` configured, `eslint --cache`, `prettier --cache`, `go build` with
  `GOCACHE` inside the tree — all the same shape. Point the cache at `/tmp` or disable it.
- `black --check` and plain `mypy` need no writes and work read-only.

Exporting `PYTHONDONTWRITEBYTECODE=1` (or `NODE_OPTIONS` equivalents) prevents the subtler
variant: `__pycache__` directories appearing in the tree and showing up as untracked files.

### Rebuilding often? Add a test image

The in-container `pip install` re-runs on every invocation. For a suite you run frequently:

```dockerfile
FROM <name>:dev
COPY requirements-dev.txt /tmp/
RUN pip install -q -r /tmp/requirements-dev.txt
```

`docker build -t <name>:test -f Dockerfile.test .` once, then `docker run --rm -v "$PWD:/app:ro" -w /app <name>:test pytest -q`.

### Exit codes

`docker run` returns the container's exit code, so `&&`/`set -e` chains work — but do not pipe
the output through `tee`/`tail` without `set -o pipefail`, or a failing suite looks like a
success. And read the *first* failure, not the last line: pytest prints the short summary last,
which is rarely the cause.

## 2. Stack hygiene before `up`

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'
docker compose ls
```

```powershell
docker ps --format "{{.Names}}`t{{.Image}}`t{{.Ports}}"
docker compose ls
```

- **Give every stack an explicit project name** (`COMPOSE_PROJECT_NAME`, or the top-level
  `name:` in the compose file). Compose otherwise derives it from the directory name, so
  every `/opt/*/dev` directory on a host shares the project `dev` — and
  `docker compose down --remove-orphans` in one stack removes the other's containers.
- **Volumes are named `<project>_<volume>`.** Changing the project name gives the new stack a
  fresh empty database. Either accept the reseed, or copy the old volume deliberately; do not
  pre-create a volume and expect compose to adopt it (label checks differ by compose version).
- **Pick ports outside the used set**, or publish none and reach the service by container name.
  New stacks generally should not publish their database port at all.
- Migrating an existing stack: remove legacy containers **by exact name** after checking their
  `com.docker.compose.project` and `com.docker.compose.project.working_dir` labels — never by
  project name alone, for the reason above.
- Ports belong in **one** place. The listener (`PORT`), the `ports:` mapping and the
  `HEALTHCHECK` must all read the same variable:

```yaml
services:
  app:
    environment:
      PORT: ${PORT:-8090}
    ports:
      - "${PORT:-8090}:${PORT:-8090}"
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:${PORT:-8090}/health || exit 1"]
```

A healthcheck with a hardcoded port keeps probing the old number after the port moves, so a
healthy container is reported unhealthy and `restart: unless-stopped` recycles it forever.

## 3. Verify

```bash
docker compose config            # resolved values: ports, env, volumes, project name
docker compose up -d
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
docker inspect -f '{{.State.Health.Status}}' <container>    # healthy, not "starting"
docker compose logs --tail=50 <service>
```

`config` is the cheap check that catches variable expansion mistakes before anything starts;
`inspect` is the only trustworthy source for health because `docker ps` shows `(healthy)` only
after the first successful probe.

## Pitfalls

- **Docker Desktop must be running** before any of this on Windows/macOS; the CLI failing with
  "cannot connect to the Docker daemon" is usually that, not a broken install.
- **WSL2 path performance.** Code under `/mnt/c/...` is slow enough to dominate a test suite.
  Keep repos inside the WSL filesystem (`~/repos`) and use Docker Desktop's from-WSL
  integration, or accept minutes instead of seconds.
- **Bind-mount paths differ by shell**: `-v "$PWD/x:/app"` in bash/zsh,
  `-v "${PWD}\x:/app"` in PowerShell, `-v "%CD%\x:/app"` in cmd. A path with spaces needs
  quoting around the whole `src:dst` argument.
- **`:ro` is not enforced the same everywhere.** On some Docker Desktop setups (and with
  remote contexts) a read-only bind can still be written through; verify with a deliberate
  `touch` inside the container if cleanliness matters.
- **`docker compose down -v` deletes named volumes** — including the database you just spent an
  hour seeding. Prefer `down` without `-v` unless you mean it.
- **Images pile up.** After a few rebuilds: `docker image prune -f`, and `docker system df` to
  see where the space went. Never `docker system prune -a` on a host running other stacks.
- **`depends_on` does not wait for readiness**, only for start. A service that connects in its
  entrypoint needs a retry loop or a `condition: service_healthy` dependency, or it fails on a
  cold start and works on the second try — the classic "works when I re-run it".
- Container CPU/memory limits hide in `docker stats`; a suite that times out in CI but passes
  locally can be a memory ceiling, not a slow test.
