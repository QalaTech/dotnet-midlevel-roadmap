# Part 3 — Containers & Docker

## Why This Matters

Containers solve "it works on my machine" by packaging your application with its entire runtime environment. Same code, same dependencies, same behavior — everywhere.

> "The container is the unit of deployment. If you can run it locally, you can run it anywhere." — Docker philosophy

A well-crafted container is:
- **Reproducible**: Same image, same behavior, every time
- **Portable**: Runs on any container runtime
- **Secure**: Minimal attack surface, non-root user
- **Efficient**: Small size, fast startup

---

## What Goes Wrong Without This

### The "Works on My Machine" Container

```dockerfile
# The bad Dockerfile everyone writes first
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

# 2GB image, SDK included, running as root
# Takes 10 minutes to build
# Different behavior than production
```

**Problems:**
- 2GB image (should be 200MB)
- SDK included (security risk)
- Running as root (security risk)
- No .dockerignore (slow builds, secrets leaked)
- No health checks

### The Docker Compose Chaos

```
Developer A: "My compose file uses ports 5000, 5432, 6379"
Developer B: "Those ports are already in use on my machine"
Developer C: "I changed the environment variables, forgot to commit"

Monday standup: "Has anyone gotten the local setup working?"
Everyone: "No, I've been fighting Docker for 2 hours"
```

### The Production Surprise

```
Local: Container runs fine
Staging: Container runs fine
Production: Container OOM killed after 5 minutes

Investigation:
- Local: 16GB RAM, container uses 4GB
- Staging: 8GB RAM, container uses 4GB
- Production: 1GB container limit, no memory settings

"Why didn't we test with memory limits?"
```

---

## Multi-Stage Dockerfiles

### The Pattern

```dockerfile
# Stage 1: Build (large, has SDK)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files first (better layer caching)
COPY ["src/OrderFlow.Api/OrderFlow.Api.csproj", "src/OrderFlow.Api/"]
COPY ["src/OrderFlow.Domain/OrderFlow.Domain.csproj", "src/OrderFlow.Domain/"]
COPY ["src/OrderFlow.Infrastructure/OrderFlow.Infrastructure.csproj", "src/OrderFlow.Infrastructure/"]
COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]

# Restore dependencies (cached unless csproj changes)
RUN dotnet restore "src/OrderFlow.Api/OrderFlow.Api.csproj"

# Copy everything else and build
COPY . .
WORKDIR "/src/src/OrderFlow.Api"
RUN dotnet build -c Release -o /app/build

# Publish (creates self-contained output)
RUN dotnet publish -c Release -o /app/publish \
    --no-restore \
    -p:PublishReadyToRun=true \
    -p:PublishTrimmed=false

# Stage 2: Runtime (small, no SDK)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime

# Security: Run as non-root user
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -D appuser

WORKDIR /app

# Copy published output from build stage
COPY --from=build --chown=appuser:appgroup /app/publish .

# Switch to non-root user
USER appuser

# Configure ASP.NET Core
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_RUNNING_IN_CONTAINER=true

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "OrderFlow.Api.dll"]
```

### Layer Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    DOCKERFILE LAYERS                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Base image (aspnet:8.0-alpine)      [Cached]      │
│           ↓                                                  │
│  Layer 2: Create non-root user                [Cached]      │
│           ↓                                                  │
│  Layer 3: Copy .csproj files                  [Cached*]     │
│           ↓                                                  │
│  Layer 4: dotnet restore                      [Cached*]     │
│           ↓                                                  │
│  Layer 5: Copy source code                    [Rebuild]     │
│           ↓                                                  │
│  Layer 6: dotnet build                        [Rebuild]     │
│           ↓                                                  │
│  Layer 7: dotnet publish                      [Rebuild]     │
│                                                              │
│  * Cached until project files change                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### .dockerignore

```
# .dockerignore
**/.git
**/.vs
**/.vscode
**/bin
**/obj
**/node_modules
**/.idea

# Build outputs
**/publish
**/TestResults
**/coverage

# Secrets and local config
**/*.env
**/appsettings.Development.json
**/appsettings.Local.json
**/*.pfx
**/*.key

# Documentation
**/README.md
**/docs

# Tests (build separately if needed)
**/tests
**/*.Tests
**/*.IntegrationTests
```

