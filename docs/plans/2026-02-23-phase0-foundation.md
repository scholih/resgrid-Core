# Connexio-Ops Phase 0: Foundation Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a running local development stack for Connexio-Ops with a Go API scaffold, Python AI service scaffold, and all infrastructure services communicating correctly via a single `docker-compose up`.

**Architecture:** New `connexio-ops` repository at `~/demos/connexio-ops/` containing a Go monorepo for the platform core and a Python FastAPI service (`connexio-ai`) extracted cleanly from kustw8-core. Infrastructure runs in Docker Compose: PostgreSQL+pgvector, Neo4j, NATS JetStream, Redis, Zitadel.

**Tech Stack:** Go 1.25, Python 3.12+, uv, FastAPI, PostgreSQL 16+pgvector, Neo4j 5 Community, NATS JetStream, Redis 7, Zitadel

**Reference:** Design doc at `~/demos/resgrid-Core/docs/plans/2026-02-23-connexio-ops-design.md`

---

## Task 1: Create connexio-ops repository

**Files:**
- Create: `~/demos/connexio-ops/` (new git repo)
- Create: `~/demos/connexio-ops/.gitignore`
- Create: `~/demos/connexio-ops/README.md`

**Step 1: Create directory and init git**

```bash
mkdir -p ~/demos/connexio-ops
cd ~/demos/connexio-ops
git init
```

Expected: `Initialized empty Git repository in .../connexio-ops/.git/`

**Step 2: Create .gitignore**

```bash
cat > ~/demos/connexio-ops/.gitignore << 'EOF'
# Go
bin/
*.exe
*.test
*.out

# Python
__pycache__/
*.pyc
.venv/
*.egg-info/
dist/
.pytest_cache/

# Environment
.env
.env.local

# Docker volumes
volumes/

# IDE
.idea/
.vscode/
*.swp

# Worktrees
.worktrees/
EOF
```

**Step 3: Create minimal README**

```bash
cat > ~/demos/connexio-ops/README.md << 'EOF'
# Connexio-Ops

Defense-grade operational platform: CAD + Knowledge Management + AI Intelligence.

## Quick Start

```bash
docker-compose up -d
```

See `docs/` for architecture and development guides.
EOF
```

**Step 4: Initial commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "chore: init connexio-ops repository"
```

---

## Task 2: Go module and project structure

**Files:**
- Create: `~/demos/connexio-ops/go.mod`
- Create: `~/demos/connexio-ops/cmd/api/main.go`
- Create: `~/demos/connexio-ops/cmd/worker/main.go`
- Create: `~/demos/connexio-ops/cmd/migrate/main.go`
- Create: `~/demos/connexio-ops/internal/domain/.gitkeep`
- Create: `~/demos/connexio-ops/internal/store/.gitkeep`
- Create: `~/demos/connexio-ops/internal/services/.gitkeep`
- Create: `~/demos/connexio-ops/internal/transport/.gitkeep`
- Create: `~/demos/connexio-ops/internal/adapters/.gitkeep`

**Step 1: Init Go module**

```bash
cd ~/demos/connexio-ops
go mod init github.com/relogiks/connexio-ops
```

Expected: creates `go.mod` with `module github.com/relogiks/connexio-ops` and `go 1.25`

**Step 2: Create directory structure**

```bash
mkdir -p ~/demos/connexio-ops/cmd/{api,worker,migrate}
mkdir -p ~/demos/connexio-ops/internal/{domain,store,services,transport,adapters}
mkdir -p ~/demos/connexio-ops/pkg
touch ~/demos/connexio-ops/internal/{domain,store,services,transport,adapters}/.gitkeep
```

**Step 3: Create cmd/api/main.go**

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintln(w, `{"status":"ok","service":"connexio-api"}`)
	})

	log.Printf("connexio-api starting on :%s", port)
	if err := http.ListenAndServe(":"+port, mux); err != nil {
		log.Fatalf("server error: %v", err)
	}
}
```

**Step 4: Create cmd/worker/main.go**

```go
package main

import (
	"log"
)

func main() {
	log.Println("connexio-worker starting...")
	// Phase 1: NATS subscription setup goes here
	select {} // block forever
}
```

**Step 5: Create cmd/migrate/main.go**

