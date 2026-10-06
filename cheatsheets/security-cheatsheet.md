# Cheatsheet — Container security (run hardened, scan, sign, verify)

| Need | Command / snippet |
|---|---|
| Run hardened locally | `docker run --user 10001 --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges --memory 512m app` |
| Check who the image runs as | `docker inspect --format '{{.Config.User}}' <img>` (empty = **root**, bad) |
| Scan for CRITICALs (the CI gate) | `trivy image --severity CRITICAL --ignore-unfixed <ref>` |
| Generate an SBOM | `syft <ref> -o cyclonedx-json=sbom.json` |
| Sign (keyless) + attest SBOM | `cosign sign --yes <ref@digest>` ; `cosign attest --predicate sbom.json --type cyclonedx <ref@digest>` |
| Verify provenance | `cosign verify <ref@digest> --certificate-oidc-issuer … --certificate-identity-regexp …` |
| Build-time secret (no leak) | `docker buildx build --secret id=ca,src=./ca.crt .` (never `--build-arg`) |
| See the live admission policies | `kubectl get constrainttemplates,constraints -A` (Gatekeeper) |

**Runtime securityContext (the K8s half — what Gatekeeper enforces):**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

**The 7 defaults for a prod image**
1. non-root `USER` · 2. minimal/distroless base · 3. multi-stage (no build tools) · 4. no secrets in
layers/ARG/ENV (runtime via Vault/ESO) · 5. read-only rootfs + emptyDir · 6. drop ALL caps + no-privilege
· 7. scanned + SBOM'd + signed.

**Enforced here, not hoped:** Gatekeeper HARD-denies root / bare-image-names / NET_RAW / hostname-pins;
Trivy CRITICAL gate blocks the push (it caught a real stale Spring Boot); cosign+syft on every prod image.

**Don't:** `--privileged` · root container · secrets in `--build-arg` (baked in history — immutable) ·
ignore a CRITICAL to "unblock" · trust that the container isolates you from the kernel (it doesn't).

See also → `../07-security/container-security.md` · `checklists/prod-image-security-checklist.md` (todo)
