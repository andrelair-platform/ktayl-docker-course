# Cheatsheet — Networking

| Problem | Command |
|---|---|
| List networks | `docker network ls` |
| Create a user-defined bridge (gets name-DNS) | `docker network create app-net` |
| Run a container on it (internal only) | `docker run -d --name db --network app-net postgres:16` |
| Resolve a name → IP (embedded DNS) | `docker run --rm --network app-net alpine nslookup db` |
| Who's on a network / subnet / gateway | `docker network inspect app-net` |
| What ports are published | `docker port <c>` |
| Publish a port to the host | `docker run -p 8000:8000 app` (host:container) |
| Reach a service from another container | use the **name** (`db:5432`), no publish needed |
| **K8s:** resolve a Service | `kubectl exec -n <ns> <p> -- nslookup <svc>.<ns>.svc` |
| **K8s:** see Services + policies | `kubectl get svc,netpol -n <ns>` |
| **K8s:** is egress allowed to X? | read the `NetworkPolicy` (default-deny → only explicit allows) |

**"Container can't reach X" — 60-second checklist**
1. Same network? (`network inspect`; K8s: a Service exists + same cluster)
2. DNS resolves the name? (`nslookup X`) — legacy default `bridge` has **no name DNS**; always use a user-defined net.
3. Right **port**, and is X listening on **`0.0.0.0`** not `127.0.0.1`? (loopback bind = unreachable)
4. Policy allows it? (K8s **default-deny** egress — needs an explicit allow, e.g. ktayl-core→`nats.messaging.svc`)
5. Just rolled out? `connect=000` right after = transient **CNI** programming, not a bug — wait, retry.

**Rules:** `EXPOSE` documents; `-p`/Service+Ingress publishes. Service-to-service = **DNS name**, never a
hard IP (Compose service name → K8s `<svc>.<ns>.svc`). Don't publish databases.

See also → `../04-networking/docker-networking-deep-dive.md`
