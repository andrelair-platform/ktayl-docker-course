# Docker usage inventory — the ktayl/minicloud laboratory

Every place the organisation uses Docker/OCI containers. This is the **map of specimens** the course
teaches from. (Compiled by scanning the repo tree — 25 Dockerfiles, 4 compose files, 9 `.dockerignore`,
+ the Kubernetes/CI/registry usage.) Columns: what the Docker usage is · the image / base · registry ·
runtime · *why* Docker is used here.

## A. Application images we BUILD (own Dockerfiles)

| Service | Stack | Dockerfile pattern | Base (build → runtime) | Registry | Runtime | Why Docker |
|---|---|---|---|---|---|---|
| **platform-demo** | Go | multi-stage + BuildKit cache mounts | `golang:1.25-alpine` → `gcr.io/distroless/static-debian12:nonroot` | ghcr (public) + Harbor | k3s Deploy + KEDA | ship a ~9 MiB static binary; the gold-standard tiny/secure image |
| **ktayl-core** | Java 21 / Spring Boot + Spring Modulith | multi-stage Maven | `maven:3.9-eclipse-temurin-21` → `eclipse-temurin:21-jre-jammy` (non-root 10001) | ghcr + Harbor | k3s (BILL-016) · Kargo **git**-Warehouse | the insurance-LOB modular monolith (Billing) |
| **ktayl-claims** | Java 21 / Spring Boot (ACL) | multi-stage Maven | `maven:3.9-eclipse-temurin-21` → `eclipse-temurin:21-jre` (non-root 10001) | ghcr + Harbor | k3s + Debezium CDC | the legacy ACL/Strangler over globalcore |
| **ktayl-policy-service** | Go/gRPC (PAS) | multi-stage | Go → distroless-ish | ghcr + Harbor | k3s · Kargo git-Warehouse | the core policy registry |
| **ktayl-underwriting** | Python 3.12 / FastAPI | multi-stage venv | `python:3.12-slim` (build venv) → `python:3.12-slim` (non-root 10001) | ghcr + Harbor | k3s · Kargo git-Warehouse | pricing math is Python-native |
| **ktayl-iam** (backend) | NestJS (Node 22) | multi-stage | `node:22-alpine` | ghcr + Harbor | k3s · Kargo git-Warehouse | custom IGA |
| **ktayl-iam** (frontend) | Next.js (Node 22) | multi-stage **standalone** | `node:22-alpine` → `node:22-alpine` (`USER node`) | ghcr + Harbor | k3s | the IAM console (BFF) |
| **ktayl-underwriting/frontend**, **ktayl-claims/frontend**, **retrieva/frontend** | Next.js | multi-stage standalone | node-alpine | ghcr + Harbor | k3s | the workbenches |
| **retrieva-backend** | Express 5 / Node 20 | multi-stage | node | ghcr + Harbor | k3s · Kargo **git**-Warehouse (2 images) | the RNCP cert product |
| **minicloud-agent** / **minicloud-crew-agent** | Python (LangGraph / CrewAI) | multi-stage | python-slim | ghcr (Internal) + Harbor | k3s · Kargo image-Warehouse | AI research agents |
| **minicloud-plane** | Go | multi-stage | Go | ghcr + Harbor | k3s Rollout | Plane webhook→NATS→REST |
| **minicloud-rag-ingest**, **minicloud-rag-eval**, **minicloud-markitdown-proxy**, **minicloud-litellm-custom** | Python | multi-stage | python | Harbor | k3s (ai ns) | the RAG/AI pipeline |
| **minicloud-backstage** | Node/TS (Backstage) | custom image (CA cert) | node | Harbor | k3s (single prod) | the developer portal |
| **minicloud-open-webui**, **minicloud-onlyoffice**, **minicloud-erpnext** | custom (vendor + CA/config) | derive from vendor image | vendor | Harbor | k3s | workplace apps that need a CA/config bake |
| **ktayl-solution-web** | Astro | build → static | — (GitHub Pages, **off-cluster**) | — | GitHub Pages | public site; deliberately NOT containerised on-cluster |
| **ktayl-itsm** | GLPI (PHP) | custom image (GLPI + CA) | glpi | Harbor | k3s (itsm ns) | GLPI ITSM; setcap/no_new_privs gotcha |

