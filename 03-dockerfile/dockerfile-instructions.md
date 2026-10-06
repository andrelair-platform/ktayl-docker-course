# Module 3 — Dockerfile Engineering (every instruction, when & why)

> Level 3 · Learning note (Type A). After this you can read **and write** any Dockerfile in the estate.

## Concept
A Dockerfile is a **recipe**: each instruction either changes the image filesystem (→ a layer) or sets
image **metadata/config** (env, user, entrypoint, exposed ports, labels, healthcheck). The build runs
top-to-bottom; the result is an OCI image.

## Problem it solves
It makes "how to assemble the runnable environment" **declarative, versioned, and reproducible** — vs a
human running install commands on a server (drift, no record).

## How it works internally — the instruction set
| Instruction | What it does | Layer? | When/why (and the gotcha) |
|---|---|---|---|
| `FROM img AS stage` | base image / start a build stage | yes | multi-stage: build in one, copy the artifact to a clean runtime |
| `RUN cmd` | run a command at build time | yes | install deps/compile; combine related commands to cut layers; `--mount=type=cache` for pkg caches (M13) |
| `COPY src dst` | copy from build context (or `--from=stage`) | yes | **prefer over ADD**; copy deps manifest first (caching, M2) |
| `ADD` | COPY + auto-untar + remote URL fetch | yes | **avoid** unless you need its magic; it's surprising/insecure |
| `WORKDIR /app` | set + create the working dir | metadata | avoid `RUN cd …` (doesn't persist) |
| `ENV K=V` | runtime+build env var | metadata | defaults only; real config comes at **runtime** (env-agnostic rule) |
| `ARG K` | **build-time** variable | — | `ARG CACHEBUST`, versions; not present at runtime |
| `EXPOSE 8080` | **documentation** of the listen port | metadata | does NOT publish anything; publishing is `-p` / a K8s Service |
| `CMD […]` | default command/args (overridable) | metadata | the "what to run" if nothing overrides |
| `ENTRYPOINT […]` | the fixed executable | metadata | CMD becomes its args; use exec-form `["…"]` for signals |
| `USER 10001` | drop to a non-root uid | metadata | **mandatory here** (Gatekeeper require-non-root) |
| `HEALTHCHECK` | liveness probe (for Docker/Compose) | metadata | in K8s, probes replace it (M12) |
| `LABEL` | image metadata (source, version) | metadata | provenance; OCI labels link image↔repo |

**CMD vs ENTRYPOINT (the one everyone gets wrong):** `ENTRYPOINT` = the program; `CMD` = its default
args. Use **exec form** `["bin","arg"]` (not shell form `bin arg`) so your process is **PID 1** and
receives **SIGTERM** for graceful shutdown. Shell form wraps you in `/bin/sh -c`, which may swallow signals.

## Example
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
USER 10001
EXPOSE 8000
CMD ["uvicorn","app.main:app","--host","0.0.0.0","--port","8000"]
```

## Example from our organisation (four real styles, read them)
- **`ktayl-underwriting/Dockerfile` (Python)** — builder installs a **venv** (`python -m venv /opt/venv`;
  `pip install .`), runtime copies `/opt/venv` + app, `useradd … uid 10001`, `USER 10001`, `CMD ["uvicorn"
  …]`. Note `ENV PATH="/opt/venv/bin:$PATH"` and `DATABASE_URL` injected at **runtime** (env-agnostic).
- **`platform-demo/Dockerfile` (Go)** — `ENTRYPOINT ["/platform-demo"]` (exec form, PID 1, distroless has
  no shell), `USER nonroot`, `ARG VERSION/COMMIT` baked into the binary via `-ldflags`.
- **`ktayl-iam/frontend/Dockerfile` (Next)** — `ARG CACHEBUST` (declared so the shared CI build-arg
  doesn't warn), copies only `.next/standalone`+`static`+`public`, **`RUN rm -rf …/npm`** (cut CVEs),
  `USER node`, `CMD ["node","server.js"]`.
- **`ktayl-core/Dockerfile` (Java)** — `ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar /app/app.jar"]`:
  the `exec` is deliberate — it **replaces** the shell so Java becomes PID 1 and gets SIGTERM (otherwise
  `sh` is PID 1 and the JVM never sees the stop signal → ungraceful kills). A subtle, real exec-form lesson.

## Architecture
```
build context ──(minus .dockerignore)──▶ BuildKit
     FROM build-stage         FROM runtime-stage (clean, minimal)
       RUN compile  ──COPY --from=build──▶  COPY artifact
                                            USER 10001 · EXPOSE · ENTRYPOINT/CMD
                                                   │
                                                   ▼  the final, shipped image
```

## Commands
```bash
docker build -t svc:dev --build-arg VERSION=dev .
docker build --target build -t svc:builder .     # build only a stage (debug the builder)
docker run --rm svc:dev                           # CMD/ENTRYPOINT in action
docker inspect --format '{{.Config.User}} {{.Config.Entrypoint}} {{.Config.Cmd}}' svc:dev
```

## Debugging
- **Container exits 0 immediately** → your CMD isn't a long-running foreground process (e.g. it forks +
  returns). Run the server in the foreground.
- **SIGTERM ignored / slow kills** → shell-form ENTRYPOINT or missing `exec` (see ktayl-core).
- **"permission denied" after `USER`** → the non-root user can't write a path it needs; fix ownership at
  build (`chown`) or mount an `emptyDir`/volume (M5).
- **Build fails at `COPY x`** → `x` is outside the context or excluded by `.dockerignore`.

## Production considerations
Multi-stage (builder tools never reach runtime) · `USER` non-root · exec-form ENTRYPOINT for graceful
stop · env at runtime not baked · pin the base (`:3.12-slim`, not `:latest`) · LABEL provenance.

## Security considerations
No secrets in `ENV`/`ARG` (they're in the image/history) · drop to non-root · minimal runtime stage ·
avoid `ADD` (remote fetch/untar surprises) · remove unneeded tooling (the Next `rm npm` trick).

## Common mistakes
`ADD` where `COPY` is meant · shell-form ENTRYPOINT (signal loss) · `RUN cd` · secrets in ARG/ENV ·
installing build tools in the runtime stage · no `USER` (root container).

## Key takeaways
1. Each FS instruction = a layer; metadata instructions configure the image.
2. **Multi-stage** keeps the runtime minimal; `COPY --from=` grabs just the artifact.
3. **ENTRYPOINT = program, CMD = args; exec form + `exec` ⇒ PID 1 ⇒ SIGTERM works.**
4. `EXPOSE` documents, it doesn't publish. `ARG` is build-time, `ENV` is runtime.
5. Always `USER` non-root here (Gatekeeper will reject root anyway).

## Exercise (on our project)
1. Read all four Dockerfiles above. For each: how many stages? what's the runtime base? which user? which
   port? what's the start command and is it exec-form?
2. Explain *why* `ktayl-core` wraps Java in `sh -c "exec java …"` and what would break without `exec`.
3. Take a naive single-stage Dockerfile (write one for a FastAPI app) and convert it to the
   ktayl-underwriting multi-stage + non-root shape. Compare `docker history` sizes.

## See also
- Prev → `02-images/…` · Next → `04-networking/…` · Prod → `12-production/…` · Security → `07-security/…`
- Fast → `cheatsheets/dockerfile-cheatsheet.md` (todo) · Case → `case-studies/ktayl-underwriting-fastapi.md` (todo)
