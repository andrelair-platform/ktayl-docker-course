# Module 6 — Docker Compose (local development stacks)

> Level 6 · Learning note (Type A). Compose = "declare a multi-container app in one YAML, bring it up with
> one command." It's where you'll run the dependency chain *locally*; Kubernetes is its production cousin.

## Concept
A `compose.yaml` declares **services** (containers), the **network** they share, the **volumes** they
persist to, and their **env/healthchecks/dependencies**. `docker compose up` creates it all; `down`
tears it down. One file, reproducible local environment.

## Problem it solves
Running "frontend + API + Postgres + Redis" by hand is 4 `docker run`s, a hand-made network, and
remembering the flags. Compose makes the **whole local stack declarative and shareable** — a new dev runs
`docker compose up` and has the same environment.

## How it works internally
- Compose creates a **project** (named by the dir) with its own **user-defined bridge network** → services
  reach each other by **service name** (DNS, Module 4).
- `depends_on` controls **start order only**, not readiness — "started" ≠ "ready to accept connections".
  Use `depends_on: { x: { condition: service_healthy } }` + a `healthcheck` for real readiness.
- `volumes:` (named) persist data; `./src:/app` (bind) gives hot-reload; `profiles:` gate optional services.
- `environment`/`env_file` inject config; `ports:` publish to the host (only what you need).

## Example (the classic chain — Lab L9)
```yaml
services:
  frontend:
    build: ./web
    ports: ["3000:3000"]
    depends_on: { api: { condition: service_healthy } }
  api:
    build: ./api
    environment: { DATABASE_URL: postgres://app:app@db:5432/app }
    healthcheck:
      test: ["CMD","curl","-f","http://localhost:8000/health"]
      interval: 5s
    depends_on: { db: { condition: service_healthy } }
  db:
    image: postgres:16
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app }
    volumes: ["pgdata:/var/lib/postgresql/data"]      # persist
    healthcheck: { test: ["CMD-SHELL","pg_isready -U app"], interval: 5s }
volumes: { pgdata: {} }
```
`docker compose up` → `db` becomes healthy → `api` starts → becomes healthy → `frontend` starts. Only
`frontend:3000` is on the host; `api`/`db` are internal.

## Example from our organisation
- **`retrieva/docker-compose.yml` + `docker-compose.production.yml`** — the RNCP product's local stack
  (backend + frontend + datastores). The `*.production.yml` override shows the dev-vs-prod split in
  Compose (different images/env/replication intent) before any of it reaches k3s.
- **`globalcore-legacy/docker-compose.yml` (+ `.oracle.yml`)** — the simulated **Java/SOAP/MySQL** legacy
  core. This is the perfect Compose teaching specimen: a legacy app you *run* locally to develop the
  **ACL** (`ktayl-claims`) against, with an alternate `oracle` compose for the other DB backend. It shows
  Compose's real job: **stand up a dependency you don't own** so you can build against it.
- **Why Compose is NOT how we deploy**: the platform deploys via Helm→ArgoCD→k3s, not `compose up`.
  Compose here is strictly **local dev / legacy simulation**. Knowing where Compose stops and Kubernetes
  starts is the point of Module 12.

## Architecture
```
docker compose up
   project bridge network ("retrieva_default")
   ┌──────────┐  name DNS  ┌────────┐  name DNS  ┌──────────┐
   │ frontend │ ─────────▶ │  api   │ ─────────▶ │   db     │
   │ :3000→host│           │(health)│            │ pgdata vol│
   └──────────┘            └────────┘            └──────────┘
        (published)        (internal)            (internal + volume)
```

## Commands
```bash
docker compose up -d                 # start the stack (detached)
docker compose up --build            # rebuild changed images first
docker compose ps                    # service status + health
docker compose logs -f api           # follow one service
docker compose exec db psql -U app   # shell into a service
docker compose --profile heavy up    # include profile-gated services
docker compose down -v               # stop + remove (incl. volumes!)
docker compose config                # render the final merged config (base + override)
```

## Debugging
- **"api can't reach db"** → wrong service name in `DATABASE_URL`, or `db` not healthy yet (add
  `condition: service_healthy`). (Module 4 network checks apply.)
- **"it starts then dies"** → `depends_on` ordered it but the dep wasn't *ready*; add a healthcheck.
- **"my code change isn't picked up"** → missing bind mount, or you rebuilt the image but didn't
  `--build`. Dev uses bind mounts + hot reload; prod uses the immutable image.
- **`down -v` ate my data** → `-v` removes named volumes too; omit it to keep `pgdata`.

## Production considerations
Compose is fine for a single box / local dev / CI integration tests. For HA, scheduling, self-heal,
rolling updates, secrets, and policy → **Kubernetes** (we use k3s). Don't run the insurance platform on
`compose up`; do use it to iterate on one service + its deps fast.

## Security considerations
Don't publish datastores (`ports:` only the edge); keep secrets in an `env_file` that's git-ignored (or
Docker secrets) — never commit `POSTGRES_PASSWORD`; the production override should drop dev conveniences
(debug ports, bind mounts). These map to the K8s controls (ESO→Vault, netpol) later.

## Common mistakes
Relying on `depends_on` for readiness (it's order-only) · publishing the DB · `down -v` on data you care
about · treating Compose as the production platform · committing secrets in the compose file.

## Key takeaways
1. Compose = declarative multi-service **local** stack; service name = DNS name.
2. `depends_on` is **start order**, not readiness — pair with `healthcheck` + `condition: service_healthy`.
3. Named **volumes** persist; **bind mounts** give dev hot-reload; publish only the edge.
4. Override files (`*.production.yml`) model dev-vs-prod — the idea Kubernetes overlays formalise.
5. Compose is for dev/legacy-sim here; **k3s is production** (Module 12).

## Exercise (on our project)
1. `docker compose config` on `globalcore-legacy` — list its services, which are published, and which the
   ACL (`ktayl-claims`) talks to. Why run the legacy locally at all?
2. Compare `retrieva/docker-compose.yml` with `docker-compose.production.yml` — what changes dev→prod, and
   which of those changes become Helm values in k3s?
3. Lab L9: build the frontend+api+db stack above; break it (wrong service name), diagnose with
   `compose logs`/`exec`, fix it.

## See also
- Networking → `04-networking/…` · Storage → `05-storage/…` · K8s mapping → `_meta/docker-to-kubernetes-map.md`
- Case → `case-studies/globalcore-legacy-compose.md` (todo) · Fast → `cheatsheets/compose-cheatsheet.md` (todo)
