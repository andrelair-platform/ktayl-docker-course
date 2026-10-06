# Cheatsheet — Docker CLI (daily core)

| Problem | Command |
|---|---|
| What's running? | `docker ps` |
| What ran/stopped (+ **exit code**)? | `docker ps -a` |
| Run a throwaway shell | `docker run --rm -it alpine sh` |
| Run detached + named | `docker run -d --name api app:dev` |
| Run with env + port + volume | `docker run -e DATABASE_URL=… -p 8000:8000 -v data:/var/lib/… app` |
| Logs | `docker logs <c>` |
| Follow logs, last 100 | `docker logs -f --tail=100 <c>` |
| Logs of a **crashed** prior run | `docker logs <c>` (already the last run) · K8s: `kubectl logs -p` |
| Shell into a running container | `docker exec -it <c> sh` (or `bash`) |
| …image has **no shell** (distroless) | run a sidecar debugger: `docker run --rm -it --pid container:<c> --net container:<c> nicolaka/netshoot` |
| The full truth (state, env, mounts, net) | `docker inspect <c>` |
| Just the exit code | `docker inspect --format '{{.State.ExitCode}}' <c>` |
| Live CPU/mem | `docker stats` |
| Processes inside | `docker top <c>` |
| Copy a file out/in | `docker cp <c>:/path ./local`  ·  `docker cp ./f <c>:/path` |
| Stop / kill / remove | `docker stop <c>` · `docker kill <c>` · `docker rm -f <c>` |
| Clean up everything unused | `docker system prune -af --volumes` ⚠️ (deletes volumes!) |
| List images / remove one | `docker images` · `docker rmi <img>` |
| Disk usage | `docker system df` |

**Exit codes:** `0` = PID1 exited (not a server?) · `1`/`2` = app error · `137` = SIGKILL/**OOM** ·
`143` = SIGTERM (graceful). A container lives exactly as long as its PID 1 process.

See also → `../08-debugging/debugging-methodology.md` · twins → `docker-vs-kubectl-cheatsheet.md`
