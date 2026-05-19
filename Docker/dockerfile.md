# Building images with a Dockerfile

A `Dockerfile` is a text recipe that describes how to build an image. Each instruction produces one layer in the resulting image — layers are cached and reused between builds, which makes the order of instructions surprisingly important. This page covers the common instructions, the `.dockerignore` file, layer caching, the difference between `ARG` and `ENV`, and multi-stage builds, with two complete worked examples (a simple Python service and a multi-stage .NET app).

## Anatomy of a build

You write a `Dockerfile` (no extension), put it next to your application code, and run `docker build` from that directory:

```bash
# Build the image from the current directory, tag as myapp:1.0
docker build -t myapp:1.0 .

# Specify a different Dockerfile location and build context
docker build -t myapp:1.0 -f deploy/Dockerfile .

# Don't use the layer cache (force everything to rebuild)
docker build --no-cache -t myapp:1.0 .
```

The `.` at the end is the **build context** — the directory tree sent to the Docker daemon. Anything outside the context is invisible to `COPY` and `ADD`. This matters: passing `/` as the context will try to ship your entire filesystem to the daemon. Always build from a focused subdirectory and use [`.dockerignore`](#dockerignore) to exclude noise.

## Dockerfile instructions

Each instruction is one keyword in uppercase followed by arguments. Conventionally there's one instruction per line, and lines starting with `#` are comments.

| Instruction | Purpose |
|---|---|
| `FROM` | Base image. Required, must be the first non-comment instruction. |
| `RUN` | Execute a command at build time and commit the result as a new layer. |
| `COPY` | Copy files/dirs from the build context into the image. |
| `ADD` | Like `COPY` but also handles remote URLs and auto-extracts tar archives. Prefer `COPY` unless you specifically need those features. |
| `WORKDIR` | Set the working directory for subsequent `RUN`/`CMD`/`ENTRYPOINT`/`COPY`. Created if it doesn't exist. |
| `ENV` | Set an environment variable available at build time AND at container run time. |
| `ARG` | Build-time-only variable (not in the final image's env). Can be set via `--build-arg`. |
| `CMD` | Default command to run when a container starts. Overridden by `docker run image <args>`. |
| `ENTRYPOINT` | Like `CMD` but not overridden by run args — those become arguments to it. The pair `ENTRYPOINT` + `CMD` is a common idiom (see below). |
| `EXPOSE` | Documents the port the container listens on. Doesn't actually publish anything — that still needs `-p` at run time. |
| `USER` | Run subsequent `RUN`/`CMD`/`ENTRYPOINT` as a specific user instead of root. |
| `VOLUME` | Declare a path as a volume mount point. Anonymous volume created if user doesn't provide one. |
| `LABEL` | Add arbitrary metadata (key=value). Used for image tagging, OCI annotations, etc. |
| `HEALTHCHECK` | Command Docker runs periodically to test container health. |
| `STOPSIGNAL` | Signal sent on `docker stop`. Defaults to SIGTERM. |
| `ONBUILD` | Defer an instruction to be run when this image is used as a base for another. Rare. |

### CMD vs ENTRYPOINT

The distinction trips up most beginners. The simplest mental model:

- **`CMD` alone** — the default command, fully overridable. `docker run image foo` runs `foo`.
- **`ENTRYPOINT` alone** — fixed command, run args are appended. `docker run image foo` runs `entrypoint foo`.
- **Both** — `ENTRYPOINT` is the program, `CMD` is its default args. `docker run image` runs `entrypoint cmd_args`; `docker run image other` runs `entrypoint other`.

Most well-designed application images use both: `ENTRYPOINT ["/app/myapp"]` and `CMD ["--serve"]` so you can override flags without re-specifying the binary.

Both instructions have a **shell form** (`CMD echo hello`) and an **exec form** (`CMD ["echo", "hello"]`). Always prefer the exec form — the shell form wraps your command in `/bin/sh -c`, which means signals don't reach your application (Ctrl+C and `docker stop` won't terminate it cleanly).

## .dockerignore

Same idea as `.gitignore`, but it controls what's sent as the build context. Without it, Docker ships everything under the build directory to the daemon — including `.git/`, `node_modules/`, build artifacts, secrets, IDE caches. That makes builds slow and can leak files into images via accidental `COPY .` patterns.

```gitignore
# .dockerignore
.git
.gitignore
.dockerignore
Dockerfile
README.md

# Build artifacts
bin/
obj/
build/
dist/
target/
*.log

# Dependency directories
node_modules/
__pycache__/
.venv/

# IDE / OS junk
.idea/
.vscode/
.DS_Store

# Secrets
.env
*.pem
*.key
```

Always add a `.dockerignore` early. It speeds up builds and reduces the chance of leaking secrets into the image's layers.

## Layer caching — why instruction order matters

Each instruction produces a layer that's cached based on its inputs. On the next build, Docker walks the Dockerfile top-to-bottom and reuses a layer if its instruction text and inputs are unchanged. The first instruction whose inputs differ invalidates all subsequent layers.

This means **changing a source file should not invalidate `RUN apt-get install` or `npm install`**. To make that true, copy dependency manifests first, install dependencies, and only then copy the rest of the source:

```dockerfile
# BAD — every source change re-runs npm install
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

```dockerfile
# GOOD — npm install only re-runs when package.json/package-lock.json change
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .
CMD ["node", "server.js"]
```

The same pattern applies in every ecosystem: `requirements.txt` then `pip install` then code; `go.mod` + `go.sum` then `go mod download` then code; `*.csproj` then `dotnet restore` then code.

## ARG vs ENV

Both look like variables, but they have very different lifetimes:

| | `ARG` | `ENV` |
|---|---|---|
| Available at build time | Yes | Yes |
| Available at run time (in container env) | **No** | Yes |
| Can be overridden from CLI | `--build-arg KEY=VALUE` | `-e KEY=VALUE` at `docker run` |
| Persisted in image metadata | Argument name only (not value) | Yes (key + value) |
| Default declaration | `ARG KEY=default` | `ENV KEY=default` |

Use `ARG` for things that affect the build but shouldn't be visible to the running app — version numbers, build flags, mirror URLs. Use `ENV` for things the application reads at runtime — log level, ports, feature flags. Never use `ARG` for secrets: the value can leak into the image's build history.

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
LABEL org.opencontainers.image.version=${APP_VERSION}
```

```bash
# Override the build-time variable
docker build --build-arg NODE_VERSION=22 --build-arg APP_VERSION=1.2.3 -t myapp:1.2.3 .
```

## Multi-stage builds

A multi-stage build uses multiple `FROM` instructions in the same Dockerfile. Each `FROM` starts a new stage; only the **last** stage becomes the final image, but earlier stages can be referenced by `COPY --from=...`. The benefit is huge: you can use a big SDK image for compiling, then copy just the resulting binary into a tiny runtime image. Final images are smaller, faster to ship, and have less attack surface.

```dockerfile
# Stage 1: build with the SDK
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/myapp ./cmd/myapp

# Stage 2: tiny runtime image
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/myapp /myapp
ENTRYPOINT ["/myapp"]
```

To build only a specific stage (useful in CI for unit-testing the builder), pass `--target`:

```bash
docker build --target builder -t myapp:builder .
```

## Example 1 — simple Python web service

A single-stage build for a small Python app served by `uvicorn`. Demonstrates ordering for cache friendliness, dropping privileges with `USER`, and using `EXPOSE` + `HEALTHCHECK`.

```dockerfile
FROM python:3.12-slim

# System deps first (most stable layer)
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Dependencies before code — this layer is cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Now the application code
COPY . .

# Run unprivileged
RUN useradd --create-home --uid 1000 appuser
USER appuser

ENV PORT=8000
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:${PORT}/healthz || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Build and run
docker build -t mypyapp:1.0 .
docker run -d --name mypyapp -p 8000:8000 mypyapp:1.0
```

## Example 2 — multi-stage .NET app

The right pattern for ASP.NET Core (and any .NET) apps: do `dotnet restore` and `dotnet publish` in the SDK image, then copy only the published output into the smaller `aspnet` runtime image. The result is roughly 200 MB instead of 800+ MB and contains no compilers.

```dockerfile
# Stage 1 — build with the SDK
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy csproj files first for layer caching (restore depends only on these)
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Now copy the rest of the source and publish
COPY . .
RUN dotnet publish "src/MyApp.Api/MyApp.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Stage 2 — runtime only
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# Copy published output from the build stage
COPY --from=build /app/publish .

# Run as a non-root user (the aspnet image includes the `app` user with UID 1654)
USER app

ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

```bash
# Build (run from repo root, where the solution lives)
docker build -t myapp-api:1.0 -f src/MyApp.Api/Dockerfile .

# Run
docker run -d --name myapp -p 8080:8080 \
  -e ConnectionStrings__Default="Host=db;Username=app;Password=...;Database=app" \
  myapp-api:1.0
```

A few .NET-specific notes:

- The official .NET images live on Microsoft's Container Registry (`mcr.microsoft.com/dotnet/...`), not Docker Hub.
- Use `aspnet:9.0` for web apps (includes the ASP.NET Core shared framework) and `runtime:9.0` for non-web .NET console apps (smaller).
- `dotnet publish` produces framework-dependent output by default. For self-contained / AOT builds, add `-r linux-x64 --self-contained true` and switch the runtime image to `runtime-deps:9.0`.

## Tagging conventions

What you tag matters more than people realize. A few conventions that pay off later:

- **Always tag with a version.** Never push only `:latest` — you lose the ability to roll back.
- **Tag with both a specific version and a moving tag.** E.g., `myapp:1.2.3` and `myapp:1.2`. Consumers pick the precision they want.
- **Include the git SHA.** `myapp:1.2.3-abc1234` makes "which commit produced this image?" a one-liner. Build it as a label too.
- **Avoid mutable tags in production manifests.** `myapp:latest` is fine for `docker pull` ergonomics; use a digest (`myapp@sha256:...`) where you need byte stability.

```bash
docker build \
  -t myapp:1.2.3 \
  -t myapp:1.2 \
  -t myapp:latest \
  --label org.opencontainers.image.revision=$(git rev-parse HEAD) \
  .
```

## Useful `docker build` options

| Option | Effect |
|---|---|
| `-t NAME[:TAG]` | Tag the resulting image. Can be repeated. |
| `-f FILE` | Path to Dockerfile (default `./Dockerfile`). |
| `--build-arg KEY=VAL` | Set a build-time `ARG`. |
| `--target STAGE` | Stop at a specific stage (multi-stage builds). |
| `--platform P` | Build for a specific platform (e.g., `linux/arm64`). |
| `--no-cache` | Don't use cached layers. |
| `--pull` | Always pull a newer base image, even if cached. |
| `--label K=V` | Add a label to the resulting image. |
| `--progress=plain` | Show full build output (useful for debugging). |

Once your image is built, push it to a registry — see [registries.md](registries.md).
