# Module 11 — Docker and Kubernetes (Level 12)

> Learning note (Type A). What happens to every Docker concept when you move to Kubernetes — and the fact
> that confuses everyone: **modern Kubernetes doesn't use Docker Engine, yet runs the Docker images you
> build.** Pairs with `_meta/docker-to-kubernetes-map.md` (the quick table).

## Concept
Kubernetes is a **cluster scheduler** for containers. It takes your OCI images and runs them as **Pods**
across nodes, adding what single-host Docker lacks: scheduling, self-heal, scaling, rolling updates,
service discovery, config/secrets, storage orchestration, and policy. Docker concepts don't disappear —
they get a cluster-grade object.

## Problem it solves
`docker run` / Compose run on **one box**, by hand, with no self-heal, no HA, no rollout safety. An
enterprise needs "run N replicas across nodes, restart on failure, roll out safely, discover by name,
inject config/secrets, attach durable storage, enforce policy" — that's Kubernetes.

## How it works internally — the runtime truth
```
kubectl apply ─▶ API server ─▶ (scheduler picks a node) ─▶ kubelet on that node
                                                              │  speaks CRI (Container Runtime Interface)
                                                              ▼
                                                        containerd  ─▶ runc ─▶ namespaces/cgroups
                                                        (pulls the OCI image, runs the container)
```
- **No Docker Engine on the node.** k3s runs **containerd** directly. `kubectl` is not `docker`; the
  kubelet talks **CRI** to containerd, which runs **runc** — the same low-level pieces from Module 0,
  minus `dockerd`.
- **dockershim was removed in k8s 1.24.** That removed the *Docker Engine* shim, **not** Docker images.
  Your `docker build` output is a generic **OCI image** → containerd runs it unchanged.
- A **Pod** = one or more containers sharing a network namespace (one IP) + volumes — the unit Kubernetes
  schedules. `docker run` ≈ a one-container Pod.

## The mapping (every Docker idea → its K8s form, grounded)
| Docker | Kubernetes | Real example here |
|---|---|---|
| `docker run` (stateless) | Deployment → ReplicaSet → Pods | platform-demo, ktayl-iam |
| a container w/ identity+disk | **StatefulSet** + volumeClaimTemplates | CNPG Postgres, Vault, Harbor |
| `-e VAR` | **ConfigMap** / **Secret** | ktayl-core env + `ktayl-iam-secret` (ESO→Vault) |
| `-v vol:/path` | **PV + PVC** (Longhorn) | every CNPG DB |
| container DNS / `--network` | **Service** + cluster DNS `<svc>.<ns>.svc` | `nats.messaging.svc:4222` |
| `-p host:ctr` | **Service** + **Ingress** | nginx → `*.10.0.0.200.nip.io` |
| `HEALTHCHECK` | **liveness/readiness/startup probes** | `/actuator/health` |
| Compose `service` | Deployment/StatefulSet | retrieva compose → k3s |
| `depends_on` | initContainers + readiness + retries | CA-trust initContainer |
| one-shot run | **Job** | `UNDERWRITING_EVENTS` stream-init |
| cron run | **CronJob** | kine-backup, vault-snapshot |
| helper container | **sidecar** | Vault agent injector, cloudflared |
| `--memory/--cpus` | `resources.requests/limits` | Gatekeeper + quotas |
| `--read-only`/`--user`/`--cap-drop` | `securityContext.*` | enforced by Gatekeeper |
| `docker build/push/pull` | **unchanged** (build with Docker/buildx in CI; containerd pulls) | the CI + nodes |

## Example from our organisation
- **The whole k3s cluster is the reference.** `ktayl-core`'s Deployment is `docker run` grown up: 2 prod
  replicas, env from a ConfigMap + ESO Secret, `/actuator/health` → readiness/liveness probes,
  `securityContext` non-root + read-only, image pulled by containerd from ghcr via `ghcr-pull`.
- **StatefulSet vs Deployment, decided for real**: ktayl services are Deployments (stateless — the image
  is the whole app, data is elsewhere); Postgres/Vault/Harbor are StatefulSets (stable identity + PVC).
