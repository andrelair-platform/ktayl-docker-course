# Module 0 — Why Containers Exist

> Level 0 · Learning note (Type A). The mental model everything else hangs on. **Do not skip this.**
> We build the "why" before a single `docker` command — because if you understand the *problem*, every
> later feature (layers, volumes, networking, probes) becomes obvious instead of memorised.

---

## Concept

A **container** packages an application **together with everything it needs to run** — its runtime, its
libraries, its dependencies, and its filesystem layout — into one immutable artifact (an **image**) that
runs the **same way everywhere**, isolated from other processes on the host.

The mental model to hold:

```
  An application does NOT run in a vacuum. It silently depends on:

        Application code
            +
        Language runtime        (Go 1.25 / Python 3.12 / Node 22 / JRE 21 …)
            +
        Dependencies / libraries (pip pkgs, npm modules, .so files, pgvector…)
            +
        Configuration           (env vars, file paths, ports)
            +
        Operating-system assumptions (which libc? which CPU instructions? which user? which certs?)

  "It works on my machine" = all five happened to line up on YOUR machine.
  A container FREEZES all five into one image, so they line up EVERYWHERE.
```

The last line — **OS assumptions** — is the one beginners forget, and it's exactly where this project
got bitten (see the org example below).

---

## Problem it solves

**Before containers**, you shipped *code* and hoped the target environment matched the one you built on.
It rarely did. The classic failure modes:

- **"Works on my machine."** Dev has Python 3.12 + a system `libpq`; the server has 3.9 + a different
  one. The app crashes in a way it never did locally.
- **Dependency hell / conflict.** App A needs library v1, App B needs v2, both on the same VM — one wins,
  one breaks.
- **Slow, drift-prone provisioning.** A human (or a 300-line Ansible role) installs the runtime + deps on
  each server. Two servers drift. Nobody can say exactly what's installed.
- **Heavyweight isolation.** The old fix was "one VM per app" — a full guest OS (GBs, minutes to boot)
  just to isolate a 10 MB binary.

**What Docker solves:** it makes the *unit of delivery* the **whole runnable environment**, not just the
code. Build once → the image carries the runtime + deps + filesystem; run anywhere a container runtime
exists, identically, isolated, in milliseconds.

---

## How it works internally

A container is **not** a tiny VM. It is **a normal Linux process** that the kernel has been told to
*lie to*, using three native kernel features:

1. **Namespaces** — isolate *what a process can see*. Each container gets its own:
   - `pid` (its processes look like PID 1 + children; can't see the host's),
   - `net` (its own network interfaces, IPs, ports),
   - `mnt` (its own filesystem view — from the image),
   - `uts` (its own hostname), `ipc`, `user` (map container-root to a harmless host uid).
2. **cgroups** (control groups) — limit *how much a process can use*: CPU, memory, pids, IO. This is
   what `--memory`/`resources.limits` ride on. OOM-kill lives here.
3. **A layered (union) filesystem** — the image is a **stack of read-only layers**; the running container
   adds one thin **writable layer** on top (copy-on-write). This is why images are cache-friendly and why
   the container's own writes are **ephemeral** (Level 5).

So "containerisation" = `clone()` a process into fresh namespaces + attach cgroups + `pivot_root` into the
image's filesystem. There is no guest kernel — the container shares the **host kernel** (which is why a
Linux image can't run on a Windows kernel, and why CPU-instruction assumptions leak through — see below).

**The architecture of "Docker" itself** (and why the pieces matter later):

```
  docker CLI / buildx         ← you type here (a client)
        │  (REST API)
        ▼
  dockerd (Docker Engine)     ← builds images, manages containers  [DEV machines, CI runners]
        │
        ▼
  containerd                  ← the real container lifecycle manager (pull, unpack, run)
        │
        ▼
  runc                        ← the low-level tool that actually calls clone()/namespaces/cgroups
        │
        ▼
  Linux kernel
```

**OCI** (Open Container Initiative) standardises two things: the **image format** and the **runtime**. The
payoff is huge and shows up all over this project: an image you build with Docker/buildx is a *generic OCI
image* — so **containerd on the k3s nodes runs it without Docker Engine being installed at all** (see the
org example + `_meta/docker-to-kubernetes-map.md`).

