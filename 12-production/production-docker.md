# Module 12 — Production Docker (Level 8)

> Learning note (Type A). The difference between "it runs on my laptop" and "it's a safe, operable
> production artifact." Most of these are *defaults you'd never bother with locally* and *mandatory* in prod.

## Concept
A production image is **minimal, immutable, non-root, observable, limited, and gracefully stoppable**, and
reads its config at **runtime** (not baked). It's designed to be scheduled, killed, and replaced freely.

## Problem it solves
Dev images optimise for *iteration* (big, root, bind-mounted source, debug tools). Prod needs the
opposite: small attack surface, reproducibility, safe rollouts, and operability under a scheduler. Shipping
a dev image to prod = security + reliability debt.

## How it works internally — the production checklist (each a real control)
| Practice | Why | Here |
|---|---|---|
| **multi-stage** | build tools never reach runtime | every app image |
| **minimal/distroless base** | fewer CVEs, faster pulls | platform-demo distroless; slim/jre elsewhere |
| **non-root `USER`** | blast-radius | `USER 10001`/`nonroot` + Gatekeeper |
| **env at runtime, not baked** | the *same* image runs dev+prod → promotable | the **env-agnostic image rule** (Kargo needs it) |
| **graceful shutdown (SIGTERM)** | finish in-flight work, clean connections | exec-form ENTRYPOINT (`ktayl-core` `sh -c "exec java…"`) |
| **HEALTHCHECK → probes** | don't route to a not-ready pod; restart a dead one | `/actuator/health` → readiness/liveness |
| **resource requests/limits** | scheduling + OOM protection | Gatekeeper + namespace quotas |
| **logs to stdout/stderr** | collected centrally, no shell needed | → Loki |
| **secrets at runtime** | never in a layer | ESO→Vault |
| **read-only rootfs + emptyDir** | immutable runtime FS | crew-agent |
| **immutable SHA tag** | provable, reproducible deploys | ghcr `:<sha>` |
| **pinned base (`:3.12-slim`)** | reproducibility | all Dockerfiles |

## Example (dev vs prod, same app)
```
DEV                              PROD
- bind-mount ./src (hot reload)  - immutable image (code baked, config at runtime)
- single stage, dev deps         - multi-stage, runtime deps only
- root is fine                   - USER 10001, read-only rootfs, caps dropped
- debug port open                - no debug port
- no limits                      - requests/limits + probes
- :dev tag (Harbor)              - :<sha> tag (ghcr), signed + SBOM
```

## Example from our organisation
- **Env-agnostic is a hard rule** (`gitops.md`): the image must read env at runtime so Kargo can promote
  *one artifact* dev→prod. A frontend that baked `NEXT_PUBLIC_*` had to be fixed to server-inject
  `window.__ENV__` + a `getRuntimeEnv()` accessor — because a baked-env image is **not promotable**. This
  is "production immutability" made concrete: config is injected, never built in.
- **Graceful stop, done right**: `ktayl-core`'s `ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar …"]` —
  the `exec` makes Java PID 1 so it receives SIGTERM on a rollout and shuts down cleanly (Module 3). Miss
  the `exec` → `sh` is PID 1 → the JVM is SIGKILLed after the grace period → dropped requests.
- **Probes replace HEALTHCHECK**: in k3s, readiness gates traffic + liveness restarts — the production
  form of Docker's HEALTHCHECK (Module 11).
- **Limits + non-root are enforced**, not hoped: Gatekeeper rejects a pod with no limits-able config / as
  root; quotas size the namespace. "Production practice" here is *admission-enforced*.
- **Logs → stdout**: every service logs to stdout (the alembic incident proved why — a disabled logger =
  blind in prod); Loki collects; no `exec` needed to see what happened.

## Architecture
```
 source → multi-stage build → minimal non-root image (:sha, signed, SBOM)
            │ config NOT baked
            ▼
 k3s: Deployment(replicas) · ConfigMap/Secret(runtime env) · probes · resources · securityContext(read-only, non-root)
            │ rolling update + canary brake
            ▼
 SIGTERM on replace → exec-form PID 1 drains → Pod replaced, zero/low downtime
```

## Commands
```bash
docker run --read-only --tmpfs /tmp --user 10001 --cap-drop ALL \
  --memory 512m --cpus 1 --security-opt no-new-privileges app:<sha>
docker inspect --format '{{.Config.User}} {{.Config.Healthcheck}}' app:<sha>
ssh controller "kubectl -n ktayl-core get deploy ktayl-core -o yaml | yq '.spec.template.spec.containers[0].{resources,readinessProbe,securityContext}'"
```

## Debugging
- **OOMKilled (137)** → limit too low or a leak; raise limit/fix leak (Module 8).
- **Rollout drops requests** → missing `exec` (ungraceful SIGTERM) or no readiness probe (traffic before ready).
- **"works in dev, fails in prod"** → baked env (not env-agnostic), or a dev-only bind mount the prod
  image doesn't have.
- **Pod rejected** → Gatekeeper (root / no limits / bare image) — fix the manifest, not a retry.

## Production considerations
This module *is* the production considerations. Meta-rules: same artifact dev→prod; config at runtime;
observable by default; killed-and-replaced safely; admission-enforced hardening; immutable signed SHA.

## Security considerations
Minimal base + non-root + read-only + dropped caps + no debug surface + runtime secrets — the Module 7
controls, applied as *defaults* for every prod image. A prod image with a shell + root + baked secrets is
a liability even if it "works."

## Common mistakes
Shipping the dev image to prod · baking env/config (breaks promotion) · shell-form ENTRYPOINT (no graceful
stop) · no probes/limits · logging to a file instead of stdout · `:latest` · root + writable rootfs in prod.

## Key takeaways
1. Prod image = **minimal · immutable · non-root · observable · limited · gracefully stoppable**.
2. **Config at runtime, never baked** → the *same* artifact is promotable (Kargo); this is the #1 rule here.
3. **exec-form ENTRYPOINT ⇒ PID 1 ⇒ SIGTERM ⇒ graceful drain** on rollout.
4. HEALTHCHECK → **K8s probes**; `--memory/--cpus` → **requests/limits**; both enforced by Gatekeeper.
5. Logs to **stdout** (Loki); secrets at runtime (Vault/ESO); deploy by **signed SHA**.

## Exercise (on our project)
1. Find the env-agnostic-image rule in `gitops.md` and the frontend that violated it — explain why a baked
   `NEXT_PUBLIC_*` makes an image un-promotable and how it was fixed.
2. Compare a dev overlay vs prod overlay for one service (`values-dev.yaml` vs `values-prod.yaml`) — list
   every dev→prod difference and classify each as size/security/observability/immutability.
3. Remove the `exec` from `ktayl-core`'s ENTRYPOINT in your head: trace what happens to the JVM on a
   `kubectl rollout restart`. Why do requests drop?

## See also
- Security → `07-security/…` · K8s probes/limits → `11-kubernetes/…` · CI/immutable SHA → `09-ci-cd/…`
- Checklist → `checklists/prod-image-checklist.md` (todo) · Dockerfile → `03-dockerfile/…`