- **initContainer + sidecar in the wild**: the CA-trust initContainer runs *before* nextcloud to append
  the minicloud CA; the Vault agent injector rides *alongside* a pod as a sidecar. Both are "extra
  containers in one Pod" — a concept Compose only hints at.
- **Proof the runtime is containerd**: there is no Docker Engine on the nodes; the images are the exact
  OCI artifacts CI built with buildx. (Lab L16: inspect a running container's namespaces/cgroups on a node
  — same primitives as `docker run`, scheduled.)

## Architecture
```
 DEV (one box)                         PROD (k3s cluster)
 docker run app ──────────────────▶   Deployment(app, replicas:2)
   -e / --network / -v / -p             └ ConfigMap/Secret · Service+Ingress · PVC · probes · securityContext
 docker build → local image      ──▶   CI buildx → ghcr → containerd pulls → Pod(runc, namespaces/cgroups)
```

## Commands
```bash
ssh controller "kubectl --context minicloud get deploy,sts,job,cronjob -A | head"
ssh controller "kubectl -n ktayl-core get pod -o jsonpath='{.items[0].spec.containers[0].image}'"
ssh controller "kubectl -n <ns> describe pod <p>"     # events, probes, state (the 'docker inspect')
ssh controller "crictl ps"                            # containerd's 'docker ps' on a node (CRI tool)
```

## Debugging
Same decision tree as Module 8, cluster edition: `kubectl get pod` (Running? Restarts?) → `describe`
(Events: ImagePull/OOM/FailedScheduling/probe-fail) → `logs [-p]` → `exec`/`debug` (distroless) →
`get netpol/pvc/events`. New cluster-only layers: **scheduling** (quota/affinity) and **admission**
(Gatekeeper denials).

## Production considerations
Probes (don't send traffic before ready; restart on dead); resource requests/limits (scheduling + OOM);
rolling update / canary brake; PodDisruptionBudgets; don't node-pin (Longhorn is network-attached);
env-agnostic images so the *same* artifact runs dev+prod (the thing Kargo promotes).

## Security considerations
`securityContext` enforces what Dockerfiles should set (non-root, caps, read-only) — and **Gatekeeper
admission** makes it non-optional; NetworkPolicies (default-deny) are the "don't publish the DB" rule at
cluster scale; Secrets via ESO→Vault, not plain k8s Secrets where avoidable.

## Common mistakes
"K8s removed Docker so my image won't run" (false — OCI images run) · using a Deployment for a database ·
expecting `depends_on`-style ordering (use initContainers + readiness) · no probes/limits · node-pinning
a stateful pod · assuming `kubectl exec` works on distroless (use `kubectl debug`).

## Key takeaways
1. K8s runs **OCI images** via **kubelet→CRI→containerd→runc** — **no Docker Engine** on the node.
2. **dockershim removal ≠ Docker-image removal**; keep building with Docker/buildx.
3. Every Docker concept has a cluster object (Pod/Deployment/StatefulSet/Service/PVC/ConfigMap/probe).
4. **Deployment = stateless, StatefulSet = stateful**; initContainers/sidecars = multi-container Pods.
5. The hardening you'd set in a Dockerfile is **enforced by securityContext + Gatekeeper** here.

## Exercise (on our project)
1. Pick `ktayl-core`'s Deployment: map each field back to a `docker run` flag (image, env, ports, probes,
   securityContext, resources). Which `docker run` flag has no single equivalent and why?
2. Find a StatefulSet (CNPG Postgres) and a Deployment (platform-demo) — justify each choice.
3. Confirm the node runtime is containerd (`crictl ps` or the kubelet config) and write 3 sentences on why
   "Kubernetes dropped Docker" does **not** mean your Docker images are obsolete.

## See also
- Quick table → `_meta/docker-to-kubernetes-map.md` · Storage/PVC → `05-storage/…` · Networking/Service → `04-networking/…`
- Production → `12-production/…` · Security/admission → `07-security/…` · Fundamentals → `01-fundamentals/module-00-…`
