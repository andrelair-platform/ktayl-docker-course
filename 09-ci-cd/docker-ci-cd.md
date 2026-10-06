# Module 9 — Docker + CI/CD (Level 11)

> Learning note (Type A). How source code becomes a running container **without a human touching a
> server**: build → prove → sign → push → GitOps → reconcile. This is where Docker meets the delivery
> machine.

## Concept
CI builds + proves + signs the **immutable image**; GitOps (a repo of desired state) + a reconciler
(ArgoCD) + a promoter (Kargo) move that *one artifact* through environments. **CI builds & proves; Kargo
promotes; ArgoCD deploys.** Nobody runs `docker push` or `kubectl apply` by hand.

## Problem it solves
Manual build/push/deploy is slow, unauditable, and drift-prone ("what's actually in prod?"). A pipeline
makes delivery **reproducible, auditable, and gated** — and makes the image, not the code, the unit that
flows to prod.

## How it works internally — the pipeline
```
git push (main)
   ▼  GitHub Actions (CI)
 L0/L1/L2 tests  →  docker buildx build  →  Trivy CRITICAL gate  →  cosign sign + syft SBOM
   ▼                     dual-push: Harbor(:sha, dev)  +  ghcr(:sha, prod)
 image published (immutable SHA)
   ▼  Kargo (watches the built image / the commit)
 opens a GitOps PR bumping the prod image tag  →  CODEOWNERS review  →  squash-merge (signed)
   ▼  ArgoCD (reconciles the GitOps repo → cluster)
 k3s rolls out the new image (canary/Rollout brake) → Pods on containerd
```

## Example (the shape of the CI job)
```yaml
- uses: docker/setup-buildx-action@v4
- uses: docker/build-push-action@v7
  with: { context: ., push: true, tags: "harbor…/app:${{sha}}\nghcr…/app:${{sha}}",
          cache-from: type=gha, cache-to: type=gha,mode=max }
- uses: aquasecurity/trivy-action@…   # severity: CRITICAL, exit-code 1
- run: cosign sign --yes ghcr…/app@${digest}
- run: syft ghcr…/app:${sha} -o cyclonedx-json=sbom.json && cosign attest --predicate sbom.json …
```

## Example from our organisation (read `ktayl-core/.github/workflows/ci.yml` + `platform-demo`)
- **`build-test` job** runs `mvn verify` (incl. the Modularity + L2 Testcontainers tests) on every PR;
  **`build-and-push`** runs only on `main` push.
- **Tailscale + CA trust**: the runner joins the tailnet (`tag:ci`) and trusts the **minicloud CA** so it
  can `docker push` to the internal Harbor (TLS). (Only needed for Harbor; ghcr is public TLS.)
- **Dual-push**: Harbor (`:<sha>`, dev) **and** ghcr (`:<sha>`, prod, durable, off local disk).
- **Gate + provenance**: **Trivy CRITICAL** (which *rejected* the stale Spring Boot — Module 7), **cosign
  keyless** signs the digest, **syft** attaches a CycloneDX SBOM.
- **Promotion is Kargo, not CI**: CI stops at "image published". **Kargo** then promotes the one immutable
  SHA dev→prod via a **CODEOWNERS-gated PR** (`git`-Warehouse for ktayl-core: Freight = a commit, both
  images pinned to `commitFrom(...).ID[0:7]`). ArgoCD reconciles. This separation is deliberate: driving a
  deploy synchronously from CI is a sync-over-async anti-pattern.
- **The CA-as-BuildKit-secret-file pattern** (platform-demo): a multi-line PEM can't be a `--build-arg`
  (truncates to line 1 + bakes into history) → staged to a file and mounted via `secret-files:`; a
  `CACHEBUST` arg busts the CA layer per commit. (ktayl-core needs no CA in-image — in-cluster HTTP.)