```go
package main

import (
	"log"
	"os"
)

func main() {
	log.Println("connexio-migrate starting...")
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		log.Fatal("DATABASE_URL not set")
	}
	log.Printf("connected to: %s", dbURL)
	log.Println("migrations: none defined yet")
}
```

**Step 6: Verify Go builds**

```bash
cd ~/demos/connexio-ops
go build ./cmd/api/
go build ./cmd/worker/
go build ./cmd/migrate/
```

Expected: no errors, three binaries produced in current directory

**Step 7: Write failing test for health endpoint**

Create `~/demos/connexio-ops/cmd/api/main_test.go`:

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestHealthEndpoint(t *testing.T) {
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`{"status":"ok","service":"connexio-api"}`))
	})

	req := httptest.NewRequest(http.MethodGet, "/health", nil)
	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Errorf("expected status 200, got %d", w.Code)
	}
	if !strings.Contains(w.Body.String(), `"status":"ok"`) {
		t.Errorf("expected ok status in body, got: %s", w.Body.String())
	}
}
```

**Step 8: Run test to verify it passes**

```bash
cd ~/demos/connexio-ops
go test ./cmd/api/ -v
```

Expected: `PASS TestHealthEndpoint`

**Step 9: Commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "feat: Go module scaffold with health endpoint"
```

---

## Task 3: Docker Compose infrastructure stack

**Files:**
- Create: `~/demos/connexio-ops/docker-compose.yml`
- Create: `~/demos/connexio-ops/.env.example`
- Create: `~/demos/connexio-ops/infra/postgres/init.sql`

**Step 1: Create postgres init script (enables pgvector)**

```bash
mkdir -p ~/demos/connexio-ops/infra/postgres
cat > ~/demos/connexio-ops/infra/postgres/init.sql << 'EOF'
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify
SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';
EOF
```

**Step 2: Create .env.example**

```bash
cat > ~/demos/connexio-ops/.env.example << 'EOF'
# Copy to .env and adjust values

# Deployment
DEPLOYMENT_MODE=airgapped
TENANT_NAME=connexio
LANGUAGE_PRIMARY=nl
LANGUAGE_SECONDARY=en

# PostgreSQL
POSTGRES_DB=connexio
POSTGRES_USER=connexio
POSTGRES_PASSWORD=connexio_dev_password
DATABASE_URL=postgresql://connexio:connexio_dev_password@localhost:5432/connexio

# Redis
REDIS_URL=redis://localhost:6379

# NATS
NATS_URL=nats://localhost:4222

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=connexio_dev_password

# Zitadel
ZITADEL_DOMAIN=localhost:8081

# connexio-ai
AI_SERVICE_URL=http://localhost:8000

# Ollama
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b
EMBEDDING_MODEL=bge-m3

# API
PORT=8080
EOF

cp ~/demos/connexio-ops/.env.example ~/demos/connexio-ops/.env
```

**Step 3: Create docker-compose.yml**

