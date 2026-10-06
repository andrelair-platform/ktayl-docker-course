# Module 1 — Docker Basics (images, containers, run · exec · logs · ports · env)

> Level 1 · Learning note (Type A). Module 0 built the *why*; this builds the **muscle memory** of driving
> a container. Everything here has a `kubectl` twin (the cluster is "Docker, scheduled") — we show both.

---

## Concept
- An **image** is the immutable, read-only template (the "class"): runtime + deps + filesystem + config.
- A **container** is a **running (or stopped) instance** of an image (the "object"). One image → **many**
  containers. You `run` an image → you get a container with its own ephemeral writable layer + namespaces.
- A container's life is tied to its **PID 1** process: when PID 1 exits, the container stops and leaves an
  **exit code**.

## Problem it solves
Module 0 said containers freeze the environment. Level 1 is the *operator's* half: how do I **start, feed
(env), expose (ports), observe (logs), enter (exec), and stop/remove** a container — reliably and the
same way every time — without SSHing into a box and running the app by hand.

## How it works internally — the lifecycle
```
 docker create     ┌─────────┐  docker start   ┌─────────┐  SIGTERM/exit   ┌─────────┐  docker rm
 (image → config) ▶│ created │ ───────────────▶│ running │ ───────────────▶│ exited  │ ──────────▶ (gone)
                   └─────────┘                 └────┬────┘   (exit code)    └─────────┘
                                  docker pause ─────┤                            ▲
                                                 (paused) ── unpause ────────────┘
    `docker run` = create + start in one step.   PID 1 dies ⇒ container stops.   writable layer dies on rm.
```
- **`run` = pull (if needed) + create + start.** `-d` detaches (background); without it you attach to the
  container's stdout/stderr (foreground).
- **Exit codes** tell you *how* it ended: `0` clean · `1`/`2` app error · **`137`** SIGKILL/OOM ·
  **`143`** SIGTERM (graceful stop). `docker ps -a` shows them.
- **Env & ports are set at `run` time**, not baked: `-e KEY=val` injects config; `-p host:container`
  publishes a port to the host. `EXPOSE` in the image is *documentation only* — it publishes nothing.
- **`--rm`** auto-removes on exit (throwaway); **`--restart`** sets a restart policy (the ancestor of K8s
  `restartPolicy` / CrashLoopBackOff).

## Example
```bash
docker pull nginx:1.27                                   # fetch the image (layers) to the local cache
docker run -d --name web -p 8080:80 nginx:1.27           # create+start, detached, publish 80→8080
docker ps                                                # it's running
curl localhost:8080                                      # reach it via the published port
docker logs -f web                                       # watch its stdout
docker exec -it web sh                                   # step inside (nginx has a shell)
docker run -d --name api -e DATABASE_URL=postgres://… app:dev   # config via env at run time
docker stop web && docker rm web                         # stop (SIGTERM→SIGKILL) then remove
docker run --rm -it alpine sh                            # throwaway: removed the moment you exit
```

## Example from our organisation
You won't `docker run` on the cluster (k3s schedules it) — but **every running Pod *is* a container you
drive with the exact same verbs**, via `kubectl`:
- **A pod is a container**: `platform-demo` / `ktayl-iam` pods are instances of the images CI built. The
  image is replaceable; the container is the running instance (Module 0).
- **Env at run time, not baked**: `ktayl-core` gets `DATABASE_HOST`, `AUTHENTIK_ISSUER`, etc. from a
  **ConfigMap** + secrets from **ESO→Vault** — the cluster's `-e` (the env-agnostic-image rule, Module 12).
  `kubectl get pod <p> -o jsonpath='{.spec.containers[0].env}'` is "show me its `-e` flags."
- **Logs → stdout → Loki**: every service logs to stdout (never a file), so `kubectl logs` just works —
  the alembic "zero logs" incident proved why: a *disabled logger* looked like a silent app. Logs are the
  **first reflex** after "is it running?".
- **exec, and when you can't**: `kubectl exec -it <p> -- sh` enters a pod — but **`platform-demo` is
  distroless (no shell)**, so you use `kubectl debug` (an ephemeral debug container) instead (Module 8).
- **Exit codes in the wild**: a pod `OOMKilled` = **137** (raise the memory limit); a clean rollout sends
  **143** (SIGTERM) — which `ktayl-core`'s `exec`-form entrypoint catches to shut down gracefully (Module 3).
- **Ports**: Dockerfiles `EXPOSE 8000/8080/3000/9898` (doc), but users reach apps only through the nginx
  **Ingress → Service → Pod** — the cluster's "publish", not a `-p` on the container.

