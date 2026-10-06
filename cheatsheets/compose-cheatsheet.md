# Cheatsheet — Docker Compose

| Problem | Command |
|---|---|
| Start the stack (detached) | `docker compose up -d` |
| Rebuild changed images first | `docker compose up -d --build` |
| Status + health | `docker compose ps` |
| Follow one service's logs | `docker compose logs -f api` |
| Shell into a service | `docker compose exec db psql -U app` |
| Run a one-off command | `docker compose run --rm api <cmd>` |
| Include profile-gated services | `docker compose --profile heavy up` |
| Use an override file explicitly | `docker compose -f compose.yaml -f compose.prod.yaml up` |
| See the final merged config | `docker compose config` |
| Stop + remove (keep volumes) | `docker compose down` |
| Stop + remove **incl. volumes** | `docker compose down -v` ⚠️ |
| Scale a service | `docker compose up -d --scale worker=3` |

**Readiness (the #1 mistake):** `depends_on` is **start order only**. For real readiness:
```yaml
depends_on: { db: { condition: service_healthy } }
db: { healthcheck: { test: ["CMD-SHELL","pg_isready -U app"], interval: 5s, retries: 10 } }
```

**Local dev vs the override:** base `compose.yaml` = dev (bind mounts, debug); `*.production.yml` drops
those + swaps images/env (retrieva does this). Service **name = DNS name** on the project network; only
publish the edge (frontend), keep `db`/`api` internal.

**Here, Compose = local dev / legacy-sim only** (retrieva, globalcore-legacy). Production is k3s, not
`compose up`.

See also → `../06-compose/docker-compose.md`
