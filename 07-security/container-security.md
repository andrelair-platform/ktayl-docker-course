# Module 7 — Container Security (Level 9)

> Learning note (Type A). A container is **not** a security boundary by default (shared kernel, Module 0).
> Enterprise security = many thin layers: non-root, dropped caps, seccomp, read-only rootfs, scanned +
> signed images, trusted registries, runtime secrets, admission control. For a **regulated insurer**
> (DORA/GDPR) this is mandatory, not optional.

## Concept
Defence in depth across the image's whole life: **build** (minimal base, no baked secrets, scan, SBOM,
sign) → **admit** (policy gate at deploy) → **run** (non-root, least capability, read-only, isolated).

## Problem it solves
A single control fails open: a root container + a kernel CVE = host compromise; an unsigned image = supply
-chain swap; a baked secret = leaked key. Layering controls means one failure doesn't become a breach.

## How it works internally — the controls
| Control | What it does | Our enforcement |
|---|---|---|
| **non-root** (`USER`, runAsNonRoot) | process isn't uid 0 → kernel-exploit blast radius shrinks | every app Dockerfile `USER 10001`/`nonroot`; **Gatekeeper `require-non-root` HARD-denies root** (ns NotIn exemptions) |
| **drop capabilities** | remove Linux caps the app doesn't need (NET_RAW, SYS_ADMIN…) | **Gatekeeper `block-net-raw`** (must drop NET_RAW) |
| **no privileged** | `--privileged` = basically host-root; banned | admission denies it |
| **seccomp / AppArmor** | restrict syscalls / file ops | `seccompProfile: RuntimeDefault` on jobs/pods |
| **read-only rootfs** | image FS immutable at runtime → can't drop a binary | `readOnlyRootFilesystem: true` (+ emptyDir, Module 5) |
| **minimal base** | fewer packages = fewer CVEs | distroless (platform-demo), slim (Python), jre (Java) |
| **no baked secrets** | secrets never in a layer/ARG/ENV | **ESO→Vault**; BuildKit `--mount=type=secret` for build-time (the CA) |
| **vulnerability scan** | fail the build on known CRITICALs | **Trivy CRITICAL gate** in CI |
| **SBOM** | a bill of materials for the image | **syft** CycloneDX attached to the ghcr image |
| **signing** | prove provenance; detect tampering | **cosign keyless** signs the digest |
| **trusted registries** | only pull from approved, fully-qualified refs | **Gatekeeper allowed-registries** (bare names denied) |

## Example
```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot   # minimal, no shell, non-root by default
COPY --from=build /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```
```yaml
securityContext:               # the runtime half (K8s)
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

## Example from our organisation
- **Admission is the backstop, not hope**: Gatekeeper/OPA denies root, bare image names, NET_RAW, and
  hostname-pins at the API server — so even if a Dockerfile forgot `USER`, the deploy is rejected. (The
  nats-box stream-init Jobs run `runAsNonRoot:1000`, drop ALL caps, read-only, seccomp RuntimeDefault —
  a textbook hardened one-shot container.)
- **The scan gate caught a real one**: `ktayl-core` BILL-010 shipped on Spring Boot 3.4.1; the **Trivy
  CRITICAL gate rejected the image** (vulnerable transitive tomcat/spring-security). The fix was Boot
  3.5.16 + a pinned tomcat. *The gate did its job — a CVE never reached a registry.* Best proof that
  "scan in CI" is not ceremony.
- **Keyless cosign + SBOM** on every prod (ghcr) image → supply-chain provenance (a cornerstone of the
  DORA ICT-risk story; also the Retrieva thesis).
- **Secrets never in images**: DB/Stripe/ERPNext creds come from **ESO→Vault** at runtime; the one
  build-time secret (the minicloud CA) is passed as a **BuildKit secret file**, never a build-arg —
  because a multi-line PEM in `--build-arg` truncates to its first line *and* a build-arg would be baked
  into image history (a leak). Real gotcha, real fix.
- **Insurance framing**: PII + money → identity-is-the-perimeter (BYOD), default-deny egress (exfil), and
  auditability. Containers alone don't give any of that; these controls do.

## Architecture
```
BUILD                         ADMIT                      RUN
minimal base ─ no secrets ─▶  Gatekeeper/OPA        ─▶   non-root · drop caps · read-only rootfs
Trivy scan (CRITICAL gate)    (non-root, registries,     seccomp · no-privilege · netpol default-deny
syft SBOM · cosign sign        no NET_RAW, no hostpin)   secrets from Vault at runtime
```

## Commands
```bash
trivy image --severity CRITICAL --ignore-unfixed ghcr.io/andrelair-platform/ktayl-core:<sha>
syft <image> -o cyclonedx-json
cosign verify <image@digest> --certificate-identity-regexp … --certificate-oidc-issuer …
docker run --user 10001 --read-only --cap-drop ALL --security-opt no-new-privileges <img>
ssh controller "kubectl get constrainttemplates,constraints -A"     # the live Gatekeeper policies
```

## Debugging
- **Pod denied at deploy** → read the Gatekeeper message (`require-non-root` / allowed-registries /
  block-net-raw) — the fix is in the manifest/Dockerfile, not a retry.
- **EROFS under read-only rootfs** → add emptyDir at the write path (Module 5).
- **cosign verify fails** → wrong identity/issuer, or the tag moved (verify the **digest**).
- **Trivy fails CI** → a real fixable CVE (bump the dep/base, as with the Boot 3.5.16 fix) — don't
  `--ignore` your way past a CRITICAL.

## Production considerations
All controls on by default; exemptions are explicit + justified (e.g. the linuxserver s6 images that need
root land in the `require-non-root` NotIn list — documented, not silent). Immutable, signed, scanned
images only. Secrets via Vault/ESO; rotate; break-glass in Vaultwarden.

## Security considerations
This whole module *is* the security consideration. The meta-point: **a green build is necessary, not
sufficient** — the live QA gate + admission + runtime policy are the rest.

## Common mistakes
Root containers · `--privileged` for convenience · secrets in ARG/ENV/layers · `:latest` from an
untrusted registry · ignoring a CRITICAL to "unblock" · assuming the container isolates you from the
kernel (it doesn't).

## Key takeaways
1. A container is **not** a boundary alone — layer controls (non-root, caps, seccomp, read-only).
2. **Admission (Gatekeeper) enforces** what Dockerfiles should do — root/bare-image/NET_RAW denied.
3. **Scan (Trivy) + SBOM (syft) + sign (cosign)** in CI = supply-chain integrity (it caught a real CVE).
4. **No baked secrets** — runtime from Vault/ESO; build-time via BuildKit secret, never build-arg.
5. For a regulated insurer, this is compliance evidence (DORA/GDPR), not polish.

## Exercise (on our project)
1. Read `ktayl-core/.github/workflows/ci.yml` — find the Trivy, syft, and cosign steps. What severity gate?
   what does cosign sign — the tag or the digest?
2. Find the Gatekeeper `require-non-root` exemption list — which namespaces are exempt and *why* (hint:
   linuxserver/s6 images)? Why is "exempt + documented" better than "disable the policy"?
3. Open the postgresql-noavx512 or a frontend Dockerfile — identify every security control present (USER,
   minimal base, npm removal) and one you'd add.

## See also
- Production → `12-production/…` · CI (where scan/sign live) → `09-ci-cd/…` · Registries (trust) → `10-registries/…`
- Checklist → `checklists/prod-image-security-checklist.md` (todo) · ADR → `adr/adr-001-distroless-vs-slim-vs-jre.md` (todo)
