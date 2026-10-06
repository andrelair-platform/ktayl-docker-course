# Cheatsheet — Images, layers, caching, tags

| Problem | Command |
|---|---|
| Build an image | `docker build -t app:dev .` |
| Build with BuildKit explicitly | `DOCKER_BUILDKIT=1 docker build -t app:dev .` |
| Build a specific stage (debug builder) | `docker build --target build -t app:builder .` |
| Build on an **arm Mac → amd64 cluster** | `DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build …` |
| See layers + size (find the fat one) | `docker history --no-trunc app:dev` |
| See image config (user/entrypoint/env/ports) | `docker inspect app:dev` |
| Just the digest (immutable id) | `docker inspect --format '{{index .RepoDigests 0}}' app:dev` |
| Prove a cache bust (time it) | `time docker build . && time docker build --no-cache .` |
| Tag for a registry (SHA) | `docker tag app:dev harbor.10.0.0.200.nip.io/library/app:$(git rev-parse --short HEAD)` |
| Read a remote image's manifest | `docker buildx imagetools inspect ghcr.io/andrelair-platform/ktayl-core:<sha>` |
| Remove dangling layers | `docker image prune` |

**Caching rules (memorise):** order **stable→volatile**; put `COPY <deps-manifest>` + install **before**
`COPY . .` (the `go.mod`/`pom.xml`/`package*.json` split). First changed layer busts cache for all below.

**Tag vs digest:** tag = movable pointer; **digest `@sha256:…` = immutable** (what cosign signs; what prod
should pin). Quote all-digit SHAs (`tag: "9248482"`) or YAML parses them as a number → `InvalidImageName`.

**Size lever:** base image. distroless (Go) ≪ slim (Python) < jre (Java). `docker history` shows where the
bytes went — usually build tools left in the runtime stage (→ multi-stage) or `COPY . .` dragging junk (→ `.dockerignore`).

See also → `../02-images/images-layers-caching.md`