```yaml
# ~/demos/connexio-ops/docker-compose.yml
name: connexio-ops

services:

  # ── PostgreSQL + pgvector ─────────────────────────────────────
  postgres:
    image: pgvector/pgvector:pg16
    container_name: connexio-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-connexio}
      POSTGRES_USER: ${POSTGRES_USER:-connexio}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-connexio_dev_password}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-connexio}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - connexio-network

  # ── Redis ────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: connexio-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - connexio-network

  # ── NATS JetStream ───────────────────────────────────────────
  nats:
    image: nats:2-alpine
    container_name: connexio-nats
    command: ["-js", "-m", "8222"]
    ports:
      - "4222:4222"   # client
      - "8222:8222"   # monitoring
    volumes:
      - nats_data:/data
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8222/healthz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - connexio-network

  # ── Neo4j Community ──────────────────────────────────────────
  neo4j:
    image: neo4j:5-community
    container_name: connexio-neo4j
    environment:
      NEO4J_AUTH: neo4j/${NEO4J_PASSWORD:-connexio_dev_password}
      NEO4J_PLUGINS: '["apoc"]'
    ports:
      - "7474:7474"   # browser
      - "7687:7687"   # bolt
    volumes:
      - neo4j_data:/data
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:7474 || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s
    networks:
      - connexio-network

  # ── Zitadel (OIDC identity provider) ─────────────────────────
  zitadel:
    image: ghcr.io/zitadel/zitadel:latest
    container_name: connexio-zitadel
    command: start-from-init --masterkey "MasterkeyNeedsToHave32Characters" --tlsMode disabled
    environment:
      ZITADEL_DATABASE_POSTGRES_HOST: postgres
      ZITADEL_DATABASE_POSTGRES_PORT: 5432
      ZITADEL_DATABASE_POSTGRES_DATABASE: zitadel
      ZITADEL_DATABASE_POSTGRES_USER_USERNAME: connexio
      ZITADEL_DATABASE_POSTGRES_USER_PASSWORD: connexio_dev_password
      ZITADEL_DATABASE_POSTGRES_USER_SSL_MODE: disable
      ZITADEL_DATABASE_POSTGRES_ADMIN_USERNAME: connexio
      ZITADEL_DATABASE_POSTGRES_ADMIN_PASSWORD: connexio_dev_password
      ZITADEL_DATABASE_POSTGRES_ADMIN_SSL_MODE: disable
      ZITADEL_EXTERNALSECURE: "false"
      ZITADEL_EXTERNALPORT: 8081
      ZITADEL_EXTERNALDOMAIN: localhost
    ports:
      - "8081:8080"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - connexio-network

networks:
  connexio-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  nats_data:
  neo4j_data:
```

**Step 4: Start infrastructure**

```bash
cd ~/demos/connexio-ops
docker-compose up -d postgres redis nats neo4j
```

Expected: 4 containers starting (skip Zitadel for now — it needs postgres healthy first)

**Step 5: Wait for healthy state**

```bash
docker-compose ps
```

Expected: postgres, redis, nats, neo4j all showing `healthy` or `running`

**Step 6: Verify pgvector extension**

```bash
docker exec connexio-postgres psql -U connexio -d connexio -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

Expected:
```
 extname | extversion
---------+------------
 vector  | 0.8.0
(1 row)
```

**Step 7: Verify NATS JetStream**

```bash
curl -s http://localhost:8222/healthz
```

Expected: `{"status":"ok"}`

**Step 8: Verify Redis**

```bash
docker exec connexio-redis redis-cli ping
```

Expected: `PONG`

**Step 9: Verify Neo4j**

```bash
curl -s http://localhost:7474 | head -5
```

Expected: HTML response (Neo4j browser)

**Step 10: Start Zitadel**

```bash
cd ~/demos/connexio-ops
docker-compose up -d zitadel
sleep 30
docker-compose logs zitadel | tail -20
```

Expected: Zitadel starts and shows no fatal errors. First boot takes ~30s.

**Step 11: Commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "feat: Docker Compose infrastructure stack (postgres+pgvector, redis, nats, neo4j, zitadel)"
```

---

## Task 4: Go API connects to PostgreSQL

**Files:**
- Modify: `~/demos/connexio-ops/go.mod` (add pgx dependency)
- Create: `~/demos/connexio-ops/internal/store/db.go`
- Create: `~/demos/connexio-ops/internal/store/db_test.go`

**Step 1: Add pgx dependency**

```bash
cd ~/demos/connexio-ops
go get github.com/jackc/pgx/v5
```

Expected: `go.mod` updated with pgx/v5

**Step 2: Write failing test for DB connectivity**

Create `~/demos/connexio-ops/internal/store/db_test.go`:

```go
package store_test

import (
	"context"
	"os"
	"testing"

	"github.com/relogiks/connexio-ops/internal/store"
)

func TestConnect(t *testing.T) {
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		t.Skip("DATABASE_URL not set — skipping integration test")
	}

	db, err := store.Connect(context.Background(), dbURL)
	if err != nil {
		t.Fatalf("expected connection, got error: %v", err)
	}
	defer db.Close()

	if err := db.Ping(context.Background()); err != nil {
		t.Fatalf("expected ping to succeed, got: %v", err)
	}
}
```

**Step 3: Run test — verify it fails**

```bash
cd ~/demos/connexio-ops
DATABASE_URL=postgresql://connexio:connexio_dev_password@localhost:5432/connexio go test ./internal/store/ -v -run TestConnect
```

