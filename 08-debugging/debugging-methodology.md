# Module 8 — Debugging Containers: a systematic methodology

> Learning note (Type A) + the hub for `troubleshooting/`. The goal is a **repeatable method**, not a bag
> of commands — you narrow the layer first, then reach for the right tool.

## Concept
Container failures cluster into a few layers: **image/config → start/lifecycle → app → networking →
storage/permissions → resources**. Diagnose top-down; don't guess. The same method works for `docker` and
for `kubectl` (the cluster is just "docker, scheduled").

## Problem it solves
"It's broken" wastes hours when you poke random layers. A decision tree turns an opaque failure into a
5-minute narrow-down and a root cause you can *name* (the transferable skill; the fix syntax is not).

## How it works internally — the decision tree
```
Container not working
   │
   ├─ Is it even running?  docker ps / kubectl get pod
   │     └─ No → WHY did it stop?  docker ps -a (EXIT CODE) / kubectl describe (State, Reason, restartCount)
   │            137=SIGKILL/OOM · 143=SIGTERM · 1/2=app error · 0=PID1 wasn't a foreground server
   ▼
   Logs                    docker logs / kubectl logs (--previous for a crashed prior container)
   │   (nothing in logs? maybe logging was disabled — see the alembic case)
   ▼
   Config / env            docker inspect / kubectl describe · are the env vars actually set?
   ▼
   Image pull              ImagePullBackOff → tag exists? registry auth? digest present?
   ▼
   Networking / DNS        exec + nslookup/curl · right port? bound to 0.0.0.0? netpol allows? (Module 4)
   ▼
   Storage / permissions   mounts present? read-only rootfs + write path? uid owns the volume? (Module 5)
   ▼
   Resources               OOMKilled? CPU-throttled? quota rejected the pod? (stats/top/describe)
```

## Example
```bash
docker ps -a                       # STATUS + exit code of the stopped container
docker logs --tail=100 <ctr>       # why it crashed
docker inspect <ctr>               # State.ExitCode, Mounts, NetworkSettings, Config.Env
docker exec -it <ctr> sh           # poke from inside (if the image has a shell!)
```

## Example from our organisation (real incidents — each a layer)
- **ImagePullBackOff, "postgresql tag not found"** → Harbor GC had deleted a tag still referenced
  (`postgresql*` wasn't retention-protected). Layer = *image pull*. Fix = retention rule, not a retry.
  (Cousin: the **MinIO docker.io paywall** → digest-pinned refs stopped resolving → mirror to Harbor.)
- **"Add request logging" shipped, zero logs live** → the app self-migrates on startup and alembic's
  `fileConfig()` (default `disable_existing_loggers=True`) **killed every logger** after the migration.
  Layer = *app*, found only by reading the running pod — mocks/CI were green. The lesson in the decision
  tree's "nothing in logs?" branch.
- **CrashLoop: EROFS** → `readOnlyRootFilesystem` + a write to `~/.cache` → add emptyDir (Module 5). Layer
  = *storage/permissions*.
- **`InvalidImageName` `9.248482e+06`** → an all-digit SHA parsed as a YAML number. Layer = *config*. Fix
  = quote the tag (Kargo does this). You'd never find it in logs — it's an admission/parse error.
- **`connect=000` right after a rollout** → transient CNI programming, **not** a netpol bug. Layer =
  *networking, but transient* — the method's "is it transient?" check saves you chasing a ghost.
- **Distroless has no shell** → `docker exec sh` fails on platform-demo's image. Use an **ephemeral debug
  container** (`kubectl debug`) that attaches a shell-bearing image to the pod's namespaces. This is the
  trade-off of a tiny image: harder to poke live → rely on logs/metrics + ephemeral debug.

## Architecture (where to look in K8s)
```
kubectl get pod            → Running? Restarts? Age
kubectl describe pod       → Events (ImagePull, FailedScheduling, OOMKilled, Readiness fail), State/Reason
kubectl logs [-p]          → app output (‑p = the crashed previous container)
kubectl exec / kubectl debug → inside the container (debug = for distroless/no-shell images)
kubectl get events -n ns   → cluster-level clues (quota, admission denials)
```

## Commands (and *when* each earns its place)
```bash
docker ps / ps -a          # running / stopped + exit code      → "is it up, why did it die"
docker logs [-f] [--tail]  # app output                          → the first thing after ps
docker inspect <ref>       # full truth: state, mounts, env, net → config/mount/net questions
docker exec -it <ctr> sh   # inside (needs a shell in the image) → app/FS/DNS checks
docker stats / top         # live CPU/mem / processes            → resource/OOM/hang
docker history <img>       # layers + sizes                      → "why huge", cache questions
docker network inspect     # a network's members/subnet          → connectivity
docker volume inspect      # a volume's mountpoint                → persistence/permissions
```

## Production considerations
Logs to **stdout/stderr** (collected by Loki) so you don't need a shell; expose **health + metrics** so
most issues surface without `exec`; keep a **debug image** pattern for distroless (`kubectl debug`). The
platform's `sdlc_loop` + Alertmanager catch band-breaches before a human looks.

## Security considerations
`exec` into a prod container is a privileged action — audited; prefer logs/metrics. A shell-less
(distroless) image is *more secure* precisely because an attacker also can't `exec` a shell — accept the
debugging trade-off, don't "fix" it by adding a shell to prod.

## Common mistakes
Restarting blindly instead of reading the exit code/events · ignoring "no logs" as "no problem" (it was a
disabled logger) · chasing a transient CNI blip as a bug · trying `exec sh` on distroless and concluding
"broken" · not using `--previous`/`-p` on a crashed container.

## Key takeaways
1. **Narrow the layer first** (running? → logs → config → pull → net → storage → resources).
2. **Exit codes talk**: 137 OOM/kill, 143 SIGTERM, 0 = PID1 wasn't a server.
3. "No logs" can be a *disabled logger*, not silence — verify on the **running** pod.
4. Distroless ⇒ `kubectl debug` (ephemeral container), not `exec sh`.
5. Some failures are **config/admission/transient**, invisible to logs — know which layer you're in.

## Exercise (on our project)
1. `kubectl get pods -A | grep -v Running` on dev — pick a non-Running pod, run the decision tree, and
   name the layer + root cause before fixing anything.
2. Try `kubectl exec` a shell into `platform-demo` (distroless) — watch it fail; then do it correctly with
   `kubectl debug`. Why is the failure actually a *security feature*?
3. Re-read the alembic "zero logs" incident; add a step to the decision tree that would have caught a
   disabled-logger faster.

## See also
- Networking fixes → `04-networking/…` · Storage fixes → `05-storage/…` · Each incident → `troubleshooting/*.md` (todo)
- Fast → `cheatsheets/debugging-cheatsheet.md` (todo)