- **The recurring CI-secret trap**: a stale **repo-level** `HARBOR_PASSWORD` *shadows* the org-level one →
  `docker login harbor` 401. Check `gh secret list --repo` AND `--org`.
- **Signed-commits gotcha**: Kargo's promotion commits are unsigned but `main` requires signatures →
  **merge the prod PR with `--squash`** (GitHub makes one signed squash commit).

## Architecture
```
Developer ─push─▶ GitHub ─▶ CI(build·test·scan·sign·SBOM) ─▶ Harbor(dev)+ghcr(prod)
                                                               │
                                                     Kargo ────┘  (promote 1 immutable SHA)
                                                       │ opens CODEOWNERS PR → squash-merge
                                                       ▼
                                                 GitOps repo ─▶ ArgoCD ─▶ k3s/containerd ─▶ Pods
```

## Commands
```bash
gh run list --repo andrelair-platform/ktayl-core          # CI runs
gh run view <id> --log | grep -iE 'trivy|cosign|syft|push'# trace build→scan→sign→push
docker buildx build --cache-to type=gha …                 # the CI cache mechanism
ssh controller "kubectl get application -n argocd"         # ArgoCD sync state
# Kargo: Freight/Stage/Promotion CRs in the service's kargo-project namespace
```

## Debugging
- **CI fails at `docker login harbor` (401)** → repo-level secret shadowing the org one (delete it).
- **Trivy fails** → a real CRITICAL; bump the dep/base (the Boot 3.5.16 fix), don't ignore it.
- **Build re-runs everything** → no `cache-from/to` or a changed `CACHEBUST`/lockfile.
- **Prod PR won't merge** ("commits must be verified") → merge with `--squash`.
- **Deployed SHA ≠ expected** → check Kargo Freight/Promotion + the ArgoCD synced revision (Module 12).

## Production considerations
Prod = immutable SHA from ghcr, signed + SBOM'd + scanned; promotion is CODEOWNERS-gated; ArgoCD auto-sync
+ a canary/Rollout brake is the runtime safety. The **live QA gate** (adversarial test on dev) must pass
before the prod PR — green CI is necessary, not sufficient.

## Security considerations
Least-privilege CI creds (Vault-sourced org secrets), keyless OIDC signing (no long-lived key), SBOM +
scan as supply-chain controls, no secrets in build-args/logs, signed commits on `main`.

## Common mistakes
Pushing/deploying by hand · `:latest` to prod · skipping the scan/sign · driving the deploy from CI
instead of GitOps · repo-vs-org secret shadowing · un-squashed Kargo prod merges.

## Key takeaways
1. **CI builds & proves; Kargo promotes; ArgoCD deploys** — one immutable SHA flows, nobody hand-deploys.
2. Build → **Trivy gate** → **cosign** → **syft SBOM** → dual-push (Harbor dev / ghcr prod).
3. The CA goes in as a **BuildKit secret file**, never a build-arg (truncation + history leak).
4. Promotion is a **CODEOWNERS-gated GitOps PR**, squash-merged (signatures).
5. Green CI ≠ shippable — the live QA gate + canary are the rest.

## Exercise (on our project)
1. Open `ktayl-core/.github/workflows/ci.yml`: identify the test gate, the Trivy severity, where cosign
   signs, and why `build-and-push` is `main`-only. Trace one real run in `gh run view`.
2. Explain why promotion is Kargo (a GitOps PR) and not a `kubectl apply` in the CI job — what does that
   buy (audit, gate, reconciliation)?
3. Find the `platform-demo` CA-as-secret-file steps; explain what breaks if you pass the CA as a
   `--build-arg` instead.

## See also
- Registries → `10-registries/…` · Security (scan/sign) → `07-security/…` · K8s deploy → `11-kubernetes/…`
- Runbook → `runbooks/push-an-image-to-harbor.md` (todo) · Case → `case-studies/ktayl-core-ci-pipeline.md` (todo)
