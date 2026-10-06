# Cheatsheet — Registries (Harbor dev · ghcr prod), scan/sign

| Problem | Command |
|---|---|
| Tag for Harbor (dev) by SHA | `docker tag app:dev harbor.10.0.0.200.nip.io/library/app:$(git rev-parse --short HEAD)` |
| Login to Harbor | `docker login harbor.10.0.0.200.nip.io -u admin` (pw: controller `~/.harbor-admin`) |
| Push to Harbor | `docker push harbor.10.0.0.200.nip.io/library/app:<sha>` |
| Login to ghcr (prod) | `echo $GH_PAT \| docker login ghcr.io -u <user> --password-stdin` |
| Pull prod image | `docker pull ghcr.io/andrelair-platform/ktayl-core:<sha>` |
| List Harbor repos (API, from controller) | `/usr/bin/curl -sk -u admin:$(cat ~/.harbor-admin) https://harbor.10.0.0.200.nip.io/api/v2.0/projects/library/repositories \| jq '.[].name'` |
| Read a remote manifest + digest | `docker buildx imagetools inspect ghcr.io/andrelair-platform/ktayl-core:<sha>` |
| **Scan** for CRITICALs (CI gate) | `trivy image --severity CRITICAL --ignore-unfixed <ref>` |
| **SBOM** (CycloneDX) | `syft <ref> -o cyclonedx-json=sbom.json` |
| **Sign** the digest (keyless) | `cosign sign --yes ghcr.io/andrelair-platform/app@<digest>` |
| **Verify** signature | `cosign verify <ref@digest> --certificate-oidc-issuer … --certificate-identity-regexp …` |
| See signatures + SBOM attached | `cosign tree <ref>` |
| K8s pull secret present? | `kubectl get secret ghcr-pull -n <ns> -o jsonpath='{.type}'` → `kubernetes.io/dockerconfigjson` |

**Model:** dev→**Harbor** `:<sha>` (keep-3 + daily GC) · prod→**ghcr** `:<sha>` (keep-5, signed+SBOM).
Promote the **same SHA** (Kargo), never rebuild. Pin prod by **digest/SHA**, never `:latest`.

**ImagePullBackOff — fast triage:** tag still exists? (retention/GC may have removed it — `postgresql*` is
always-retain) · `imagePullSecret` present + valid? · registry reachable? (Harbor on Longhorn flapping /
a paywalled public base) · verifying a **digest** not a moved tag?

**Gotchas:** repo-level `HARBOR_PASSWORD` shadows the org one → `docker login` 401 (`gh secret list --repo`
AND `--org`). Mirror critical third-party images you don't own (the MinIO paywall). Cosign signs the
**digest**, so verify the digest.

See also → `../10-registries/registries-and-promotion.md` · `../09-ci-cd/docker-ci-cd.md`
