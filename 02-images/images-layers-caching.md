# Module 2 — Images, Layers, Caching, Tags & Base Images

> Level 2 · Learning note (Type A). An image is the *artifact*; a container is a *running instance* of it.
> This module is where build speed, image size, and reproducibility are won or lost.

## Concept
An **image** is an immutable, content-addressed stack of **read-only layers** + a small **config** (env,
entrypoint, user, exposed ports). Each Dockerfile instruction that changes the filesystem produces **one
layer**. A container = those layers + a thin writable layer on top (Module 0).

## Problem it solves
Shipping a whole filesystem would be huge and slow. Layers make images **deduplicated, cacheable, and
incrementally pullable**: if two images share a base layer, it's stored/pulled **once**; if only your code
changed, only your code's layer rebuilds and re-pushes.

## How it works internally
- **Layer** = a tar of filesystem changes from the previous layer, identified by a **sha256 digest**.
- **Build cache**: for each instruction Docker computes a cache key (the instruction + the digest of its
  inputs). If the key is unchanged, it reuses the cached layer. **The first instruction whose input
  changed busts the cache for itself and everything after it** — so *order matters*.
- **Tag vs ID vs digest**: a **tag** (`:v1`, `:a7c17b4`) is a human pointer that can move; the **image ID**
  is the local config digest; the **repo digest** (`@sha256:…`) is the immutable content address. Prod
  should pin by digest or an immutable SHA tag.
- `.dockerignore` shrinks the **build context** sent to the daemon (and prevents `COPY . .` from baking
  `.git`, `node_modules`, secrets).

## Example
```dockerfile
COPY package*.json ./   # layer A — changes rarely
RUN npm ci              # layer B — cached as long as A is unchanged (fast rebuilds!)
COPY . .                # layer C — your source (changes often)
RUN npm run build       # layer D
```
If you instead `COPY . .` *before* `npm ci`, every source edit busts the deps layer → `npm ci` re-runs
every build. **Order cheap-and-stable first, expensive-and-volatile last.**

## Example from our organisation
- **The dep-cache split** is in every build: `platform-demo/Dockerfile` does `COPY go.mod` → `go mod
  download` *before* `COPY . .`; `ktayl-core`/`ktayl-claims` do `COPY pom.xml` → `mvn dependency:go-offline`
  before `COPY src`. Result: a one-line code change doesn't re-download the dependency universe.
- **Size via base choice**: `platform-demo` lands on `gcr.io/distroless/static-debian12:nonroot`
  (~2 MiB base + ~7 MiB Go binary); `ktayl-core` on `eclipse-temurin:21-jre` (hundreds of MiB — a JVM
  can't go static-distroless). Same multi-stage technique, very different final size — that's the
  base-image trade-off made concrete.
- **Digests in prod**: ghcr images are promoted by immutable short-SHA tag (`ktayl-core:a7c17b4`) and
  cosign signs the **digest**, not the tag — the signature can't be moved under you.
- **The all-digit-SHA bug** (`gitops.md`): a 7-hex SHA that happens to be all decimal (`9248482`) parses
  as a YAML *number* → `9.248482e+06` → `InvalidImageName`. Lesson: a tag is a *string*; quote it. Kargo's
  `yaml-update` wraps tags in `quote(...)` for exactly this.

## Architecture
```
Dockerfile                 Image = ordered read-only layers            Registry
  FROM  ──────────▶  [L0 base]                                   stored/pulled by digest,
  COPY deps ───────▶ [L1 deps]      (cached across code changes)  shared layers deduped
  RUN install ─────▶ [L2 vendored deps]
  COPY src ────────▶ [L3 your code]  ← first changed layer busts cache here down
  RUN build ───────▶ [L4 artifact]
                       + container writable layer (ephemeral)
```

## Commands
```bash
docker build -t app:dev .            # build (sends the context minus .dockerignore)
docker images                        # local layer cache on disk
docker history --no-trunc app:dev    # every layer + the instruction + size (find the fat layer)
docker inspect app:dev               # config: Env, Entrypoint, User, ExposedPorts, RootFS layers
docker image inspect --format '{{index .RepoDigests 0}}' ghcr.io/andrelair-platform/ktayl-core:a7c17b4
docker buildx build --no-cache .     # prove a cache-bust; compare build time vs cached
```

## Debugging
- **"Why is my build slow every time?"** → a volatile instruction (e.g. `COPY . .`) sits *above* an
  expensive one. Reorder. Check with `docker history`.
- **"Image is huge."** → `docker history` shows the heavy layer; usually build tools left in the runtime
  stage (fix: multi-stage, Module 3/12) or `COPY . .` dragging `node_modules`/`.git` (fix: `.dockerignore`).
- **"CI rebuilds everything."** → the CI cache key changed (a `CACHEBUST` arg, a lockfile reshuffle) or
  no registry/gha cache is configured.

## Production considerations
- Pin prod to an **immutable SHA tag or digest** (never `:latest`). Our hybrid: dev tags in Harbor, prod
  SHA tags in ghcr; retention keep-3/keep-5.
- Keep the runtime image **minimal** — fewer layers/packages = smaller attack surface + faster pulls +
  faster node cold-starts (matters on a 5-node lab).

## Security considerations
- Every layer is kept — **a secret `COPY`'d then `rm`'d in a later layer is still in the earlier layer**.
  Never bake secrets (use BuildKit `--mount=type=secret`, Module 13; or runtime env via ESO→Vault).
- A bloated base ships more CVEs → the Trivy CRITICAL gate (Module 9) rejects them. Distroless/slim pass more easily.

## Common mistakes
`COPY . .` before installing deps · no `.dockerignore` · tagging prod `:latest` · assuming a later `rm`
removes a secret from the image · confusing tag (movable) with digest (immutable).

## Key takeaways
1. Image = ordered **read-only layers** + config; container adds one ephemeral writable layer.
2. **Order instructions stable→volatile** so the dep layer caches across code changes.
3. Tag ≠ ID ≠ **digest**; pin prod by digest/immutable-SHA; quote all-digit SHAs.
4. `.dockerignore` controls the build context and keeps junk/secrets out.
5. Base-image choice is the biggest size/security lever (distroless vs slim vs jre).

## Exercise (on our project)
1. `docker history` the `platform-demo` image and the `ktayl-core` image — find each one's biggest layer
   and explain the size gap (distroless‑static vs JRE).
2. In `ktayl-iam/frontend/Dockerfile`, identify which instruction caches `npm ci` and what would break the
   cache. Then open its `.dockerignore` (or its repo's) — what would `COPY . .` have leaked without it?
3. Find where the project pins prod images by SHA (hint: `services/*/helm/values-prod.yaml`) and explain
   why not `:latest`.

## See also
- Next → `03-dockerfile/dockerfile-instructions.md` · Fast → `cheatsheets/images-cheatsheet.md` (todo)
- Prod/size → `12-production/production-docker.md` · Registries/digests → `10-registries/registries-and-promotion.md`
- Case → `case-studies/platform-demo-go-distroless.md` (todo)
