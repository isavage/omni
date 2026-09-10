# omni — Docker wrapper for OmniRoute

Minimal wrapper to run [OmniRoute](https://github.com/diegosouzapw/OmniRoute) (`diegosouzapw/omniroute`) via Docker Compose and deploy to a VPS with [`isavage/deploy@v3`](https://github.com/marketplace/actions/zero-config-vps-docker-deploy).

Derived from [Docker Guide `v3.8.51`](https://github.com/diegosouzapw/OmniRoute/blob/release/v3.8.51/docs/guides/DOCKER_GUIDE.md).

## Stack

- `docker-compose.yml` — `omniroute` (`diegosouzapw/omniroute:${OMNIROUTE_IMAGE_TAG:-latest}`) + `redis:8.6.5-alpine` on `omni.net` bridge, `restart: unless-stopped`, `stop_grace_period: 40s`, healthchecks, `omniroute-data` / `omniroute-redis-data` volumes. Secrets `JWT_SECRET`/`API_KEY_SECRET`/`STORAGE_ENCRYPTION_KEY` are mapped explicitly in `environment:` and also loaded via `env_file: .env`.
- `.env.example` — copy to `.env` and fill secrets.
- `.github/workflows/deploy.yml` — `push` to `main`/`master` or `workflow_dispatch` (`no_cache`/`cleanup`) via `isavage/deploy@v3` (`VPS_HOST`/`VPS_USER`/`VPS_SSH_KEY`).

## Quick start

```bash
cp .env.example .env
# generate secrets
openssl rand -base64 48  # JWT_SECRET
openssl rand -hex 32     # API_KEY_SECRET
openssl rand -hex 32     # STORAGE_ENCRYPTION_KEY (64 hex chars, leave empty to disable)
openssl rand -base64 32  # OMNIROUTE_WS_BRIDGE_SECRET
# edit .env: set above + INITIAL_PASSWORD
docker compose up -d
docker compose logs -f
open http://localhost:20128
```

`STORAGE_ENCRYPTION_KEY` enables SQLite at-rest encryption. `STORAGE_ENCRYPTION_KEY_VERSION=v1` — bump to `v2` on rotation. If you lose the key, the DB is unrecoverable.

`OMNIROUTE_WS_BRIDGE_SECRET` is the shared secret for the internal Codex Responses WS bridge (`src/app/api/internal/codex-responses-ws`). Required in production; bridge returns `401` if unset.

## VPS deploy (GHA)

### 1. Prepare VPS

Ubuntu/Debian with Docker + Compose plugin, SSH key auth, user in `docker` group or `root`.

### 2. GitHub Secrets

Settings → Secrets and variables → Actions:

| Secret | Required | Description |
|---|---|---|
| `VPS_HOST` | yes | Hostname/IP |
| `VPS_USER` | yes | SSH user |
| `VPS_SSH_KEY` | yes | Private key for `~/.ssh/authorized_keys` |
| `DOPPLER_TOKEN` | no | Doppler `dp.st...` (alternative to `.env`) |
| `ENV_FILE` | no | Full `.env` as secret |

### 3. Environment on VPS

Action syncs to `/docker/<repo-name>` via `rsync --delete` but never deletes `.env`/`.env.*`. Use one of: Doppler (`doppler_token`), GitHub `ENV_FILE` (`env_content`), or persistent VPS `.env` (`ssh` + create `/docker/omni/.env`).

Push to `main` or Run workflow (`workflow_dispatch`) to deploy. Polls `docker inspect` and logs `docker compose logs --tail=100` on failure.

### 4. Manual run

```bash
mkdir -p /docker/omni && cd /docker/omni
# copy docker-compose.yml + .env
docker compose pull && docker compose up -d
```

## Configuration

- **Image tags** — `latest` = stable, `next` = `release/v*` pre-release, `3.8.51` = pinned. Pin for GitOps: `OMNIROUTE_IMAGE_TAG=3.8.51`.
- **Ports** — `PORT=20128` → `${OMNIROUTE_BIND_HOST:-127.0.0.1}:${PORT}:20128` (loopback by default, proxy forwards to it).
- **Subdomain vs subpath**
  - Subdomain `https://omni.example.com` (recommended): `OMNIROUTE_BASE_PATH=` (empty), `NEXT_PUBLIC_BASE_URL=https://omni.example.com`, `AUTH_COOKIE_SECURE=true`, no compose change.
  - Subpath `https://example.com/omniroute`: `OMNIROUTE_BASE_PATH=/omniroute`, `NEXT_PUBLIC_BASE_URL=https://example.com/omniroute` (prebuilt image patches at startup).
- **Reverse proxy** — host nginx/Caddy: `proxy_pass http://127.0.0.1:20128;` / `reverse_proxy 127.0.0.1:20128`. Or add Caddy in compose:
  ```yaml
  caddy:
    image: caddy:latest
    ports: ["80:80","443:443"]
    command: caddy reverse-proxy --from https://omni.example.com --to http://omniroute:20128
  ```

### Runtime RAM

`OMNIROUTE_MEMORY_MB=1024` is only for dashboard/light chat.

| Workload | `OMNIROUTE_MEMORY_MB` | cgroup `--memory` |
|---|---|---|
| One coding agent | `8192` | `≥10g` |
| Two concurrent long `/v1/responses` | `10240–12288` | `12–16g` |

## Useful commands

```bash
docker compose ps
docker compose logs -f omniroute
docker compose exec redis redis-cli ping
docker compose down        # keep volumes
docker compose down -v     # delete data
docker compose pull && docker compose up -d
```

## References

- https://github.com/diegosouzapw/OmniRoute/blob/release/v3.8.51/docs/guides/DOCKER_GUIDE.md
- https://github.com/marketplace/actions/zero-config-vps-docker-deploy
- https://github.com/isavage/deploy
