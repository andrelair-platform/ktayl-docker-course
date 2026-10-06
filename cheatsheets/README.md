# Cheatsheets — fast "problem → command" (Type B)

Optimised to find an answer in **seconds**. Each is one screen of `Problem → Command`. For *why*, follow
the link to the matching deep-dive (Type A). Project-grounded: real registries, real `kubectl`.

> **Environment shorthands used everywhere below**
> - Cluster is reached from the controller: `ssh controller "kubectl --context minicloud <...>"`
>   (default context is `myAKSCluster` — **always pass `--context minicloud`**).
> - Internal HTTPS (minicloud CA): `/usr/bin/curl --cacert ~/minicloud-ca.crt ...` (NOT anaconda curl).
> - Registries: **dev** `harbor.10.0.0.200.nip.io/library/<name>` · **prod** `ghcr.io/andrelair-platform/<name>:<sha>`.
> - The nodes run **containerd**, not Docker Engine → on a node use `crictl`, not `docker`.

| Cheatsheet | Use when |
|---|---|
| [docker-cli](docker-cli-cheatsheet.md) | run / ps / logs / exec / inspect / stats — the daily core |
| [docker ↔ kubectl twins](docker-vs-kubectl-cheatsheet.md) | "what's the `kubectl` for this `docker` command?" |
| [images](images-cheatsheet.md) | build / tag / history / size / digest |
| [dockerfile](dockerfile-cheatsheet.md) | which instruction, exec-form, non-root, multi-stage |
| [networking](networking-cheatsheet.md) | DNS, ports, "can't reach service X" |
| [storage](storage-cheatsheet.md) | volumes / bind / tmpfs / "data vanished" / EROFS |
| [compose](compose-cheatsheet.md) | up/down/logs/exec, local dev stacks |
| [debugging](debugging-cheatsheet.md) | "it's broken" — the narrow-down commands + exit codes |
| [registry](registry-cheatsheet.md) | Harbor/ghcr push/pull, cosign/trivy/syft, ImagePullBackOff |
| [security](security-cheatsheet.md) | run hardened, scan, sign, verify |

See also → `../08-debugging/debugging-methodology.md` (the method) · `../_meta/docker-to-kubernetes-map.md`.