---

## Example (simple)

```bash
docker run --rm hello-world        # pulls an OCI image, runs it as an isolated process, exits
docker run --rm -it alpine sh      # a 7 MB Linux userland; `ps` shows PID 1 = your shell (pid namespace)
```
Inside that `alpine sh`, `cat /etc/os-release` says Alpine even if your host is Ubuntu — the **mnt
namespace** gave the process the *image's* filesystem, not the host's. That's the whole trick in one line.

---

## Example from our organisation (the one that proves the point)

Two real specimens in **this** repo make "why containers" concrete — one shows the win, one shows the
subtle limit.

### (a) The win — `ktayl-underwriting` (Python/FastAPI)
`ktayl-underwriting/Dockerfile` freezes **Python 3.12 + a venv with the exact pinned deps + a non-root
user + the app**, and reads its `DATABASE_URL` **at runtime**. The identical image runs on the dev
namespace and the prod namespace with zero "install Python / pip install" steps and zero drift. That's the
*application + runtime + deps + config* model, boxed.

### (b) The subtle limit — `minicloud-postgresql-noavx512` (the SIGILL story)
This is the best teaching specimen in the whole estate. The platform runs Postgres with the `pgvector`
extension. The upstream (Bitnami) image's `pgvector` was compiled with **AVX-512** CPU instructions. The
cluster's older ThinkPads (i7-8565U / i7-10510U — Whiskey/Comet Lake) **don't have AVX-512**, so Postgres
crashed with **SIGILL (illegal instruction)** the moment a vector query ran.

The lesson is exact: **a container shares the host kernel and the host CPU.** The image was byte-identical
everywhere — but "the same image" still met a *different CPU* and failed. The container didn't magically
fix the OS/hardware assumption; it made the assumption **explicit and portable**, and then the fix was
clean: build a custom base image that recompiles pgvector `-mno-avx512*` (`minicloud-postgresql-noavx512/
Dockerfile`, `FROM harbor.../library/postgresql:18.4.0`). **Containers standardise *userspace*, not the
kernel or the silicon.** Remember this when someone says "containers mean it runs anywhere."

### (c) The runtime proof — k3s runs containerd, not Docker
On the cluster nodes there is **no Docker Engine**. We build images with Docker/buildx in CI, push them to
Harbor/ghcr, and **containerd** pulls + runs them. Proof that "Docker image" (an OCI artifact) and "Docker
Engine" (one daemon) are *different things* — a distinction most people never internalise. (More in L12.)

---

## Architecture — containers vs virtual machines

```
   VIRTUAL MACHINES                        CONTAINERS
   ┌───────────────┐ ┌───────────────┐     ┌──────────┐ ┌──────────┐ ┌──────────┐
   │   App A        │ │   App B        │     │  App A   │ │  App B   │ │  App C   │
   │   Bins/Libs    │ │   Bins/Libs    │     │Bins/Libs │ │Bins/Libs │ │Bins/Libs │
   │  GUEST OS (GB) │ │  GUEST OS (GB) │     └──────────┘ └──────────┘ └──────────┘
   ├───────────────┤ ├───────────────┤          │  namespaces + cgroups  │
   │         Hypervisor              │          ├────────────────────────┤
   ├─────────────────────────────────┤          │   SHARED HOST KERNEL   │
   │          Host OS / kernel        │         ├────────────────────────┤
   ├─────────────────────────────────┤          │        Host OS         │
   │            Hardware              │          ├────────────────────────┤
   └─────────────────────────────────┘          │        Hardware        │
                                                 └────────────────────────┘
   each app carries a full guest OS          apps share ONE kernel; isolate via namespaces/cgroups
   GBs, boots in minutes                     MBs, starts in milliseconds
   stronger isolation (separate kernel)      lighter, denser; isolation = kernel features (+ seccomp/caps)
```
**When each wins:** VMs when you need a *different kernel* or the strongest isolation boundary
(multi-tenant untrusted code). Containers when you want *density, speed, and identical userspace* — which
is 95% of application delivery, and why this whole platform is containers on k3s, not VM-per-app.

