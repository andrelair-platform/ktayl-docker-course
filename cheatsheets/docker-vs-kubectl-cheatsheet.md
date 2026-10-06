# Cheatsheet — `docker` ↔ `kubectl` twins

Because our containers run on **k3s/containerd**, not a local Docker daemon. Prefix cluster commands with
`ssh controller "kubectl --context minicloud …"`.

| You want (docker) | On the cluster (kubectl) |
|---|---|
| `docker ps` | `kubectl get pods -A` |
| `docker ps -a` (+ why dead) | `kubectl get pods -A` then `kubectl describe pod <p> -n <ns>` (Events/State/Reason) |
| `docker logs <c>` | `kubectl logs <p> -n <ns>` |
| `docker logs` of the crashed prior run | `kubectl logs <p> -n <ns> -p` |
| `docker exec -it <c> sh` | `kubectl exec -it <p> -n <ns> -- sh` |
| exec into a **distroless** pod (no shell) | `kubectl debug -it <p> -n <ns> --image=nicolaka/netshoot --target=<container>` |
| `docker inspect <c>` | `kubectl get pod <p> -n <ns> -o yaml` |
| `docker stats` | `kubectl top pod -n <ns>` (needs metrics-server) |
| which image is running? | `kubectl get pod <p> -n <ns> -o jsonpath='{.spec.containers[0].image}'` |
| why won't it schedule / pull? | `kubectl describe pod <p> -n <ns>` · `kubectl get events -n <ns> --sort-by=.lastTimestamp` |
| restart it | `kubectl rollout restart deploy/<d> -n <ns>` (never `docker restart`) |
| `docker run` one-off | a **Job** (`kubectl create job … --image=…`) — e.g. the stream-init Jobs |
| the node's "docker ps" | on a node: `crictl ps` (containerd's CRI tool) |
| is the runtime Docker? | **no** — `kubectl get node -o wide` → CONTAINER-RUNTIME = `containerd` |

**Don't:** `argocd app sync` / `kubectl apply` by hand to "deploy" — delivery is GitOps (Kargo→ArgoCD).
Reading (`get/describe/logs/exec`) is always fine.

See also → `../11-kubernetes/docker-to-kubernetes.md` · `../_meta/docker-to-kubernetes-map.md`