Expected: `FAIL` — `store.Connect undefined`

**Step 4: Implement store.Connect**

Create `~/demos/connexio-ops/internal/store/db.go`:

```go
package store

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// DB wraps a pgx connection pool.
type DB struct {
	pool *pgxpool.Pool
}

// Connect creates a new connection pool for the given PostgreSQL URL.
func Connect(ctx context.Context, dbURL string) (*DB, error) {
	pool, err := pgxpool.New(ctx, dbURL)
	if err != nil {
		return nil, fmt.Errorf("store.Connect: %w", err)
	}
	return &DB{pool: pool}, nil
}

// Ping verifies the connection is alive.
func (db *DB) Ping(ctx context.Context) error {
	return db.pool.Ping(ctx)
}

// Close releases all connections in the pool.
func (db *DB) Close() {
	db.pool.Close()
}
```

**Step 5: Run test — verify it passes**

```bash
cd ~/demos/connexio-ops
DATABASE_URL=postgresql://connexio:connexio_dev_password@localhost:5432/connexio go test ./internal/store/ -v -run TestConnect
```

Expected: `PASS TestConnect`

**Step 6: Write failing test for pgvector extension presence**

Add to `~/demos/connexio-ops/internal/store/db_test.go`:

```go
func TestPgvectorExtensionEnabled(t *testing.T) {
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		t.Skip("DATABASE_URL not set — skipping integration test")
	}

	db, err := store.Connect(context.Background(), dbURL)
	if err != nil {
		t.Fatalf("connect: %v", err)
	}
	defer db.Close()

	enabled, err := db.PgvectorEnabled(context.Background())
	if err != nil {
		t.Fatalf("PgvectorEnabled: %v", err)
	}
	if !enabled {
		t.Error("expected pgvector extension to be enabled")
	}
}
```

**Step 7: Run test — verify it fails**

```bash
cd ~/demos/connexio-ops
DATABASE_URL=postgresql://connexio:connexio_dev_password@localhost:5432/connexio go test ./internal/store/ -v -run TestPgvectorExtensionEnabled
```

Expected: `FAIL` — `db.PgvectorEnabled undefined`

**Step 8: Implement PgvectorEnabled**

Add to `~/demos/connexio-ops/internal/store/db.go`:

```go
// PgvectorEnabled reports whether the pgvector extension is installed.
func (db *DB) PgvectorEnabled(ctx context.Context) (bool, error) {
	var count int
	err := db.pool.QueryRow(ctx,
		"SELECT COUNT(*) FROM pg_extension WHERE extname = 'vector'",
	).Scan(&count)
	if err != nil {
		return false, fmt.Errorf("PgvectorEnabled: %w", err)
	}
	return count > 0, nil
}
```

**Step 9: Run all store tests — verify they pass**

```bash
cd ~/demos/connexio-ops
DATABASE_URL=postgresql://connexio:connexio_dev_password@localhost:5432/connexio go test ./internal/store/ -v
```

Expected: `PASS` for both tests

