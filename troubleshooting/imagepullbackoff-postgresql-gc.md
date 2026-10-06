# Troubleshooting — `ImagePullBackOff`: the registry GC'd a tag still in use

> Type D (troubleshooting). A **real incident** (#1146): litellm was down ~6 days because Harbor's
> retention GC deleted the custom Postgres image a running workload still referenced. Symptom → decision
> tree → root cause → fix → prevention. This is the registry/debugging modules made painful and concrete.

## Symptom
A pod stuck in **`ImagePullBackOff`** / `ErrImagePull`; `kubectl describe pod` Events show:
```
Failed to pull image "harbor.10.0.0.200.nip.io/library/postgresql:18.4.0-noavx512":
  manifest unknown / not found
```
The app had been running fine for days, then a reschedule (node drain, restart, scale) couldn't pull the
image that *used to* be there. (A running pod survives on the node's cached image until it's rescheduled —
so the registry deletion is invisible until something forces a fresh pull. Classic time-bomb.)

## Decision tree (narrow the layer — Module 8)
```
ImagePullBackOff
   │
   ├─ Does the tag/digest exist in the registry NOW?   ← start here
   │     crane manifest harbor…/library/postgresql:18.4.0-noavx512   → "not found"  → YES, this is it
   │        why gone?  retention/GC deleted it · or a digest-pinned public base got paywalled
   │
   ├─ Auth?            imagePullSecret present + valid?  (ghcr-pull) · shadowed HARBOR_PASSWORD?
   ├─ Registry up?     Harbor reachable? (Longhorn volume flapping → 502/timeout)
   └─ Right ref?       verifying a moved tag vs the digest? typo in the image name?
```
Order matters: **"does it exist in the registry"** is the cheapest, most common cause — check it first,
not auth.

## Root cause (this incident)
Harbor runs a **tag-retention policy (keep-N) + daily GC** to bound the controller disk. The custom
**`postgresql:…-noavx512`** image (rebuilt without AVX-512 so pgvector doesn't SIGILL on the old
ThinkPads — see `case-studies`/Module 0) lives **ONLY in Harbor** — it's not on any public registry. The
keep-N retention didn't know it was irreplaceable, aged it out, and GC deleted the blobs. Next reschedule
→ nothing to pull → `ImagePullBackOff` → every CNPG Postgres that used it degraded (litellm's DB among
them), ~6 days until noticed.

**The deeper lesson:** a GC policy that only knows "keep the newest N" will delete a **rarely-rebuilt but
load-bearing** image. Retention must understand *irreplaceability*, not just recency.

## Fix (what resolved it)
1. **Rebuild + re-push** the image to Harbor (restore the artifact) — `runbooks/rebuild-custom-postgres-base.md`.
2. **Add an always-retain rule** so GC can never delete it again: a Harbor retention rule
   `retain latestPushedK=100` scoped to `repository: postgresql*` (the repo has 1–2 tags → effectively
   keep-all). Verified via a retention **dry-run** (`postgresql retained=1 total=1`).
3. Let the pod reschedule / `kubectl rollout restart` → it pulls the restored image → Running.

```bash
# confirm the tag is back
crane manifest harbor.10.0.0.200.nip.io/library/postgresql:18.4.0-noavx512 >/dev/null && echo OK
# re-apply the retention rules after ANY Harbor DB wipe/rebuild (they're live-API config, not GitOps)
# → see ops-runbooks "Harbor retention + GC"; the postgresql* always-retain is rule #2
```

## Prevention
- **Always-retain irreplaceable images** (the `postgresql*` rule). Any image built only in Harbor and not
  reproducible on demand = protect it.
- **Never GC while a Longhorn volume is rebuilding** (heavy delete-I/O worsens the rebuild).
- **Durability upgrade (deferred):** move the custom base to **ghcr** (off Harbor's GC path, survives a
  Harbor rebuild) — blocked on a ghcr *write* token. Until then, re-apply the retention rule after any
  Harbor DB wipe.
- **Mirror + pin third-party bases you depend on** (the cousin incident: `docker.io` `minio/minio` went
  paywalled → digest-pinned refs `ImagePullBackOff` → mirrored to Harbor). Don't depend on an anonymous
  public pull for anything load-bearing.
- **Alerting:** a reschedule shouldn't be the first time you learn an image is gone — a periodic
  "does every referenced image still resolve?" check closes the time-bomb gap.

## Why mocks/CI never caught this
Nothing in the code or CI changed — the image built and passed months ago. This is a **registry
lifecycle** failure, invisible until a pull. It's the kind of thing only *operating* the system surfaces
(the reliability/Game-Day point, Module 8) — and exactly why "it built fine" ≠ "it'll run tomorrow."

## See also
- Registries/retention → `../10-registries/registries-and-promotion.md` · Debug method → `../08-debugging/debugging-methodology.md`
- The image itself → Module 0 (the SIGILL/noavx512 story) · Fast → `../cheatsheets/registry-cheatsheet.md` (ImagePullBackOff triage)
- Runbook → `../runbooks/rebuild-custom-postgres-base.md` (todo)
