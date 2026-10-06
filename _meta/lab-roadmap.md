# Lab roadmap — project-based, hands-on

Every lab uses a **real service from the org**, not a toy app. Each lab: a goal, the specimen, the steps,
and a "done when" bar. Labs are stored in `labs/` as you complete them (with your notes + what broke).

> **Where you run them.** Image builds: locally (`docker`/`buildx`) or read the CI that does it. Runtime
> inspection: `kubectl --context minicloud` against the live cluster (the containers already run there) —
> `ssh controller "kubectl …"`. No local Docker daemon is required for the K8s-side labs.

| # | Lab | Specimen | Level | Done when |
|---|---|---|---|---|
| L1 | Run, inspect, exec, read logs of a live container | any pod (e.g. `platform-demo`) | 1 | you can get logs, exec a shell, read exit codes + restart count |
| L2 | Read a Dockerfile & answer the investigation questions | `ktayl-underwriting/Dockerfile` | 2–3 | base? stages? root? port? what's copied? what to optimise? |
| L3 | Build an image locally + read `docker history` | `platform-demo` (Go) | 2 | you see the layers + the cache-mount effect on a rebuild |
| L4 | Containerise a FastAPI service from scratch | mimic ktayl-underwriting | 3,7 | multi-stage, non-root, env-agnostic, < the slim base + your code |
| L5 | Containerise a Next.js frontend (standalone) | mimic ktayl-iam frontend | 3,7 | standalone output, `USER node`, no dev deps in runtime |
| L6 | Optimise an image | take a naive single-stage build → multi-stage/distroless | 2,8 | measurable size drop; `docker history` explains it |
| L7 | Convert a root container to non-root | a Dockerfile without `USER` | 9 | runs as uid≠0; survives Gatekeeper `require-non-root` |
| L8 | Add a HEALTHCHECK + map it to K8s probes | ktayl-core | 8,12 | HEALTHCHECK locally + readiness/liveness in the chart |
| L9 | Compose stack: frontend + API + Postgres | retrieva compose as reference | 6 | `docker compose up` → the 3 talk over the compose network |
| L10 | Debug container networking | break a service's DNS/netpol on dev | 4,8 | you diagnose with `exec`+`nslookup`/`curl` and fix it |
| L11 | Persist data correctly | a Postgres container: volume vs no volume | 5 | data survives `rm`+recreate with a volume; is lost without |
| L12 | Push an image to a registry | Harbor (dev) | 10 | `:<sha>` appears in Harbor; pull it back |
| L13 | Build through CI (read + trigger) | ktayl-core CI | 11 | you can trace build→Trivy→cosign→SBOM→ghcr in the run log |
| L14 | Deploy the resulting image to K8s | via Kargo→ArgoCD | 11,12 | a new `:<sha>` promoted dev→prod and reconciled |
| L15 | Rebuild the custom Postgres base | `minicloud-postgresql-noavx512` | 7,13 | you understand the AVX-512/SIGILL fix + push to Harbor |
| L16 | Inspect namespaces & cgroups of a running container | any pod on a node | 0 | you see the pid/net/mnt namespaces + the cgroup limits |
| L17 | Multi-arch reality check | the `--platform linux/amd64` pin | 13 | explain why amd64-only + what breaks on the arm Mac |

**Investigation drills** (do these repeatedly — section 6 of the brief): point at a service, answer:
base image? why chosen? number of stages? running as root? exposed port? what's copied in? what can be
optimised? what security issues? Then compare to the corrected analysis in the matching `case-studies/` note.