---

## Complete OrderFlow Dockerfile

### API Service

```dockerfile
# src/OrderFlow.Api/Dockerfile
ARG DOTNET_VERSION=8.0

# ============================================
# Stage 1: Build
# ============================================
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build

ARG VERSION=1.0.0
ARG CONFIGURATION=Release

WORKDIR /src

# Copy solution and project files for restore
COPY ["OrderFlow.sln", "./"]
COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]
COPY ["src/OrderFlow.Api/OrderFlow.Api.csproj", "src/OrderFlow.Api/"]
COPY ["src/OrderFlow.Domain/OrderFlow.Domain.csproj", "src/OrderFlow.Domain/"]
COPY ["src/OrderFlow.Application/OrderFlow.Application.csproj", "src/OrderFlow.Application/"]
COPY ["src/OrderFlow.Infrastructure/OrderFlow.Infrastructure.csproj", "src/OrderFlow.Infrastructure/"]

# Restore (this layer is cached until csproj changes)
RUN dotnet restore "src/OrderFlow.Api/OrderFlow.Api.csproj"

# Copy source code
COPY src/ src/

# Build
WORKDIR "/src/src/OrderFlow.Api"
RUN dotnet build -c ${CONFIGURATION} -o /app/build \
    -p:Version=${VERSION} \
    --no-restore

# ============================================
# Stage 2: Publish
# ============================================
FROM build AS publish

ARG VERSION=1.0.0
ARG CONFIGURATION=Release

RUN dotnet publish -c ${CONFIGURATION} -o /app/publish \
    -p:Version=${VERSION} \
    -p:PublishReadyToRun=true \
    -p:PublishSingleFile=false \
    --no-restore

# ============================================
# Stage 3: Runtime
# ============================================
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-alpine AS runtime

# Install curl for health checks
RUN apk add --no-cache curl

# Security hardening
RUN addgroup -g 1000 orderflow && \
    adduser -u 1000 -G orderflow -D orderflow && \
    # Remove unnecessary packages
    apk del --purge apk-tools && \
    rm -rf /var/cache/apk/*

WORKDIR /app

# Copy published output
COPY --from=publish --chown=orderflow:orderflow /app/publish .

# Use non-root user
USER orderflow

# Environment configuration
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    TZ=UTC

EXPOSE 8080

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl --fail http://localhost:8080/health/live || exit 1

# Labels for container metadata
LABEL org.opencontainers.image.title="OrderFlow API" \
      org.opencontainers.image.description="OrderFlow Order Management API" \
      org.opencontainers.image.vendor="OrderFlow" \
      org.opencontainers.image.version="${VERSION}"

ENTRYPOINT ["dotnet", "OrderFlow.Api.dll"]
```

### Worker Service

```dockerfile
# src/OrderFlow.Worker/Dockerfile
ARG DOTNET_VERSION=8.0

FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build

WORKDIR /src
COPY ["OrderFlow.sln", "./"]
COPY ["Directory.Build.props", "./"]
COPY ["Directory.Packages.props", "./"]
COPY ["src/OrderFlow.Worker/OrderFlow.Worker.csproj", "src/OrderFlow.Worker/"]
COPY ["src/OrderFlow.Domain/OrderFlow.Domain.csproj", "src/OrderFlow.Domain/"]
COPY ["src/OrderFlow.Application/OrderFlow.Application.csproj", "src/OrderFlow.Application/"]
COPY ["src/OrderFlow.Infrastructure/OrderFlow.Infrastructure.csproj", "src/OrderFlow.Infrastructure/"]

RUN dotnet restore "src/OrderFlow.Worker/OrderFlow.Worker.csproj"

COPY src/ src/

WORKDIR "/src/src/OrderFlow.Worker"
RUN dotnet publish -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/runtime:${DOTNET_VERSION}-alpine AS runtime

RUN addgroup -g 1000 orderflow && \
    adduser -u 1000 -G orderflow -D orderflow

WORKDIR /app
COPY --from=build --chown=orderflow:orderflow /app/publish .

USER orderflow

ENV DOTNET_RUNNING_IN_CONTAINER=true

# Workers don't typically need health checks via HTTP
# but can use a file-based health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD test -f /tmp/healthy || exit 1

ENTRYPOINT ["dotnet", "OrderFlow.Worker.dll"]
```

