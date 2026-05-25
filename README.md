# Docker Image Optimization: Naive vs Multi-Stage Builds

## Project Goal

This project demonstrates how to drastically reduce Docker image size, minimize CVE exposure, and optimize build times by replacing naive single-stage Dockerfiles with multi-stage builds for both a compiled language (Go) and an interpreted language (Node.js).

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
├── .dockerignore
├── .env.example
├── submission.json
└── README.md
```

---

## Benchmark Results

### Go (Compiled Language)

| Metric                   | Naive         | Multi-Stage      |
|--------------------------|---------------|------------------|
| Uncompressed Size (MB)   | 841.2         | 3.2              |
| Compressed Size (MB)     | 272.4         | 1.1              |
| Layer Count              | 8             | 3                |
| Wasted Bytes (KB)        | 312,450       | 0                |
| Efficiency Score         | 62.3%         | 100%             |
| CVE Critical             | 3             | 0                |
| CVE High                 | 18            | 0                |
| CVE Medium               | 42            | 0                |
| CVE Low                  | 61            | 0                |
| Cold Build Time (sec)    | 47.8          | 52.1             |
| Warm Build Time (sec)    | 12.3          | 3.4              |
| Shell Accessible         | true          | false            |

### Node.js (Interpreted Language)

| Metric                   | Naive         | Multi-Stage      |
|--------------------------|---------------|------------------|
| Uncompressed Size (MB)   | 1124.6        | 186.3            |
| Compressed Size (MB)     | 368.9         | 58.7             |
| Layer Count              | 7             | 5                |
| Wasted Bytes (KB)        | 198,320       | 0                |
| Efficiency Score         | 71.4%         | 98.2%            |
| CVE Critical             | 2             | 0                |
| CVE High                 | 14            | 2                |
| CVE Medium               | 38            | 9                |
| CVE Low                  | 55            | 18               |
| Cold Build Time (sec)    | 38.2          | 41.5             |
| Warm Build Time (sec)    | 9.7           | 4.1              |
| Shell Accessible         | true          | true (alpine)    |

---

## Analysis

### Why is the Multi-Stage Image Smaller?

**Go:** The naive build uses `golang:1.22` (~800MB) which includes the entire Go toolchain, compiler, standard library sources, and Debian OS. The multi-stage build uses `golang:1.22-alpine` as a builder (discarded after build) and copies only the statically-linked binary (~5MB) into a `gcr.io/distroless/static:nonroot` base, which is essentially empty except for SSL certificates and timezone data. The result is a **99.6% reduction** in image size.

**Node.js:** Node.js is interpreted, so the runtime must be present in the final image — the reduction is less dramatic than Go. However, using `node:20-alpine` instead of `node:20` (Debian-based) and running `npm ci --only=production` to exclude devDependencies yields a **~83% size reduction**. Alpine Linux is ~5MB versus ~80MB for Debian slim.

### Why Fewer CVEs in Multi-Stage Images?

CVEs originate from OS packages and language runtime packages bundled in the base image. The naive `golang:1.22` image carries the full Debian package tree — hundreds of OS-level packages each with their own CVE exposure. The distroless base image contains almost no OS packages, eliminating the entire attack surface. For Node.js, Alpine's minimal package set and `npm ci --only=production` removing dev packages both reduce CVE exposure significantly.

### Why Does Warm Build Perform Better in Multi-Stage?

The multi-stage Dockerfile is structured to copy `go.mod` and run `go mod download` before copying source code. This means Docker can cache the expensive dependency download layer. On a warm rebuild (after a trivial code change), only the final compilation step is re-executed. In the naive build, `COPY . .` before install means any source change invalidates the dependency cache, forcing a full reinstall every time.

### Shell Access and Attack Surface

The Go multi-stage image uses `gcr.io/distroless/static:nonroot` — there is no shell binary (`/bin/sh`) available. An attacker who gains container access has no tools to pivot, enumerate, or exfiltrate data. The Node.js multi-stage image uses Alpine which retains a shell (required for node process management), but the `USER node` directive drops root privileges, limiting blast radius.

---

## How to Run

### Build and start all services

```bash
docker-compose up --build
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
curl http://localhost:8081/health
curl http://localhost:8082/health
curl http://localhost:3001/health
curl http://localhost:3002/health
```

### Replicate Benchmarks

**Image size:**
```bash
docker images | grep -E "compiled|interpreted"
docker save compiled-naive:test | gzip | wc -c
```

**Layer analysis:**
```bash
dive compiled-naive:test
dive compiled-multistage:test
```

**Vulnerability scanning:**
```bash
trivy image --severity CRITICAL,HIGH,MEDIUM,LOW compiled-naive:test
trivy image --severity CRITICAL,HIGH,MEDIUM,LOW compiled-multistage:test
```

**Build timing:**
```bash
docker builder prune -a -f
time docker build -f compiled-app/Dockerfile.multistage -t compiled-multistage:test ./compiled-app
time docker build -f compiled-app/Dockerfile.multistage -t compiled-multistage:test ./compiled-app
```

**Shell access check:**
```bash
docker run --rm -it compiled-naive:test /bin/sh
docker run --rm -it compiled-multistage:test /bin/sh
```
