# Case study — `platform-demo`: the gold-standard distroless Go image

> Type F (case study). The sharpest specimen in the estate: a **~6 MiB**, non-root, distroless, signed,
> SBOM'd production image. We read its Dockerfile line-by-line, then verify the real image from the
> registry. Source: `platform-demo/Dockerfile`.

## Why this is the reference
Tiny (fast pulls, fast node cold-start), minimal attack surface (no shell, no package manager, no libc
userland beyond what a static Go binary needs), non-root, reproducible-ish, signed + SBOM'd. It's what
"production image" should mean — and the counterpoint to the (necessarily) larger JVM images (ktayl-core).

## The Dockerfile, line by line
```dockerfile
# syntax=docker/dockerfile:1.6
FROM golang:1.25-alpine AS build          # (1) builder stage — the Go SDK (big, thrown away)
WORKDIR /src
COPY go.mod ./                            # (2) deps manifest FIRST → caches the module layer (M2)
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go mod download                       # (3) BuildKit cache mounts: persist pkg cache, NOT in the image (M13)
COPY . .                                  # (4) now the source (volatile) — below the cached deps
ARG VERSION="dev"
ARG COMMIT="unknown"                      # (5) build-time metadata (not runtime; not secret)
RUN --mount=type=cache,… \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \   # (6) STATIC build (no libc) → can run on scratch/distroless
    go build -trimpath \
      -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${COMMIT}" \  # (7) strip symbols (-s -w) → smaller
      -o /out/platform-demo .

FROM gcr.io/distroless/static-debian12:nonroot   # (8) runtime = ~2 MiB base, NO shell, nonroot by default
COPY --from=build /out/platform-demo /platform-demo   # (9) copy ONLY the binary (multi-stage payoff)
EXPOSE 9898                               # (10) documentation only — publishing is the K8s Service
USER nonroot                              # (11) non-root (Gatekeeper require-non-root is satisfied)
ENTRYPOINT ["/platform-demo"]             # (12) exec form → the binary is PID 1 → gets SIGTERM (graceful)
```
**The five techniques that make it gold:** (a) multi-stage — the 800 MB Go SDK never ships; (b)
`CGO_ENABLED=0` static build → runs on **distroless-static** (no libc/userland needed); (c) `-ldflags
"-s -w"` + `-trimpath` → smaller, reproducible-ish binary; (d) `USER nonroot` + a shell-less base =
minimal attack surface; (e) BuildKit **cache mounts** → fast rebuilds without bloating the image.

## Verified reality (pulled from ghcr with `crane`, 2026 — no Docker daemon)
```
image : ghcr.io/andrelair-platform/platform-demo:v0.1.2
index digest : sha256:59ff6034…           (+ a .sig and a .sbom tag → cosign-signed + SBOM attached)
amd64 manifest : sha256:3e5ff7ec…
config : arch=amd64 os=linux · User="nonroot" · Entrypoint=["/platform-demo"] · ExposedPorts=["9898/tcp"]
layers (13, compressed, over the wire):
   5.580 MiB   ← the Go binary (the only big layer)
   0.425 / 0.137 / 0.080 / 0.028 / 0.012 MiB  ← distroless base bits (ca-certs, base-files, tzdata…)
   0.000 MiB × 7   ← metadata/empty layers
   TOTAL ≈ 6.27 MiB
```
Everything the Dockerfile *claims* is **confirmed by the real image**: non-root, correct entrypoint,
port 9898, and the whole thing is ~6 MiB because the base is ~0.7 MiB and the stripped static binary is
~5.6 MiB. (`docker history` would show the uncompressed view; this is the compressed pull size.)

## What to notice (and reuse)
- **One fat layer, by design**: the binary dominates; nothing else matters. Contrast ktayl-core where the
  **JRE** is the fat base — a JVM app *can't* go distroless-static (it needs a JRE), so it's hundreds of
  MiB. Same multi-stage technique, different floor. That's the base-image trade-off, measured.
- **No shell = you can't `kubectl exec sh`** → use `kubectl debug` (Module 8). The security win (an
  attacker also gets no shell) costs you easy live poking — an accepted trade-off.
- **Signed + SBOM'd**: the `.sig`/`.sbom` sibling tags are cosign + syft outputs from CI (Module 9) — the
  supply-chain story is real, not slideware.

## Investigation answers (the drill, for this specimen)
- base image? `gcr.io/distroless/static-debian12:nonroot`. why? static Go binary needs no userland; smallest + non-root.
- stages? **2** (golang-alpine builder → distroless runtime).
- root? **no** — `USER nonroot` (verified in the config).
- port? `EXPOSE 9898` (doc) → published via the Service/Ingress, not the container.
- copied in? **only** `/platform-demo` (the binary). Nothing else.
- optimise? already near-optimal; the only lever left is `distroless/static:nonroot` → `scratch` (drop
  even ca-certs if it makes no TLS calls) — usually not worth losing the certs/tzdata/nonroot user.
- security issues? essentially none at the image layer — this is the target state.

## Reuse recipe (for any static-compiled service)
Multi-stage: SDK builder (+ cache mounts, deps-first) → `distroless/static:nonroot`; `CGO_ENABLED=0`
static build with `-ldflags "-s -w" -trimpath`; `COPY --from=build` the binary only; `USER nonroot`;
exec-form `ENTRYPOINT`. Then scan + sign + SBOM in CI.

## See also
- Images/size → `../02-images/images-layers-caching.md` · Dockerfile → `../03-dockerfile/dockerfile-instructions.md`
- Debug (distroless) → `../08-debugging/debugging-methodology.md` · Security/sign → `../07-security/container-security.md`
- Contrast → ktayl-core (JVM, can't go distroless-static) · Fast → `../cheatsheets/images-cheatsheet.md`
