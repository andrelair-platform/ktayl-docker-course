# Docker competency map — skills → where this project proves them

What you should be able to *do*, mapped to the real specimen that proves it. Mirrors the platform's
proof-catalog discipline: a skill is "owned" only when you can reproduce it on the real example (the **R**
column in the README progress tracker).

| Competency | What "owning it" means | Proven here on… | Level |
|---|---|---|---|
| **Why containers** | explain the problem containers solve + when NOT to use them | the postgresql-noavx512 SIGILL story; ktayl-solution-web deliberately off-cluster | 0 |
| **Containers vs VMs / internals** | namespaces, cgroups, layered FS, OCI, containerd/runc | k3s uses containerd (no Docker Engine); Gatekeeper limits/caps | 0 |
| **Run & inspect containers** | run/exec/logs/ps/inspect/stats; lifecycle & exit codes | any dev container; `kubectl` equivalents | 1 |
| **Read a Dockerfile** | explain any line + spot issues | all 25 Dockerfiles (start: ktayl-underwriting) | 2–3 |
| **Write a Dockerfile** | design a correct, small, non-root multi-stage build | you wrote ktayl-core's | 3 |
| **Image layers & caching** | order instructions for cache hits; read `docker history` | the `COPY go.mod` / `COPY pom.xml` dep-cache split | 2 |
| **Image optimisation** | choose a base; cut size; multi-stage | platform-demo distroless (~9 MiB) vs Java/JRE (~hundreds MiB) — the trade-off | 2, 8 |
| **Networking** | bridge/host/none, DNS, ports, service-to-service | ktayl-core → NATS/ERPNext/Postgres over in-cluster DNS | 4 |
| **Storage / persistence** | writable layer vs volume vs bind mount vs tmpfs | CNPG Postgres PVC; readOnlyRootFS + emptyDir (crew-agent) | 5 |
| **Docker Compose** | model a multi-service local stack | retrieva + globalcore-legacy compose | 6 |
| **Containerise each stack** | Next/Node/FastAPI/Go/Java/DB/worker/job | the per-stack Dockerfiles | 7 |
| **Production hardening** | non-root, distroless, health, limits, immutability, graceful stop | every prod image + Gatekeeper + K8s probes/limits | 8 |
| **Security** | caps, seccomp, read-only rootfs, scan, SBOM, signing, trusted registries | cosign + syft + Trivy gate + Gatekeeper admission | 9 |
| **Registries & promotion** | dev vs prod registry, tags, immutable SHA, promotion | Harbor→ghcr hybrid + Kargo | 10 |
| **CI/CD for images** | build→test→scan→sign→push→GitOps→deploy | the ktayl-core/platform-demo CI | 11 |
| **Docker in Kubernetes** | map every Docker concept to its K8s form; CRI/containerd | the Deployment/StatefulSet/Job/sidecar table | 12 |
| **Advanced build** | BuildKit cache mounts, buildx, multi-arch, SBOM/provenance, rootless | platform-demo cache mounts; the amd64-only pin + the arm ThinkPad reality | 13 |
| **Debugging** | a systematic container-failure methodology | the real incidents (ImagePullBackOff, CrashLoop, EROFS, Multi-Attach) | all |

**Self-assessment questions** you should be able to answer by the end (interview-grade):
why is *this* app containerised? how should it be? how is its image built + where does it live? how does
it reach prod? how is it secured? how is its data persisted? how does it talk to other services? how do I
troubleshoot it? how does Kubernetes actually run it (and why not via Docker Engine)?
