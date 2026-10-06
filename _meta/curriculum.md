# Curriculum — 14 levels, progression & recommended order

Each module follows the same **teaching format** (consistent across the whole course):
> Concept · Problem it solves · How it works internally · Example · **Example from our organisation** ·
> Architecture (diagram) · Commands · Debugging · Production considerations · Security considerations ·
> Common mistakes · Key takeaways · **Exercise (on our project)**.

Two depths are always present: the simple (`docker run nginx`) **and** the enterprise path
(`source → BuildKit → OCI image → SBOM → scan → registry → GitOps → containerd → Pod → cgroups/namespaces`).

## Recommended order of study
Linear **0 → 13**, with two rules: (1) **do not skip Level 0** (the mental model makes everything else
click); (2) each level's **lab + project-example** before moving on (theory alone is not "done"). Fast
path if time-boxed: 0 → 2 → 3 → 4 → 5 → 6 → 11 → 12 (covers 80% of day-to-day), then 7–10, 13.

## The levels

### Level 0 — Container foundations  → `01-fundamentals/`  ✴ **Module 0 written**
Why containers exist; the "works on my machine" problem; containers vs VMs; Docker architecture (CLI →
daemon → containerd → runc); OCI; **namespaces** (pid/net/mnt/user), **cgroups** (cpu/mem), layered
filesystem. **Org hook:** the postgresql-noavx512 SIGILL (same image, wrong CPU) + k3s runs **containerd,
not Docker Engine**. Labs: run/inspect a container, see its namespaces/cgroups.

### Level 1 — Docker basics  → `01-fundamentals/`
images vs containers; pull/run/stop/rm/inspect/logs/exec/ps; ports; env vars; the container **lifecycle**
and **exit codes**. **Org hook:** every dev container; map each command to its `kubectl` twin.

### Level 2 — Docker images  → `02-images/`
Dockerfile basics; **build context**; **layers**; **caching** (why instruction order matters); tags vs
image IDs vs digests; `.dockerignore`; base-image choice; size. **Org hook:** the `COPY go.mod` /
`COPY pom.xml` dep-cache split in platform-demo & ktayl-core; distroless vs slim vs alpine sizes.

### Level 3 — Dockerfile engineering  → `03-dockerfile/`
FROM · RUN · COPY vs ADD · WORKDIR · ENV · ARG · EXPOSE · CMD vs ENTRYPOINT · USER · HEALTHCHECK · LABEL —
when and *why* each. **Org hook:** line-by-line walk of ktayl-underwriting (Python venv), ktayl-iam
frontend (Next standalone), platform-demo (distroless), ktayl-core (Java + `sh -c "exec java $JAVA_OPTS"`).

### Level 4 — Networking  → `04-networking/`
bridge/host/none; custom networks; container **DNS**; EXPOSE vs publish; service-to-service.
**Org hook:** ktayl-core → `nats.messaging.svc` / `erpnext.erp.svc` over in-cluster HTTP; why Compose DNS
≠ K8s Service DNS; the default-deny NetworkPolicy posture.

### Level 5 — Storage  → `05-storage/`
the container **writable layer** (ephemeral!); volumes vs bind mounts vs tmpfs; persistence; permissions;
backup. **Org hook:** CNPG Postgres PVC (container ≠ storage); readOnlyRootFilesystem + emptyDir for
`~/.cache`/`/tmp` (crew-agent EROFS fix); Longhorn as the volume backend.

### Level 6 — Docker Compose  → `06-compose/`
services, networks, volumes, depends_on, env, healthchecks, profiles; local dev stacks. **Org hook:**
retrieva's compose + globalcore-legacy (Java/SOAP/MySQL). Lab: a frontend + API + Postgres stack.

### Level 7 — Application containerisation  → (per-stack, in `03`/`12`)
Containerise Next/React, Node/NestJS, FastAPI, Django, Go, Java, **workers**, **scheduled jobs**,
**databases**, **AI/LLM**. Compare strategies. **Org hook:** the stack table from the inventory — each is
a real Dockerfile; contrast Go-static-distroless vs Python-venv vs Node-standalone vs Java-JRE.

### Level 8 — Production Docker  → `12-production/`
non-root; minimal/distroless base; Alpine trade-offs (musl!); multi-stage; HEALTHCHECK vs K8s probes;
graceful shutdown (SIGTERM); resource limits; logging to stdout; secrets; config; immutability (env at
**runtime**, not baked). **Org hook:** the env-agnostic-image rule (Kargo needs it); Gatekeeper enforcing
non-root + limits; prod replicas.

### Level 9 — Security  → `07-security/`
isolation; root vs non-root; Linux capabilities; privileged; seccomp/AppArmor; read-only rootfs; secrets;
**vulnerability scanning**, **SBOM**, **image signing**, trusted registries, supply chain. **Org hook:**
Gatekeeper (require-non-root, allowed-registries/fully-qualified, drop NET_RAW); cosign keyless + syft SBOM
+ Trivy CRITICAL gate (which *rejected* a stale Spring Boot — a real catch); Vault/ESO for secrets.

### Level 10 — Registries  → `10-registries/`
Docker Hub vs GHCR vs Harbor vs ACR; auth; tags (semantic vs SHA vs immutable); promotion between envs.
**Org hook:** the **hybrid Harbor(dev)+ghcr(prod)** model, keep-3/keep-5 retention, GC, the `postgresql*`
always-retain rule, the MinIO-paywall/digest-pin gotcha.

### Level 11 — Docker + CI/CD  → `09-ci-cd/`
`git push → CI → test → docker build → scan → tag → push → GitOps update → ArgoCD → K8s`. **Org hook:**
the real ktayl-core/platform-demo GitHub Actions (Tailscale + CA trust + buildx + dual-push + Trivy +
cosign + SBOM), then **Kargo** promotes the one immutable artifact.

### Level 12 — Docker and Kubernetes  → `11-kubernetes/`
Every Docker concept → its K8s equivalent (full table → `docker-to-kubernetes-map.md`); OCI images;
containerd; **CRI**; kubelet; Pods. **Why modern K8s dropped Docker Engine (dockershim) but still runs
OCI images you build with Docker.** **Org hook:** the whole k3s cluster.

### Level 13 — Advanced container engineering  → `13-advanced/`
BuildKit (cache mounts, secrets); buildx; **multi-platform / ARM vs AMD64**; caching strategies; remote
builders; image manifests/indexes; SBOM/provenance; reproducible builds; rootless. **Org hook:**
platform-demo's `--mount=type=cache`; the `--platform linux/amd64` pin + the arm MacBook/ThinkPad reality;
the CA-as-BuildKit-secret-file pattern (why build-args truncate a multi-line PEM).

## Progression (what "done" means per level)
`Theory understood → Lab completed → Project example analysed → Can reproduce independently`.
Track in the README matrix. A level is only **R** ("owned") when you can redo the lab on the real service
without notes — that's the interview-ready bar.
