# Docker — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Docker Fundamentals](#docker-fundamentals)
- [Images & Dockerfile](#images-dockerfile)
- [Multi-Stage Builds](#multi-stage)
- [BuildKit & Build Cache](#buildkit)
- [.dockerignore](#dockerignore)
- [Containers](#containers)
- [Health Checks](#health-checks)
- [Networking](#networking)
- [Volumes & Storage](#volumes)
- [Docker Compose](#docker-compose)
- [Registry & Image Management](#registry)
- [Docker Scout & Vulnerability Scanning](#scout)
- [Docker Content Trust & Image Signing](#trust)
- [Resource Limits (cgroups)](#resource-limits)
- [Docker Context & Remote Builders](#context)
- [Docker in Docker (DinD)](#dind)
- [Docker Swarm](#swarm)
- [Security Best Practices](#security)
- [Master Cheatsheet](#master-cheatsheet)

---

## Docker Fundamentals

### 🟢 Q1. What is Docker and how does it differ from VMs?

```
Virtual Machine:
  Hardware → Hypervisor → Guest OS → App
  Full OS per VM, GB of RAM, minutes to start

Container:
  Hardware → Host OS → Docker Engine → Container (App + libs)
  Shared kernel, MB of RAM, seconds to start

Docker architecture:
  dockerd (daemon)    — manages images, containers, networks, volumes
  containerd          — container runtime (OCI-compliant)
  runc                — low-level container runner
  Docker CLI          — sends API calls to dockerd
  Docker Hub/Registry — image storage
```

```bash
# Install Docker Engine (Ubuntu)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # Add user to docker group (re-login required)

# Verify
docker version
docker info
docker system info

# Docker lifecycle
docker pull nginx:1.25-alpine          # Pull image
docker run -d --name web -p 80:80 nginx:1.25-alpine   # Run container
docker ps                              # List running containers
docker ps -a                           # All containers (including stopped)
docker stop web                        # Stop gracefully (SIGTERM → SIGKILL after 10s)
docker start web                       # Start stopped container
docker restart web                     # Stop + start
docker rm web                          # Remove container
docker rmi nginx:1.25-alpine           # Remove image
docker logs web                        # View logs
docker logs -f web                     # Follow logs
docker exec -it web sh                 # Shell inside running container
docker inspect web                     # Full JSON metadata
```

---

## Images & Dockerfile

### 🟢 Q2. What are Docker images and layers?

```bash
# Image = stack of read-only layers + metadata
# Each Dockerfile instruction creates a layer (RUN, COPY, ADD)
# Layers are cached and shared between images

docker image ls                        # List images
docker image ls --filter dangling=true # Dangling (untagged) images
docker image history nginx:latest      # Show layers
docker image inspect nginx:latest      # Full metadata
docker image prune                     # Remove dangling images
docker image prune -a                  # Remove all unused images
docker system df                       # Disk usage breakdown
docker system prune -a --volumes       # Remove EVERYTHING unused
```

---

### 🟡 Q3. How do you write an optimised, secure Dockerfile?

```dockerfile
# ===== OPTIMISED MULTI-LAYER DOCKERFILE =====

# Use specific version tag (never 'latest' in production)
FROM node:20.11-alpine3.19

# Set non-root user early
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

# Set working directory
WORKDIR /app

# Copy dependency files FIRST (cache-friendly)
# Only invalidates cache when package.json changes
COPY package.json package-lock.json ./

# Install dependencies as root, then switch user
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/*

# Copy application code (changes frequently — put last)
COPY --chown=appuser:appgroup src/ ./src/
COPY --chown=appuser:appgroup config/ ./config/

# Security: run as non-root
USER appuser

# Document exposed port (informational only)
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Use exec form (not shell form) — proper signal handling
ENTRYPOINT ["node"]
CMD ["src/index.js"]
```

```dockerfile
# ===== BEST PRACTICES SUMMARY =====

# 1. SORT multi-line args alphabetically for readability + smaller diffs
RUN apt-get update && apt-get install -y \
    curl \
    git \
    htop \
    jq \
    vim \
    && rm -rf /var/lib/apt/lists/*

# 2. Combine RUN commands to reduce layers
# BAD — 3 layers
RUN apt-get update
RUN apt-get install -y nginx
RUN rm -rf /var/lib/apt/lists/*

# GOOD — 1 layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*

# 3. Use --no-install-recommends to reduce image size
RUN apt-get install -y --no-install-recommends nginx

# 4. Use COPY instead of ADD (unless you need URL fetch or tar auto-extract)
COPY src/ /app/src/          # GOOD
ADD  src/ /app/src/          # BAD (unexpected behavior)
ADD  https://example.com/file.tar.gz /tmp/   # OK — unique ADD use case

# 5. Explicit COPY --chown instead of RUN chown (saves a layer)
COPY --chown=appuser:appgroup . /app/

# 6. Use ARG for build-time variables, ENV for runtime
ARG BUILD_DATE
ARG VERSION
ENV APP_VERSION=$VERSION
LABEL build-date=$BUILD_DATE

# 7. LABEL for metadata
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.description="My Application"
LABEL maintainer="team@example.com"
```

---

## Multi-Stage Builds

### 🟡 Q4. How do multi-stage builds work and why are they important?

**Explanation:**
Multi-stage builds use multiple `FROM` statements in one Dockerfile. Earlier stages compile/build, later stages copy only artifacts. Final image contains no build tools — dramatically smaller and more secure.

```dockerfile
# ===== Go application =====
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /build

# Download dependencies (cached separately from source)
COPY go.mod go.sum ./
RUN go mod download

# Build binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# Stage 2: Final — only the binary
FROM scratch                        # Empty base image (smallest possible)
# Or: FROM gcr.io/distroless/static:nonroot (includes TLS certs, no shell)

COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
ENTRYPOINT ["/server"]
```

```dockerfile
# ===== Node.js application =====
# Stage 1: Install ALL deps (including devDependencies for build)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build                   # TypeScript compile, webpack, etc.

# Stage 3: Production — only prod deps + built output
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/config ./config

RUN addgroup -g 1001 appgroup && adduser -D -u 1001 -G appgroup appuser
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

```dockerfile
# ===== Java Spring Boot =====
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B     # Cache dependencies

COPY src/ src/
RUN mvn package -DskipTests -B

# Use layered jar for better caching
FROM eclipse-temurin:17-jre-alpine AS extractor
WORKDIR /extract
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

RUN addgroup -g 1001 spring && adduser -D -u 1001 -G spring spring
USER spring

# Copy jar layers (most-stable first = better caching)
COPY --from=extractor /extract/dependencies/ ./
COPY --from=extractor /extract/spring-boot-loader/ ./
COPY --from=extractor /extract/snapshot-dependencies/ ./
COPY --from=extractor /extract/application/ ./

EXPOSE 8080
HEALTHCHECK CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

```bash
# Build specific stage
docker build --target builder -t myapp:builder .

# Build with build args
docker build \
  --build-arg VERSION=1.2.3 \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t myapp:1.2.3 .

# See size difference
docker image ls | grep myapp
# myapp:builder    1.2 GB   (with all build tools)
# myapp:prod       45 MB    (just the binary)
```

---

## BuildKit & Build Cache

### 🟡 Q5. What is BuildKit and how do you manage build cache?

**Explanation:**
BuildKit is Docker's next-generation build backend. It enables parallel build stages, advanced caching (including remote cache), secret injection without leaking to layers, and SSH forwarding. It's enabled by default in Docker 23+.

```bash
# Enable BuildKit (older Docker versions)
export DOCKER_BUILDKIT=1
# Or in /etc/docker/daemon.json:
# { "features": { "buildkit": true } }

# Build with BuildKit features
docker buildx build -t myapp:latest .

# ===== CACHE MOUNTS (keep package cache between builds) =====
```

```dockerfile
# BuildKit-specific Dockerfile syntax
# syntax=docker/dockerfile:1

FROM ubuntu:22.04

# Cache apt packages — not committed to image layer
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
    nginx curl jq \
    && rm -rf /var/lib/apt/lists/*

# Cache pip packages
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache npm packages
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# ===== SECRETS — inject at build time, NOT in image layers =====
# Pass secret as build argument (mounted, never stored in layer)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# ===== SSH FORWARDING — access private git repos =====
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git
```

```bash
# Build with secrets
docker buildx build \
  --secret id=npmrc,src=$HOME/.npmrc \
  --secret id=github_token,env=GITHUB_TOKEN \
  -t myapp:latest .

# Build with SSH forwarding
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
docker buildx build --ssh default -t myapp:latest .

# ===== REMOTE CACHE =====
# Push cache to registry
docker buildx build \
  --cache-to type=registry,ref=registry.example.com/myapp:cache,mode=max \
  --cache-from type=registry,ref=registry.example.com/myapp:cache \
  -t myapp:latest \
  --push .

# GitHub Actions cache
docker buildx build \
  --cache-from type=gha \
  --cache-to type=gha,mode=max \
  -t myapp:latest .

# ===== MULTI-PLATFORM BUILDS =====
# Create builder with multi-platform support
docker buildx create --name multiarch --driver docker-container --bootstrap
docker buildx use multiarch

docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myapp:latest \
  --push .

# List builders
docker buildx ls
docker buildx inspect multiarch
```

---

## .dockerignore

### 🟢 Q6. What is .dockerignore and why is it critical?

```bash
# .dockerignore — exclude files from build context
# Reduces build context size (sent to daemon), improves cache, prevents secret leaks

# Version control
.git
.gitignore
.gitattributes

# Node.js
node_modules
npm-debug.log
.npm

# Python
__pycache__
*.pyc
*.pyo
.venv
venv
dist
*.egg-info

# Build output
dist/
build/
target/
*.class

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Secrets — CRITICAL
.env
.env.*
*.key
*.pem
*.p12
secrets/
.aws/
.ssh/
kubeconfig
*credentials*

# Docker
Dockerfile*
docker-compose*

# Docs (don't include in image)
README.md
docs/
*.md

# Test files
test/
tests/
**/*_test.go
**/*.test.js
coverage/
.nyc_output/

# CI
.github/
.circleci/
Jenkinsfile
```

---

## Containers

### 🟡 Q7. How do you run and manage containers?

```bash
# Run options
docker run \
  --name web \
  -d \                              # Detached (background)
  -p 8080:80 \                      # port mapping host:container
  -e APP_ENV=production \           # Environment variable
  --env-file .env \                 # Load env file
  -v /host/data:/container/data \   # Bind mount
  -v myvolume:/data \               # Named volume
  --network my-network \            # Connect to network
  --restart unless-stopped \        # Restart policy
  --memory 512m \                   # Memory limit
  --cpus 1.5 \                      # CPU limit
  --user 1001:1001 \                # Run as user:group
  --read-only \                     # Read-only filesystem
  --security-opt no-new-privileges \# Prevent privilege escalation
  --cap-drop ALL \                  # Drop all Linux capabilities
  --cap-add NET_BIND_SERVICE \      # Only re-add what's needed
  nginx:alpine

# Container lifecycle
docker start <name/id>
docker stop <name/id>               # SIGTERM → wait 10s → SIGKILL
docker stop -t 30 <name>            # Custom grace period
docker kill <name>                  # Immediate SIGKILL
docker pause <name>                 # Freeze (SIGSTOP)
docker unpause <name>
docker restart <name>

# Exec into container
docker exec -it web bash
docker exec -it web sh              # Alpine (no bash)
docker exec -u root web bash        # As root
docker exec web ps aux              # Non-interactive

# Copy files
docker cp web:/etc/nginx/nginx.conf ./nginx.conf    # Container → host
docker cp ./nginx.conf web:/etc/nginx/nginx.conf    # Host → container

# Stats
docker stats                        # Live resource usage
docker stats --no-stream            # One-time snapshot
docker top web                      # Processes in container

# Cleanup
docker rm web                       # Remove stopped container
docker rm -f web                    # Force remove running
docker container prune              # Remove all stopped containers
docker rm $(docker ps -aq)          # Remove all containers
```

---

## Health Checks

### 🟡 Q8. How do Docker health checks work?

```dockerfile
# ===== DOCKERFILE HEALTHCHECK =====

# HTTP health check
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=30s \
            --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# With wget (Alpine — no curl)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# TCP port check
HEALTHCHECK --interval=20s --timeout=5s \
  CMD nc -z localhost 5432 || exit 1

# Script-based health check
COPY scripts/healthcheck.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/healthcheck.sh
HEALTHCHECK --interval=30s --timeout=10s \
  CMD /usr/local/bin/healthcheck.sh

# Disable inherited healthcheck
HEALTHCHECK NONE
```

```bash
# Check health status
docker inspect --format='{{.State.Health.Status}}' web
# starting / healthy / unhealthy / none

# View health check log
docker inspect --format='{{json .State.Health}}' web | jq .

# docker ps shows health status
docker ps
# CONTAINER ID  IMAGE  STATUS
# abc123        web    Up 5 min (healthy)
```

```yaml
# docker-compose.yaml healthcheck
services:
  web:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s     # Grace period before first check

  database:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    depends_on:
      database:
        condition: service_healthy    # Wait for healthy database
```

---

## Networking

### 🟡 Q9. How does Docker networking work?

```bash
# Network types
docker network ls
# bridge   — default, containers communicate via IP (172.17.0.0/16)
# host     — container uses host network (no isolation)
# none     — no networking
# overlay  — multi-host (Swarm)
# macvlan  — assign MAC address (appear as physical device on network)

# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.1.0/24 \
  --gateway 172.20.0.1 \
  my-network

# Connect container to network
docker run -d --name app --network my-network myapp
docker network connect my-network existing-container
docker network disconnect my-network container

# Container DNS — on user-defined networks, containers resolve by name
docker run -d --name db --network my-network postgres
docker run -d --name app --network my-network myapp
# App can reach db as: psql -h db -U postgres

# Inspect network
docker network inspect my-network

# Published ports
docker run -p 8080:80 nginx            # Specific host port
docker run -p 80 nginx                 # Random host port
docker run -p 127.0.0.1:8080:80 nginx  # Bind to localhost only
docker run -P nginx                    # Publish all EXPOSE ports

# Host network (no isolation)
docker run --network host nginx        # Uses host's port 80 directly
```

---

## Volumes & Storage

### 🟡 Q10. How do Docker volumes and bind mounts differ?

```bash
# Volumes — managed by Docker, stored in /var/lib/docker/volumes/
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune             # Remove unused volumes

# Use named volume
docker run -v mydata:/var/lib/postgresql/data postgres

# Bind mount — maps host path to container path
docker run -v /host/path:/container/path myapp
docker run -v $(pwd)/config:/app/config:ro myapp   # Read-only

# tmpfs mount — in-memory (not persisted)
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=100m myapp

# Modern --mount syntax (clearer)
docker run \
  --mount type=volume,source=mydata,target=/data \
  --mount type=bind,source=$(pwd)/config,target=/app/config,readonly \
  myapp

# Volume in Dockerfile
VOLUME ["/var/lib/mysql"]   # Declares a mount point (auto-creates anonymous volume)

# Backup a volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  busybox tar czf /backup/mydata-backup.tar.gz /data

# Restore a volume
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  busybox tar xzf /backup/mydata-backup.tar.gz -C /
```

---

## Docker Compose

### 🟡 Q11. How do you use Docker Compose for multi-container applications?

```yaml
# docker-compose.yaml — complete example
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
      cache_from:
        - myapp:cache
    image: myapp:${VERSION:-latest}
    container_name: myapp-api
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env.production
    volumes:
      - uploads:/app/uploads
      - ./config:/app/config:ro
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 128M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      traefik.enable: "true"
      traefik.http.routers.api.rule: "Host(`api.example.com`)"

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME:-myapp}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - frontend
    depends_on:
      - api

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  uploads:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.0.0.1,rw
      device: ":/exports/uploads"

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # No external access
```

```bash
# Compose commands
docker compose up -d                    # Start all services
docker compose up -d api                # Start specific service
docker compose down                     # Stop + remove containers
docker compose down -v                  # Stop + remove containers + volumes
docker compose ps                       # Show service status
docker compose logs -f api              # Follow service logs
docker compose exec api bash            # Shell in service
docker compose restart api              # Restart service
docker compose pull                     # Pull latest images
docker compose build --no-cache         # Rebuild images
docker compose config                   # Validate and view resolved config

# Multiple compose files (override pattern)
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up -d

# Scale service
docker compose up -d --scale api=3
```

---

## Registry & Image Management

### 🟡 Q12. How do you work with Docker registries?

```bash
# Docker Hub
docker login
docker tag myapp:latest myuser/myapp:1.0.0
docker push myuser/myapp:1.0.0
docker pull myuser/myapp:1.0.0

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0

# Google Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev
docker push us-central1-docker.pkg.dev/my-project/my-repo/myapp:1.0.0

# GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u myuser --password-stdin
docker tag myapp:latest ghcr.io/myorg/myapp:1.0.0
docker push ghcr.io/myorg/myapp:1.0.0

# Private registry with self-signed cert
# Add to /etc/docker/daemon.json:
# { "insecure-registries": ["registry.example.com:5000"] }

# Skopeo — copy images without pulling
skopeo copy docker://nginx:latest docker://registry.example.com/nginx:latest
skopeo inspect docker://nginx:latest
skopeo delete docker://registry.example.com/myapp:old

# Crane — manipulate images in registry
crane digest nginx:latest
crane ls registry.example.com/myapp
crane copy myapp:staging registry.example.com/myapp:production
```

---

## Docker Scout & Vulnerability Scanning

### 🟡 Q13. How do you scan Docker images for vulnerabilities?

```bash
# ===== DOCKER SCOUT (built-in, Docker Desktop 4.17+) =====
docker scout quickview myapp:latest       # Overview of vulnerabilities
docker scout cves myapp:latest            # Detailed CVE list
docker scout recommendations myapp:latest # Suggest fixes
docker scout compare myapp:1.0.0 myapp:1.1.0  # Compare two images

# Enable in CI
docker scout cves \
  --format sarif \
  --output scout-report.sarif \
  myapp:latest

docker scout cves \
  --exit-code \                            # Exit 1 if vulnerabilities found
  --only-severity critical,high \          # Only critical+high
  myapp:latest

# ===== TRIVY (popular open-source) =====
# Install
brew install trivy
# Or: docker pull aquasec/trivy

# Scan image
trivy image myapp:latest
trivy image --severity HIGH,CRITICAL myapp:latest
trivy image --exit-code 1 --severity CRITICAL myapp:latest
trivy image --format sarif --output trivy-results.sarif myapp:latest

# Scan filesystem
trivy fs --security-checks vuln,config .

# Scan IaC (Dockerfile, K8s manifests)
trivy config .

# In CI/CD (GitHub Actions)
# - uses: aquasecurity/trivy-action@master
#   with:
#     image-ref: myapp:latest
#     format: sarif
#     output: trivy-results.sarif
#     severity: CRITICAL,HIGH
#     exit-code: 1

# ===== GRYPE (Anchore) =====
grype myapp:latest
grype myapp:latest --fail-on critical

# ===== SNYK =====
snyk container test myapp:latest
snyk container monitor myapp:latest      # Continuous monitoring
snyk container test myapp:latest --file=Dockerfile --severity-threshold=high
```

---

## Docker Content Trust & Image Signing

### 🔴 Q14. How do you sign Docker images?

```bash
# ===== DOCKER CONTENT TRUST (DCT) — legacy signing =====
# Enable DCT (enforces signed images)
export DOCKER_CONTENT_TRUST=1

# Generate keys
docker trust key generate mykey

# Sign and push
docker trust sign myapp:1.0.0

# Inspect trust data
docker trust inspect --pretty myapp:latest

# Add signer
docker trust signer add --key cosign.pub ci-signer registry.example.com/myapp

# Remove signature
docker trust untag myapp:1.0.0

# ===== COSIGN (modern, OIDC-based — recommended) =====
# Install: go install github.com/sigstore/cosign/cmd/cosign@latest

# Generate key pair
cosign generate-key-pair

# Sign image (key-based)
cosign sign --key cosign.key myapp:latest

# Sign with OIDC (keyless — no key management needed)
# Works in GitHub Actions with GITHUB_TOKEN
cosign sign --yes myapp:latest

# Verify signature
cosign verify --key cosign.pub myapp:latest

# Keyless verify (uses Fulcio + Rekor transparency log)
cosign verify \
  --certificate-identity-regexp="https://github.com/myorg/myrepo/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:latest

# Attach SBOM
cosign attach sbom --sbom sbom.spdx myapp:latest

# Sign with annotations
cosign sign \
  --key cosign.key \
  -a "git-commit=$(git rev-parse HEAD)" \
  -a "build-date=$(date -u +%Y-%m-%d)" \
  myapp:latest
```

---

## Resource Limits (cgroups)

### 🟡 Q15. How do you set resource limits on containers?

```bash
# ===== MEMORY =====
docker run \
  --memory 512m \            # Hard limit (OOM kill at 512MB)
  --memory-swap 512m \       # Disable swap (swap = memory - memory = 0)
  --memory-reservation 256m \ # Soft limit (reclaimed under pressure)
  --oom-kill-disable \        # Disable OOM kill (use with care!)
  --memory-oom-kill-disable \ # Same
  myapp

# Check memory limit inside container
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# ===== CPU =====
docker run \
  --cpus 1.5 \               # Max 1.5 CPU cores
  --cpu-shares 512 \         # Relative weight (default 1024)
  --cpu-period 100000 \      # Microseconds per period
  --cpu-quota 50000 \        # Max microseconds per period (=50% of 1 CPU)
  --cpuset-cpus "0,2" \      # Pin to specific CPUs
  myapp

# ===== IO =====
docker run \
  --blkio-weight 500 \       # IO weight (default 500, range 10-1000)
  --device-read-bps /dev/sda:10mb \   # Read limit
  --device-write-bps /dev/sda:10mb \  # Write limit
  --device-read-iops /dev/sda:1000 \  # IOPS limit
  --device-write-iops /dev/sda:1000 \
  myapp

# ===== IN DOCKER COMPOSE =====
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 128M

# ===== CHECK RESOURCE USAGE =====
docker stats myapp
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
# Container stats show:
# CPU % / MEM USAGE / LIMIT / MEM % / NET I/O / BLOCK I/O
```

---

## Docker Context & Remote Builders

### 🟡 Q16. What are Docker contexts and remote builders?

```bash
# Docker context = named connection to a Docker daemon
# Allows switching between local, remote, and cloud builders

# List contexts
docker context ls

# Create remote context (SSH)
docker context create remote-server \
  --docker "host=ssh://user@server.example.com"

# Create remote context (TCP — less secure, use SSH)
docker context create remote-tcp \
  --docker "host=tcp://server.example.com:2376,ca=/certs/ca.pem,cert=/certs/cert.pem,key=/certs/key.pem"

# Switch context
docker context use remote-server

# Run commands on remote
docker ps                              # Shows containers on remote
docker build -t myapp .                # Builds on remote server

# Back to local
docker context use default

# ===== DOCKER BUILDX WITH REMOTE BUILDER =====
# Build on a remote server
docker buildx create \
  --name remote-builder \
  --driver remote \
  tcp://buildkitd.example.com:1234

# Or via SSH
docker buildx create \
  --name remote-builder \
  --driver docker-container \
  --driver-opt network=host \
  "ssh://user@build-server.example.com"

docker buildx use remote-builder

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push .

# ===== CLOUD BUILDERS =====
# Docker Build Cloud (paid service)
docker buildx create --driver cloud myorg/mybuilder
docker buildx use cloud-myorg-mybuilder
docker buildx build -t myapp:latest --push .
```

---

## Docker in Docker (DinD)

### 🔴 Q17. What is Docker in Docker and when do you use it?

```bash
# DinD = running Docker inside a Docker container
# Use case: CI/CD (Jenkins, GitLab CI, GitHub Actions runners in containers)

# ===== METHOD 1: DinD (privileged — NOT recommended for prod) =====
docker run \
  --privileged \
  --name dind \
  -d \
  docker:24-dind

# Connect to inner docker
docker exec -it dind docker ps

# ===== METHOD 2: Docker socket mount (less isolation but safer) =====
docker run \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  myci-image

# ===== METHOD 3: Kaniko (no Docker needed — builds OCI images) =====
# Popular in Kubernetes CI/CD
docker run \
  -v $(pwd):/workspace \
  -v ~/.docker:/kaniko/.docker:ro \
  gcr.io/kaniko-project/executor:latest \
  --context /workspace \
  --dockerfile /workspace/Dockerfile \
  --destination myregistry/myimage:tag

# Kaniko in Kubernetes CI
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      containers:
        - name: kaniko
          image: gcr.io/kaniko-project/executor:latest
          args:
            - "--context=git://github.com/org/repo"
            - "--dockerfile=Dockerfile"
            - "--destination=registry.example.com/myapp:$(BUILD_TAG)"
          volumeMounts:
            - name: registry-credentials
              mountPath: /kaniko/.docker
      volumes:
        - name: registry-credentials
          secret:
            secretName: registry-credentials
            items:
              - key: .dockerconfigjson
                path: config.json
      restartPolicy: Never

# ===== JENKINS WITH DinD =====
# Jenkinsfile (K8s pod with DinD sidecar)
agent {
  kubernetes {
    yaml """
spec:
  containers:
  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
      - name: DOCKER_TLS_CERTDIR
        value: ""
  - name: docker
    image: docker:24
    env:
      - name: DOCKER_HOST
        value: tcp://localhost:2375
    command: [sleep]
    args: [infinity]
"""
  }
}
steps {
  container('docker') {
    sh 'docker build -t myapp .'
  }
}
```

---

## Docker Swarm

### 🟡 Q18. What are Docker Swarm basics?

```bash
# ===== SWARM SETUP =====
# Initialize swarm on manager node
docker swarm init --advertise-addr 10.0.0.1

# Get join token
docker swarm join-token worker
docker swarm join-token manager

# Join as worker from another node
docker swarm join \
  --token SWMTKN-1-xxxxx \
  10.0.0.1:2377

# List nodes
docker node ls
docker node inspect node-1

# Promote worker to manager
docker node promote node-2
docker node demote node-2

# ===== SERVICES =====
# Create a service (like 'docker run' but distributed)
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  --network my-overlay \
  --mount type=volume,source=mydata,target=/data \
  --update-delay 10s \
  --update-order start-first \
  --rollback-delay 5s \
  nginx:alpine

# Scale service
docker service scale web=5

# Update service (rolling update)
docker service update \
  --image nginx:latest \
  --update-parallelism 1 \
  --update-delay 10s \
  web

# Rollback
docker service rollback web

# List services and tasks
docker service ls
docker service ps web
docker service logs -f web

# Remove service
docker service rm web

# ===== STACKS (Compose for Swarm) =====
docker stack deploy -c docker-compose.yaml mystack
docker stack ls
docker stack services mystack
docker stack ps mystack
docker stack rm mystack

# docker-compose.yaml for Swarm
version: "3.8"
services:
  web:
    image: nginx:alpine
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == us-east
    ports:
      - "80:80"
```

---

## Security Best Practices

### 🔴 Q19. What are Docker security best practices?

```dockerfile
# 1. Use minimal base images
FROM scratch                           # Truly minimal
FROM gcr.io/distroless/nodejs:18       # Distroless (no shell, no package manager)
FROM alpine:3.19                       # Small but has shell

# 2. Non-root user
RUN addgroup -g 1001 -S app && adduser -u 1001 -S app -G app
USER app

# 3. Read-only filesystem
# Run with: --read-only
# If app needs writable dirs:
VOLUME /tmp
VOLUME /var/cache/app

# 4. No secrets in ENV or image
# BAD:
ENV DB_PASSWORD=supersecret
# GOOD: inject at runtime via Kubernetes secrets or Vault

# 5. Specific version tags (never latest)
FROM node:20.11.0-alpine3.19    # Pinned to exact version

# 6. Scan before push (CI)
# trivy image myapp:latest --exit-code 1 --severity CRITICAL

# 7. Sign images (cosign)
# cosign sign myapp:latest
```

```bash
# Runtime security

# Drop ALL capabilities, add only what's needed
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  myapp

# Use seccomp profiles
docker run \
  --security-opt seccomp=/etc/docker/seccomp/default.json \
  myapp

# Run with AppArmor profile
docker run \
  --security-opt apparmor=docker-default \
  myapp

# Audit docker daemon
docker info | grep -i security
# SecurityOptions: apparmor, seccomp

# Docker Bench Security
docker run -it --net host --pid host \
  --userns host --cap-add audit_control \
  -v /var/lib:/var/lib \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/lib/systemd:/usr/lib/systemd \
  -v /etc:/etc \
  docker/docker-bench-security
```

---

## Master Cheatsheet

### Docker CLI Quick Reference
```bash
# Images
docker pull image:tag
docker build -t name:tag .
docker build --no-cache -t name:tag .
docker push name:tag
docker image ls
docker image rm name:tag
docker image prune -a

# Containers
docker run -d --name c -p 8080:80 -e KEY=VAL -v vol:/data image
docker start/stop/restart/rm c
docker ps -a
docker logs -f c
docker exec -it c bash
docker inspect c
docker stats

# Networks
docker network create mynet
docker network ls
docker network inspect mynet

# Volumes
docker volume create myvol
docker volume ls
docker volume rm myvol

# Compose
docker compose up -d
docker compose down -v
docker compose logs -f
docker compose exec service bash
docker compose build --no-cache

# System
docker system df
docker system prune -a --volumes
```

### Dockerfile Quickref
```dockerfile
FROM image:tag
LABEL key=value
ARG BUILD_ARG=default
ENV RUNTIME_VAR=value
WORKDIR /app
COPY --chown=user:group src/ dest/
ADD https://url.com/file /dest/
RUN command && cleanup
EXPOSE 8080
VOLUME ["/data"]
USER nonroot
HEALTHCHECK --interval=30s CMD curl -f http://localhost/health
ENTRYPOINT ["executable"]
CMD ["arg1", "arg2"]
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Docker fundamentals vs VMs | 🟢 |
| Q2 | Images and layers | 🟢 |
| Q3 | Optimized Dockerfile | 🟡 |
| Q4 | Multi-stage builds | 🟡 |
| Q5 | BuildKit & remote cache | 🟡 |
| Q6 | .dockerignore | 🟢 |
| Q7 | Container lifecycle | 🟡 |
| Q8 | Health checks | 🟡 |
| Q9 | Docker networking | 🟡 |
| Q10 | Volumes & storage | 🟡 |
| Q11 | Docker Compose | 🟡 |
| Q12 | Registry & image management | 🟡 |
| Q13 | Docker Scout & Trivy scanning | 🟡 |
| Q14 | Content trust & Cosign signing | 🔴 |
| Q15 | Resource limits (cgroups) | 🟡 |
| Q16 | Docker context & remote builders | 🟡 |
| Q17 | Docker in Docker (DinD/Kaniko) | 🔴 |
| Q18 | Docker Swarm | 🟡 |
| Q19 | Security best practices | 🔴 |
