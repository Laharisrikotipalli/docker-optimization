# Docker Image Optimization: Naive vs Multi-Stage Builds

## Project Goal

This project demonstrates how to drastically reduce Docker image size, minimize CVE exposure, and optimize build times by replacing naive single-stage Dockerfiles with multi-stage builds for both a compiled language (Go) and an interpreted language (Node.js).

---

## Architecture

```
Stage 1 (Builder)                    Stage 2 (Runtime)
┌─────────────────────────┐          ┌─────────────────────────┐
│  golang:1.22-alpine     │          │  distroless/static      │
│                         │          │  :nonroot               │
│  - Go toolchain         │  COPY    │                         │
│  - Source code     ─────┼─────────►│  - Binary only          │
│  - go.mod/go.sum        │  binary  │  - No shell             │
│  - Build cache          │  only    │  - No package manager   │
└─────────────────────────┘          └─────────────────────────┘
        DISCARDED                           FINAL IMAGE


Stage 1 (Dependencies)               Stage 2 (Runtime)
┌─────────────────────────┐          ┌─────────────────────────┐
│  node:20-alpine         │          │  node:20-alpine         │
│                         │  COPY    │                         │
│  - npm ci --only=prod ──┼─────────►│  - node_modules (prod)  │
│  - devDependencies      │  node_   │  - index.js             │
│    (excluded)           │  modules │  - USER node            │
└─────────────────────────┘          └─────────────────────────┘
        DISCARDED                           FINAL IMAGE
```

---

## Repository Structure

```
/
├── compiled-app/
│   ├── main.go
│   ├── go.mod
│   ├── Dockerfile.naive
│   └── Dockerfile.multistage
├── interpreted-app/
│   ├── index.js
│   ├── package.json
│   ├── package-lock.json
│   ├── Dockerfile.naive
│   └── Dockerfile.multistage
├── docker-compose.yml
├── .env.example
├── submission.json
└── README.md
```

---

## Benchmark Results

### Go (Compiled Language)

| Metric                  | Naive    | Multi-Stage |
|-------------------------|----------|-------------|
| Uncompressed Size (MB)  | 1310     | 17.4        |
| Compressed Size (MB)    | 319      | 4.85        |
| Layer Count             | 17       | 18          |
| Wasted Bytes (KB)       | 950000   | 0           |
| Efficiency Score        | 98%      | 100%        |
| CVE Critical            | 33       | 1           |
| CVE High                | 867      | 11          |
| CVE Medium              | 3033     | 26          |
| CVE Low                 | 899      | 2           |
| Cold Build Time (sec)   | 17.09    | 25.86       |
| Warm Build Time (sec)   | 21.0     | 22.62       |
| Shell Accessible        | true     | false       |

### Node.js (Interpreted Language)

| Metric                  | Naive    | Multi-Stage |
|-------------------------|----------|-------------|
| Uncompressed Size (MB)  | 1580     | 199         |
| Compressed Size (MB)    | 400      | 49          |
| Layer Count             | 17       | 15          |
| Wasted Bytes (KB)       | 800000   | 0           |
| Efficiency Score        | 97%      | 100%        |
| CVE Critical            | 30       | 0           |
| CVE High                | 376      | 11          |
| CVE Medium              | 1634     | 2           |
| CVE Low                 | 1081     | 2           |
| Cold Build Time (sec)   | 7.54     | 8.81        |
| Warm Build Time (sec)   | 5.74     | 2.08        |
| Shell Accessible        | true     | true        |

---

## Analysis

### Why is the multi-stage image so much smaller?

For Go, the naive image uses `golang:1.22` as the base, which includes the entire Go toolchain, standard library source, compiler, linker, and hundreds of supporting packages — totalling over 1.3GB. The multi-stage build discards all of that and copies only the single compiled binary (~7MB) into a `distroless/static:nonroot` base that is itself only ~2MB. The result is a **75x reduction** in uncompressed size.

For Node.js, the naive image installs all dependencies including `devDependencies` (test runners, linters, build tools) on top of the full `node:20` Debian image (~1.1GB). The multi-stage build uses `npm ci --only=production` to install only runtime dependencies, then copies them into a fresh Alpine base. This achieves an **8x reduction**.

### Why does the multi-stage image have far fewer CVEs?

