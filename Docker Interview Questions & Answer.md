# 🐳 Docker Interview Questions & Answers — Complete Guide (Basic to Advanced)

> Comprehensive collection of **104 real-world Docker interview questions & answers** with clear explanations, practical terminal examples, production Dockerfiles, Compose specs, kernel internals, and troubleshooting scenarios.

> **Quick Tip:** Click any question to expand its full answer and code snippets.

---

## 📑 Table of Contents

- [1. Docker Fundamentals](#fundamentals) (8 questions)
- [2. Images, Layers & Build Cache](#images) (10 questions)
- [3. Dockerfile Deep Dive](#dockerfile) (11 questions)
- [4. Running Containers & Runtime Behavior](#runtime) (10 questions)
- [5. Docker Internals (Linux)](#internals) (7 questions)
- [6. Docker Networking](#networking) (7 questions)
- [7. Storage, Volumes & Data Persistence](#storage) (5 questions)
- [8. Docker Compose](#compose) (5 questions)
- [9. Docker Security](#security) (6 questions)
- [10. Registries & Image Distribution](#registry) (4 questions)
- [11. Docker Swarm & Orchestration](#orchestration) (4 questions)
- [12. CI/CD, Docker-in-Docker & Best Practices](#cicd) (4 questions)
- [13. Troubleshooting & Debugging](#troubleshooting) (12 questions)
- [14. Rapid-Fire & Tricky Questions](#misc) (11 questions)
- [15. Command Cheat Sheet](#cheatsheet)

---


## <a id="fundamentals"></a>1. Docker Fundamentals

<details>
<summary><b>What is Docker and what problem does it solve?</b> `🟢 Basic`</summary>
<br>

Docker is a platform for building, shipping, and running applications inside **containers**. A container packages an application with its runtime, libraries, and config so it runs the same way on a laptop, in CI, and in production.
- **Problem solved:** "It works on my machine". It removes environment drift between dev, test, and prod.
- **Benefits:** fast startup (milliseconds), high density, immutable artifacts, reproducible builds, and easy CI/CD and microservices.
- **Core pieces:** Docker Engine (daemon), CLI, images, containers, registries, volumes, networks, Compose, BuildKit.

```bash
docker run -d -p 8080:80 --name web nginx:1.27
curl localhost:8080
```

</details>

<details>
<summary><b>Container vs Virtual Machine - what is the difference?</b> `🟢 Basic`</summary>
<br>

|  | Container | Virtual Machine |
| --- | --- | --- |
| Isolation | Process-level, using kernel namespaces and cgroups | Hardware-level, using a hypervisor |
| Kernel | Shares the host kernel | Each VM runs its own guest kernel |
| Size | MBs | GBs |
| Boot time | Milliseconds to seconds | Seconds to minutes |
| Density | Hundreds per host | Tens per host |
| Security boundary | Weaker (shared kernel) | Stronger |
| OS flexibility | Linux containers need a Linux kernel | Any OS |

**Key line:** a container is just an isolated Linux process. A VM is a whole emulated machine. For stronger isolation of containers you can use gVisor, Kata Containers, or Firecracker.

</details>

<details>
<summary><b>What is the difference between an image and a container?</b> `🟢 Basic`</summary>
<br>

An **image** is a read-only, layered template: the filesystem plus metadata such as CMD, ENV, and exposed ports. A **container** is a running (or stopped) instance of an image. It adds a thin **writable layer** on top, plus runtime config such as networks, mounts, and limits.

Analogy: image = class, container = object. One image can run many containers.
```bash
docker images          # list images
docker ps -a           # list containers (running + stopped)
docker run -d --name a1 nginx && docker run -d --name a2 nginx   # 2 containers, 1 image
```

</details>

<details>
<summary><b>Explain the Docker architecture (client, daemon, containerd, runc, shim).</b> `🟡 Intermediate`</summary>
<br>

```bash
docker CLI ──REST API (unix:///var/run/docker.sock)──► dockerd
                                                        │ gRPC
                                                        ▼
                                                   containerd  (image pull, snapshots, lifecycle)
                                                        │
                                                        ▼
                                          containerd-shim-runc-v2  (one per container)
                                                        │
                                                        ▼
                                                      runc  (creates namespaces/cgroups, then exits)
                                                        │
                                                        ▼
                                                 your process (PID 1 in container)
```
- **docker CLI**: client that sends REST calls to the daemon.
- **dockerd**: the API server. Handles builds (BuildKit), networking, volumes, and orchestration.
- **containerd**: a CNCF-graduated runtime that manages the container lifecycle, image storage, and snapshots. Kubernetes uses it directly.
- **shim**: keeps the container running if dockerd or containerd restarts, holds its STDIO, and reports the exit status.
- **runc**: the OCI reference low-level runtime. It sets up namespaces, cgroups, and seccomp, then execs the process.

> 💡 **Tip:** Because of the shim, enabling "live-restore": true in daemon.json lets containers keep running while dockerd is upgraded or restarted.

</details>

<details>
<summary><b>What is OCI? How do Docker, containerd, CRI-O, and Podman relate?</b> `🟡 Intermediate`</summary>
<br>

The **Open Container Initiative** defines three specs: **Image spec** (image format), **Runtime spec** (how to run a filesystem bundle), and **Distribution spec** (registry API). Any compliant tool can build or run any compliant image.
- **Docker**: full developer platform (build + run + Compose). It uses containerd and runc underneath.
- **containerd**: high-level runtime used by Docker and by Kubernetes (through CRI).
- **CRI-O**: lightweight runtime built only for Kubernetes. OpenShift uses it.
- **Podman**: daemonless, rootless by default, with a Docker-compatible CLI (`alias docker=podman`). Supports pods.
- **Buildah / Kaniko**: build images without a Docker daemon (useful in CI and Kubernetes).

</details>

<details>
<summary><b>Kubernetes removed "dockershim". Does that mean Docker images no longer work in Kubernetes?</b> `🟡 Intermediate`</summary>
<br>

**No.** Kubernetes 1.24 removed *dockershim*, the adapter that let the kubelet talk to dockerd. Kubelets now talk to CRI runtimes such as containerd or CRI-O directly. Images built with Docker are standard OCI images and run unchanged.

What changed: you can no longer rely on `/var/run/docker.sock` on nodes (for example, Docker-in-Docker builds or log agents that read Docker paths). Use `crictl` instead of `docker` on nodes.
```bash
crictl ps            # instead of docker ps on a k8s node
crictl logs <id>
crictl inspecti <image>
```

</details>

<details>
<summary><b>Explain the container lifecycle and states.</b> `🟢 Basic`</summary>
<br>

States: **created**, **running**, **paused**, **restarting**, **exited**, **dead**, and **removing**.
```bash
docker create --name app nginx    # created
docker start app                  # running
docker pause app                  # paused (cgroup freezer, SIGSTOP-like)
docker unpause app
docker stop app                   # SIGTERM -> wait 10s -> SIGKILL  => exited
docker rm app                     # removed (writable layer deleted)
docker inspect -f '{{.State.Status}} {{.State.ExitCode}}' app
```

</details>

<details>
<summary><b>What is Docker Hub / a registry / a repository / a tag?</b> `🟢 Basic`</summary>
<br>

- **Registry**: a server that stores images. Examples: Docker Hub, Harbor, ECR, GCR/Artifact Registry, ACR, Quay, the OpenShift internal registry.
- **Repository**: a collection of related images, for example `library/nginx`.
- **Tag**: a mutable label pointing to an image, for example `1.27`, `latest`.
- **Digest**: an immutable content hash, for example `nginx@sha256:ab12...`.


Full reference format: `[registry[:port]/][namespace/]repo[:tag|@digest]`, for example `registry.example.com:5000/team/api:1.4.2`.

</details>


## <a id="images"></a>2. Images, Layers & Build Cache

<details>
<summary><b>What are image layers and how does copy-on-write work?</b> `🟡 Intermediate`</summary>
<br>

Each filesystem-changing Dockerfile instruction (`RUN`, `COPY`, `ADD`) creates a read-only **layer**, which is a tar diff of the filesystem. Layers are content-addressed, cached, and shared between images. When a container starts, Docker stacks the layers with a union filesystem (**overlay2**) and adds a writable container layer on top.

**Copy-on-write:** when a container modifies a file from a lower layer, the file is first copied up into the writable layer and then changed. The original layer stays untouched, which lets 100 containers share one image on disk.
```bash
docker history nginx:1.27            # see layers and sizes
docker inspect -f '{{json .RootFS.Layers}}' nginx:1.27
docker inspect -f '{{json .GraphDriver.Data}}' <container>   # LowerDir/UpperDir/MergedDir
```
> ⚠️ **Warning:** Gotcha: deleting a file in a later layer does not shrink the image. It adds a "whiteout" marker while the bytes remain in the earlier layer. Install and clean up in the same RUN.

</details>

<details>
<summary><b>How does the Docker build cache work and how do you optimize for it?</b> `🟡 Intermediate`</summary>
<br>

For each instruction, the builder checks whether a cached layer exists with the same parent and the same instruction. For `COPY`/`ADD` it also compares file checksums. The **first cache miss invalidates every following layer**.

**Rule:** order instructions from least to most frequently changing. Copy dependency manifests before source code.
```bash
# BAD - any source change re-installs all dependencies
COPY . .
RUN npm ci

# GOOD - dependencies are cached until package*.json changes
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

Other tips: use `.dockerignore`, BuildKit cache mounts, `--cache-from` in CI, and pin base image versions. Use `--no-cache` to force a rebuild and `--pull` to refresh the base image.

</details>

<details>
<summary><b>How do you reduce Docker image size? (Very frequently asked)</b> `🟡 Intermediate`</summary>
<br>

1. **Multi-stage builds**: compile in a fat image and ship only the artifacts.
2. **Smaller base images**: `-slim`, `alpine` (musl libc caveats), `distroless`, `scratch` for static binaries.
3. **Combine RUN steps and clean up in the same layer** (apt caches, temp files).
4. `--no-install-recommends` (apt), `pip install --no-cache-dir`, `apk add --no-cache`.
5. **.dockerignore** to keep `.git`, `node_modules`, tests, and secrets out.
6. Install only production dependencies (`npm ci --omit=dev`).
7. Analyze with `docker history`, `dive`, or `docker scout`.

```bash
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

Typical result: a Go app goes from ~900 MB (`golang` image) to under 15 MB (`scratch`/`distroless`).

</details>

<details>
<summary><b>What is a multi-stage build? Give an example.</b> `🟡 Intermediate`</summary>
<br>

A Dockerfile with multiple `FROM` stages. Earlier stages build or test, and the final stage copies only the needed artifacts with `COPY --from=`. Compilers, source code, and build tools stay out of the final image, so it is smaller and has less attack surface.
```bash
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/app ./cmd/app

FROM build AS test
RUN go test ./...

FROM gcr.io/distroless/static-debian12:nonroot AS final
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```
```bash
docker build -t app:1.0 .                 # builds final stage (test stage skipped by BuildKit if unused)
docker build --target test -t app:test .  # run only up to the test stage
```
> 💡 **Tip:** BuildKit builds only the stages the target depends on, and builds independent stages in parallel.

</details>

<details>
<summary><b>What is the difference between docker save/load and docker export/import?</b> `🟡 Intermediate`</summary>
<br>

| save / load | export / import |
| --- | --- |
| Works on images | Works on containers |
| Keeps all layers, tags, history, and metadata (CMD, ENV...) | Flattens to a single filesystem layer and loses history and metadata |
| Use for air-gapped transfer of images | Use for snapshotting a container filesystem |
```bash
docker save -o app.tar app:1.0 && scp app.tar host: && ssh host docker load -i app.tar
docker save app:1.0 | gzip > app.tar.gz
docker export mycontainer > fs.tar && docker import fs.tar myimg:flat
```

</details>

<details>
<summary><b>What are dangling images and how do you clean up disk space?</b> `🟢 Basic`</summary>
<br>

Dangling images are untagged layers shown as `<none>:<none>`. They usually appear after rebuilding a tag.
```bash
docker system df                     # what uses space
docker image prune                   # dangling images
docker image prune -a --filter "until=168h"   # unused images older than 7 days
docker container prune
docker volume prune                  # careful: data loss
docker builder prune --keep-storage 10GB
docker system prune -a --volumes     # nuke everything unused - be careful!
```

</details>

<details>
<summary><b>Why should you avoid the :latest tag? Tag vs digest?</b> `🟡 Intermediate`</summary>
<br>

- `latest` is just a default tag name. It does not mean "newest" and it is **mutable**, so deployments are not reproducible and rollbacks are ambiguous.
- Tags can be re-pointed. **Digests** (`@sha256:...`) are immutable content hashes.
- Best practice: tag with a semantic version plus the git SHA (`app:1.4.2`, `app:1.4.2-a1b2c3d`), and pin by digest in production manifests.

```bash
docker pull nginx:1.27
docker inspect --format='{{index .RepoDigests 0}}' nginx:1.27
# FROM nginx:1.27@sha256:<digest>   <- reproducible base
```

</details>

<details>
<summary><b>What is BuildKit and what advanced features does it provide?</b> `🔴 Advanced`</summary>
<br>

BuildKit is the modern build engine (the default since Docker 23). Its features:
- Parallel builds of independent stages, with unused stages skipped.
- **Cache mounts** (`--mount=type=cache`) for package manager caches that persist across builds.
- **Secret mounts** (`--mount=type=secret`) that never get written to a layer.
- **SSH forwarding** (`--mount=type=ssh`) for cloning private repos.
- External cache import/export (registry, local, GitHub Actions, S3).
- Multi-platform builds with `buildx`, heredocs, and `COPY --link`.
- Build attestations (SBOM, provenance).

```bash
# syntax=docker/dockerfile:1
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    pip install private-package
RUN --mount=type=ssh git clone git@github.com:org/private.git /opt/private
COPY . .
```
```bash
docker build --secret id=netrc,src=$HOME/.netrc --ssh default -t app .
```

</details>

<details>
<summary><b>How do you build multi-architecture images (amd64 + arm64)?</b> `🔴 Advanced`</summary>
<br>

Use `docker buildx`. It produces a **manifest list** (OCI image index) that points to one image per platform. The client pulls the matching one automatically. Cross-builds use QEMU emulation or native remote builders. Cross-compilation in the Dockerfile with `$BUILDPLATFORM`/`$TARGETARCH` is the fastest option.
```bash
docker run --privileged --rm tonistiigi/binfmt --install all   # QEMU emulators
docker buildx create --name multi --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.example.com/app:1.0 --push .
docker buildx imagetools inspect registry.example.com/app:1.0
```
```bash
FROM --platform=$BUILDPLATFORM golang:1.23 AS build
ARG TARGETOS TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app .
```

</details>

<details>
<summary><b>How do you share build cache in CI where every runner is fresh?</b> `🔴 Advanced`</summary>
<br>

Export the BuildKit cache to a registry (or the GHA cache backend) and import it on the next build. `mode=max` caches the layers of all intermediate stages, not just the final one.
```bash
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/app:buildcache \
  --cache-to   type=registry,ref=registry.example.com/app:buildcache,mode=max \
  -t registry.example.com/app:${GIT_SHA} --push .

# GitHub Actions
#   --cache-from type=gha --cache-to type=gha,mode=max
```

</details>


## <a id="dockerfile"></a>3. Dockerfile Deep Dive

<details>
<summary><b>CMD vs ENTRYPOINT - what is the difference and how do they combine? (Top question)</b> `🟡 Intermediate`</summary>
<br>

- **ENTRYPOINT**: the executable that always runs. It is overridden only with `--entrypoint`.
- **CMD**: default arguments (or a default command). It is overridden by any arguments after the image name in `docker run`.
- When both are in exec form, CMD is appended to ENTRYPOINT as arguments.

```bash
FROM alpine
ENTRYPOINT ["ping"]
CMD ["-c", "3", "localhost"]
```
```bash
docker run img                 # ping -c 3 localhost
docker run img -c 1 8.8.8.8    # ping -c 1 8.8.8.8   (CMD replaced)
docker run --entrypoint sh img -c 'echo hi'   # entrypoint replaced
```
|  | No ENTRYPOINT | ENTRYPOINT exec_entry p1 | ENTRYPOINT ["exec_entry","p1"] |
| --- | --- | --- | --- |
| No CMD | error | /bin/sh -c exec_entry p1 | exec_entry p1 |
| CMD ["c1","c2"] | c1 c2 | /bin/sh -c exec_entry p1 (CMD ignored) | exec_entry p1 c1 c2 |
| CMD c1 c2 | /bin/sh -c c1 c2 | /bin/sh -c exec_entry p1 | exec_entry p1 /bin/sh -c c1 c2 |
> 💡 **Tip:** Pattern: ENTRYPOINT = fixed binary or wrapper script, CMD = default flags. This makes the image behave like a CLI tool.

</details>

<details>
<summary><b>Shell form vs exec form - why does it matter?</b> `🟡 Intermediate`</summary>
<br>

**Shell form** (`CMD node server.js`) runs as `/bin/sh -c "node server.js"`, so `sh` becomes PID 1. Most shells do **not forward signals**, so `docker stop` waits 10s and then SIGKILLs the app with no graceful shutdown.

**Exec form** (`CMD ["node","server.js"]`) runs the binary directly as PID 1, which receives SIGTERM. Exec form does not expand variables, though, so `$HOME` is not expanded.
```bash
CMD ["node", "server.js"]            # preferred
CMD ["sh", "-c", "exec node server.js --port $PORT"]   # need env expansion: use exec
```
> ⚠️ **Warning:** Exec form must use double quotes because it is JSON. CMD ['node','app.js'] silently falls back to shell form.

</details>

<details>
<summary><b>COPY vs ADD?</b> `🟢 Basic`</summary>
<br>

Both copy files into the image. **ADD** can also (1) auto-extract **local** tar archives (gzip/bzip2/xz) and (2) fetch remote URLs, and recent versions can add git repos. Because that "magic" is implicit, the best practice is to use **COPY** by default and ADD only when you need tar auto-extraction.
```bash
COPY --chown=app:app src/ /app/src/
ADD rootfs.tar.gz /             # auto-extracts
# For URLs prefer RUN curl ... && verify checksum && cleanup (one layer), or:
ADD --checksum=sha256:<hash> https://example.com/tool.tar.gz /tmp/
```

</details>

<details>
<summary><b>ARG vs ENV?</b> `🟡 Intermediate`</summary>
<br>

| ARG | ENV |
| --- | --- |
| Build-time only | Build-time and runtime (persists in the image) |
| Set with --build-arg | Override with docker run -e |
| Not in the final container's environment (but visible in docker history!) | Visible in docker inspect |
```bash
ARG BASE_TAG=3.12-slim           # ARG before FROM: only usable in FROM lines
FROM python:${BASE_TAG}
ARG APP_VERSION=dev              # must re-declare after FROM to use
ENV APP_VERSION=${APP_VERSION} \
    PYTHONUNBUFFERED=1
LABEL org.opencontainers.image.version=$APP_VERSION
```
```bash
docker build --build-arg APP_VERSION=1.4.2 --build-arg BASE_TAG=3.12-alpine -t app .
```
> ⚠️ **Warning:** Never pass secrets with ARG or ENV. They are recorded in image metadata and history. Use BuildKit --secret.

</details>

<details>
<summary><b>What does EXPOSE do? Is it required to publish a port?</b> `🟢 Basic`</summary>
<br>

`EXPOSE` is **documentation/metadata only**. It does not publish anything. Ports are published at runtime with `-p`. `-P` publishes all EXPOSEd ports to random high host ports.
```bash
docker run -p 8080:80 nginx          # host 8080 -> container 80 (all interfaces)
docker run -p 127.0.0.1:8080:80 nginx  # bind only to localhost (safer)
docker run -P nginx && docker port $(docker ps -lq)
```

</details>

<details>
<summary><b>What is .dockerignore and why is it important?</b> `🟡 Intermediate`</summary>
<br>

It excludes files from the **build context** sent to the builder. Benefits: faster builds (smaller context), better cache hits, smaller images when you `COPY . .`, and it prevents secrets from leaking into images.
```bash
# .dockerignore
.git
node_modules
**/*.log
.env
*.pem
Dockerfile*
docker-compose*.yml
coverage/
!README.md
```

</details>

<details>
<summary><b>Explain HEALTHCHECK. How is it different from Kubernetes probes?</b> `🟡 Intermediate`</summary>
<br>

HEALTHCHECK defines a command Docker runs periodically. Exit code 0 means **healthy** and 1 means **unhealthy**. Status shows in `docker ps`. Compose's `depends_on: condition: service_healthy` uses it, and Swarm replaces unhealthy tasks.
```bash
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://localhost:8080/healthz || exit 1
```
```bash
docker inspect --format='{{json .State.Health}}' app | jq
```

**Difference:** plain Docker only *reports* unhealthy and does not restart the container by itself. Kubernetes **ignores** Dockerfile HEALTHCHECK and uses liveness, readiness, and startup probes instead.

</details>

<details>
<summary><b>WORKDIR, USER, VOLUME, STOPSIGNAL, SHELL, ONBUILD, LABEL - what do they do?</b> `🟡 Intermediate`</summary>
<br>

- **WORKDIR**: sets the working directory and creates it if missing. Prefer it over `RUN cd`, which does not persist between instructions.
- **USER**: the user/UID for following RUN, CMD, and ENTRYPOINT instructions. Use it to run as non-root.
- **VOLUME**: declares a mount point. An anonymous volume is created at runtime if nothing is mounted there. Changes made to that path after the VOLUME line in the build may be discarded.
- **STOPSIGNAL**: the signal sent by `docker stop` (for example `SIGQUIT` for nginx graceful shutdown).
- **SHELL**: changes the default shell for shell form (for example `["/bin/bash","-o","pipefail","-c"]`).
- **ONBUILD**: a trigger that runs when another image uses this one as a base.
- **LABEL**: metadata (OCI annotations such as `org.opencontainers.image.source`).

```bash
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -fsSL https://example.com/x | tar xz   # fails build if curl fails
RUN groupadd -r app && useradd -r -g app -u 10001 app
WORKDIR /app
USER 10001
STOPSIGNAL SIGTERM
```

</details>

<details>
<summary><b>How do you write a correct entrypoint script? Why use "exec $@"?</b> `🔴 Advanced`</summary>
<br>

Wrapper scripts do setup (render config, wait for dependencies, run migrations) and then must **replace themselves** with the main process using `exec`. That way the app becomes PID 1, receives signals, and its exit code propagates. `"$@"` passes CMD through as arguments.
```bash
#!/bin/sh
set -eu
: "${DB_HOST:?DB_HOST is required}"
envsubst < /etc/app/config.tmpl > /etc/app/config.yaml
if [ "${1:-}" = "migrate" ]; then exec /app/migrate; fi
exec "$@"
```
```bash
COPY --chmod=755 docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["/app/server", "--port=8080"]
```

</details>

<details>
<summary><b>What are Dockerfile heredocs and COPY --link?</b> `🔴 Advanced`</summary>
<br>

**Heredocs** (dockerfile syntax 1.4+) let you write multi-line scripts or inline files without long `&& \` chains:
```bash
# syntax=docker/dockerfile:1
RUN <<EOF
set -e
apt-get update
apt-get install -y --no-install-recommends curl
rm -rf /var/lib/apt/lists/*
EOF
COPY <<EOF /etc/app/config.ini
[server]
port=8080
EOF
```

**COPY --link** creates the copied layer independently of the layers below it. If the base image changes, the layer does not need to be rebuilt and can be rebased, which speeds up rebuilds and cache reuse.

</details>

<details>
<summary><b>Write a production-grade Dockerfile for a Node.js / Python app.</b> `🟡 Intermediate`</summary>
<br>

```bash
# syntax=docker/dockerfile:1
# ---------- Node.js ----------
FROM node:20-bookworm-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:20-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM node:20-bookworm-slim
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK CMD node -e "require('http').get('http://localhost:3000/healthz',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"
CMD ["node", "dist/server.js"]
```
```bash
# syntax=docker/dockerfile:1
# ---------- Python ----------
FROM python:3.12-slim AS builder
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
WORKDIR /app
RUN python -m venv /venv
ENV PATH=/venv/bin:$PATH
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH=/venv/bin:$PATH
RUN useradd -r -u 10001 app
COPY --from=builder /venv /venv
WORKDIR /app
COPY --chown=app . .
USER 10001
EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "-w", "4", "app:app"]
```

</details>


## <a id="runtime"></a>4. Running Containers & Runtime Behavior

<details>
<summary><b>Explain common docker run flags.</b> `🟢 Basic`</summary>
<br>

```bash
docker run -d \                       # detached
  --name api \
  -it \                               # -i keep STDIN open, -t allocate a TTY
  --rm \                              # delete container on exit
  -p 8080:8080 \                      # publish port
  -e ENV=prod --env-file .env \       # env vars
  -v data:/var/lib/app \              # named volume
  --mount type=bind,src=$PWD/conf,dst=/conf,readonly \
  --network backend \
  --restart unless-stopped \
  --memory 512m --cpus 1.5 --pids-limit 200 \
  --user 10001:10001 \
  --read-only --tmpfs /tmp \
  --health-cmd 'curl -f localhost:8080/healthz || exit 1' \
  --log-opt max-size=10m --log-opt max-file=3 \
  myorg/api:1.4.2
```

Detach from an attached TTY without stopping the container: `Ctrl-P Ctrl-Q`.

</details>

<details>
<summary><b>docker exec vs docker attach?</b> `🟢 Basic`</summary>
<br>

- **exec** starts a **new process** inside a running container (debugging, shells). Exiting it does not stop the container.
- **attach** connects your terminal to the **main process's** STDIN/STDOUT. Ctrl-C may send SIGINT to PID 1 and stop the container.

```bash
docker exec -it api sh
docker exec -u 0 api id                # exec as root
docker attach --sig-proxy=false api
```

</details>

<details>
<summary><b>Explain restart policies.</b> `🟡 Intermediate`</summary>
<br>

| Policy | Behavior |
| --- | --- |
| no (default) | Never restart |
| on-failure[:N] | Restart only on a non-zero exit, optionally at most N times |
| always | Always restart. Also starts on daemon boot, even if the container was manually stopped earlier |
| unless-stopped | Like always, but not if you manually stopped it before the daemon restarted |

Restarts use exponential backoff (100ms, doubling, capped). A restart policy does not react to "unhealthy" status. That needs an orchestrator.
```bash
docker update --restart unless-stopped api
```

</details>

<details>
<summary><b>The PID 1 problem: signals, zombies, and --init. (Advanced favorite)</b> `🔴 Advanced`</summary>
<br>

The Linux kernel treats PID 1 specially:
1. **No default signal handlers.** Unless PID 1 explicitly handles SIGTERM, the signal is ignored. `docker stop` then waits for the timeout and sends SIGKILL, so there is no graceful shutdown.
2. **Zombie reaping.** PID 1 must `wait()` on orphaned children. Apps that are not init systems leave zombies, which can exhaust the PID table.


**Fixes:**
- Use exec form and handle SIGTERM in the app.
- Use `docker run --init`, which injects **tini** as PID 1 to forward signals and reap zombies.
- Or bake in tini or dumb-init: `ENTRYPOINT ["/usr/bin/tini","--"]`.

```bash
docker run --init -d myapp
docker stop -t 30 myapp           # custom grace period
# compose:  init: true   and   stop_grace_period: 30s
```

</details>

<details>
<summary><b>What do container exit codes 0, 1, 125, 126, 127, 137, 139, 143 mean?</b> `🟡 Intermediate`</summary>
<br>

| Code | Meaning |
| --- | --- |
| 0 | Success / normal exit |
| 1 | Application error |
| 125 | docker run itself failed (bad flag, daemon error) |
| 126 | Command cannot be invoked (permission denied / not executable) |
| 127 | Command not found (wrong path, missing binary, or a CRLF line ending in the script shebang) |
| 137 | 128+9 = SIGKILL: OOM killed, or docker kill, or the stop timeout was exceeded |
| 139 | 128+11 = SIGSEGV segmentation fault (often a glibc/musl mismatch) |
| 143 | 128+15 = SIGTERM, graceful stop |
```bash
docker inspect -f 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}} err={{.State.Error}}' app
```

</details>

<details>
<summary><b>docker stop vs docker kill vs docker rm -f?</b> `🟡 Intermediate`</summary>
<br>

- **stop**: sends STOPSIGNAL (default SIGTERM), waits for the grace period (default 10s), then SIGKILL.
- **kill**: sends SIGKILL immediately (or any signal: `docker kill -s HUP nginx` to reload config).
- **rm -f**: kills and removes the container.

</details>

<details>
<summary><b>How do you limit CPU and memory? What happens when the limit is exceeded?</b> `🟡 Intermediate`</summary>
<br>

```bash
docker run -d --memory=512m --memory-swap=512m \   # no swap
  --memory-reservation=256m \                       # soft limit
  --cpus=1.5 \                                       # CFS quota: 1.5 cores max
  --cpu-shares=512 \                                 # relative weight under contention
  --cpuset-cpus=0,1 \                                # pin to cores
  --pids-limit=256 \                                 # fork-bomb protection
  --ulimit nofile=65535:65535 app
docker stats --no-stream
docker update --memory 1g --memory-swap 1g app       # live update
```
- **Memory over the limit**: the kernel OOM killer kills the process (exit 137, `OOMKilled=true`).
- **CPU over the limit**: the process is **throttled**, not killed (watch `cpu.stat nr_throttled`).
- No limits means a container can use all host resources (noisy neighbor).

> ⚠️ **Warning:** JVM, Node, and Go runtimes should be container-aware. Modern JVMs read cgroup limits (-XX:MaxRAMPercentage=75). Node needs --max-old-space-size on older versions. Go 1.19+ supports GOMEMLIMIT.

</details>

<details>
<summary><b>How do you view and manage container logs?</b> `🟢 Basic`</summary>
<br>

Docker captures the STDOUT/STDERR of PID 1. Apps should log to stdout rather than to files (12-factor). Official images symlink log files to `/dev/stdout`.
```bash
docker logs -f --tail 100 --since 10m -t api
docker logs api 2>&1 | grep ERROR
```

Logging drivers: `json-file` (default), `local`, `journald`, `syslog`, `fluentd`, `gelf`, `awslogs`, `splunk`, `none`. Configure rotation globally, because the default json-file grows without limit:
```bash
# /etc/docker/daemon.json
{
  "log-driver": "local",
  "log-opts": { "max-size": "10m", "max-file": "5" }
}
```

</details>

<details>
<summary><b>What does docker inspect give you? Show useful Go-template queries.</b> `🟡 Intermediate`</summary>
<br>

```bash
docker inspect -f '{{.State.Pid}}' app
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' app
docker inspect -f '{{json .Mounts}}' app | jq
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' app
docker inspect -f '{{.Config.Env}}' app
docker inspect -f '{{.HostConfig.Memory}}' app
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps --filter status=exited --filter label=team=core -q
```

</details>

<details>
<summary><b>How do you copy files between host and container? Can you modify a running container?</b> `🟡 Intermediate`</summary>
<br>

```bash
docker cp app:/var/log/app.log ./app.log
docker cp ./fix.conf app:/etc/app/fix.conf
docker diff app            # A=added, C=changed, D=deleted vs image
docker commit app debug:snapshot   # snapshot for forensic use only
```

Containers should be **immutable**. Changes made inside are lost on recreate. Fix the Dockerfile and rebuild instead of using `docker commit` in pipelines, because commit gives opaque, unreproducible images.

</details>


## <a id="internals"></a>5. Docker Internals (Linux)

<details>
<summary><b>What are Linux namespaces? Which ones does Docker use?</b> `🔴 Advanced`</summary>
<br>

Namespaces limit **what a process can see**.
| Namespace | Isolates |
| --- | --- |
| pid | Process IDs (the container has its own PID 1) |
| net | Interfaces, IPs, routes, iptables, ports |
| mnt | Mount points / filesystem view |
| uts | Hostname and domain name |
| ipc | SysV IPC, POSIX message queues, shared memory |
| user | UID/GID mapping (root in the container = unprivileged on the host). Used with userns-remap or rootless mode |
| cgroup | The cgroup hierarchy view |
```bash
PID=$(docker inspect -f '{{.State.Pid}}' app)
ls -l /proc/$PID/ns
sudo nsenter -t $PID -n ip addr        # enter only the container's net namespace
sudo nsenter -t $PID -n ss -lntp       # debug with HOST tools in container netns
docker run --pid=host --network=host ...   # share host namespaces (avoid in prod)
```

</details>

<details>
<summary><b>What are cgroups? cgroup v1 vs v2?</b> `🔴 Advanced`</summary>
<br>

Control groups limit and account for **how much** a process can use: CPU, memory, block I/O, PIDs, and devices. Namespaces handle isolation (what it sees), cgroups handle resources (what it uses).
- **v1**: separate hierarchies per controller (`/sys/fs/cgroup/memory/...`).
- **v2**: a single unified hierarchy, better memory accounting (including page cache and PSI pressure metrics), safer delegation for rootless. It is the default on modern distros (RHEL 9, Ubuntu 22.04+).

```bash
stat -fc %T /sys/fs/cgroup        # cgroup2fs => v2
docker info | grep -i cgroup
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/cpu.stat
```

</details>

<details>
<summary><b>How does the overlay2 storage driver work?</b> `🔴 Advanced`</summary>
<br>

OverlayFS merges directories into one view:
- **lowerdir**: read-only image layers (stacked).
- **upperdir**: the container's writable layer.
- **workdir**: internal scratch space for overlayfs.
- **merged**: the unified view mounted as the container's root.


Reads fall through to the highest layer that has the file. The first write to a lower file triggers a **copy-up** of the whole file, which is expensive for large files such as databases. That is why write-heavy data should live in **volumes**, which bypass overlayfs.
```bash
docker info --format '{{.Driver}}'          # overlay2
docker inspect -f '{{.GraphDriver.Data.UpperDir}}' app
mount | grep overlay
```

</details>

<details>
<summary><b>What are Linux capabilities and how does Docker use them?</b> `🔴 Advanced`</summary>
<br>

Root's privileges are split into ~40 capabilities (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_NET_BIND_SERVICE`...). Docker gives containers a **reduced default set** (for example CHOWN, SETUID, NET_BIND_SERVICE, KILL) and drops dangerous ones such as SYS_ADMIN, NET_ADMIN, and SYS_PTRACE.
```bash
# Least privilege: drop all, add only what is needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
docker run --cap-add NET_ADMIN nicolaka/netshoot tc qdisc ...
docker inspect -f '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}' app
# inside a container:  grep Cap /proc/1/status ; capsh --decode=<hex>
```

</details>

<details>
<summary><b>What is seccomp and AppArmor/SELinux in Docker?</b> `🔴 Advanced`</summary>
<br>

- **seccomp**: filters **syscalls**. Docker's default profile blocks dozens of dangerous syscalls (for example `kexec_load`, `mount`, `reboot`, `bpf` for unprivileged users). Custom profiles: `--security-opt seccomp=profile.json`.
- **AppArmor** (Ubuntu/Debian): the `docker-default` profile restricts file and mount access.
- **SELinux** (RHEL/Fedora/OpenShift): labels such as `container_t`. For bind mounts use `:z` (shared label) or `:Z` (private label), otherwise you get "permission denied" even as root.

```bash
docker run -v /srv/data:/data:Z app
docker run --security-opt seccomp=unconfined app    # debug only!
docker info --format '{{.SecurityOptions}}'
```

</details>

<details>
<summary><b>What exactly does --privileged do? Why is it dangerous?</b> `🔴 Advanced`</summary>
<br>

`--privileged` grants **all capabilities**, disables seccomp and AppArmor/SELinux confinement, exposes all host **devices** (`/dev/*`), and makes `/sys` writable. A privileged container can mount the host disk and escape trivially:
```bash
# proof of why it is dangerous (do not do this in prod):
docker run --rm -it --privileged alpine sh -c 'mkdir /h && mount /dev/sda1 /h && ls /h/etc'
```

Instead, use targeted flags: `--cap-add`, `--device /dev/net/tun`, or `--security-opt`. Legitimate uses are rare (DinD, some CNF/DPDK workloads, hardware access).

</details>

<details>
<summary><b>How would you create a "container" without Docker? (Shows deep understanding)</b> `🔴 Advanced`</summary>
<br>

A container is just namespaces + cgroups + a root filesystem + security filters:
```bash
mkdir -p rootfs && docker export $(docker create alpine) | tar -C rootfs -x
sudo unshare --pid --fork --mount --uts --ipc --net --mount-proc=rootfs/proc \
     chroot rootfs /bin/sh
# inside: hostname demo; ps aux  -> you are PID 1, isolated net, own hostname

# add resource limits via cgroup v2
sudo mkdir /sys/fs/cgroup/demo
echo 100M | sudo tee /sys/fs/cgroup/demo/memory.max
echo $$   | sudo tee /sys/fs/cgroup/demo/cgroup.procs
```

runc does exactly this from an OCI `config.json` (`runc spec` generates one).

</details>


## <a id="networking"></a>6. Docker Networking

<details>
<summary><b>What network drivers does Docker support?</b> `🟡 Intermediate`</summary>
<br>

| Driver | Description / Use case |
| --- | --- |
| bridge (default) | A private network on a Linux bridge (docker0), with NAT to the outside. Single host. |
| host | Shares the host network stack. No isolation and no port mapping, best performance. Linux only. |
| none | Loopback only. Fully isolated. |
| overlay | Multi-host network over VXLAN (Swarm). Containers on different hosts talk as if on one L2. |
| macvlan | The container gets its own MAC/IP on the physical LAN. Good for legacy apps and network appliances. |
| ipvlan | Like macvlan but shares the host MAC (L2 or L3 mode). Useful where switches limit MACs per port. |
| container:<name> | Joins another container's net namespace (sidecar pattern, like a Kubernetes pod). |
```bash
docker network ls
docker network create -d bridge --subnet 172.30.0.0/24 backend
docker network create -d macvlan --subnet 192.168.1.0/24 --gateway 192.168.1.1 -o parent=eth0 lan
docker run --network container:app nicolaka/netshoot ss -lntp
```

</details>

<details>
<summary><b>Default bridge vs user-defined bridge - why prefer user-defined?</b> `🟡 Intermediate`</summary>
<br>

- **Automatic DNS**: on user-defined networks, containers resolve each other **by name** through Docker's embedded DNS at `127.0.0.11`. The default bridge supports only IPs (or the legacy `--link`).
- **Better isolation**: only containers on the same network can talk to each other.
- You can connect and disconnect running containers on the fly.
- Configurable subnets, MTU, and options per network.

```bash
docker network create app-net
docker run -d --name db  --network app-net -e POSTGRES_PASSWORD=x postgres:16
docker run -d --name api --network app-net -e DB_HOST=db myapi   # "db" resolves
docker run --rm --network app-net busybox nslookup db
docker network connect frontend api     # multi-homed container
```

</details>

<details>
<summary><b>How does port publishing work under the hood?</b> `🔴 Advanced`</summary>
<br>

On the bridge network each container gets a **veth pair**: one end is `eth0` inside the container, the other is attached to `docker0` or the user bridge on the host. For `-p 8080:80` Docker adds:
- A **DNAT** rule in the `nat/DOCKER` chain (hostIP:8080 to containerIP:80).
- **MASQUERADE** (SNAT) for outbound traffic from the container subnet.
- FORWARD rules, plus a userland `docker-proxy` for hairpin and localhost cases.

```bash
sudo iptables -t nat -L DOCKER -n -v
sudo iptables -L DOCKER-USER -n          # put YOUR filtering rules here
ip link show type veth ; bridge link
```
> ⚠️ **Warning:** Classic gotcha: published ports bypass UFW/firewalld INPUT rules because DNAT happens in PREROUTING and the traffic goes through FORWARD. Bind to 127.0.0.1:port or filter in the DOCKER-USER chain.

</details>

<details>
<summary><b>How does overlay networking work across hosts? Which ports are needed?</b> `🔴 Advanced`</summary>
<br>

Overlay networks encapsulate container L2 frames in **VXLAN** (UDP) between hosts. A gossip-based control plane distributes the endpoint and IP/MAC mappings. Swarm required ports:
- **TCP 2377**: cluster management (Raft).
- **TCP/UDP 7946**: node gossip / discovery.
- **UDP 4789**: VXLAN data plane.
- ESP (IP protocol 50) if you use `--opt encrypted`.

```bash
docker network create -d overlay --attachable --opt encrypted mesh
```
> ⚠️ **Warning:** VXLAN adds 50 bytes of header. If the underlay MTU is 1500 (or less on VPN/cloud), set the overlay MTU to 1450 or lower, otherwise large packets (TLS handshakes, big SIP INVITEs) silently hang.

</details>

<details>
<summary><b>How do containers resolve DNS? How do you customize it?</b> `🟡 Intermediate`</summary>
<br>

On user-defined networks, `/etc/resolv.conf` points to the embedded DNS at `127.0.0.11`. It answers container names, service names, and aliases itself and forwards everything else to the host's resolvers. On the default bridge, the container gets a copy of the host's resolv.conf (localhost entries such as systemd-resolved's 127.0.0.53 are replaced with public fallbacks).
```bash
docker run --dns 10.0.0.2 --dns-search corp.local --add-host api.local:10.1.1.5 app
docker run --network app-net --network-alias cache redis   # extra DNS name
# daemon.json global:  { "dns": ["10.0.0.2", "8.8.8.8"] }
```

</details>

<details>
<summary><b>How do you access a service on the host from inside a container?</b> `🟡 Intermediate`</summary>
<br>

- Docker Desktop: use `host.docker.internal`.
- Linux: add `--add-host=host.docker.internal:host-gateway`, which maps to the bridge gateway IP.
- Or use `--network host`.
- The host service must listen on an interface reachable from the bridge, not only 127.0.0.1.

```bash
docker run --add-host=host.docker.internal:host-gateway curlimages/curl \
  curl http://host.docker.internal:9000
```

</details>

<details>
<summary><b>Two containers cannot talk to each other. How do you troubleshoot?</b> `🟣 Scenario`</summary>
<br>

1. Are they on the **same network**? `docker inspect -f '{{json .NetworkSettings.Networks}}' a b`
2. On the default bridge? Name resolution will not work there. Use a user-defined network.
3. DNS: `docker exec a nslookup b` or `getent hosts b`.
4. Is the target listening on **0.0.0.0** and the right port? `docker exec b ss -lntp`, or `nsenter -t PID -n ss -lntp` if the image has no tools.
5. Use the **container port**, not the published host port, for container-to-container traffic.
6. Check firewall, iptables, and ICC (`"icc": false`), and network policies.
7. Run a debug sidecar: `docker run --rm -it --network container:b nicolaka/netshoot`, then use `tcpdump -i eth0 port 5432`.

</details>


## <a id="storage"></a>7. Storage, Volumes & Data Persistence

<details>
<summary><b>Volumes vs bind mounts vs tmpfs?</b> `🟢 Basic`</summary>
<br>

|  | Named Volume | Bind Mount | tmpfs |
| --- | --- | --- | --- |
| Location | Managed by Docker (/var/lib/docker/volumes) | Any host path | Host memory |
| Persistence | Survives container removal | Yes (host files) | No, gone on stop |
| Pre-populated from image | Yes (if empty) | No, host content hides the image content | No |
| Portability | High; supports drivers (NFS, cloud) | Host-dependent | - |
| Best for | Databases, app data | Dev live reload, config files | Secrets, scratch, caches |
```bash
docker volume create pgdata
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16
docker run -d --mount type=bind,src="$PWD"/src,dst=/app/src,readonly node:20
docker run -d --mount type=tmpfs,dst=/tmp,tmpfs-size=64m app
```
> 💡 **Tip:** --mount is more explicit than -v. With -v, a missing host path is silently created as a directory owned by root, while --mount errors out.

</details>

<details>
<summary><b>How do you back up and restore a Docker volume?</b> `🟡 Intermediate`</summary>
<br>

```bash
# backup
docker run --rm -v pgdata:/data:ro -v "$PWD":/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tgz -C /data .
# restore
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine \
  sh -c "cd /data && tar xzf /backup/pgdata-2026-10-26.tgz"
```
> ⚠️ **Warning:** For databases, prefer logical or consistent backups (pg_dump, mysqldump), or stop the container first. Copying live DB files can give a corrupt backup.
```bash
docker exec db pg_dump -U postgres appdb | gzip > appdb.sql.gz
```

</details>

<details>
<summary><b>"Permission denied" writing to a bind mount. How do you fix it?</b> `🟣 Scenario`</summary>
<br>

Root cause: the container process UID/GID does not match the host directory owner. Bind mounts do not translate UIDs. On SELinux hosts, the files may also be missing the right label.
- Run as your UID: `docker run --user "$(id -u):$(id -g)" ...`
- Or `chown` the host directory to the container's UID (for example 999 for postgres, 10001 for your app).
- Build the image with a matching UID: `ARG UID=1000`, then `useradd -u $UID`.
- SELinux: add `:z` or `:Z`. Check with `ls -Z` and `ausearch -m avc`.
- Rootless or userns-remap: the host UID is shifted (for example 100000+). Use `podman unshare chown` or the matching subuid range.

</details>

<details>
<summary><b>What happens to data when a container is removed? Anonymous vs named volumes?</b> `🟡 Intermediate`</summary>
<br>

- The writable layer is deleted with the container.
- **Named volumes** persist until you run `docker volume rm`.
- **Anonymous volumes** (from `VOLUME` in a Dockerfile or `-v /path`) persist as orphans unless you use `docker rm -v` or `--rm`. They are a common cause of disk bloat.

```bash
docker volume ls -f dangling=true
docker rm -v old-container
```

</details>

<details>
<summary><b>How do you use NFS or shared storage with volumes? How do you move Docker's data-root?</b> `🔴 Advanced`</summary>
<br>

```bash
docker volume create --driver local \
  --opt type=nfs --opt o=addr=10.0.0.10,rw,nfsvers=4 \
  --opt device=:/exports/shared shared-nfs
docker run -v shared-nfs:/data app
```

Move `/var/lib/docker` to a bigger disk:
```bash
sudo systemctl stop docker
sudo rsync -aHAX /var/lib/docker/ /data/docker/
echo '{ "data-root": "/data/docker" }' | sudo tee /etc/docker/daemon.json
sudo systemctl start docker && docker info | grep "Docker Root Dir"
```

</details>


## <a id="compose"></a>8. Docker Compose

<details>
<summary><b>What is Docker Compose? Write a full-stack example.</b> `🟢 Basic`</summary>
<br>

Compose defines and runs multi-container applications from a YAML file: services, networks, volumes, secrets, and configs. It runs as `docker compose` (v2, a Go plugin). The top-level `version:` key is obsolete.
```bash
# compose.yaml
name: shop
services:
  web:
    build:
      context: .
      target: final
      args: { APP_VERSION: "1.4.2" }
    image: registry.example.com/shop/web:1.4.2
    ports: ["8080:8080"]
    environment:
      DB_HOST: db
      REDIS_URL: redis://cache:6379
    env_file: .env.web
    depends_on:
      db:    { condition: service_healthy }
      cache: { condition: service_started }
    restart: unless-stopped
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
    networks: [front, back]

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pw
    secrets: [db_pw]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [back]

  cache:
    image: redis:7-alpine
    networks: [back]

networks:
  front: {}
  back: { internal: true }     # no external access

volumes:
  pgdata: {}

secrets:
  db_pw:
    file: ./secrets/db_pw.txt
```
```bash
docker compose up -d --build
docker compose ps ; docker compose logs -f web
docker compose exec db psql -U postgres
docker compose down          # keep volumes
docker compose down -v       # also delete volumes
```

</details>

<details>
<summary><b>depends_on does not wait for my DB to be ready. Why, and how do you fix it?</b> `🟡 Intermediate`</summary>
<br>

Plain `depends_on` only controls **start order**. It waits for the container to start, not for the app inside to be ready. Fix:
1. Add a `healthcheck` to the dependency and use `condition: service_healthy`.
2. Use `condition: service_completed_successfully` for one-shot jobs such as migrations.
3. Most important: make the app **retry connections** with backoff. In production, dependencies restart independently anyway.

```bash
services:
  migrate:
    image: app
    command: ["./migrate", "up"]
    depends_on: { db: { condition: service_healthy } }
  api:
    image: app
    depends_on:
      migrate: { condition: service_completed_successfully }
```

</details>

<details>
<summary><b>How do you manage multiple environments (dev/stage/prod) in Compose?</b> `🟡 Intermediate`</summary>
<br>

- **Override files**: `compose.yaml` is the base, and `compose.override.yaml` is auto-merged for dev. Add explicit `-f` files for other environments.
- **Variable interpolation** with `${VAR:-default}` from `.env` or the shell.
- **Profiles** for optional services.
- **include:** / **extends:** to reuse definitions.

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose --profile debug up -d          # starts services with profiles: [debug]
docker compose --env-file .env.staging config  # render final merged config
```
```bash
services:
  web:
    image: myorg/web:${TAG:-latest}
  adminer:
    image: adminer
    profiles: ["debug"]
```
> ⚠️ **Warning:** The .env file is used for interpolation in the compose file. It is NOT injected into containers unless you reference it with env_file: or in environment:.

</details>

<details>
<summary><b>How do you scale a service in Compose? What are the limitations?</b> `🟡 Intermediate`</summary>
<br>

```bash
docker compose up -d --scale worker=5
```

Limitations: you cannot pin a fixed host port on a scaled service (port conflict), so use a port range (`"8000-8005:8000"`) or put a reverse proxy in front. Do not set `container_name`. Compose is single-host with no self-healing across nodes, so use Swarm or Kubernetes for real scaling. DNS round-robin works: `worker` resolves to all replicas.

</details>

<details>
<summary><b>What is docker compose watch / how do you set up hot reload for dev?</b> `🟡 Intermediate`</summary>
<br>

```bash
services:
  web:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: package.json
```
```bash
docker compose watch
```

Alternative: bind-mount the source plus a dev server with a file watcher (nodemon, uvicorn `--reload`). Use an anonymous volume for `/app/node_modules` so the host directory does not hide it.

</details>


## <a id="security"></a>9. Docker Security

<details>
<summary><b>What are Docker security best practices? (Must-know list)</b> `🟡 Intermediate`</summary>
<br>

1. **Run as non-root** (`USER 10001`). Use rootless Docker or userns-remap.
2. **Minimal base images** (distroless/slim) and multi-stage builds, so there is less to exploit.
3. **Scan images** (Trivy, Grype, Docker Scout, Snyk) in CI and fail on critical CVEs.
4. **Pin and verify**: pin base images by digest and sign images (cosign / Notation).
5. **No secrets in images**: use BuildKit secrets for builds and runtime secret stores (Swarm/K8s secrets, Vault).
6. **Least privilege at runtime**: `--cap-drop ALL`, `--security-opt no-new-privileges`, `--read-only`, keep seccomp/AppArmor on, never `--privileged`.
7. **Resource limits** (memory, CPU, pids) against DoS.
8. **Protect the daemon**: never expose TCP 2375 unauthenticated. Use TLS (2376) or SSH. Limit the `docker` group.
9. **Never mount docker.sock** into untrusted containers.
10. Keep the Engine, kernel, and runc patched (for example runc CVE-2019-5736, CVE-2024-21626 "Leaky Vessels").

```bash
docker run -d --user 10001:10001 --read-only --tmpfs /tmp \
  --cap-drop ALL --security-opt no-new-privileges:true \
  --memory 256m --pids-limit 100 myapp:1.0@sha256:...
```

</details>

<details>
<summary><b>Why is mounting /var/run/docker.sock dangerous? Why is the docker group root-equivalent?</b> `🔴 Advanced`</summary>
<br>

Anyone who can talk to the Docker API can start a privileged container that mounts the host root filesystem, which gives full host root:
```bash
docker run --rm -it -v /:/host alpine chroot /host sh    # root shell on the host
```

So membership in the `docker` group or access to the socket equals root on the host. Mitigations: rootless Docker, a socket proxy with an API allow-list (for example a read-only proxy for Traefik or monitoring), daemonless builders (Kaniko, Buildah, BuildKit rootless) in CI, and authorization plugins.

</details>

<details>
<summary><b>Rootless Docker vs userns-remap vs USER in Dockerfile?</b> `🔴 Advanced`</summary>
<br>

| Approach | What runs as non-root | Notes |
| --- | --- | --- |
| USER in Dockerfile | The app process inside the container | The daemon is still root. Simplest; always do it. |
| userns-remap | Container root maps to an unprivileged host UID range | Daemon is root. Set "userns-remap": "default" in daemon.json. Breaks some bind mounts and host namespaces. |
| Rootless mode | The daemon and containers run as a normal user | Strongest. Limitations: ports below 1024 need sysctl, no AppArmor, overlay needs newer kernels, slirp4netns/pasta networking is slower. |
```bash
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

</details>

<details>
<summary><b>How do you manage secrets in Docker?</b> `🟡 Intermediate`</summary>
<br>

- **Never** put secrets in `ENV`, `ARG`, or COPY'd files. They persist in layers and history.
- **Build time**: `RUN --mount=type=secret` (BuildKit). The secret is tmpfs-mounted only during that RUN.
- **Runtime**: Swarm/Compose secrets mounted at `/run/secrets/<name>`, Kubernetes Secrets, or Vault / cloud secret managers.
- Use the `*_FILE` env var convention that official images support (for example `POSTGRES_PASSWORD_FILE`).

```bash
echo -n 'S3cr3t' | docker secret create db_pw -           # swarm
docker service create --secret db_pw -e POSTGRES_PASSWORD_FILE=/run/secrets/db_pw postgres:16
# detect leaks
docker history --no-trunc myimg | grep -i -E 'pass|token|key'
```

</details>

<details>
<summary><b>How do you scan images for vulnerabilities and sign them?</b> `🟡 Intermediate`</summary>
<br>

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 myorg/api:1.4.2
docker scout cves myorg/api:1.4.2
grype myorg/api:1.4.2
syft myorg/api:1.4.2 -o spdx-json > sbom.json          # SBOM
docker buildx build --sbom=true --provenance=mode=max -t myorg/api:1.4.2 --push .

# signing with Sigstore cosign
cosign sign --key cosign.key myorg/api@sha256:<digest>
cosign verify --key cosign.pub myorg/api@sha256:<digest>
```

Enforce verification at deploy time with an admission controller (Kyverno, Sigstore policy-controller, Connaisseur). Docker Content Trust (`DOCKER_CONTENT_TRUST=1`, Notary v1) is legacy. cosign and Notation are the modern options.

</details>

<details>
<summary><b>How do you secure the Docker daemon remote API?</b> `🔴 Advanced`</summary>
<br>

- Never use `-H tcp://0.0.0.0:2375` without TLS. Attackers scan for it to run cryptominers.
- Prefer SSH: `docker -H ssh://user@host ps` or `docker context create remote --docker host=ssh://user@host`.
- Or mutual TLS on 2376 with `--tlsverify` and CA, server, and client certificates.

```bash
docker context create prod --docker "host=ssh://deploy@10.0.0.5"
docker context use prod && docker ps
```

</details>


## <a id="registry"></a>10. Registries & Image Distribution

<details>
<summary><b>How do you push an image to a private registry?</b> `🟢 Basic`</summary>
<br>

```bash
docker login registry.example.com
docker tag app:1.0 registry.example.com/team/app:1.0
docker push registry.example.com/team/app:1.0
# run your own registry
docker run -d -p 5000:5000 -v reg:/var/lib/registry --name registry registry:2
docker tag app:1.0 localhost:5000/app:1.0 && docker push localhost:5000/app:1.0
curl -s localhost:5000/v2/_catalog
```

For enterprise use Harbor, which adds RBAC, scanning, replication, and signing, or a cloud registry (ECR, ACR, GAR, Quay).

</details>

<details>
<summary><b>"x509: certificate signed by unknown authority" / insecure registry error - how do you fix it?</b> `🟡 Intermediate`</summary>
<br>

```bash
# Correct fix: trust the registry's CA
sudo mkdir -p /etc/docker/certs.d/registry.example.com:5000
sudo cp ca.crt /etc/docker/certs.d/registry.example.com:5000/ca.crt
# Lab only: allow plain HTTP / untrusted
# /etc/docker/daemon.json
{ "insecure-registries": ["registry.lab:5000"] }
sudo systemctl restart docker
```

</details>

<details>
<summary><b>How do you handle Docker Hub rate limits and air-gapped environments?</b> `🟡 Intermediate`</summary>
<br>

- Authenticate pulls (`docker login`). Anonymous and free-tier pulls are rate-limited.
- Run a **pull-through cache / mirror** (Harbor proxy cache, a registry:2 proxy, Artifactory, Nexus).
- Mirror base images into your private registry (`skopeo copy`, `crane copy`) and pin by digest.
- Air-gapped: `docker save`/`load` or `skopeo copy docker://... oci-archive:...`.

```bash
{ "registry-mirrors": ["https://mirror.example.com"] }
skopeo copy docker://docker.io/library/nginx:1.27 docker://registry.example.com/mirror/nginx:1.27
```

</details>

<details>
<summary><b>What is a manifest / manifest list (image index)?</b> `🔴 Advanced`</summary>
<br>

A **manifest** describes one image: its config blob digest plus an ordered list of layer digests. A **manifest list / OCI index** points to multiple platform-specific manifests (amd64, arm64...). When you pull, the client picks the manifest matching your platform. Layers are fetched only if they are not already present (content-addressed dedup).
```bash
docker manifest inspect nginx:1.27 | jq '.manifests[].platform'
docker buildx imagetools inspect nginx:1.27
crane manifest nginx:1.27
```

</details>


## <a id="orchestration"></a>11. Docker Swarm & Orchestration

<details>
<summary><b>What is Docker Swarm? Explain its key concepts.</b> `🟡 Intermediate`</summary>
<br>

- **Nodes**: managers (Raft consensus, keep an odd count of 3 or 5) and workers.
- **Service**: the desired state (image, replicas, ports, update policy).
- **Task**: one container instance of a service, scheduled on a node.
- **Routing mesh**: a published port is reachable on *every* node and load-balanced through IPVS to the tasks.
- **Stack**: a Compose file deployed as a set of services.
- Built-in secrets/configs, mTLS between nodes, overlay networking, and rolling updates.

```bash
docker swarm init --advertise-addr 10.0.0.1
docker swarm join-token worker
docker service create --name web --replicas 3 -p 80:80 \
  --update-parallelism 1 --update-delay 10s --update-order start-first \
  --rollback-parallelism 1 nginx:1.27
docker service scale web=5
docker service update --image nginx:1.27.2 web
docker service rollback web
docker stack deploy -c compose.yaml shop
```

</details>

<details>
<summary><b>Docker Swarm vs Kubernetes?</b> `🟡 Intermediate`</summary>
<br>

|  | Swarm | Kubernetes |
| --- | --- | --- |
| Complexity | Very simple, built into Docker | Steep learning curve |
| Ecosystem | Small | Huge (Helm, operators, service mesh, CNI/CSI) |
| Scaling | Manual / basic | HPA, VPA, cluster autoscaler |
| Networking | Overlay + routing mesh | Pluggable CNI, Services, Ingress/Gateway API, NetworkPolicy |
| Storage | Volume plugins | PV/PVC/StorageClass, CSI |
| Use when | Small teams, simple apps | Production at scale, multi-team, enterprise (OpenShift) |

</details>

<details>
<summary><b>How does Swarm Raft quorum work? What happens if you lose managers?</b> `🔴 Advanced`</summary>
<br>

Managers replicate cluster state with Raft. A majority (**quorum** = floor(N/2)+1) is needed for changes. 3 managers tolerate 1 failure, 5 tolerate 2, and 7 tolerate 3. Even counts add no fault tolerance. If quorum is lost, running tasks keep running, but you cannot schedule, update, or scale. Recover with:
```bash
docker swarm init --force-new-cluster --advertise-addr <surviving-manager-ip>
```

</details>

<details>
<summary><b>How do you do zero-downtime deployments with Docker?</b> `🟡 Intermediate`</summary>
<br>

- **Swarm**: rolling update with `--update-order start-first`, health checks, and `--update-failure-action rollback`.
- **Compose (single host)**: a reverse proxy (Traefik, Nginx) with blue/green. Start the new container, wait for it to be healthy, switch traffic, then stop the old one.
- The app must handle **SIGTERM gracefully**: stop accepting connections, drain in-flight requests, then exit.
- Use readiness health checks so traffic only reaches ready instances.

```bash
services:
  api:
    image: myorg/api:${TAG}
    deploy:
      replicas: 3
      update_config: { parallelism: 1, delay: 10s, order: start-first, failure_action: rollback }
      restart_policy: { condition: on-failure }
    stop_grace_period: 30s
```

</details>


## <a id="cicd"></a>12. CI/CD, Docker-in-Docker & Best Practices

<details>
<summary><b>Docker-in-Docker (DinD) vs Docker-outside-of-Docker (DooD) vs Kaniko/Buildah?</b> `🔴 Advanced`</summary>
<br>

| Approach | How | Pros | Cons |
| --- | --- | --- | --- |
| DinD | Run docker:dind with --privileged; a nested daemon | Isolated, clean state per job | Needs privileged, no shared cache, storage driver issues |
| DooD | Mount the host /var/run/docker.sock | Fast, shares cache | Host root exposure, container name clashes, jobs see each other |
| Kaniko / Buildah / BuildKit rootless | Builds images in userspace without a daemon | No privileged mode; Kubernetes-friendly | Some feature differences, slower without cache |
```bash
# GitLab CI DinD
build:
  image: docker:27
  services: [docker:27-dind]
  variables: { DOCKER_TLS_CERTDIR: "/certs" }
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

# Kaniko in Kubernetes
/kaniko/executor --context . --dockerfile Dockerfile \
  --destination registry.example.com/app:$SHA --cache=true
```

</details>

<details>
<summary><b>Describe a typical Docker CI/CD pipeline.</b> `🟡 Intermediate`</summary>
<br>

1. Lint the Dockerfile (`hadolint`) and the code.
2. Build with BuildKit and registry cache, tagging with the git SHA.
3. Run unit tests in a test stage (`--target test`) and integration tests with Compose.
4. Scan (Trivy), generate an SBOM, and fail on critical CVEs.
5. Push to the registry and sign with cosign.
6. Promote the **same digest** through dev, stage, and prod. Never rebuild per environment.
7. Deploy (Helm/ArgoCD/Swarm), run smoke tests, and roll back automatically on failure.

```bash
# GitHub Actions snippet
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/org/app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
    platforms: linux/amd64,linux/arm64
```

</details>

<details>
<summary><b>What are Dockerfile and container best practices (12-factor)?</b> `🟡 Intermediate`</summary>
<br>

- One concern per container. The container lifecycle equals the process lifecycle.
- Ephemeral and stateless: state goes in volumes or external services.
- Configuration through environment variables or mounted config, not baked into the image.
- Log to stdout/stderr.
- Build once and promote the same image across environments.
- Pin versions, use non-root, run a HEALTHCHECK, handle SIGTERM gracefully.
- Lint with hadolint and keep images small and scanned.
- Use OCI labels (source, revision, version) for traceability.

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
```

</details>

<details>
<summary><b>Alpine vs Debian-slim vs distroless vs scratch - how do you choose?</b> `🟡 Intermediate`</summary>
<br>

| Base | Size | Pros | Cons |
| --- | --- | --- | --- |
| scratch | 0 MB | Nothing to attack | Static binaries only; no shell, CA certs, or tzdata unless you copy them in |
| distroless | ~2-20 MB | No shell or package manager; has CA certs and nonroot user | Harder to debug (use :debug tags or ephemeral debug containers) |
| alpine | ~5-8 MB | Tiny and has a shell | musl libc: Python wheels compile slowly, DNS and locale quirks, possible performance differences |
| debian-slim / ubi-minimal | ~30-80 MB | glibc compatibility, familiar tooling | Larger, more CVEs to patch |

</details>


## <a id="troubleshooting"></a>13. Troubleshooting & Debugging

<details>
<summary><b>A container exits immediately after starting. How do you debug it?</b> `🟣 Scenario`</summary>
<br>

1. `docker ps -a` shows the exit code. `docker logs <c>` shows the error output.
2. `docker inspect -f '{{.State}}'` shows OOMKilled and the error.
3. Common causes:
<ul>
<li>The main process finished. Containers live only as long as PID 1, so a `CMD` that runs a background daemon and exits will stop the container. Run the daemon in the foreground (`nginx -g 'daemon off;'`).</li>
<li>Wrong CMD or entrypoint path (127), script not executable (126), Windows CRLF line endings in the script.</li>
<li>Missing env vars or config, or the app crashes on startup.</li>
<li>Architecture mismatch: `exec format error` (arm64 image on amd64).</li>
</ul>
4. Override the entrypoint to look around: `docker run -it --entrypoint sh myimg`.

```bash
docker run --rm -it --entrypoint sh myimg -c 'ls -l /app; cat /app/entrypoint.sh | od -c | head'
docker image inspect -f '{{.Os}}/{{.Architecture}}' myimg
```

</details>

<details>
<summary><b>App works locally but the port is not reachable when it runs in a container. Why?</b> `🟣 Scenario`</summary>
<br>

#1 cause: the app binds to **127.0.0.1** (localhost inside the container), so it is unreachable through the published port. It must bind to **0.0.0.0**.
```bash
flask run --host=0.0.0.0
uvicorn app:app --host 0.0.0.0 --port 8000
docker exec app ss -lntp           # check LISTEN address
```

Other causes: a wrong port mapping (`-p host:container` reversed), the container is not running, a host firewall or cloud security group, or the port is published only on 127.0.0.1.

</details>

<details>
<summary><b>Container keeps getting killed with exit code 137. What do you do?</b> `🟣 Scenario`</summary>
<br>

1. Check `docker inspect -f '{{.State.OOMKilled}}'`. If true, the memory limit was hit.
2. Check host logs: `dmesg -T | grep -i -E 'killed process|oom'`, `journalctl -k`.
3. Check `docker stats` trends, then decide whether it is a memory leak or an undersized limit.
4. Make the runtime aware of the limit: JVM `-XX:MaxRAMPercentage`, `GOMEMLIMIT`, Node `--max-old-space-size`.
5. If OOMKilled is false, something sent SIGKILL: `docker kill`, a stop timeout (app ignores SIGTERM), or the host OOM killer under node memory pressure.

</details>

<details>
<summary><b>The host disk is full because of Docker. How do you investigate and fix it?</b> `🟣 Scenario`</summary>
<br>

```bash
docker system df -v
sudo du -sh /var/lib/docker/* | sort -h
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail   # huge logs?
# immediate relief for a huge log (container keeps running)
sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log
docker image prune -a ; docker builder prune ; docker volume ls -f dangling=true
```

**Permanent fixes:** log rotation in daemon.json, scheduled pruning in CI agents, avoid writing to the container layer (use volumes), and move `data-root` to a larger disk.

</details>

<details>
<summary><b>How do you debug a distroless/scratch container that has no shell?</b> `🟣 Scenario`</summary>
<br>

Attach a tools container that shares the target's namespaces:
```bash
# share network + PID namespace: see its processes and sockets
docker run --rm -it --network container:app --pid container:app nicolaka/netshoot
  ps aux ; ss -lntp ; tcpdump -i eth0 -nn port 8080 ; curl localhost:8080/healthz
  ls /proc/1/root/app        # browse target's filesystem via /proc

# or use host tools with nsenter
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' app) -n -p tcpdump -i any

# Docker Desktop: docker debug app ; Kubernetes: kubectl debug -it pod --image=busybox --target=app
```

</details>

<details>
<summary><b>The container has no internet access / DNS resolution fails. How do you troubleshoot?</b> `🟣 Scenario`</summary>
<br>

1. `docker run --rm busybox ping -c1 8.8.8.8` tests IP connectivity. `nslookup google.com` tests DNS.
2. IP works but DNS fails: check `/etc/resolv.conf` in the container, a corporate DNS that is unreachable from the bridge, or systemd-resolved at 127.0.0.53. Fix with `"dns"` in daemon.json.
3. No IP connectivity: check `sysctl net.ipv4.ip_forward` (must be 1), the iptables MASQUERADE rule, firewalld/nftables flushing Docker rules (restart docker), proxy settings (`HTTP_PROXY`), and a subnet conflict between the Docker bridge and your VPN or corporate LAN (change `"bip"` / `"default-address-pools"`).
4. Small requests work but big ones hang: an **MTU** problem (VPN, overlay, OpenStack). Set `"mtu": 1400`.

```bash
# /etc/docker/daemon.json
{
  "dns": ["10.10.0.2"],
  "mtu": 1400,
  "bip": "172.26.0.1/16",
  "default-address-pools": [{ "base": "172.80.0.0/12", "size": 24 }]
}
```

</details>

<details>
<summary><b>Docker builds are slow. How do you speed them up?</b> `🟣 Scenario`</summary>
<br>

- Shrink the build context (`.dockerignore`). Watch the "transferring context: 2GB" line.
- Reorder layers for cache hits (dependency manifests first).
- Use BuildKit cache mounts for package managers.
- Use multi-stage builds with parallel stages.
- Use remote cache (`--cache-from/--cache-to`) in ephemeral CI.
- Avoid QEMU emulation for multi-arch: cross-compile, or use native arm64 builders.
- Pin and pre-pull base images, and use a local registry mirror.
- Profile with `docker build --progress=plain` to see per-step timings.

</details>

<details>
<summary><b>The Docker daemon will not start or is unresponsive. What do you check?</b> `🟣 Scenario`</summary>
<br>

```bash
sudo systemctl status docker
sudo journalctl -u docker -n 200 --no-pager
sudo dockerd --debug --validate --config-file /etc/docker/daemon.json   # validate config JSON
df -h /var/lib/docker ; df -i /var/lib/docker                           # disk or inodes full
ls -l /var/run/docker.sock ; id                                          # permission (docker group)
```

Common causes: invalid `daemon.json` (a flag set both in the systemd unit `-H` and the JSON), full disk or inodes, a storage driver incompatible with the filesystem, iptables/nftables backend conflicts, a stale containerd, or corrupt metadata after a crash.

</details>

<details>
<summary><b>Your application inside a container does not shut down gracefully and loses in-flight requests.</b> `🟣 Scenario`</summary>
<br>

1. Check what PID 1 is: `docker exec app ps -o pid,cmd 1`. If it is `sh -c`, switch to exec form or use `exec` in the script.
2. Make the app trap SIGTERM: stop accepting connections, drain, close the DB pool, then exit.
3. Increase the grace period: `docker stop -t 30` or `stop_grace_period: 30s`.
4. Set the correct `STOPSIGNAL` (nginx uses SIGQUIT for graceful shutdown).
5. Add `--init` if the app spawns child processes.

```bash
# Node.js
process.on('SIGTERM', () => { server.close(() => { db.end(); process.exit(0); }); });
```

</details>

<details>
<summary><b>A secret was accidentally baked into an image that was pushed. What do you do?</b> `🟣 Scenario`</summary>
<br>

1. **Rotate or revoke the secret immediately**. Assume it is compromised, because layers can be pulled by anyone with access.
2. Delete the affected tags and digests from the registry and run registry garbage collection. Check mirrors and caches.
3. Fix the Dockerfile: use BuildKit `--secret` and add the file to `.dockerignore`. Rebuild.
4. Add secret scanning to CI (gitleaks, trufflehog, `trivy image --scanners secret`).


Removing the file in a later layer does **not** remove it. It is still in the earlier layer tarball.

</details>

<details>
<summary><b>The timezone or locale in the container is wrong. How do you fix it?</b> `🟣 Scenario`</summary>
<br>

```bash
RUN apt-get update && apt-get install -y --no-install-recommends tzdata && rm -rf /var/lib/apt/lists/*
ENV TZ=Asia/Tokyo
```
```bash
docker run -e TZ=Asia/Tokyo -v /etc/localtime:/etc/localtime:ro app
```

Containers share the host **kernel clock**, so time itself is the same. Only the timezone database and setting differ. Prefer UTC in servers and logs.

</details>

<details>
<summary><b>How do you monitor Docker containers in production?</b> `🟣 Scenario`</summary>
<br>

- **Metrics**: cAdvisor or node-exporter, scraped by Prometheus and shown in Grafana. Enable the daemon metrics endpoint (`"metrics-addr": "127.0.0.1:9323"`).
- **Logs**: a logging driver or agent (Fluent Bit, Vector, Promtail) shipping to Loki or ELK.
- **Events**: `docker events --filter event=die --filter event=oom`.
- **Health**: HEALTHCHECK status, restart counts (`{{.RestartCount}}`).
- **Tracing**: OpenTelemetry in the app.

```bash
docker stats --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.PIDs}}'
docker events --since 1h --filter type=container --filter event=oom
docker run -d -p 8081:8080 -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro gcr.io/cadvisor/cadvisor
```

</details>


## <a id="misc"></a>14. Rapid-Fire & Tricky Questions

<details>
<summary><b>Can you run Windows containers on Linux, or Linux containers on Mac/Windows?</b> `🟡 Intermediate`</summary>
<br>

Containers share the host kernel, so **Linux containers need a Linux kernel**. Docker Desktop on Mac/Windows runs a lightweight Linux VM (WSL2/HyperKit/Virtualization.framework). Windows containers need a Windows kernel, so they cannot run on Linux hosts.

</details>

<details>
<summary><b>Can a container run more than one process? Should it?</b> `🟡 Intermediate`</summary>
<br>

It can (supervisord, s6-overlay), but the best practice is **one concern per container**. That gives independent scaling, cleaner logs, proper signal handling, and simpler health checks. Exceptions exist (for example a tightly coupled helper). In Kubernetes, use sidecars in a pod instead.

</details>

<details>
<summary><b>How many containers can run on a host?</b> `🟡 Intermediate`</summary>
<br>

There is no fixed Docker limit. It is bounded by host resources: memory, CPU, PIDs (`kernel.pid_max`), file descriptors, IP addresses in the bridge subnet (a /16 gives about 65k), iptables rule scale, and the storage driver.

</details>

<details>
<summary><b>What happens to data written in the container layer when the container stops (not removed)?</b> `🟡 Intermediate`</summary>
<br>

It is **preserved**. Stop and start keeps the writable layer. Only `docker rm` deletes it. It still should not be used for persistent data, because writes are slower (copy-up) and the data is tied to that one container.

</details>

<details>
<summary><b>What is the difference between docker run, docker create, and docker start?</b> `🟡 Intermediate`</summary>
<br>

`create` builds the container (writable layer and config) without starting it. `start` starts an existing container. `run` = pull if needed + create + start (+ attach).

</details>

<details>
<summary><b>Does Docker's "--network host" work on Docker Desktop? What about -p with host network?</b> `🟡 Intermediate`</summary>
<br>

With host networking, `-p` is ignored because the container uses the host ports directly. Host networking is native on Linux. Docker Desktop added host networking support only in recent versions (opt-in), and it applies to the Linux VM.

</details>

<details>
<summary><b>What is the difference between --cpus and --cpu-shares?</b> `🔴 Advanced`</summary>
<br>

- `--cpus=1.5` is a **hard cap** (CFS quota/period). The container is throttled even if the host is idle.
- `--cpu-shares=512` is a **relative weight** that only matters under contention. Idle CPU is still usable (default 1024, which maps to `cpu.weight` in v2).


Analogy to Kubernetes: shares are like CPU requests, quota is like CPU limits.

</details>

<details>
<summary><b>What is the "--network container:" or "--pid container:" mode used for?</b> `🔴 Advanced`</summary>
<br>

It joins another container's namespace. That is how Kubernetes pods work: a pause container holds the network namespace and app containers join it, sharing `localhost`. Use it for sidecars (proxy, log shipper) and for debugging minimal images with tools from another image.

</details>

<details>
<summary><b>Why might an Alpine-based image behave differently (DNS, performance, segfaults)?</b> `🔴 Advanced`</summary>
<br>

Alpine uses **musl libc** instead of glibc. Differences include the DNS resolver (historically no TCP fallback, and `search`/`ndots` handling), memory allocator performance, missing glibc symbols for prebuilt binaries (segfault or "not found" when the binary exists), and Python wheels that must compile from source. Use `-slim` (glibc) when compatibility matters.

</details>

<details>
<summary><b>How does Docker handle container logs if the app writes huge amounts to stdout? Can that slow the app down?</b> `🔴 Advanced`</summary>
<br>

Yes. In the default **blocking** mode, if the log driver cannot keep up (for example a remote driver that is slow or down), writes to stdout block and the app stalls. Use `--log-opt mode=non-blocking --log-opt max-buffer-size=4m`, which may drop logs under pressure, or use local drivers with an agent shipping asynchronously.

</details>

<details>
<summary><b>What are the important daemon.json settings for production?</b> `🔴 Advanced`</summary>
<br>

```bash
{
  "log-driver": "local",
  "log-opts": { "max-size": "20m", "max-file": "5" },
  "live-restore": true,
  "storage-driver": "overlay2",
  "data-root": "/data/docker",
  "default-ulimits": { "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 } },
  "default-address-pools": [{ "base": "172.80.0.0/12", "size": 24 }],
  "registry-mirrors": ["https://mirror.example.com"],
  "metrics-addr": "127.0.0.1:9323",
  "icc": false,
  "no-new-privileges": true,
  "userland-proxy": false,
  "features": { "buildkit": true }
}
```

</details>


## <a id="cheatsheet"></a>15. Command Cheat Sheet

*Quick revision before the interview.*

### Images

```bash
docker build -t app:1.0 .
docker build --target test --no-cache .
docker buildx build --platform linux/amd64,linux/arm64 --push -t r/app:1 .
docker images / docker image ls
docker pull / push / tag
docker history --no-trunc app:1.0
docker image inspect app:1.0
docker rmi app:1.0 ; docker image prune -a
docker save -o a.tar app:1.0 ; docker load -i a.tar
```

### Containers

```bash
docker run -d --name c -p 8080:80 img
docker ps -a ; docker ps -q -f status=exited
docker start|stop|restart|pause|kill c
docker rm -f c ; docker container prune
docker exec -it c sh
docker logs -f --tail 100 c
docker cp c:/path ./ ; docker diff c
docker top c ; docker stats ; docker port c
docker update --memory 1g --cpus 2 c
docker wait c ; docker rename c c2
```

### Networking

```bash
docker network ls
docker network create -d bridge net1
docker network inspect net1
docker network connect|disconnect net1 c
docker run --network host|none|container:x
docker run --add-host h:host-gateway
docker network prune
```

### Volumes

```bash
docker volume create v1
docker volume ls -f dangling=true
docker volume inspect v1
docker run -v v1:/data img
docker run --mount type=bind,src=$PWD,dst=/app,ro img
docker run --tmpfs /tmp img
docker volume rm v1 ; docker volume prune
```

### Compose

```bash
docker compose up -d --build
docker compose ps / logs -f svc
docker compose exec svc sh
docker compose run --rm svc cmd
docker compose up -d --scale w=3
docker compose config
docker compose pull ; docker compose watch
docker compose down -v --remove-orphans
```

### System / Debug

```bash
docker info ; docker version
docker system df -v ; docker system prune
docker events --filter event=die
docker inspect -f '{{.State.Pid}}' c
nsenter -t PID -n ss -lntp
docker run --rm -it --network container:c nicolaka/netshoot
docker context ls / use
journalctl -u docker -f
```

### Swarm

```bash
docker swarm init / join
docker node ls
docker service create --replicas 3 img
docker service ls / ps svc / logs svc
docker service scale svc=5
docker service update --image img:2 svc
docker service rollback svc
docker stack deploy -c compose.yaml st
docker secret create s -
```

### Security

```bash
docker run --user 10001 --read-only \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --pids-limit 100 --memory 256m img
trivy image img ; docker scout cves img
cosign sign / verify
hadolint Dockerfile
```