---

## Commands (orientation only — Level 1 goes deep)

```bash
docker run --rm -it alpine sh      # start an isolated process from an image
docker ps                          # running containers
docker ps -a                       # + stopped ones (and their EXIT CODE)
docker images                      # local image cache (layers on disk)
docker inspect <id>                # the full truth: config, mounts, networks, state
# the K8s twins (what you'll actually use here):
ssh controller "kubectl --context minicloud get pods -A"      # the cluster's 'docker ps'
ssh controller "kubectl logs <pod> -n <ns>"                   # the cluster's 'docker logs'
```

---

## Debugging (the Level-0 reflexes)
- **"It worked locally, not on the server."** → ask *which of the five* (runtime/deps/config/OS/CPU)
  differs. The container narrows it; the SIGILL case shows it can still be CPU.
- **Container exits immediately.** → `docker ps -a` for the **exit code**; `docker logs` for *why*. A
  container lives exactly as long as its PID 1 process; when PID 1 exits, the container stops (Level 1).
- **"Is this even a container problem?"** → `docker inspect`/`kubectl describe` before guessing.

---

## Production considerations
- The image is the **immutable unit of deployment** — same artifact dev→prod (this is why the platform's
  rule is *env-agnostic images*: read config at runtime, never bake it; it's what lets Kargo promote one
  SHA). 
- Containers are **cattle, not pets** — designed to be killed and recreated; anything that must survive
  goes on a volume (Level 5) or an external store.
- Density + fast start are *why* a 5-node lab can run ~90 services — a VM-per-service model couldn't.

## Security considerations
- A container is **weaker isolation than a VM** (shared kernel). That's why this platform layers
  **non-root + dropped capabilities + seccomp + read-only rootfs + admission control (Gatekeeper)** on top
  (Level 9) — containers alone are not a security boundary against a kernel exploit.
- "container-root" is still mapped toward host-root unless you drop it — hence the `USER 10001` in every
  app Dockerfile here and the Gatekeeper `require-non-root` gate.

## Common mistakes
- Thinking a container is a tiny VM (it's a *process* with a costume).
- Thinking "containerised" ⇒ "runs anywhere" (kernel + CPU still leak — the SIGILL case).
- Treating the container's writable layer as storage (it's ephemeral — Level 5).
- Confusing **Docker image** (OCI artifact) with **Docker Engine** (one daemon K8s no longer needs).

## Key takeaways
1. A container = **app + runtime + deps + config + userspace OS**, frozen into an image, run as an
   isolated **process** via **namespaces + cgroups + a layered filesystem**.
2. It standardises **userspace, not the kernel or the CPU** (SIGILL proof).
3. **OCI** decouples image from engine → you build with Docker, k3s runs it with **containerd**.
4. Lighter/faster/denser than VMs; weaker isolation → harden it (Level 9).
5. Image = immutable unit; writable layer = ephemeral; persistence is a separate decision (Level 5).

## Exercise (on our project)
1. `ssh controller "kubectl --context minicloud get pods -A | head"` — pick any Running pod. That *is* a
   container. Get its image (`kubectl get pod <p> -n <ns> -o jsonpath='{.spec.containers[0].image}'`).
2. Open `minicloud-postgresql-noavx512/Dockerfile`. In your own words: *what OS/hardware assumption broke,
   why did a byte-identical image still fail, and how did a custom base image fix it?* Write 4 sentences.
3. Confirm the runtime: find out what container runtime the k3s nodes use (hint: it's **not** Docker).
   What does that tell you about "Docker image" vs "Docker Engine"? → then read
   `_meta/docker-to-kubernetes-map.md` §"The one thing people get wrong".

## See also
- Next → Level 1 Docker basics (`01-fundamentals/` — images/containers/run/exec/logs) *(to be written)*
- Map  → `_meta/docker-to-kubernetes-map.md` (OCI, containerd, CRI)
- Case → `case-studies/postgresql-noavx512.md` (the SIGILL deep dive) *(to be written)*
- Inventory → `_meta/docker-inventory.md` (the whole laboratory)
