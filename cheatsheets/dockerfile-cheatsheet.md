# Cheatsheet — Dockerfile instructions

| Need | Do | Gotcha |
|---|---|---|
| start a stage | `FROM img:tag AS build` | pin the tag (`:3.12-slim`, never `:latest`) |
| copy the artifact only | `COPY --from=build /out/app /app` | multi-stage keeps build tools out of runtime |
| copy deps manifest first | `COPY pom.xml .` → `RUN mvn … go-offline` → `COPY src .` | caching (M2) |
| copy files | `COPY src dst` | prefer over `ADD` (ADD auto-untars + fetches URLs = surprises) |
| set working dir | `WORKDIR /app` | not `RUN cd` (doesn't persist) |
| build-time var | `ARG VERSION` | **not** at runtime; never a secret (baked in history) |
| runtime default env | `ENV PORT=8000` | defaults only — real config at runtime (env-agnostic) |
| document the port | `EXPOSE 8000` | **doesn't publish**; publishing = `-p` / a Service |
| drop to non-root | `RUN useradd -u 10001 app` + `USER 10001` | mandatory (Gatekeeper denies root) |
| the start command | `ENTRYPOINT ["bin"]` + `CMD ["arg"]` | **exec form** `[…]`, not shell form, so PID1 gets SIGTERM |
| run via a shell but keep signals | `ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar app.jar"]` | the `exec` replaces sh → Java is PID1 (ktayl-core) |
| build-time secret (a CA/token) | `RUN --mount=type=secret,id=ca,target=/tmp/ca …` | NEVER `--build-arg` (truncates multi-line + bakes) |
| persistent build cache | `RUN --mount=type=cache,target=/go/pkg/mod …` | stays in BuildKit, not in the image (platform-demo) |

**Multi-stage skeleton (the house pattern):**
```dockerfile
FROM <sdk> AS build
WORKDIR /src
COPY <deps-manifest> . && RUN <fetch deps>
COPY . . && RUN <build>
FROM <minimal-runtime>
RUN useradd -u 10001 app
COPY --from=build /artifact /app
USER 10001
EXPOSE <port>
ENTRYPOINT ["<exec-form>"]
```

See also → `../03-dockerfile/dockerfile-instructions.md`
