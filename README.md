# bpdash

Docker Compose stack for the bpdash app. Three services: **db**, **backend**, **frontend**.

## Services

| Container  | Hostname   | Image                  | Port (host -> container) |
|------------|------------|------------------------|-------------------------|
| `db`       | `db`       | `postgres:16-alpine`   | internal (5432)    |
| `backend`  | `backend`  | `blackpearlfsae/bpdash-backend`      | internal (3000)    |
| `frontend` | `frontend` | `blackpearlfsae/bpdash-frontend`     | `80:80` (HTTP) — HTTPS disabled for now, see TLS section |

Hostnames are the Compose service names — services reach each other at `http://backend:3000`, `db:5432`, etc. on the default Compose network.

---

## Development setup

After cloning this repo, the `frontend/` and `backend/` directories need to be populated from their own repos for CD:

```bash
git clone https://github.com/BlackPearlFSAE/BP_dashboard_FE.git frontend
git clone https://github.com/BlackPearlFSAE/BlackPearl_WS.git backend
```

Then bring the stack up locally<br>
(builds from the `frontend/` and `backend/` Dockerfiles):

```bash
docker compose up --build
```

Frontend will be available at <http://localhost> (port 80).

### Environment variables

Optional overrides (defaults in parentheses):

- `POSTGRES_USER` (`bpdash`)
- `POSTGRES_PASSWORD` (`bpdash`)
- `POSTGRES_DB` (`bpdash`)
- `PUBLISH_INTERVAL` (`200`)
- `FRONTEND_URL` (`http://localhost`)
- `FRONTEND_DEPLOY_URL` (empty)

Put them in a `.env` file next to `compose.yml`.

---

## Deployment procedure

### First deploy on a VPS

Prereqs on the server:
- Docker engine + Docker Compose plugin installed (`docker --version`, `docker compose version`)
- Domainname pointing to VPS public ip (optional for non HTTPS setup )
  - Recommend DuckDNS as it's free, but it gives subdomain.
- Ports 80 (and later 443) open in the firewall rule (typically on , but make sure to check)

Steps:
```bash
# 1. Clone the stack + its two submodule repos
git clone <this-repo-url> bpdash && cd bpdash
git clone https://github.com/BlackPearlFSAE/BP_dashboard_FE.git frontend
git clone https://github.com/BlackPearlFSAE/BlackPearl_WS.git backend

# 2. Create the root .env (overrides defaults in compose.yml)
#    At minimum set POSTGRES_PASSWORD to something non-default.
cp .env.example .env   # if an example exists; otherwise create it manually
nano .env

# 3. Build and start in the background
docker compose up -d --build

# 4. Verify
docker compose ps                    # all services should be "healthy" / "running"
docker compose logs -f backend       # tail backend logs, Ctrl-C to exit
curl http://localhost/               # frontend should respond
curl http://localhost/api/           # backend proxied through nginx
```

### Updating a running deployment

**Case 1. You push new images (from your dev machine):**
```bash
docker compose build
docker compose push
```
Make sure you're logged into the Docker CLI for the target registry first.

**Case 2. You pull on the server:**
```bash
docker compose pull && docker compose up -d
```

### Day-to-day ops

```bash
docker compose ps                    # status
docker compose logs -f <service>     # tail logs (backend, frontend, db)
docker compose restart <service>     # restart one service
docker compose down                  # stop all (data volume preserved)
docker compose down -v               # ⚠ also wipes db_data — destroys the DB
```
---

## HTTPS

Currently disabled — the stack serves plain HTTP on port 80. When you're ready to enable TLS, see [docs/tls-auto-renewal.md](docs/tls-auto-renewal.md).

---

## Database persistence

Postgres data lives in the named volume `db_data` (see [compose.yml](compose.yml)). It survives `docker compose down` but **not** `docker compose down -v`. There is no automated backup — if you care about the data, run a periodic `pg_dump`:

```bash
docker compose exec db pg_dump -U bpdash bpdash > backup-$(date +%F).sql
```

Restore with:

```bash
cat backup-YYYY-MM-DD.sql | docker compose exec -T db psql -U bpdash bpdash
```