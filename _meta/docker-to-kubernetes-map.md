# Docker → Kubernetes concept map (with the real org example)

The single most useful translation table in the course: every Docker concept and its Kubernetes
equivalent, grounded in a running specimen. Read this *after* Levels 4–6 (you need networking/storage/
compose first), then it makes Level 12 click.

| Docker concept | Kubernetes equivalent | Real example here |
|---|---|---|
| `docker run <img>` | a **Pod** (usually via a **Deployment**) | platform-demo / ktayl-iam Deployments |
| a container that must keep identity + storage | a **StatefulSet** + volumeClaimTemplates | CNPG Postgres, Vault, Harbor, Longhorn |
| `-e FOO=bar` env var | **ConfigMap** (non-secret) / **Secret** (secret) | ktayl-core env (ConfigMap) + `ktayl-iam-secret` (ESO→Vault) |
| `-v data:/var/lib/postgresql` volume | **PersistentVolume + PVC** (Longhorn) | every CNPG Postgres |
| a bind mount (dev) | **hostPath** (rare) / configMap/secret mount | CA-trust configMap mounted into nextcloud |
| `--network` + container DNS | **Service** + cluster DNS (`<svc>.<ns>.svc`) | ktayl-core → `nats.messaging.svc:4222` |
| `-p 8080:8080` publish | **Service** (ClusterIP/NodePort/LB) + **Ingress** | nginx Ingress → `*.10.0.0.200.nip.io` |
| `HEALTHCHECK` | **liveness / readiness / startup probes** | `/actuator/health` → probes in the GAP chart |
| a Compose `service` | a **Deployment** / **StatefulSet** | retrieva compose → retrieva K8s |
| `depends_on` | there is no hard ordering → **initContainers** + readiness + retries | CA-trust initContainer before the app |
| a one-shot `docker run … ; exit` | a **Job** | the `UNDERWRITING_EVENTS`/`HR_LIFECYCLE` stream-init |
| a cron `docker run` on a timer | a **CronJob** | kine-backup, vault-snapshot, Harbor GC |
| one Pod, helper container | a **sidecar** | Vault agent injector; cloudflared |
| `--memory`/`--cpus` limits | `resources.requests/limits` | enforced by Gatekeeper + namespace quotas |
| `--read-only` rootfs | `securityContext.readOnlyRootFilesystem` | crew-agent (+ emptyDir for writable paths) |
| `--user 10001` | `securityContext.runAsNonRoot/runAsUser` | enforced by Gatekeeper `require-non-root` |
| `--cap-drop`/`--cap-add` | `securityContext.capabilities` | drop NET_RAW (Gatekeeper block-net-raw) |
| `docker build` | **you still build an OCI image** (Docker/buildx in CI) | the CI buildx jobs |
| `docker push`/`pull` | same OCI registries (Harbor/ghcr) + `imagePullSecrets` | `ghcr-pull` ESO secret |
| the Docker Engine daemon | **the container runtime = containerd** (via **CRI**, kubelet) | k3s nodes run **containerd**, no Docker Engine |

## The one thing people get wrong
> **"Kubernetes removed Docker, so my Docker images won't work."** False.
Kubernetes removed **dockershim** (the shim to the Docker *Engine* daemon) in v1.24. It still runs **OCI
images** — exactly what `docker build` / `buildx` produce. We build images with Docker/BuildKit in CI,
push them to Harbor/ghcr, and **containerd** (not Docker Engine) pulls + runs them on the k3s nodes.
Docker-the-image-format and Docker-the-engine are different things; only the engine left the K8s node.

## Reading path
`docker run` mental model (L0–1) → networking (L4) → storage (L5) → Compose (L6) → **this table** → L12
(`11-kubernetes/`). The platform's own spine (`ktayl-core`, CNPG, NATS, Kargo→ArgoCD) is the live
reference for every row.
