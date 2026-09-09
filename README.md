# omni — Docker wrapper for OmniRoute

Minimal git project to run [OmniRoute](https://github.com/diegosouzapw/OmniRoute) (`diegosouzapw/omniroute`) as a Docker container and deploy it to a VPS via GitHub Actions using [`isavage/deploy`](https://github.com/marketplace/actions/zero-config-vps-docker-deploy) (`isavage/deploy@v3`).

Derived from upstream [Docker Guide `v3.8.51`](https://github.com/diegosouzapw/OmniRoute/blob/release/v3.8.51/docs/guides/DOCKER_GUIDE.md).

## Stack

- `docker-compose.yml:6` — `omniroute` (prebuilt `diegosouzapw/omniroute:${OMNIROUTE_IMAGE_TAG:-latest}`) + `redis` sidecar (`redis:8.6.5-alpine`) with healthchecks, `restart: unless-stopped`, `stop_grace_period: 40s`, persistent named volumes `omniroute-data` / `omniroute-redis-data`.
- `.env.example:1` — copy to `.env` and fill secrets.
- `.github/workflows/deploy.yml:1` — deploy on push to `main`/`master` or manual `workflow_dispatch` via SSH + `docker compose up -d` with health polling.

## Quick start (local)

```bash
cp .env.example .env
# edit JWT_SECRET, API_KEY_SECRET, INITIAL_PASSWORD, OMNIROUTE_WS_BRIDGE_SECRET
# generate: openssl rand -base64 48 ; openssl rand -hex 32 ; openssl rand -base64 32
docker compose up -d
docker compose logs -f
open http://localhost:20128
```

Production coding-agent workloads need more RAM than the `1024` default — see [Runtime RAM](#runtime-ram).

## VPS deploy (GHA)

### 1. Prepare VPS

- Ubuntu/Debian with Docker + Compose plugin, SSH key auth.
- User in `docker` group or `root` (action handles `sudo`).

### 2. GitHub Secrets

Settings → Secrets and variables → Actions (**or** Environment `production` if you uncomment `environment: production` in `.github/workflows/deploy.yml:17`):

| Secret | Required | Description |
|---|---|---|
| `VPS_HOST` | yes | Hostname/IP |
| `VPS_USER` | yes | SSH user (`root` or sudo-enabled) |
| `VPS_SSH_KEY` | yes | Private key matching `~/.ssh/authorized_keys` on VPS |
| `DOPPLER_TOKEN` | no | Doppler service token `dp.st...` — if used, app reads secrets via `doppler run -- docker compose up -d` |
| `ENV_FILE` | no | Alternative: full `.env` contents as a secret, passed as `env_content` |

Only set `environment: production` on the job when secrets live in that Environment; otherwise omit it.

### 3. Environment on VPS

The action syncs the repo to `target_dir` (default `/docker/<repo-name>`) via `rsync --delete` but **never deletes `.env` / `.env.*`** (auto-protected). Three modes:

- **Doppler** — set `doppler_token` input, reference vars in compose `environment:`.
- **GitHub `ENV_FILE`** — set `env_content: ${{ secrets.ENV_FILE }}` and file is written as `600` on VPS.
- **Persistent VPS `.env`** — `ssh` in, create `/docker/omni/.env` manually; it survives deploys.

Push to `main` or Run workflow → `workflow_dispatch` (`no_cache`/`cleanup` toggles) to deploy. The action polls `docker inspect` health and streams `docker compose logs --tail=100` on failure.

### 4. Manual VPS run

```bash
# on VPS
mkdir -p /docker/omni && cd /docker/omni
# copy docker-compose.yml + .env
docker compose pull
docker compose up -d
```

## Configuration

- **Image tags** — `latest` = highest published stable semver; `next` = current `release/v*` pre-release; `3.8.51` = pinned. Pin for GitOps: `OMNIROUTE_IMAGE_TAG=3.8.51`. See Docker Guide → Image Tags.
- **Ports** — `PORT=20128` → `${OMNIROUTE_BIND_HOST:-127.0.0.1}:${PORT}:20128`. Set `OMNIROUTE_BIND_HOST=0.0.0.0` to expose, or front with Caddy/Traefik.
- **Subpath** — `OMNIROUTE_BASE_PATH=/omniroute` + `NEXT_PUBLIC_BASE_URL=https://host/omniroute` (rebuild if building locally; prebuilt root image is patched at startup).
- **HTTPS** — uncomment Caddy profile in `docker-compose.yml:55` or front externally:

  ```yaml
  caddy:
    image: caddy:latest
    ports: ["80:80","443:443"]
    command: caddy reverse-proxy --from https://your-domain.com --to http://omniroute:20128
  ```

### Runtime RAM

Default `OMNIROUTE_MEMORY_MB=1024` is only for dashboard/light chat. Per guide:

| Workload | `OMNIROUTE_MEMORY_MB` | cgroup `--memory` |
|---|---|---|
| One coding agent | `8192` | `≥10g` |
| Two concurrent long `/v1/responses` | `10240–12288` | `12–16g` |

Set `OMNIROUTE_MEMORY_MB` in `.env` and uncomment `deploy.resources.limits.memory` in compose.

### CLI tools when OmniRoute runs in Docker

`setup-codex`/`setup-claude` writes `~/.codex/*.config.toml` — inside the container that's ephemeral (`/home/node`). Preferred: run `omniroute` CLI on host and `omniroute connect http://localhost:20128`. To let the container write host config, use the upstream `host` profile bind-mounts; avoid `cli` profile's `docker.sock` on public VPS (see guide security warning).

## Useful commands

```bash
docker compose ps
docker compose logs -f omniroute
docker compose exec redis redis-cli ping
docker compose down          # keep volumes
docker compose down -v       # also delete data
docker compose pull && docker compose up -d  # update to latest tag
```

## References

- https://github.com/diegosouzapw/OmniRoute/blob/release/v3.8.51/docs/guides/DOCKER_GUIDE.md
- https://github.com/marketplace/actions/zero-config-vps-docker-deploy
- https://github.com/isavage/deploy