---

## Docker Compose for Local Development

### Complete Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # Application Services
  # ============================================
  api:
    build:
      context: .
      dockerfile: src/OrderFlow.Api/Dockerfile
      args:
        - VERSION=local
    container_name: orderflow-api
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Database=Host=postgres;Database=orderflow;Username=orderflow;Password=orderflow
      - ConnectionStrings__Redis=redis:6379
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=guest
      - RabbitMQ__Password=guest
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - orderflow-network
    restart: unless-stopped

  worker:
    build:
      context: .
      dockerfile: src/OrderFlow.Worker/Dockerfile
    container_name: orderflow-worker
    environment:
      - DOTNET_ENVIRONMENT=Development
      - ConnectionStrings__Database=Host=postgres;Database=orderflow;Username=orderflow;Password=orderflow
      - ConnectionStrings__Redis=redis:6379
      - RabbitMQ__Host=rabbitmq
    depends_on:
      - api
      - rabbitmq
    networks:
      - orderflow-network
    restart: unless-stopped

  # ============================================
  # Infrastructure Services
  # ============================================
  postgres:
    image: postgres:16-alpine
    container_name: orderflow-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow
      POSTGRES_DB: orderflow
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orderflow -d orderflow"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - orderflow-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: orderflow-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - orderflow-network
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: orderflow-rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - orderflow-network
    restart: unless-stopped

  # ============================================
  # Observability
  # ============================================
  seq:
    image: datalust/seq:latest
    container_name: orderflow-seq
    ports:
      - "5341:5341"   # Ingestion
      - "8081:80"     # UI
    environment:
      ACCEPT_EULA: "Y"
    volumes:
      - seq-data:/data
    networks:
      - orderflow-network
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: orderflow-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - orderflow-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: orderflow-grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./observability/grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - prometheus
    networks:
      - orderflow-network
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: orderflow-jaeger
    ports:
      - "16686:16686" # UI
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    networks:
      - orderflow-network
    restart: unless-stopped

networks:
  orderflow-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  rabbitmq-data:
  seq-data:
  prometheus-data:
  grafana-data:
```

### Development Override

```yaml
# docker-compose.override.yml (automatically loaded with docker-compose up)
version: '3.8'

services:
  api:
    build:
      target: build  # Stop at build stage for faster rebuilds
    volumes:
      # Mount source for hot reload
      - ./src:/src:cached
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    command: ["dotnet", "watch", "run", "--project", "src/OrderFlow.Api"]

  worker:
    build:
      target: build
    volumes:
      - ./src:/src:cached
    environment:
      - DOTNET_ENVIRONMENT=Development
    command: ["dotnet", "watch", "run", "--project", "src/OrderFlow.Worker"]
```

### Production Override

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    image: ghcr.io/yourorg/orderflow-api:${VERSION:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

  worker:
    image: ghcr.io/yourorg/orderflow-worker:${VERSION:-latest}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
```

---

## Container Security

### Security Checklist

```
┌─────────────────────────────────────────────────────────────┐
│                   CONTAINER SECURITY                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Build Time:                                                 │
│  ☑ Use official base images                                 │
│  ☑ Pin image versions (not :latest)                         │
│  ☑ Use multi-stage builds (no SDK in runtime)               │
│  ☑ Scan for vulnerabilities (Trivy, Grype)                  │
│  ☑ No secrets in Dockerfile                                 │
│  ☑ Use .dockerignore                                        │
│                                                              │
│  Runtime:                                                    │
│  ☑ Run as non-root user                                     │
│  ☑ Read-only filesystem where possible                      │
│  ☑ Drop all capabilities                                    │
│  ☑ Use security contexts                                    │
│  ☑ Limit resources (CPU, memory)                            │
│  ☑ Health checks configured                                 │
│                                                              │
│  Network:                                                    │
│  ☑ Don't expose unnecessary ports                           │
│  ☑ Use internal networks between services                   │
│  ☑ TLS for external traffic                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Secure Dockerfile Patterns

```dockerfile
# Use specific version, not :latest
FROM mcr.microsoft.com/dotnet/aspnet:8.0.1-alpine3.18

