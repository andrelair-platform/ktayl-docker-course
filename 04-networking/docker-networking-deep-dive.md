# Module 4 — Container Networking

> Level 4 · Learning note (Type A). How containers get an IP, find each other by name, and expose ports —
> and how all of that changes shape in Kubernetes.

## Concept
Each container lives in its own **network namespace** (its own interfaces, IP, routing, ports). Docker
attaches it to a **network**; containers on the **same user-defined network** reach each other **by name**
via an embedded DNS resolver. Publishing (`-p`) bridges a host port to a container port.

## Problem it solves
Services must talk without hard-coding IPs (which change every restart). Name-based service discovery +
network isolation solve "how does the API find Postgres?" and "how do I keep DB traffic off the public
interface?"

## How it works internally — the network drivers
| Driver | What it gives | Use when |
|---|---|---|
| **bridge** (default, user-defined) | private virtual LAN; **DNS by container name**; NAT to host | the normal case; Compose makes one per project |
| **host** | container shares the host's network stack (no isolation, no port mapping) | max perf / needs host ports; e.g. media (JVB), some probes |
| **none** | no networking at all | fully isolated batch work |
| the default `bridge` (legacy) | works but **no name DNS** (only `--link`, deprecated) | avoid — always make a user-defined network |

- **DNS**: on a user-defined bridge, Docker runs a resolver at `127.0.0.11`; `ping api` resolves to the
  `api` container's IP. **Compose service name = DNS name.**
- **EXPOSE vs publish**: `EXPOSE 8080` is *documentation*; nothing is reachable from the host until you
  `-p 8080:8080` (host:container). Container-to-container traffic on the same network needs **no publish**.

## Example (Compose: the classic chain)
```yaml
services:
  frontend: { build: ./web, ports: ["3000:3000"] }   # published to the host
  api:      { build: ./api, environment: { DB_HOST: db } }  # talks to "db" by name
  db:       { image: postgres:16 }                    # NOT published → internal only
```
`frontend → api → db`: the browser hits `localhost:3000` (published); `api` reaches `db:5432` by DNS over
the project's bridge network; `db` is never exposed to the host. **Only publish what a human/outside needs.**

## Example from our organisation
- **In-cluster service-to-service by DNS**: `ktayl-core` reaches NATS at `nats.messaging.svc:4222` and
  ERPNext at `erpnext.erp.svc` — Kubernetes DNS (`<service>.<namespace>.svc`), the grown-up version of
  Compose's service-name DNS. It reaches Stripe over the **public internet** (the one governed egress
  exception). Nothing is "published to the host"; users reach apps only via the **nginx Ingress**
  (`*.10.0.0.200.nip.io`) → Service → Pod.
- **EXPOSE is doc-only here too**: every Dockerfile `EXPOSE`s its port (8000/8080/3000/9898) but exposure
  to users is done by the Ingress + Service, not the container.
- **Isolation by default-deny**: the platform runs **default-deny NetworkPolicies** — a pod can only reach
  what's explicitly allowed (`ktayl-core` egress = DNS + NATS + ERPNext + Postgres + `api.stripe.com`).
  That's "don't publish the DB" taken to the cluster: Postgres/NATS/Vault are never reachable except by
  the pods that need them. (The Docker analogue: keep `db` off any published port + on a private network.)
- **The `connect=000` gotcha**: right after a rollout, a pod may briefly fail to connect — transient CNI
  (Cilium) programming, **not** a netpol bug. Lesson: distinguish "DNS/route not programmed yet" from a
  real policy denial before debugging the wrong layer.

## Architecture
```
 Docker (local)                                  Kubernetes (here)
 browser ─▶ host:3000 ─▶ [frontend]              user ─▶ Ingress(nginx) ─▶ Service ─▶ Pod
                           │ bridge DNS "api"                               (ClusterIP, kube-dns)
                           ▼                                   pod ─▶ "nats.messaging.svc:4222"
                        [api] ─▶ "db:5432" [db] (unpublished)  egress gated by NetworkPolicy (default-deny)
```

## Commands
```bash
docker network ls
docker network create app-net
docker run -d --name db --network app-net postgres:16        # no -p → internal only
docker run --rm --network app-net alpine nslookup db          # name → IP (embedded DNS)
docker network inspect app-net                                # containers, subnet, gateway
docker port <ctr>                                             # what's published
# K8s twins:
ssh controller "kubectl exec -n ktayl-core deploy/ktayl-core -- nslookup nats.messaging.svc"
ssh controller "kubectl get svc,netpol -n <ns>"
```

## Debugging — "container can't reach X"
1. Same network? (`docker network inspect`; in K8s, same cluster + a Service exists).
2. DNS resolves? (`nslookup`/`getent hosts X` inside the container).
3. Right port + is X listening on `0.0.0.0` (not `127.0.0.1`)? (binding to loopback is a classic miss).
4. Policy allows it? (default `bridge` blocks name DNS; in K8s a **NetworkPolicy** may deny egress).
5. Transient? (just rolled out → wait for CNI; don't chase a ghost).

## Production considerations
Expose the minimum; put datastores on private networks / behind default-deny; terminate TLS at the edge
(Ingress) not in every container; prefer a Service/DNS name over an IP (IPs are cattle).

## Security considerations
Default-deny egress (DLP, exfil prevention) · never publish a DB to the host/world · one governed egress
exception per real need (the Stripe `api.stripe.com` allow) · network policy is *admission-enforced* here,
not convention.

## Common mistakes
Using the legacy default `bridge` and wondering why name-DNS fails · binding a server to `127.0.0.1`
inside a container (nothing else can reach it) · publishing a database port · confusing EXPOSE with
publish · blaming netpol for a transient post-rollout `connect=000`.

## Key takeaways
1. Each container has its own net namespace; **user-defined networks give name-based DNS**.
2. `EXPOSE` documents; **publish (`-p`) / Service+Ingress** is what actually exposes.
3. Service-to-service = **DNS name**, never a hard-coded IP (Compose service name → K8s `<svc>.<ns>.svc`).
4. Isolate datastores; **default-deny egress** is the enterprise posture here.
5. Bind servers to `0.0.0.0` inside containers; `127.0.0.1` is a trap.

## Exercise (on our project)
1. From the `ktayl-core` architecture doc, list every network destination it has and classify each as
   internal‑DNS vs public‑egress. Why is Stripe the only external one?
2. `kubectl get netpol -n <a ktayl ns>` — find a default-deny policy and the explicit allows. Which
   Compose habit does this generalise?
3. Exec into a pod and `nslookup` a Service in another namespace; then one it's *not* allowed to reach —
   explain the difference between "DNS resolves but connection refused" and "policy denied".

## See also
- Storage → `05-storage/…` · K8s Services → `11-kubernetes/…` + `_meta/docker-to-kubernetes-map.md`
- Fix → `troubleshooting/container-cannot-reach-postgres.md` (todo) · Fast → `cheatsheets/networking-cheatsheet.md` (todo)
