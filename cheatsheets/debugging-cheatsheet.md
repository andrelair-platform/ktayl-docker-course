# Cheatsheet — Debugging ("it's broken")

**Narrow the layer top-down** (don't guess): running? → logs → config → pull → network → storage → resources.

| Step | docker | kubectl (cluster) |
|---|---|---|
| Is it up? restarts? | `docker ps -a` | `kubectl get pod <p> -n <ns>` |
| **Why** did it die / not start | `docker inspect --format '{{.State.ExitCode}} {{.State.Error}}' <c>` | `kubectl describe pod <p> -n <ns>` (Events, State, Reason) |
| Logs | `docker logs --tail=200 <c>` | `kubectl logs <p> -n <ns>` |
| Logs of the crashed prior run | — | `kubectl logs <p> -n <ns> -p` |
| Inside (has a shell) | `docker exec -it <c> sh` | `kubectl exec -it <p> -n <ns> -- sh` |
| Inside (distroless, no shell) | sidecar: `--pid container:<c> netshoot` | `kubectl debug -it <p> -n <ns> --image=nicolaka/netshoot --target=<ctr>` |
| Resources / OOM | `docker stats` | `kubectl top pod -n <ns>` ; describe → `OOMKilled` |
| Network / DNS | `docker exec <c> nslookup <svc>` | `kubectl exec <p> -n <ns> -- nslookup <svc>.<ns>.svc` |
| Image pull failing | `docker pull <ref>` | describe → `ImagePullBackOff` reason |
| Cluster-level clues | — | `kubectl get events -n <ns> --sort-by=.lastTimestamp` |

**Exit codes:** `0` PID1 exited (not a foreground server) · `1/2` app error · `137` SIGKILL/**OOM** · `143` SIGTERM.

**Symptom → likely layer (from real incidents)**
| Symptom | Layer / cause |
|---|---|
| `ImagePullBackOff` | tag GC'd (retention) · auth (missing `imagePullSecret`, shadowed `HARBOR_PASSWORD`) · paywalled public pull |
| `CrashLoopBackOff` + `EROFS` | storage: read-only rootfs + unmounted write path → emptyDir |
| app runs but **no logs** | not "fine" — a disabled logger (the alembic `fileConfig` case); verify on the live pod |
| `InvalidImageName` `9.xe+06` | config: all-digit SHA parsed as a number → quote the tag |
| `connect=000` right after rollout | networking, **transient** CNI programming → wait, not a bug |
| `exec format error` | arch mismatch (arm image on amd64 node) → fix `--platform` |
| `exec sh` fails | distroless (no shell) → `kubectl debug` |

**Golden rule:** read the exit code / Events **before** restarting. Restarting hides the evidence.

See also → `../08-debugging/debugging-methodology.md` · `docker-vs-kubectl-cheatsheet.md`