# Create non-root user
RUN addgroup -g 1000 app && adduser -u 1000 -G app -D app

# Remove unnecessary tools
RUN apk del --purge apk-tools && rm -rf /var/cache/apk/*

# Set file permissions
COPY --chown=app:app --chmod=550 ./publish /app

# Switch to non-root
USER app

# Read-only root filesystem
# Set in docker-compose or kubernetes:
# security_context:
#   read_only_root_filesystem: true
```

### Security Scanning in CI

```yaml
# Trivy scan
- name: Scan container image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ env.IMAGE }}'
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'

# Grype scan (alternative)
- name: Scan with Grype
  uses: anchore/scan-action@v3
  with:
    image: '${{ env.IMAGE }}'
    fail-build: true
    severity-cutoff: high
    output-format: sarif

# Hadolint for Dockerfile best practices
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: src/OrderFlow.Api/Dockerfile
    failure-threshold: warning
```

### Runtime Security with Docker Compose

```yaml
services:
  api:
    image: orderflow-api:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
    user: "1000:1000"
```

---

## Container Registry Management

### GitHub Container Registry

```yaml
# Login and push
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ghcr.io/${{ github.repository }}/api:${{ github.sha }}
      ghcr.io/${{ github.repository }}/api:latest
```

### Azure Container Registry

```bash
# Create ACR
az acr create --name orderflowacr --resource-group orderflow-rg --sku Basic

# Enable admin access (for dev only)
az acr update --name orderflowacr --admin-enabled true

# Build and push (using ACR Tasks)
az acr build --registry orderflowacr --image orderflow-api:v1 .

# Enable vulnerability scanning
az acr config content-trust update --registry orderflowacr --status enabled
```

### Image Tagging Strategy

```
Tag Strategy:
  - :latest           → Current production release
  - :v1.2.3           → Semantic version (immutable)
  - :sha-abc123       → Git commit SHA (immutable)
  - :main             → Latest from main branch
  - :develop          → Latest from develop branch
  - :pr-123           → Pull request builds

Best Practice:
  - Production uses :v1.2.3 (immutable)
  - Never deploy :latest to production
  - Always scan before pushing
```

---

## Local Development Scripts

### Makefile

```makefile
# Makefile
.PHONY: up down build logs clean test migrate

# Default target
help:
	@echo "OrderFlow Development Commands"
	@echo "────────────────────────────────"
	@echo "make up       - Start all services"
	@echo "make down     - Stop all services"
	@echo "make build    - Build containers"
	@echo "make logs     - Follow all logs"
	@echo "make clean    - Remove all containers and volumes"
	@echo "make test     - Run tests in container"
	@echo "make migrate  - Run database migrations"
	@echo "make shell    - Open shell in API container"

up:
	docker compose up -d
	@echo "Services starting..."
	@echo "API: http://localhost:5000"
	@echo "Seq: http://localhost:8081"
	@echo "Grafana: http://localhost:3000"
	@echo "RabbitMQ: http://localhost:15672"

down:
	docker compose down

build:
	docker compose build --no-cache

logs:
	docker compose logs -f

clean:
	docker compose down -v --rmi local --remove-orphans
	docker system prune -f

test:
	docker compose run --rm api dotnet test

migrate:
	docker compose run --rm api dotnet ef database update

shell:
	docker compose exec api /bin/sh

# Development with hot reload
dev:
	docker compose -f docker-compose.yml -f docker-compose.override.yml up

# Production-like environment
prod:
	docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Check health of all services
health:
	@echo "Checking service health..."
	@curl -sf http://localhost:5000/health && echo "API: OK" || echo "API: FAILED"
	@curl -sf http://localhost:8081 && echo "Seq: OK" || echo "Seq: FAILED"
	@curl -sf http://localhost:9090/-/healthy && echo "Prometheus: OK" || echo "Prometheus: FAILED"
```

### PowerShell Script (Windows)

```powershell
# scripts/dev.ps1
param(
    [Parameter(Position=0)]
    [string]$Command = "help"
)

function Show-Help {
    Write-Host "OrderFlow Development Commands" -ForegroundColor Cyan
    Write-Host "────────────────────────────────"
    Write-Host "  up      - Start all services"
    Write-Host "  down    - Stop all services"
    Write-Host "  build   - Build containers"
    Write-Host "  logs    - Follow all logs"
    Write-Host "  clean   - Remove everything"
    Write-Host "  test    - Run tests"
    Write-Host "  shell   - Open API shell"
}

switch ($Command) {
    "up" {
        docker compose up -d
        Write-Host "`nServices:" -ForegroundColor Green
        Write-Host "  API:      http://localhost:5000"
        Write-Host "  Seq:      http://localhost:8081"
        Write-Host "  Grafana:  http://localhost:3000"
    }
    "down" { docker compose down }
    "build" { docker compose build --no-cache }
    "logs" { docker compose logs -f }
    "clean" {
        docker compose down -v --rmi local
        docker system prune -f
    }
    "test" { docker compose run --rm api dotnet test }
    "shell" { docker compose exec api /bin/sh }
    default { Show-Help }
}
```

---

## Health Checks and Probes

### Kubernetes-Style Health Checks

```csharp
// Program.cs - Health check configuration
builder.Services.AddHealthChecks()
    // Liveness: Is the app alive?
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])

    // Readiness: Is the app ready to serve traffic?
    .AddNpgSql(
        connectionString,
        name: "database",
        tags: ["ready"])
    .AddRedis(
        redisConnectionString,
        name: "redis",
        tags: ["ready"])
    .AddRabbitMQ(
        rabbitConnectionString,
        name: "rabbitmq",
        tags: ["ready"]);

// Map endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Dockerfile Health Check

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl --fail http://localhost:8080/health/live || exit 1

# Or for containers without curl
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1
```

### Docker Compose Health Dependencies

```yaml
services:
  api:
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

---

## Hands-On Exercise: Containerize OrderFlow

### Step 1: Create Optimized Dockerfile

Create `src/OrderFlow.Api/Dockerfile`:
- Multi-stage build
- Non-root user
- Health check
- Proper layer caching

### Step 2: Create Docker Compose Stack

Create `docker-compose.yml`:
- API service
- PostgreSQL
- Redis
- RabbitMQ
- Seq for logging

### Step 3: Add Development Override

Create `docker-compose.override.yml`:
- Hot reload with volume mounts
- Development environment variables

### Step 4: Create Local Dev Scripts

Create `Makefile` or `scripts/dev.ps1`:
- up, down, build, logs, clean commands
- Health check command

### Step 5: Test the Stack

```bash
# Build and start
make up

# Verify health
curl http://localhost:5000/health

# Check logs
make logs

# Clean up
make clean
```

---

## Deliverables

1. **Dockerfile**: Multi-stage, secure, optimized
2. **docker-compose.yml**: Complete development stack
3. **docker-compose.override.yml**: Development hot reload
4. **Makefile/scripts**: Developer convenience commands
5. **.dockerignore**: Proper exclusions
6. **Documentation**: README with setup instructions

---

## Reflection Questions

1. Why use multi-stage builds instead of single-stage?
2. What's the security risk of running containers as root?
3. How do health checks differ between Docker and Kubernetes?
4. What's the trade-off between image size and build time?

---

## Resources

### Docker
- [Docker Documentation](https://docs.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker for .NET Developers](https://www.youtube.com/watch?v=vmnvOITMoIg) - Nick Chapsas
- [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

### .NET and Containers
- [.NET Container Images](https://hub.docker.com/_/microsoft-dotnet)
- [Containerize .NET Apps](https://learn.microsoft.com/en-us/dotnet/core/docker/introduction)
- [Health Checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)

### Container Security
- [Trivy](https://trivy.dev/)
- [Grype](https://github.com/anchore/grype)
- [Hadolint](https://github.com/hadolint/hadolint)
- [Snyk Container](https://snyk.io/product/container-vulnerability-management/)

### Docker Compose
- [Compose Specification](https://docs.docker.com/compose/compose-file/)
- [Awesome Docker Compose](https://github.com/docker/awesome-compose)

---

**Next:** [Part 4 — Infrastructure as Code](./part-4.md)
