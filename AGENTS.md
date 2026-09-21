# AGENTS.md

## Model routing

- You own planning, architecture, and final verification. Don't write bulk code yourself.
- Break work into independent, clearly scoped tasks with success criteria and hand each to a subagent.
- Review subagent output before reporting done.

## Project purpose

This repository runs a small Dockerized FastAPI service that receives MikroTik
RouterOS `.rsc` exports, removes the generated export header, and commits and
pushes changed configurations to a Git remote. Treat router configuration
files as sensitive infrastructure data.

## Repository map

- `server/app.py` — FastAPI endpoints, bearer-token authentication, request
  handling, RouterOS header normalization, and graceful shutdown.
- `server/git_ops.py` — repository initialization, Git command execution,
  commit/push behavior, timeouts, and credential redaction.
- `server/requirements.txt` — pinned Python dependencies.
- `server/Dockerfile` — production image containing Python and Git.
- `docker-compose.yml` — service configuration, environment variables, health
  check, port mapping, and the persistent `./repo_data` mount.
- `router-scripts/mikrotik-backup.rsc` — RouterOS health check, sensitive export,
  upload, and temporary-file cleanup script. Treat it as sensitive integration
  code.
- `README.md` — operator setup and configuration documentation.

## Architecture and data flow

- On startup, `server/app.py` validates `ROUTER_AUTH_TOKEN` and calls
  `initialise_repo()` to clone or initialize `/data/repo`.
- `GET /health` is the liveness endpoint used by the Compose healthcheck.
- `POST /backup/config` requires a bearer token, accepts the raw RSC body, reads
  `X-Router-Name`, removes the changing leading RouterOS comment header, and
  writes `<router_name>.rsc` into `/data/repo`.
- Blocking Git work runs through `asyncio.to_thread` under one lock. A changed
  file is staged, committed, and pushed; an unchanged upload returns HTTP 204,
  while a committed upload returns HTTP 200 with `{"committed": true}`.

## Working agreement

- Plan changes before editing and keep each change narrowly scoped.
- For larger or independent changes, split the work into clearly scoped tasks
  with explicit success criteria and review the resulting changes before
  reporting completion.
- Preserve unrelated user changes in the worktree. Do not reset, checkout,
  or overwrite files unrelated to the request.
- Update `README.md` when behavior, configuration, deployment, or operator
  steps change.

## Development and verification

Run commands from the repository root unless a command specifies another
directory.

```powershell
# Check the working tree before and after a change
git status --short

# Validate Python syntax without requiring third-party packages
python -m compileall -q server

# Validate the Compose file when Docker is installed
docker compose config

# Build/start the service for an integration check (requires Docker)
docker compose up --build -d

# Check liveness and inspect service logs
Invoke-WebRequest http://localhost:8080/health
docker compose logs --tail 100 backup-server

# Stop the local integration environment
docker compose down
```

Configure the `environment` block in `docker-compose.yml`. The required values
are `ROUTER_AUTH_TOKEN`, `GIT_REPO_URL`, and `GIT_PAT`; optional values include
`GIT_BRANCH`, Git author identity, `COMMIT_MESSAGE_FORMAT`, `LISTEN_PORT`,
`GIT_TIMEOUT`, and `SHUTDOWN_TIMEOUT`. If `LISTEN_PORT` changes, update the
Compose `ports` mapping too. Replace `serverUrl` and `authToken` in the
RouterOS script, test it with `/system script run git-backup`, then schedule it
under RouterOS System → Scheduler.

There is currently no automated test, lint, or type-check configuration. For
changes to request handling or Git behavior, at minimum run the syntax check
and manually verify `/health`, authentication failures, empty-body rejection,
and the changed-success path when a safe test repository is available. Do not
use a production router or production backup repository for testing.

When changing Docker or dependency behavior, run `docker compose config` and
an image build if Docker is available. If a required tool is unavailable,
report that limitation rather than claiming the check passed.

## Code and configuration conventions

- Follow the existing Python style: standard-library imports first, typed
  helper signatures where practical, small functions, and clear logging.
- Keep blocking Git/network work off the FastAPI event loop and preserve the
  existing lock around repository mutations.
- Preserve the existing Git timeout, non-interactive credential behavior,
  PAT redaction, and graceful shutdown semantics unless the change explicitly
  addresses one of those concerns.
- Keep dependency versions pinned in `server/requirements.txt`.
- Keep container/runtime configuration in `docker-compose.yml` and document
  new environment variables in `README.md`.
- Keep RouterOS syntax compatible with the script's supported RouterOS
  environment; manually review any change that uses `show-sensitive` or sends
  request headers/body data.

## Security and operational safety

- Never commit `GIT_PAT`, `ROUTER_AUTH_TOKEN`, real router exports, or other
  credentials. Use local environment overrides or untracked files for tests.
- Do not weaken bearer-token comparison, Git command timeouts, credential
  redaction, or the HTTPS-only PAT injection path without documenting and
  reviewing the security impact.
- Treat `router-scripts/mikrotik-backup.rsc` as secret-bearing code: it exports
  sensitive settings and must not log or expose the token or uploaded content.
- Do not change the default port, volume, branch, or environment contract
  casually; check the README and Compose health check together.
- Avoid destructive Git or filesystem operations. The `repo_data` directory
  is persistent runtime state; inspect it before changing or removing it.
