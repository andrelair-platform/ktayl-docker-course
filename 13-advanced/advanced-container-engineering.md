# Module 13 — Advanced Container Engineering

> Learning note (Type A). The build-system internals + supply-chain + multi-arch topics that separate
> "I can write a Dockerfile" from "I understand how images are actually produced and trusted." Grounded in
> the real gotchas this platform hit.

## Concept
Modern image builds run on **BuildKit** (the engine behind `docker build`/`buildx`): a DAG of build steps
with content-addressed caching, secret mounts, cache mounts, and **multi-platform** output, producing OCI
**image manifests/indexes** with optional **SBOM + provenance** attestations.

## Problem it solves
Naive builds are slow (no cache), leak secrets (build-args in history), are single-arch, and are
unverifiable. BuildKit + buildx + attestations make builds **fast, secret-safe, multi-arch, and
provable** — the enterprise/supply-chain bar.

## How it works internally — the topics
| Topic | What it is | The gotcha / why it matters here |
|---|---|---|
| **BuildKit** | the DAG build engine (`# syntax=docker/dockerfile:1`) | parallel steps, better cache; enabled via buildx |
| **cache mounts** | `RUN --mount=type=cache,target=…` | persist a pkg cache across builds **without** baking it into a layer (platform-demo: Go mod + build cache) |
| **secret mounts** | `RUN --mount=type=secret,id=…` / `secret-files:` | a build-time secret that **never lands in a layer/history** (the minicloud **CA**) |
| **buildx** | the multi-driver, multi-platform front end | `docker buildx build --platform …` |
| **multi-arch** | one tag → a manifest **index** of per-arch images | our cluster is **amd64**; the MacBook is **arm64** |
| **image manifest / index** | the JSON that lists layers (manifest) or per-arch manifests (index) | `buildx imagetools inspect` to read it; digests live here |
| **SBOM / provenance** | CycloneDX bill-of-materials + how/where built | syft SBOM + cosign attest (Module 7/9) |
| **reproducible builds** | byte-identical output across builds (pinned `SOURCE_DATE_EPOCH`) | **broke Kargo's NewestBuild** — see below |
| **rootless** | build/run without a root daemon | smaller attack surface for the builder |
| **remote/gha cache** | share the build cache across CI runs | `cache-from/to type=gha` |

## Example
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app .
RUN --mount=type=secret,id=ca,target=/tmp/ca.crt  update-ca-certificates   # secret never in a layer
```
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t app:multi --push .
docker buildx imagetools inspect app:multi      # the manifest index (both arches + digests)
```

## Example from our organisation (the advanced gotchas are all real)
- **Cache mounts** — `platform-demo/Dockerfile` uses `--mount=type=cache` for the Go module + build cache:
  fast rebuilds, and the cache is **not** in the final image (it stays in BuildKit). Textbook.
- **Secret-file, not build-arg** — the minicloud **CA** is a *multi-line PEM*. As a `--build-arg` it
  **truncates to its first line** *and* bakes into image history (leak). Fix: stage it to a file and mount
  via `secret-files:` (platform-demo CI) + a `CACHEBUST` arg to bust the CA layer per commit. (ktayl-core
  needs no CA in-image — it talks in-cluster HTTP.)
- **Multi-arch reality** — CI pins `--platform linux/amd64` because the **k3s cluster is amd64**; builds
  run fine on an amd64 runner. On the **arm64 MacBook**, `docker build` of these images needs emulation or
  a cross-build — which is why the local-build path sets `DOCKER_DEFAULT_PLATFORM=linux/amd64`. Lesson:
  *an image is arch-specific; "runs anywhere" stops at the CPU architecture* (cousin of the Module 0 SIGILL
  point — one is instruction set within an arch, this is the arch itself).
- **Reproducible build broke promotion** — `ktayl-policy-service` (Go/distroless) pinned `SOURCE_DATE_EPOCH`
  → `config.created` was *identical to the nanosecond* across commits → **Kargo's `NewestBuild` couldn't
  rank tags** and auto-promotion stalled silently. The fix was to switch that service's Warehouse to the
  **git model** (Freight keyed on the commit, not the image timestamp). A gorgeous advanced lesson: a
  *good* practice (reproducibility) interacting badly with a *build-date-based* promoter. (Also: a
  temurin/JRE base has a **fixed base-layer date** → same NewestBuild failure → git-Warehouse — why
  ktayl-core/claims use git.)
