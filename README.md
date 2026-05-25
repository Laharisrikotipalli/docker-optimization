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
| Layer Count             | 17       | 15          |
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
curl http://localhost:8091/health
curl http://localhost:8092/health
curl http://localhost:3011/health
curl http://localhost:3012/health
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