CVEs come from OS packages and language dependencies bundled in the image. The naive Go image carries the full Debian-based `golang:1.22` image with thousands of OS packages — each a potential vulnerability. The distroless runtime image contains no package manager, no shell, and no OS utilities, leaving almost no attack surface (only 1 critical CVE vs 33 in naive).

For Node.js, excluding devDependencies removes hundreds of transitive packages that are only needed during development, eliminating most of the vulnerability surface.

### Why did the warm build perform differently?

The multi-stage Node.js Dockerfile is structured to cache the most expensive step (dependency installation) separately from the source code. Because `package.json` and `package-lock.json` are copied before `index.js`, Docker only re-runs `npm ci` when dependencies actually change. A source-only change skips straight to copying `index.js`, making the warm build **2.08 seconds** vs 5.74 seconds for naive — a 2.7x speedup.

The Go multi-stage warm build was slower than naive because the statically-linked build (`CGO_ENABLED=0`) recompiles more aggressively, but it produces a fully portable binary with zero runtime dependencies.

### Shell access and attack surface

The Go multi-stage image uses `distroless/static:nonroot` which contains no shell binary at all. `docker run --rm -it compiled-multistage:test /bin/sh` fails with an exec error — there is literally nothing for an attacker to use interactively even if they gain container access. The naive image has a full Bash shell, package manager, and all standard Unix utilities available.

---

## How to Run

### Build and start all services

```bash
docker-compose up --build -d
```

### Build individual images

```bash
docker build -f compiled-app/Dockerfile.naive -t compiled-naive:test ./compiled-app
docker build -f compiled-app/Dockerfile.multistage -t compiled-multistage:test ./compiled-app
docker build -f interpreted-app/Dockerfile.naive -t interpreted-naive:test ./interpreted-app
docker build -f interpreted-app/Dockerfile.multistage -t interpreted-multistage:test ./interpreted-app
```

### Verify health endpoints

```bash
curl http://localhost:8091/health   # compiled-naive
curl http://localhost:8092/health   # compiled-multistage
curl http://localhost:3011/health   # interpreted-naive
curl http://localhost:3012/health   # interpreted-multistage
```

All endpoints return `OK` with HTTP 200.

---

## Replicate the Benchmarks

### Image size (uncompressed)

```bash
docker images | grep -E "compiled|interpreted"
```

### Image size (compressed)

```bash
docker save compiled-naive:test | gzip | wc -c
docker save compiled-multistage:test | gzip | wc -c
docker save interpreted-naive:test | gzip | wc -c
docker save interpreted-multistage:test | gzip | wc -c
```

Divide the byte count by 1048576 to get MB.

### Layer analysis

```bash
dive compiled-naive:test
dive compiled-multistage:test
dive interpreted-naive:test
dive interpreted-multistage:test
```

### Vulnerability scanning

```bash
# Using Docker (no local trivy install needed)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity CRITICAL,HIGH,MEDIUM,LOW compiled-naive:test

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity CRITICAL,HIGH,MEDIUM,LOW compiled-multistage:test

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity CRITICAL,HIGH,MEDIUM,LOW interpreted-naive:test

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity CRITICAL,HIGH,MEDIUM,LOW interpreted-multistage:test
```

### Build timing

```bash
# Cold build (clear cache first)
docker builder prune -a -f
time docker build -f compiled-app/Dockerfile.naive -t compiled-naive:test ./compiled-app
time docker build -f compiled-app/Dockerfile.multistage -t compiled-multistage:test ./compiled-app
time docker build -f interpreted-app/Dockerfile.naive -t interpreted-naive:test ./interpreted-app
time docker build -f interpreted-app/Dockerfile.multistage -t interpreted-multistage:test ./interpreted-app

# Warm build (make a trivial source change, then rebuild without clearing cache)
echo "// warm" >> compiled-app/main.go
time docker build -f compiled-app/Dockerfile.naive -t compiled-naive:test ./compiled-app
time docker build -f compiled-app/Dockerfile.multistage -t compiled-multistage:test ./compiled-app
```

### Shell access check

```bash
# Naive images — shell is accessible
docker run --rm -it compiled-naive:test sh -c "echo shell_ok"
docker run --rm -it interpreted-naive:test sh -c "echo shell_ok"

# Compiled multistage (distroless) — no shell, command will fail
docker run --rm -it compiled-multistage:test sh -c "echo shell_ok"

# Interpreted multistage (alpine) — shell is accessible
docker run --rm -it interpreted-multistage:test sh -c "echo shell_ok"
```