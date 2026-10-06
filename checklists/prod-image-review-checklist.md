# Checklist — production image review (the gate you run on every Dockerfile/image)

> Type G (checklist). Run this before an image goes to prod. Each item is **pass/fail**. Most are also
> **admission-enforced** here (Gatekeeper) or **CI-enforced** (Trivy/cosign) — but review them *before*
> the gate rejects you. A fail = fix the Dockerfile/manifest, not a retry.

## 1. Dockerfile hygiene
- [ ] **Multi-stage** — build tools/SDK are NOT in the runtime stage.
- [ ] **Pinned base** — `:3.12-slim` / `:21-jre` / a digest, never `:latest`.
- [ ] **Minimal runtime base** — distroless/static (static binaries) · slim (interpreted) · jre (JVM); not a full OS.
- [ ] **Deps-manifest copied before source** — `COPY go.mod|pom.xml|package*.json` → install → `COPY . .` (cache).
- [ ] **`.dockerignore` present** — excludes `.git`, `node_modules`, `target`, secrets, local env.
- [ ] **`USER` non-root** (uid ≠ 0, e.g. 10001 / `nonroot`).
- [ ] **exec-form `ENTRYPOINT`/`CMD`** (`["…"]`) so the process is PID 1 and gets SIGTERM (graceful stop);
      if wrapped in a shell, it's `sh -c "exec …"`.
- [ ] **No secrets** in `ARG`/`ENV`/layers — build-time secrets via `--mount=type=secret`; runtime via Vault/ESO.
- [ ] **Config at runtime, not baked** — the SAME image must run dev+prod (env-agnostic → promotable by Kargo).
- [ ] **`EXPOSE`** documents the real listen port (publishing is the Service/Ingress).

## 2. Image / supply chain (CI)
- [ ] **Scanned** — Trivy CRITICAL gate passes (`--ignore-unfixed`); no ignored CRITICALs.
- [ ] **SBOM** attached (syft CycloneDX).
- [ ] **Signed** — cosign keyless signs the **digest**; `cosign verify` succeeds.
- [ ] **Immutable tag** — prod pins `:<sha>` (or a digest), dual-pushed Harbor(dev)+ghcr(prod).
- [ ] **Right arch** — built `--platform linux/amd64` (the cluster); no accidental arm-only image.
- [ ] **Small-ish** — `crane manifest`/`docker history` shows no surprise fat layer (leftover build tools / `COPY . .` junk).

## 3. Runtime / Kubernetes securityContext (what Gatekeeper enforces)
- [ ] `runAsNonRoot: true` + `runAsUser` (≠0).
- [ ] `allowPrivilegeEscalation: false`; **not** `privileged`.
- [ ] `readOnlyRootFilesystem: true` + an `emptyDir` for each real write path (no EROFS crash).
- [ ] `capabilities: { drop: ["ALL"] }` (at least drop `NET_RAW`).
- [ ] `seccompProfile: { type: RuntimeDefault }`.
- [ ] **Resource `requests`/`limits`** set (scheduling + OOM protection) + fits the namespace quota.
- [ ] **Probes** — readiness (gate traffic) + liveness (restart dead); startup if slow boot.
- [ ] **Logs to stdout/stderr** (collected by Loki) — no logging to a file; logging actually enabled.
- [ ] **No node-pin** on a stateful pod (Longhorn is network-attached; `strategy: Recreate` for RWO).
- [ ] **imagePullSecret** present for a private/internal image (ghcr-pull).

## 4. Registry / lifecycle
- [ ] Prod image on **ghcr** (durable), dev on **Harbor** — fully-qualified ref (Gatekeeper allowed-registries).
- [ ] If the image is **built only in Harbor and irreplaceable** → covered by an **always-retain** rule
      (the `postgresql*` scar — don't let GC delete a load-bearing image).
- [ ] Critical **third-party base images are mirrored** (not an anonymous public pull — MinIO paywall lesson).

## 5. Promotion / delivery
- [ ] Image reaches prod via **Kargo → CODEOWNERS PR → ArgoCD** (one immutable SHA promoted, not rebuilt).
- [ ] The **live QA gate** passed on dev (green CI is necessary, not sufficient).
- [ ] The image is **env-agnostic** (prerequisite for Kargo promotion).

---
**One-line verdict:** an image is prod-ready only if §1–§3 are all ✅ (the gate enforces §1 non-root, §2
scan/sign/registry, §3 securityContext). A ✅ across §4–§5 means it'll also *stay* runnable and reach prod safely.

## See also
- Deep dives → `../12-production/production-docker.md` · `../07-security/container-security.md`
- Reference image → `../case-studies/platform-demo-go-distroless.md` (passes every item)
- Security fast → `../cheatsheets/security-cheatsheet.md`
