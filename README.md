# Docker — a project-based course & knowledge base (ktayl-solution / minicloud)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Docker](https://img.shields.io/badge/Docker-OCI%20%7C%20BuildKit-blue)
![Kubernetes](https://img.shields.io/badge/runtime-k3s%20%2F%20containerd-326ce5)

> **Learn Docker from a real, self-hosted enterprise platform** — 25 Dockerfiles across Go / Python /
> Node / Java / legacy, a custom Postgres base image, Harbor + ghcr registries, and a full
> CI → sign → SBOM → scan → Kargo → ArgoCD → k3s(containerd) pipeline. From *"why containers exist"* to
> *"production container architecture in Kubernetes."*

> **Simulation environment.** `ktayl-solution` is a **fictional** insurance company used as an
> enterprise-architecture + engineering laboratory. No real customer data or production insurance
> operations are involved. All examples are synthetic; internal hostnames (`*.10.0.0.200.nip.io`) are
> Tailscale-only and not publicly reachable.

---

> This is **not** a generic Docker course. Every concept is taught against the **real containers in this
> organisation** — the insurance IS (ktayl) running on the self-hosted platform (minicloud). The whole
> repo tree is your **laboratory**: 25 Dockerfiles across Go / Python / Node / Java / legacy, a custom
> Postgres base image, Harbor + ghcr registries, Kargo → ArgoCD → k3s(containerd), Gatekeeper admission,
> CNPG/Vault StatefulSets, CA-trust initContainers, Vault sidecars, stream-init Jobs, and a full
> CI → sign → SBOM → scan → promote → deploy pipeline.

**Goal:** go from *"why do containers exist"* to *"I can look at any enterprise app and reason about why
it's containerised, how its image is built, where it lives, how it reaches prod, how it's secured, how
its data persists, how it talks to other services, how I debug it, and how Kubernetes runs it."*

---

## How to use this knowledge base (navigate 5 ways)

### 1. By learning objective — follow the curriculum

→ [`_meta/curriculum.md`](_meta/curriculum.md) — 14 levels, each with concept · why · internals · org
example · labs · debugging · production · security · mistakes · takeaways · exercise.
Recommended order of study is in the curriculum (don't skip Level 0).

### 2. By operational problem — "something's broken"

→ [`troubleshooting/`](troubleshooting/) + [`cheatsheets/`](cheatsheets/).
Examples: *"my container can't reach Postgres"*, *"image won't build in CI"*, *"pod CrashLoopBackOff"*.

### 3. By technology / stack

Go → [`case-studies/platform-demo-go-distroless.md`] · Python/FastAPI → ktayl-underwriting ·
Next.js → ktayl-iam-frontend · Java/Spring → ktayl-core & ktayl-claims · custom DB → postgresql-noavx512.
Mapped in → [`_meta/project-examples.md`](_meta/project-examples.md).

### 4. By project / service

Which real service demonstrates which concept → [`_meta/project-examples.md`](_meta/project-examples.md)
and the per-service deep dives in [`case-studies/`](case-studies/).

### 5. By topic

Jump to the level folder: fundamentals · images · dockerfile · networking · storage · compose · security ·
debugging · ci-cd · registries · kubernetes · production · advanced.

---

## Folder structure

```
docker/
├── README.md                 ← you are here (the entry point)
├── _meta/                     ← course deliverables (inventory, curriculum, maps, taxonomy, labs)
│   ├── docker-inventory.md          # every place we use Docker/OCI (the laboratory map)
│   ├── competency-map.md            # skills → where proven in this project
│   ├── curriculum.md                # the 14 levels + progression + recommended order
│   ├── note-taxonomy.md             # the kinds of notes + when to write each
│   ├── lab-roadmap.md               # the hands-on labs (project-based)
│   ├── project-examples.md          # concept → real service/Dockerfile map
│   └── docker-to-kubernetes-map.md  # every Docker concept → its K8s equivalent here
│
├── 01-fundamentals/           ← Level 0–1: why containers, vs VMs, engine/containerd/runc/OCI, namespaces/cgroups
├── 02-images/                 ← Level 2: images, layers, caching, tags, .dockerignore, base images
├── 03-dockerfile/             ← Level 3: every instruction, when/why
├── 04-networking/             ← Level 4: bridge/host/none, DNS, ports, service-to-service
├── 05-storage/                ← Level 5: writable layer, volumes, bind mounts, tmpfs, persistence
├── 06-compose/                ← Level 6: Compose stacks (local dev)
├── 07-security/               ← Level 9: non-root, caps, seccomp, read-only rootfs, scan, SBOM, signing
├── 08-debugging/              ← Level: the troubleshooting methodology
├── 09-ci-cd/                  ← Level 11: git push → build → scan → sign → push → GitOps → deploy
├── 10-registries/             ← Level 10: Harbor (dev) + ghcr (prod), tags, promotion
├── 11-kubernetes/             ← Level 12: Docker concepts → Pod/PVC/Service/probes; containerd/CRI
├── 12-production/             ← Level 8: non-root, distroless, health, limits, immutability
├── 13-advanced/               ← Level 13: BuildKit, buildx, multi-arch, SBOM, provenance, rootless
│
├── labs/                      ← runnable, project-based labs
├── case-studies/              ← per-service deep dives (the laboratory specimens)
├── troubleshooting/           ← problem → diagnosis → fix (operational)
├── runbooks/                  ← step-by-step operational procedures
├── adr/                       ← Docker architecture decisions (why Alpine vs distroless, etc.)
├── checklists/                ← production / security / dockerfile-review checklists
└── cheatsheets/               ← fast "problem → command" references
```

> All 14 level modules are written: Modules 0 **and 1** in `01-fundamentals/` + one learning note per
> level folder `02`–`13`. `cheatsheets/` (11, done); `labs/`, `case-studies/`, `troubleshooting/`,
> `runbooks/`, `adr/`, `checklists/` fill in as you do the labs (each module's *See also* lists the paired
> notes-to-write).
> Note taxonomy (Learning note vs Cheatsheet vs Runbook vs ADR vs Troubleshooting vs Case study vs
> Checklist) → [`_meta/note-taxonomy.md`](_meta/note-taxonomy.md).

---

## Progress tracker

Legend per module: **T** theory understood · **L** lab completed · **P** project example analysed ·
**R** can reproduce independently.

| Level | Module                                                                                        | T | L | P | R |
| ----- | --------------------------------------------------------------------------------------------- | :-: | :-: | :-: | :-: |
| 0     | Why containers exist · containers vs VMs · engine/containerd/runc/OCI · namespaces/cgroups | ⬜ | ⬜ | ⬜ | ⬜ |
| 1     | Docker basics (images, containers, run/exec/logs/ports/env)                                   | ⬜ | ⬜ | ⬜ | ⬜ |
| 2     | Images — layers, caching, tags, .dockerignore, base images                                   | ⬜ | ⬜ | ⬜ | ⬜ |
| 3     | Dockerfile engineering (every instruction)                                                    | ⬜ | ⬜ | ⬜ | ⬜ |
| 4     | Networking (bridge/host/none, DNS, service-to-service)                                        | ⬜ | ⬜ | ⬜ | ⬜ |
| 5     | Storage (volumes, bind mounts, tmpfs, persistence)                                            | ⬜ | ⬜ | ⬜ | ⬜ |
| 6     | Docker Compose (local dev stacks)                                                             | ⬜ | ⬜ | ⬜ | ⬜ |
| 7     | Application containerisation (Next/Node/FastAPI/Go/Java/DB/worker/job)                        | ⬜ | ⬜ | ⬜ | ⬜ |
| 8     | Production Docker                                                                             | ⬜ | ⬜ | ⬜ | ⬜ |
| 9     | Security                                                                                      | ⬜ | ⬜ | ⬜ | ⬜ |
| 10    | Registries (Harbor + ghcr, promotion)                                                         | ⬜ | ⬜ | ⬜ | ⬜ |
| 11    | Docker + CI/CD                                                                                | ⬜ | ⬜ | ⬜ | ⬜ |
| 12    | Docker and Kubernetes                                                                         | ⬜ | ⬜ | ⬜ | ⬜ |
| 13    | Advanced container engineering                                                                | ⬜ | ⬜ | ⬜ | ⬜ |

Update a cell to ✅ as you go. The matrix mirrors the platform's own proof discipline
(`minicloud-gitops/.claude/rules/evidence-and-proof.md`): a module is only "done" at **R**.

---

## Start here

1. Read [`_meta/docker-inventory.md`](_meta/docker-inventory.md) — see the whole Docker estate you'll learn from.
2. Read **Module 0** → [`01-fundamentals/module-00-why-containers-exist.md`](01-fundamentals/module-00-why-containers-exist.md).
3. Follow [`_meta/curriculum.md`](_meta/curriculum.md) in order.
