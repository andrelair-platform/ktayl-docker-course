# Module 10 — Registries & Image Promotion (Level 10)

> Learning note (Type A). A registry is where images **live** between build and run. The design choice
> that matters most in an enterprise: *which registry holds dev vs prod, and how does one artifact get
> promoted between them?*

## Concept
A **registry** stores OCI images as **repositories** of tagged/digested manifests + deduplicated layers.
You `push` from CI and `pull` at runtime (containerd). Tags are movable pointers; **digests** are
immutable content addresses.

## Problem it solves
You need a durable, access-controlled place to put images so any node can pull them, with a story for
retention (disk!), authentication, and **promotion** (dev image → prod image) that keeps "what's in prod"
provable.

## How it works internally — the registry landscape
| Registry | What it is | Used here for |
|---|---|---|
| **Docker Hub** | the public default | avoided for pulls (rate limits + the MinIO paywall bit us) |
| **GHCR** (`ghcr.io`) | GitHub's registry, free, durable, off our disk | **prod** images (signed + SBOM) |
| **Harbor** (self-hosted) | our registry on-cluster (`harbor.10.0.0.200.nip.io`) | **dev** images + the custom Postgres base |
| **ACR / ECR / GAR** | cloud-vendor registries | not used (we're self-hosted + ghcr) |

- **Tag schemes**: `:latest` (never for prod — a moving target), **semantic** `:v1.2.0` (human releases),
  **immutable SHA** `:a7c17b4` (what we actually deploy), **digest** `@sha256:…` (what cosign signs).
- **Auth**: a registry needs creds even to *read* a private repo → Kubernetes uses an **imagePullSecret**.
- **Retention + GC**: images pile up → disk fills. A retention policy keeps the last N; GC deletes
  unreferenced blobs.

## Example
```bash
docker tag app:dev harbor.10.0.0.200.nip.io/library/app:$(git rev-parse --short HEAD)
docker push harbor.10.0.0.200.nip.io/library/app:<sha>
docker pull ghcr.io/andrelair-platform/ktayl-core:a7c17b4
docker buildx imagetools inspect ghcr.io/andrelair-platform/ktayl-core:a7c17b4   # manifest + digest
```

## Example from our organisation — the hybrid model (`gitops.md`)
- **dev → Harbor, prod → ghcr.** CI dual-pushes `:<sha>` to both. Rationale: keep the **controller disk
  bounded** (Harbor is on-cluster storage) while prod images live durably **off local disk** on ghcr.
- **Retention**: Harbor keeps **keep-3** per repo (dev churns fast; Kargo runs the newest) + daily GC;
  ghcr mirror **keep-5**, cosign-aware. A real incident taught the exception: the custom
  **`postgresql:…-noavx512` image lives ONLY in Harbor and was GC'd out → litellm down 6 days** → a
  `postgresql*` **always-retain** rule. Lesson: *a GC policy must know which images are irreplaceable.*
- **The MinIO paywall**: `docker.io`/`quay.io` `minio/minio` stopped resolving anonymously → digest-pinned
  refs `ImagePullBackOff` → **mirror to Harbor** `library/minio` + protect it. Lesson: *pulling from a
  registry you don't control is a supply risk; mirror what you depend on.*
- **Visibility + pull secrets**: demo images → **Public** (k3s pulls anonymously); real services →
  **Internal/Private** + a `ghcr-pull` **ESO-rendered dockerconfigjson** secret referenced as
  `imagePullSecrets`. (GitHub has **no API** to set package visibility — flip it in the UI once.)
- **Signing/SBOM live with the prod registry**: cosign signs the **ghcr digest**; the SBOM is attested to
  it — so the registry is also the provenance store.

## Architecture — promotion (one immutable artifact)
```
CI build ──┬── push Harbor/library/app:<sha>   (DEV registry, keep-3 + GC)
           └── push ghcr/app:<sha>  + sign + SBOM   (PROD registry, keep-5)
                     │
                 Kargo (Warehouse watches it)
                     │  the SAME <sha> is promoted — never rebuilt
                 CODEOWNERS PR bumps prod overlay to app:<sha>
                     ▼
                 ArgoCD → k3s pulls ghcr/app:<sha> (via ghcr-pull secret)
```
Promotion = **move the digest, don't rebuild** → "what passed dev == what runs prod" is provable.

## Commands
```bash
# Harbor (dev) — from the controller
H=https://harbor.10.0.0.200.nip.io/api/v2.0
/usr/bin/curl -sk -u admin:$(cat ~/.harbor-admin) "$H/projects/library/repositories" | jq '.[].name'
# what a tag resolves to (digest) + the signature
cosign verify ghcr.io/andrelair-platform/ktayl-core:a7c17b4 --certificate-oidc-issuer … --certificate-identity-regexp …
ssh controller "kubectl get secret ghcr-pull -n ktayl-core -o jsonpath='{.type}'"   # kubernetes.io/dockerconfigjson
```

## Debugging
- **ImagePullBackOff** → tag/digest exists in the registry? (retention/GC may have deleted it — the
  postgresql case) · auth? (missing/expired `imagePullSecret`; the shadowed `HARBOR_PASSWORD`) · registry
  reachable? (Harbor on Longhorn flapping, MinIO paywall).
- **"pull works in CI, not on cluster"** → CI logged in with a token; the node needs an `imagePullSecret`.
- **"signature not found"** → verify the **digest**, not a moved tag.

## Production considerations
Prod = immutable SHA/digest from ghcr, signed + SBOM'd; dev = Harbor keep-3; GC daily but **never below
the deployed depth** and never GC while a Longhorn volume is rebuilding; always-retain irreplaceable
images; mirror critical third-party images you don't own.

## Security considerations
Only pull from **trusted, fully-qualified** registries (Gatekeeper allowed-registries); private repos +
pull secrets for real services; signed + scanned images; the registry is a supply-chain surface — a
poisoned base or a typo-squatted name is an attack (why we mirror + pin + sign).

## Common mistakes
`:latest` to prod · GC with no always-retain for irreplaceable images · depending on an anonymous public
pull (rate limits/paywalls) · forgetting the `imagePullSecret` · verifying a tag instead of a digest ·
repo-vs-org secret shadowing at push time.

## Key takeaways
1. **dev→Harbor, prod→ghcr**; CI dual-pushes the **same `:<sha>`**.
2. Deploy by **immutable SHA/digest**, never `:latest`; cosign signs the **digest**.
3. **Retention/GC must protect irreplaceable images** (the `postgresql*` always-retain scar).
4. **Mirror** critical third-party images (the MinIO paywall); don't depend on anonymous public pulls.
5. Private repos need an **imagePullSecret** (ESO→dockerconfigjson here).

## Exercise (on our project)
1. In `gitops.md`, find the Harbor retention rules — explain keep-3, the `postgresql*` always-retain rule,
   and the incident that created it.
2. Trace one image: where does `ktayl-core:<sha>` get pushed in CI, how does the cluster pull it (which
   secret), and where is its cosign signature?
3. Why is the custom `postgresql-noavx512` image the *most* dangerous one to GC? What protects it?

## See also
- CI (push) → `09-ci-cd/…` · Security (trust/sign) → `07-security/…` · Images/digests → `02-images/…`
- Runbook → `runbooks/rebuild-custom-postgres-base.md` (todo) · Fast → `cheatsheets/registry-cheatsheet.md` (todo)