## Architecture
```
 image (ghcr/Harbor)
     │ run / kubectl schedules
     ▼
 container (Pod)  ── PID 1 process (your server) ── writable layer (ephemeral)
     ├─ env:   -e / ConfigMap+Secret(ESO→Vault)
     ├─ ports: -p / Service+Ingress        (EXPOSE is just a label)
     ├─ logs:  stdout/stderr → docker logs / kubectl logs → Loki
     └─ enter: docker exec / kubectl exec  (distroless → kubectl debug)
```

## Commands (and the twin)
```bash
docker pull <img>                         # → containerd pulls on the node automatically
docker run -d --name x -p 8080:80 <img>   # → a Deployment/Pod (scheduled)
docker ps / ps -a                         # → kubectl get pods [-A]           (-a = see exited + codes)
docker logs -f --tail=100 x               # → kubectl logs -f --tail=100 <p> -n <ns>
docker logs (crashed prior run)           # → kubectl logs -p <p> -n <ns>     (previous container)
docker exec -it x sh                      # → kubectl exec -it <p> -n <ns> -- sh
docker inspect x                          # → kubectl get pod <p> -n <ns> -o yaml
docker stats / top                        # → kubectl top pod -n <ns>
docker stop/kill/rm x                     # → kubectl delete pod <p>  (controller recreates it)
```
Full twin table → `cheatsheets/docker-vs-kubectl-cheatsheet.md`; fast core → `cheatsheets/docker-cli-cheatsheet.md`.

## Debugging
- **Exits immediately (code 0)** → PID 1 isn't a long-running foreground server (it forked/returned). Run
  the server in the foreground.
- **`docker logs` empty** → wrong stream (app logs to a file, not stdout) **or** a disabled logger — not
  proof it's healthy (the alembic case). Verify on the *running* instance.
- **`-it` "the input device is not a TTY"** in CI → drop `-t` in non-interactive contexts.
- **Env var "not set"** → it was baked into the image instead of passed at run (`-e`/ConfigMap); check
  `docker inspect`/`kubectl describe`.
- **Port unreachable** → you `EXPOSE`d but didn't publish (`-p`), or the server bound `127.0.0.1` not
  `0.0.0.0` inside the container (Module 4).

## Production considerations
Never `docker run` by hand in prod — a controller (Deployment) owns lifecycle + restart + rollout. Config
comes at run time (ConfigMap/Secret), logs go to stdout, PID 1 must handle SIGTERM for graceful stop. The
container is **cattle**: killed + recreated freely; nothing important lives in it (Module 5).

## Security considerations
`docker exec`/`kubectl exec` into a prod container is a **privileged, audited** action — prefer
logs/metrics. Don't pass secrets as plain `-e` on a shared host (visible in `inspect`/`ps`); use
ESO→Vault. A shell-less (distroless) image is *more* secure — accept `kubectl debug` as the trade-off.

## Common mistakes
Confusing image with container (one image, many containers) · forgetting `-d` (terminal blocks) or
forgetting the server must run in the **foreground** · `EXPOSE` ≠ publish · baking env instead of passing
it · treating "no logs" as "healthy" · `docker rm` losing data that should've been on a volume (Module 5).

## Key takeaways
1. **Image = template; container = running instance**; one image → many containers.
2. `run` = pull+create+start; the container lives as long as **PID 1**; exit codes talk (137 OOM / 143 SIGTERM).
3. **Env & ports are set at run time** (`-e`, `-p`); `EXPOSE` only documents.
4. **Logs → stdout**; `logs` is the first reflex; "no logs" can be a disabled logger, not silence.
5. Every verb has a **`kubectl` twin** — here you drive Pods, but it's the same mental model.

## Exercise (on our project)
1. `ssh controller "kubectl --context minicloud get pods -n ktayl-core"` — pick the backend pod. Get its
   image, its env (`-o jsonpath='{.spec.containers[0].env}'`), and its restart count. Which of those is the
   cluster's `-e`? which is `EXPOSE` vs a real published port (hint: find its Service/Ingress)?
2. `kubectl logs` it; then `kubectl exec -it … -- sh`. Now try the same `exec` on a `platform-demo` pod —
   why does it fail, and what do you use instead?
3. Delete the pod (`kubectl delete pod …`) and watch it come back. What recreated it, and what does that
   tell you about "don't `docker run` by hand in prod"?

## See also
- Prev → `module-00-why-containers-exist.md` · Next → `../02-images/images-layers-caching.md`
- Fast → `../cheatsheets/docker-cli-cheatsheet.md` · Twins → `../cheatsheets/docker-vs-kubectl-cheatsheet.md`
- Debug → `../08-debugging/debugging-methodology.md` · K8s → `../11-kubernetes/docker-to-kubernetes.md`