- **gha cache** — the CI uses `cache-from/to type=gha` so layers persist across runs (fast CI).
- **The image is an OCI artifact** — which is exactly why containerd runs it without Docker Engine
  (Module 0/11); `buildx imagetools inspect` shows the manifest + the digest cosign signs.

## Architecture
```
 buildx/BuildKit (DAG, cache mounts, secret mounts)
        │ --platform linux/amd64[,arm64]
        ▼
 OCI image (manifest)  ──or──  OCI index (per-arch manifests)
        │ + SBOM (syft) + provenance (cosign attest)
        ▼
 registry (ghcr/Harbor) by DIGEST ──▶ containerd pulls the arch-matching manifest ──▶ runc
```

## Commands
```bash
docker buildx build --platform linux/amd64 --cache-to type=gha,mode=max --cache-from type=gha -t app:<sha> --push .
docker buildx build --secret id=ca,src=./ca.crt .            # build-time secret, not in history
docker buildx imagetools inspect ghcr.io/andrelair-platform/ktayl-core:a7c17b4   # manifest + digests
cosign tree ghcr.io/andrelair-platform/ktayl-core:a7c17b4    # signatures + SBOM/attestations attached
```

## Debugging
- **CI cache never hits** → missing `cache-from/to`, or a `CACHEBUST`/lockfile churn changes the key.
- **`exec format error` at runtime** → arch mismatch (arm image on amd64 node or vice-versa) — check the
  manifest arch vs the node.
- **Kargo won't auto-promote** → the image timestamp isn't monotonic (reproducible build / fixed base-layer
  date) → use the **git-Warehouse** model, don't hand-edit the tag.
- **Secret leaked into image** → it was a build-arg/ENV, not a secret-mount → move to `--mount=type=secret`;
  rebuild (history is immutable, so the old image stays poisoned — re-tag/replace).

## Production considerations
Prefer cache mounts (speed, no bloat) + secret mounts (no leaks); pin the platform you deploy to; attach
SBOM + provenance; know your promoter's ranking model (git vs image) so reproducible/JRE images don't
stall it; consider rootless builders for a smaller CI attack surface.

## Security considerations
Build-time secrets via `--mount=type=secret` only (never ARG/ENV — baked + in history); SBOM + provenance
+ signatures make the artifact verifiable; reproducible builds help *integrity* (same source → same bits)
even though they tripped our *promoter* — the fix was the promoter, not dropping reproducibility.

## Common mistakes
Secrets as build-args · assuming one image runs on every CPU arch · ignoring the promoter's build-date
assumption (NewestBuild stall) · no CI cache · leaving build tools/caches in the final image · hand-editing
a tag to route around a broken Warehouse (the anti-pattern Kargo exists to remove).

## Key takeaways
1. **BuildKit** powers modern builds: cache mounts (fast, no bloat) + **secret mounts** (no leaks).
2. A multi-line secret (the CA) must be a **secret file/mount**, never a `--build-arg` (truncates + bakes).
3. **Images are arch-specific** — pin `--platform`; the cluster is amd64, the Mac is arm64.
4. A **reproducible build / fixed base-layer date breaks timestamp-based promotion** → use Kargo's git-Warehouse.
5. Ship **SBOM + provenance + signature**; `buildx imagetools`/`cosign tree` reveal the manifest + attestations.

## Exercise (on our project)
1. Read `platform-demo/Dockerfile`'s cache mounts + its CI's CA-secret-file steps. Explain what each cache
   mount speeds up and why the CA can't be a build-arg.
2. Find where `ktayl-policy-service` / `ktayl-claims` use Kargo's **git**-Warehouse — explain the
   NewestBuild timestamp problem (reproducible build / temurin base-layer date) that forced it.
3. `docker buildx imagetools inspect` a ghcr image: read its manifest, note the arch + digest, then
   `cosign tree` it to see the signature + SBOM. Why does containerd not need Docker Engine to run it?

## See also
- Images/digests → `02-images/…` · CI → `09-ci-cd/…` · Security/SBOM/sign → `07-security/…` · Registries → `10-registries/…`
- Fundamentals (OCI) → `01-fundamentals/module-00-why-containers-exist.md` · K8s/containerd → `11-kubernetes/…`
