# Module 5 — Storage: the Writable Layer, Volumes, Bind Mounts, tmpfs & Persistence

> Level 5 · Learning note (Type A). The single most important rule: **a container is not storage.**

## Concept
A running container adds **one thin writable layer** on top of the image's read-only layers. It is
**ephemeral** — deleted when the container is removed. Anything that must survive goes to a **volume**,
a **bind mount**, or **tmpfs** — storage that lives *outside* the container's lifecycle.

## Problem it solves
Databases, uploaded files, certs, and caches must outlive container restarts/replacements. Without
external storage, "redeploy" = "lose the data." Volumes decouple **data lifecycle** from **container
lifecycle** (cattle, not pets).

## How it works internally
| Mechanism | What it is | Lifecycle | Use for |
|---|---|---|---|
| **writable layer** | copy-on-write FS on top of the image | dies with the container | scratch only; never real data |
| **volume** | Docker-managed dir (`/var/lib/docker/volumes/…`) mounted in | independent; survives `rm` | **databases, persistent app data** |
| **bind mount** | a host path mounted in | host-controlled | **dev hot-reload** (mount your source), config files |
| **tmpfs** | RAM-backed mount | dies with the container, never on disk | secrets/scratch you don't want persisted |

Copy-on-write means writing to a path from the image **copies** it up into the writable layer first
(small write amplification) — another reason heavy writers want a volume.

## Example
```bash
docker volume create pgdata
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16   # volume → persists
docker rm -f db && docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16
#   ↑ data is still there. Without -v, the rm would have destroyed it.
docker run -d -v "$PWD":/app node:22 …        # bind mount: edit on host, see it live in the container (dev)
docker run --tmpfs /tmp …                      # RAM scratch
```

## Example from our organisation
- **Databases never store in the container**: every CNPG Postgres (per service) writes to a **PVC backed
  by Longhorn** — the Kubernetes form of a named volume. Kill/reschedule the pod → the PVC re-attaches,
  data intact. This is the postgresql-noavx512 image + a PVC: the *image* is replaceable, the *data* is not.
- **read-only root filesystem + emptyDir** (the writable-layer lesson, enforced): `crew-agent` runs with
  `readOnlyRootFilesystem: true` (M9 hardening) and the app tried to write `~/.local`, `~/.cache`, `/tmp`
  → **EROFS** crash. Fix = mount `emptyDir` volumes at exactly those paths. The container's own FS is
  read-only (immutable); only the declared scratch volumes are writable. Perfect illustration that the
  writable layer is a liability, not a feature.
- **Config/certs as mounts, not baked**: the CA-trust pattern mounts a ConfigMap (the minicloud CA) into
  nextcloud/bookstack via an initContainer rather than baking it — a "bind-mount a file in" at cluster scale.
- **Backups prove persistence is separate**: kine-backup / vault-snapshot / db-backup CronJobs copy the
  *volume* data to MinIO/R2 on a schedule. You back up the **volume**, never the container.

## Architecture
```
   Image layers (read-only)        ← replaceable: redeploy any time
        + writable layer           ← EPHEMERAL (dies on rm) — scratch only
        + volume  ───────────────▶ pgdata / PVC(Longhorn)  ← SURVIVES, backed up to MinIO/R2
        + bind mount ────────────▶ host path (dev source / config)
        + tmpfs ─────────────────▶ RAM (secrets/scratch, never disk)
```

## Commands
```bash
docker volume ls / inspect pgdata
docker run -v pgdata:/data …            # named volume
docker run -v "$PWD":/app …             # bind mount (dev)
docker run --read-only --tmpfs /tmp …   # immutable rootfs + RAM scratch (prod shape)
docker exec <ctr> df -h                 # see what's mounted where
# K8s twins:
ssh controller "kubectl get pvc -A"
ssh controller "kubectl -n <ns> get pod <p> -o jsonpath='{.spec.containers[0].volumeMounts}'"
```

## Debugging
- **"Data vanished after redeploy."** → it was on the writable layer, not a volume. Add a volume/PVC.
- **EROFS / "read-only file system"** → app writes to a path not covered by a writable mount under
  `readOnlyRootFilesystem` → add an emptyDir/volume at that exact path (the crew-agent fix).
- **"permission denied" writing a volume** → the non-root uid doesn't own the mount; set `fsGroup`/chown,
  or init the dir. (Related real gotchas: a `0400` CA file the app couldn't read → `0644`; the www-data
  `before-starting` hook that couldn't write `/etc/ssl/certs` → do it in a root initContainer.)
- **Mount empty / wrong content** → bind path typo, or a volume shadowing image content at that path.

## Production considerations
Stateful workloads → **StatefulSet + PVC** (stable identity + storage), `strategy: Recreate` for RWO
(avoid multi-attach). Back up the volume (RTO/RPO). Don't pin a stateful pod to a node — Longhorn is
network-attached, so it fails over (the no-hostname-pin rule). Prefer **read-only rootfs + emptyDir** for
stateless apps.

## Security considerations
`readOnlyRootFilesystem: true` is a strong control (an attacker can't drop a binary in the image FS) —
pair with emptyDir for the few writable paths. Use **tmpfs** for in-flight secrets (never hit disk). Never
bake secrets/data into a layer (M2/M9).

## Common mistakes
Treating the container as a database · forgetting the volume on a DB · writing outside the declared
writable mounts under read-only rootfs · baking config/certs instead of mounting · backing up the
container instead of the volume · hostPath pinning a stateful pod.

## Key takeaways
1. The writable layer is **ephemeral** — never real data.
2. **Volume** = persistent app data (DBs); **bind mount** = dev source/config; **tmpfs** = RAM scratch/secrets.
3. Data lifecycle ≠ container lifecycle — redeploy freely, data stays (PVC/Longhorn here).
4. `readOnlyRootFilesystem + emptyDir` is the hardened prod shape (crew-agent EROFS lesson).
5. Back up the **volume**; watch permissions for non-root users.

## Exercise (on our project)
1. Find a CNPG Postgres in `minicloud-gitops` — locate its PVC/storage and explain what happens to the
   data when its pod is deleted vs when the PVC is deleted.
2. Read the crew-agent manifest: list the `emptyDir` mounts and match each to a path the app writes. Why
   not just drop `readOnlyRootFilesystem`?
3. Lab L11: run `postgres` with a volume, write a row, `rm -f` the container, recreate with the same
   volume → confirm the row survives. Then do it *without* the volume and watch it vanish.

## See also
- Prod stateful → `12-production/…` · Security (read-only rootfs) → `07-security/…`
- K8s PVC → `11-kubernetes/…` + `_meta/docker-to-kubernetes-map.md` · Fast → `cheatsheets/storage-cheatsheet.md` (todo)
