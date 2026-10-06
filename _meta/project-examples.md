# Project examples — concept → the real specimen that teaches it

When a module needs an example, use **this** service (not a toy). Paths are relative to the workspace root
(`~/Developer/cloudplateform/`).

| Concept | Best specimen | Path | Why it's the sharpest example |
|---|---|---|---|
| Why containers / "wrong environment" | postgresql-noavx512 | `minicloud-postgresql-noavx512/Dockerfile` | same image, wrong CPU → SIGILL; the container didn't make the AVX-512 problem go away — it made it *portable* and then *fixable* |
| Containers vs VMs / runtime | k3s + containerd | cluster | the runtime is **containerd**, Docker Engine isn't installed — proves "Docker image ≠ Docker runtime" |
| Multi-stage build | platform-demo (Go) | `platform-demo/Dockerfile` | builder (`golang:1.25-alpine`) → `distroless/static:nonroot`; ~9 MiB total |
| Multi-stage (interpreted lang) | ktayl-underwriting (Python) | `ktayl-underwriting/Dockerfile` | venv built in stage 1, copied to a clean slim runtime; explicit non-root uid 10001 |
| Multi-stage (Next.js standalone) | ktayl-iam frontend | `ktayl-iam/frontend/Dockerfile` | `.next/standalone` → tiny runtime; even removes the base image's npm to cut CVEs |
| Multi-stage (JVM) | ktayl-core / ktayl-claims | `ktayl-core/Dockerfile` | Maven build → `temurin-21-jre`; the JRE-vs-distroless size trade-off |
| Layer caching | platform-demo / ktayl-core | both Dockerfiles | `COPY go.mod`/`COPY pom.xml` *before* the source → deps layer caches across code changes |
| BuildKit cache mounts | platform-demo | `platform-demo/Dockerfile` | `--mount=type=cache,target=/go/pkg/mod` — persistent build cache without bloating the image |
| Base-image choice / optimisation | platform-demo vs ktayl-core | both | distroless-static (Go) vs jre (Java) — *why* you can't always go distroless |
| `.dockerignore` | ktayl-underwriting / platform-demo | `*/.dockerignore` | keeps the build context (and image) lean + avoids leaking `.git`/secrets |
| Networking (service-to-service) | ktayl-core | `ktayl-core/docs/architecture.md` | consumes NATS + ERPNext over **in-cluster DNS** (`*.svc`), Stripe over public TLS |
| Storage / container≠DB | CNPG Postgres | `minicloud-gitops` CNPG | the DB's data lives on a **PVC/Longhorn**, never the container layer |
| Writable-layer / readOnlyRootFS | crew-agent | gitops crew-agent | `readOnlyRootFilesystem:true` + emptyDir for `~/.cache`,`/tmp` (the EROFS fix) |
| Compose (local dev) | retrieva / globalcore-legacy | `retrieva/docker-compose.yml`, `globalcore-legacy/docker-compose.yml` | a real multi-service dev stack + the legacy Java/SOAP/MySQL sim |
| Non-root enforcement | Gatekeeper | gitops gatekeeper-policies | `require-non-root` HARD-denies root (ns NotIn exemptions) — security as admission, not hope |
| Allowed registries / fully-qualified | Gatekeeper | gitops | `docker.io/curlimages/curl` works, bare `curlimages/curl` is denied |
| Vulnerability scan / SBOM / signing | ktayl-core CI | `ktayl-core/.github/workflows/ci.yml` | Trivy CRITICAL gate + syft CycloneDX + cosign keyless |
| Registries (hybrid + promotion) | Harbor + ghcr + Kargo | gitops `.claude/rules/gitops.md` | dev→Harbor, prod→ghcr, Kargo promotes one SHA; keep-3/keep-5 + GC |
| CI → GitOps → deploy | ktayl-core / platform-demo | their CI + gitops | the full `push→build→scan→sign→push→Kargo→ArgoCD→k3s` chain |
| Docker→K8s object mapping | the whole cluster | `_meta/docker-to-kubernetes-map.md` | Deployment/StatefulSet/Job/initContainer/sidecar/probe examples |
| Custom base image | postgresql-noavx512 | `minicloud-postgresql-noavx512/Dockerfile` | the definitive "why build your own base" + the `FROM harbor…` self-hosted base |
| Multi-arch / platform pin | CI `--platform linux/amd64` | CI | amd64 cluster vs arm MacBook — why you pin + what breaks |

**Investigation targets (for the drills):** start with `ktayl-underwriting/Dockerfile` (clean, readable),
then `platform-demo/Dockerfile` (distroless gold standard), then `minicloud-postgresql-noavx512/Dockerfile`
(the custom-base story), then a frontend (`ktayl-iam/frontend/Dockerfile`). Corrected analyses land in
`case-studies/`.