**Step 10: Commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "feat: PostgreSQL connection pool with pgvector verification"
```

---

## Task 5: connexio-ai FastAPI scaffold

This extracts the AI service from kustw8-core as a clean, standalone service — removing Streamlit, OpenSearch, Keycloak, OpenFGA, and the Dropbox-local package dependencies. It keeps the core RAG capabilities and adds NATS event subscription.

**Files:**
- Create: `~/demos/connexio-ops/services/connexio-ai/` (new Python project)
- Create: `~/demos/connexio-ops/services/connexio-ai/pyproject.toml`
- Create: `~/demos/connexio-ops/services/connexio-ai/src/main.py`
- Create: `~/demos/connexio-ops/services/connexio-ai/src/config.py`
- Create: `~/demos/connexio-ops/services/connexio-ai/src/routers/health.py`
- Create: `~/demos/connexio-ops/services/connexio-ai/Dockerfile`
- Create: `~/demos/connexio-ops/services/connexio-ai/tests/test_health.py`

**Step 1: Create Python project**

```bash
mkdir -p ~/demos/connexio-ops/services/connexio-ai/src/routers
mkdir -p ~/demos/connexio-ops/services/connexio-ai/tests
cd ~/demos/connexio-ops/services/connexio-ai
```

**Step 2: Create pyproject.toml**

```toml
[project]
name = "connexio-ai"
version = "0.1.0"
description = "Connexio-Ops AI intelligence service"
requires-python = ">=3.12,<3.13"
dependencies = [
    # Web framework
    "fastapi[standard]>=0.115.0",
    "uvicorn>=0.34.0",
    "pydantic>=2.11.0",
    "pydantic-settings>=2.8.0",
    # Database
    "asyncpg>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.0",
    "pgvector>=0.3.0",
    # NATS messaging
    "nats-py>=2.9.0",
    # Neo4j
    "neo4j>=5.27.0",
    # Redis
    "redis>=7.1.0",
    # Ollama (local LLM)
    "ollama>=0.6.0",
    # Embeddings
    "sentence-transformers>=5.0.0",
    # HTTP client
    "httpx>=0.28.0",
    # Observability
    "langfuse>=3.0.0",
    "loguru>=0.7.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.25.0",
    "httpx>=0.28.0",
    "anyio[trio]>=4.0.0",
    "asgi-lifespan>=2.1.0",
    "ruff>=0.11.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"

[tool.ruff]
line-length = 120
src = ["src", "tests"]
lint.select = ["I", "E", "F"]
```

**Step 3: Write failing health test**

Create `~/demos/connexio-ops/services/connexio-ai/tests/test_health.py`:

```python
import pytest
from asgi_lifespan import LifespanManager
from httpx import AsyncClient, ASGITransport


@pytest.fixture
async def client():
    from src.main import app
    async with LifespanManager(app):
        async with AsyncClient(
            transport=ASGITransport(app=app),
            base_url="http://test"
        ) as c:
            yield c


async def test_health_returns_ok(client):
    response = await client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "ok"
    assert data["service"] == "connexio-ai"
```

**Step 4: Run test — verify it fails**

```bash
cd ~/demos/connexio-ops/services/connexio-ai
uv run pytest tests/test_health.py -v
```

Expected: `ERROR` — `ModuleNotFoundError: No module named 'src'`

**Step 5: Create src/config.py**

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # Service
    service_name: str = "connexio-ai"
    port: int = 8000
    debug: bool = False

    # PostgreSQL
    database_url: str = "postgresql+asyncpg://connexio:connexio_dev_password@localhost:5432/connexio"

    # NATS
    nats_url: str = "nats://localhost:4222"

    # Neo4j
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = "connexio_dev_password"

    # Redis
    redis_url: str = "redis://localhost:6379"

    # Ollama
    ollama_host: str = "http://localhost:11434"
    ollama_model: str = "qwen2.5:7b"
    embedding_model: str = "bge-m3"


def get_settings() -> Settings:
    return Settings()
```

**Step 6: Create src/routers/health.py**

```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/health")
async def health():
    return {"status": "ok", "service": "connexio-ai"}
```

**Step 7: Create src/main.py**

```python
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI

from src.config import get_settings
from src.routers.health import router as health_router

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    app.state.settings = settings
    logger.info("connexio-ai starting up")
    yield
    logger.info("connexio-ai shutting down")


app = FastAPI(
    title="connexio-ai",
    description="Connexio-Ops AI Intelligence Service",
    version="0.1.0",
    lifespan=lifespan,
)

app.include_router(health_router)
```

**Step 8: Create src/__init__.py**

```bash
touch ~/demos/connexio-ops/services/connexio-ai/src/__init__.py
touch ~/demos/connexio-ops/services/connexio-ai/src/routers/__init__.py
touch ~/demos/connexio-ops/services/connexio-ai/tests/__init__.py
```

**Step 9: Install deps and run test — verify it passes**

```bash
cd ~/demos/connexio-ops/services/connexio-ai
uv sync
uv run pytest tests/test_health.py -v
```

Expected: `PASSED test_health_returns_ok`

**Step 10: Create Dockerfile**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Copy dependency files first (layer caching)
COPY pyproject.toml uv.lock* ./

# Install dependencies
RUN uv sync --no-dev --frozen

# Copy source
COPY src/ ./src/

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Step 11: Build Docker image — verify it builds**

```bash
cd ~/demos/connexio-ops/services/connexio-ai
docker build -t connexio-ai:dev .
```

Expected: `Successfully built ...` — image builds without errors

**Step 12: Commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "feat: connexio-ai FastAPI scaffold with health endpoint and Dockerfile"
```

---

## Task 6: Add connexio-ai to Docker Compose and run full stack

**Files:**
- Modify: `~/demos/connexio-ops/docker-compose.yml` (add connexio-ai service)
- Modify: `~/demos/connexio-ops/docker-compose.yml` (add connexio-api service)

**Step 1: Add Go API Dockerfile**

Create `~/demos/connexio-ops/Dockerfile.api`:

```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum* ./
RUN go mod download
COPY . .
RUN go build -o bin/connexio-api ./cmd/api/

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/bin/connexio-api .
EXPOSE 8080
CMD ["./connexio-api"]
```

**Step 2: Add application services to docker-compose.yml**

Append to the `services:` section in `~/demos/connexio-ops/docker-compose.yml`:

```yaml
  # ── connexio-api (Go) ────────────────────────────────────────
  connexio-api:
    build:
      context: .
      dockerfile: Dockerfile.api
    container_name: connexio-api
    environment:
      PORT: "8080"
      DATABASE_URL: postgresql://connexio:connexio_dev_password@postgres:5432/connexio
      REDIS_URL: redis://redis:6379
      NATS_URL: nats://nats:4222
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      nats:
        condition: service_healthy
    networks:
      - connexio-network

  # ── connexio-ai (Python/FastAPI) ─────────────────────────────
  connexio-ai:
    build:
      context: ./services/connexio-ai
      dockerfile: Dockerfile
    container_name: connexio-ai
    environment:
      DATABASE_URL: postgresql+asyncpg://connexio:connexio_dev_password@postgres:5432/connexio
      NATS_URL: nats://nats:4222
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_USER: neo4j
      NEO4J_PASSWORD: connexio_dev_password
      REDIS_URL: redis://redis:6379
      OLLAMA_HOST: http://host.docker.internal:11434
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      nats:
        condition: service_healthy
    networks:
      - connexio-network
```

**Step 3: Start full stack**

```bash
cd ~/demos/connexio-ops
docker-compose up -d
```

Expected: all services start (postgres, redis, nats, neo4j, zitadel, connexio-api, connexio-ai)

**Step 4: Verify connexio-api health**

```bash
curl -s http://localhost:8080/health
```

Expected: `{"status":"ok","service":"connexio-api"}`

**Step 5: Verify connexio-ai health**

```bash
curl -s http://localhost:8000/health
```

Expected: `{"status":"ok","service":"connexio-ai"}`

**Step 6: Verify all containers healthy**

```bash
docker-compose ps
```

Expected: all services `running` or `healthy`

**Step 7: Final commit**

```bash
cd ~/demos/connexio-ops
git add .
git commit -m "feat: full Phase 0 stack running — connexio-api + connexio-ai + all infra"
```

---

## Phase 0 Complete — Acceptance Criteria

Run this checklist to confirm Phase 0 is done:

```bash
# 1. All containers running
docker-compose ps | grep -v "Exit"

# 2. Go API healthy
curl -sf http://localhost:8080/health | grep '"status":"ok"'

# 3. Python AI service healthy
curl -sf http://localhost:8000/health | grep '"status":"ok"'

# 4. pgvector enabled
docker exec connexio-postgres psql -U connexio -d connexio \
  -c "SELECT extname FROM pg_extension WHERE extname='vector';" \
  | grep vector

# 5. NATS JetStream running
curl -sf http://localhost:8222/healthz | grep '"status":"ok"'

# 6. Neo4j responding
curl -sf http://localhost:7474 > /dev/null && echo "neo4j: ok"

# 7. Redis responding
docker exec connexio-redis redis-cli ping | grep PONG

# 8. Go tests passing
cd ~/demos/connexio-ops && go test ./... -v

# 9. Python tests passing
cd ~/demos/connexio-ops/services/connexio-ai && uv run pytest -v
```

All 9 checks passing = Phase 0 complete. Ready to begin Phase 1 (CAD Core + Kustwacht MVP).
