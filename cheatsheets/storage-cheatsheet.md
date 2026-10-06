# Cheatsheet — Storage, volumes, persistence

| Problem | Command |
|---|---|
| Create a named volume | `docker volume create pgdata` |
| Run with a volume (persists) | `docker run -v pgdata:/var/lib/postgresql/data postgres:16` |
| Bind-mount source for dev hot-reload | `docker run -v "$PWD":/app node:22` |
| RAM scratch / in-flight secret | `docker run --tmpfs /tmp …` |
| Immutable rootfs + writable scratch (prod) | `docker run --read-only --tmpfs /tmp app` |
| What's mounted where | `docker exec <c> df -h` · `docker inspect <c>` (`.Mounts`) |
| Inspect a volume (host path) | `docker volume inspect pgdata` |
| Remove unused volumes | `docker volume prune` |
| **K8s:** list claims | `kubectl get pvc -A` |
| **K8s:** a pod's mounts | `kubectl get pod <p> -n <ns> -o jsonpath='{.spec.containers[0].volumeMounts}'` |

**Decide in one line**
- real data (DB, uploads) → **volume / PVC** (survives `rm`/reschedule)
- dev source / a config file → **bind mount** (host-controlled)
- secrets/scratch you don't want on disk → **tmpfs**
- never → the container **writable layer** (ephemeral, dies with the container)

**Symptoms → cause**
| Symptom | Cause / fix |
|---|---|
| "data vanished after redeploy" | it was on the writable layer → add a volume/PVC |
| `EROFS` / "read-only file system" | `readOnlyRootFilesystem:true` + a write path → mount `emptyDir` at that exact path (crew-agent) |
| "permission denied" writing a volume | non-root uid doesn't own it → `fsGroup`/chown, or init the dir (also: a `0400` file → `0644`) |
| volume looks empty | bind path typo, or the mount shadows image content at that path |
| `down -v` ate my data | `-v` removes named volumes — omit it |

**Always back up the volume, never the container.** (kine/vault/db-backup CronJobs do exactly this.)

See also → `../05-storage/storage-volumes-persistence.md`