## B. Custom BASE images we build (platform-level)

| Image | Why it exists | Base | Where used |
|---|---|---|---|
| **minicloud-postgresql-noavx512** | the Bitnami pgvector build uses AVX-512 (EVEX) → **SIGILL on the old ThinkPad CPUs**; recompiled pgvector `-mno-avx512*` | `harbor.../library/postgresql:18.4.0` (itself Harbor-hosted) | every CNPG Postgres on the cluster — the canonical *"why a custom base image"* case study |

## C. Third-party images we RUN (vendor charts — we don't build these)

StatefulSets / Deployments pulled from upstream registries (Grafana, Prometheus, Vault, Harbor itself,
ERPNext, Redpanda/ClickHouse (Langfuse), NATS, cert-manager, ESO, Cilium, Longhorn, MinIO, Authentik,
Stalwart, Nextcloud, Matrix, Jitsi, KEDA, Kargo, ArgoCD…). → teach *running/pinning/scanning* images we
don't own + the MinIO-paywall / digest-pinning gotcha.

## D. Kubernetes container PATTERNS in use (all are "Docker concepts, grown up")

| Pattern | Real example | Teaches |
|---|---|---|
| **Deployment** (stateless) | platform-demo, ktayl-iam, underwriting | `docker run` → replicated Pods |
| **StatefulSet** (stateful) | CNPG Postgres (per service), Vault, Harbor, Longhorn, ClickHouse | why a container ≠ storage; PVC |
| **Job** | the `UNDERWRITING_EVENTS` + `HR_LIFECYCLE` stream-init (nats-box) | one-shot containers |
| **CronJob** | kine-backup, vault-snapshot, db-backup, Harbor retention/GC | scheduled containers |
| **initContainer** | CA-trust (nextcloud, bookstack); ktayl-iam ghcr-pull prep | ordered pre-start containers |
| **sidecar** | Vault agent injector; cloudflared | one Pod, many containers |
| **readOnlyRootFilesystem + emptyDir** | crew-agent (`~/.cache`,`/tmp`) | the writable-layer lesson, enforced |

## E. Registries (the hybrid model)

| Registry | Role | Tag scheme | Retention |
|---|---|---|---|
| **Harbor** `harbor.10.0.0.200.nip.io/library/*` | **dev** images (+ the custom postgresql base) | `:<sha>` | keep-3 + daily GC; `postgresql*` always-retain |
| **ghcr.io/andrelair-platform/*** | **prod** images (durable, off local disk) | `:<sha>` (+ `vX.Y.Z` on release) | keep-5 (cosign-aware) |

Prod images are **cosign keyless-signed + carry a CycloneDX SBOM + pass a Trivy CRITICAL gate** in CI.

## F. Build & delivery toolchain

`docker buildx` / BuildKit (CI) · GitHub Actions dual-push (Harbor + ghcr) · **cosign** (keyless) ·
**syft** (SBOM) · **Trivy** (CRITICAL gate) · **Kargo** (promote one immutable artifact dev→prod) ·
**ArgoCD** (reconcile) · **k3s + containerd** (runtime — *not* Docker Engine) · **Gatekeeper/OPA**
(admission: non-root, allowed-registries/fully-qualified, drop NET_RAW, no hostname-pin).

## G. Local-dev containers

`retrieva/docker-compose.yml` + `docker-compose.production.yml` · `globalcore-legacy/docker-compose.yml`
(+ `.oracle.yml`) — the Java/SOAP/MySQL legacy simulation. → the Compose teaching material.

---

**How the course uses this:** each level picks the sharpest specimen above. E.g. *image optimisation* →
platform-demo (distroless, 9 MiB) vs the Java/JRE images (bigger, why); *multi-stage* → every app; *custom
base image* → postgresql-noavx512; *registries/promotion* → the Harbor→ghcr hybrid + Kargo; *Docker in
K8s* → the Deployment/StatefulSet/Job/initContainer/sidecar table; *security* → Gatekeeper + cosign/SBOM/Trivy.
