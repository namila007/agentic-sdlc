# 10 — Deployment

Single-host Docker Compose. Profiles for local models, observability, and air-gapped operation.

---

## Repo layout

```
agentic-sdlc/
├── README.md
├── docs/                          ← these documents
├── compose.yaml                   ← base stack
├── compose.local-llm.yaml         ← profile: ollama
├── compose.observability.yaml     ← profile: otel + grafana
├── .env.example
├── Makefile
├── services/
│   ├── gateway/      ← MCP facade (FastMCP)
│   ├── core/         ← runs, stages, gates, briefs, prompt registry
│   ├── artifacts/    ← store, versioning, provenance, attestations
│   ├── knowledge/    ← ingest, hybrid retrieval, curation
│   ├── indexer/      ← tree-sitter, embeddings, symbol graph
│   ├── worker/       ← arq jobs
│   └── ui/           ← Next.js approvals dashboard
├── packs/            ← agent role definitions (source of truth)
├── cli/              ← asdlc CLI
├── migrations/       ← alembic
├── config/
│   ├── litellm.yaml
│   ├── qdrant.yaml
│   └── policies/     ← per-project policy.yaml
└── ops/
    ├── backup.sh  restore.sh  seed.sh
    └── airgap/export.sh  import.sh
```

---

## compose.yaml

```yaml
name: asdlc

x-svc-common: &svc-common
  restart: unless-stopped
  env_file: [.env]
  networks: [asdlc]
  logging:
    driver: json-file
    options: { max-size: "10m", max-file: "3" }

services:
  # ── entry point ────────────────────────────────────────────────
  gateway:
    <<: *svc-common
    build: ./services/gateway
    ports: ["${ASDLC_PORT:-8080}:8080"]
    environment:
      CORE_URL: http://core:8000
      ARTIFACT_URL: http://artifacts:8000
      KNOWLEDGE_URL: http://knowledge:8000
      INDEXER_URL: http://indexer:8000
      AUTH_MODE: ${ASDLC_AUTH_MODE:-token}
    depends_on:
      core: { condition: service_healthy }
      artifacts: { condition: service_healthy }
      knowledge: { condition: service_healthy }
      indexer: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]
      interval: 15s
      timeout: 3s
      retries: 5

  # ── domain services ────────────────────────────────────────────
  core:
    <<: *svc-common
    build: ./services/core
    environment:
      DATABASE_URL: postgresql+asyncpg://asdlc:${POSTGRES_PASSWORD}@postgres:5432/asdlc
      REDIS_URL: redis://redis:6379/0
      LLM_GATEWAY_URL: http://llm-gateway:4000
      KNOWLEDGE_URL: http://knowledge:8000
      INDEXER_URL: http://indexer:8000
      ARTIFACT_URL: http://artifacts:8000
    volumes: ["./packs:/app/packs:ro", "./config/policies:/app/policies:ro"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    healthcheck: &py-health
      test: ["CMD", "python", "-c", "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/healthz').status==200 else 1)"]
      interval: 15s
      timeout: 3s
      retries: 5

  artifacts:
    <<: *svc-common
    build: ./services/artifacts
    environment:
      DATABASE_URL: postgresql+asyncpg://asdlc:${POSTGRES_PASSWORD}@postgres:5432/asdlc
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET: asdlc-artifacts
      S3_ACCESS_KEY: ${MINIO_ROOT_USER}
      S3_SECRET_KEY: ${MINIO_ROOT_PASSWORD}
      ATTEST_SIGN: ${ASDLC_ATTEST_SIGN:-none}
    volumes: ["${REPOS_PATH:-./repos}:/repos"]
    depends_on:
      postgres: { condition: service_healthy }
      minio: { condition: service_healthy }
    healthcheck: *py-health

  knowledge:
    <<: *svc-common
    build: ./services/knowledge
    environment:
      DATABASE_URL: postgresql+asyncpg://asdlc:${POSTGRES_PASSWORD}@postgres:5432/asdlc
      REDIS_URL: redis://redis:6379/1
      QDRANT_URL: http://qdrant:6333
      EMBEDDING_BASE_URL: http://llm-gateway:4000/v1
      EMBEDDING_MODEL: ${EMBEDDING_MODEL:-embed}
      RERANK_ENABLED: ${RERANK_ENABLED:-false}
      CONTEXT7_API_KEY: ${CONTEXT7_API_KEY:-}
    depends_on:
      postgres: { condition: service_healthy }
      qdrant: { condition: service_healthy }
      redis: { condition: service_healthy }
    healthcheck: *py-health

  indexer:
    <<: *svc-common
    build: ./services/indexer
    environment:
      DATABASE_URL: postgresql+asyncpg://asdlc:${POSTGRES_PASSWORD}@postgres:5432/asdlc
      REDIS_URL: redis://redis:6379/2
      QDRANT_URL: http://qdrant:6333
      EMBEDDING_BASE_URL: http://llm-gateway:4000/v1
      EMBEDDING_MODEL: ${EMBEDDING_MODEL:-embed}
    volumes: ["${REPOS_PATH:-./repos}:/repos:ro"]
    depends_on:
      postgres: { condition: service_healthy }
      qdrant: { condition: service_healthy }
    healthcheck: *py-health

  worker:
    <<: *svc-common
    build: ./services/worker
    command: ["arq", "worker.WorkerSettings"]
    environment:
      DATABASE_URL: postgresql+asyncpg://asdlc:${POSTGRES_PASSWORD}@postgres:5432/asdlc
      REDIS_URL: redis://redis:6379/3
      QDRANT_URL: http://qdrant:6333
      S3_ENDPOINT: http://minio:9000
      LLM_GATEWAY_URL: http://llm-gateway:4000
      GIT_TOKEN: ${GIT_TOKEN:-}
    volumes: ["${REPOS_PATH:-./repos}:/repos"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }

  ui:
    <<: *svc-common
    build: ./services/ui
    ports: ["${UI_PORT:-3000}:3000"]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:${ASDLC_PORT:-8080}
      CORE_URL: http://core:8000
      AUTH_MODE: ${ASDLC_AUTH_MODE:-token}
      OIDC_ISSUER: ${OIDC_ISSUER:-}
    depends_on: [gateway]

  # ── LLM gateway ────────────────────────────────────────────────
  llm-gateway:
    <<: *svc-common
    image: ghcr.io/berriai/litellm:main-stable
    ports: ["${LITELLM_PORT:-4000}:4000"]
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    volumes: ["./config/litellm.yaml:/app/config.yaml:ro"]
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      DATABASE_URL: postgresql://asdlc:${POSTGRES_PASSWORD}@postgres:5432/litellm
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }

  embeddings:
    <<: *svc-common
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.8
    command: ["--model-id", "${EMBEDDING_HF_MODEL:-BAAI/bge-m3}", "--port", "8081"]
    volumes: ["hf-cache:/data"]
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8081/health"]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 180s     # first run downloads the model

  # ── data ───────────────────────────────────────────────────────
  postgres:
    <<: *svc-common
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: asdlc
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: asdlc
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./ops/init-db.sql:/docker-entrypoint-initdb.d/10-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U asdlc -d asdlc"]
      interval: 10s
      timeout: 3s
      retries: 10

  qdrant:
    <<: *svc-common
    image: qdrant/qdrant:latest
    volumes:
      - qdrantdata:/qdrant/storage
      - ./config/qdrant.yaml:/qdrant/config/production.yaml:ro
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY}
    healthcheck:
      test: ["CMD-SHELL", "bash -c ':> /dev/tcp/127.0.0.1/6333' || exit 1"]
      interval: 15s
      timeout: 3s
      retries: 10

  minio:
    <<: *svc-common
    image: minio/minio:latest
    command: ["server", "/data", "--console-address", ":9001"]
    ports: ["${MINIO_CONSOLE_PORT:-9001}:9001"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes: [miniodata:/data]
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 15s
      timeout: 5s
      retries: 10

  redis:
    <<: *svc-common
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "${REDIS_MAXMEM:-1gb}",
              "--maxmemory-policy", "allkeys-lru"]
    volumes: [redisdata:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 10

volumes:
  pgdata: {}
  qdrantdata: {}
  miniodata: {}
  redisdata: {}
  hf-cache: {}

networks:
  asdlc:
    driver: bridge
```

**Note on Redis `allkeys-lru`:** the queue lives in db 3 and must not be evicted. Either run a second Redis for the queue, or set `maxmemory-policy volatile-lru` and ensure only cache keys carry TTLs. The safer choice for production is a separate `redis-queue` service with `noeviction`.

---

## Profiles

### `local-llm` — air-gapped / offline generation

```yaml
# compose.local-llm.yaml
services:
  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    volumes: [ollamadata:/root/.ollama]
    networks: [asdlc]
    # uncomment for GPU:
    # deploy: { resources: { reservations: { devices: [{ capabilities: [gpu] }] } } }
volumes: { ollamadata: {} }
```

```bash
docker compose -f compose.yaml -f compose.local-llm.yaml up -d
docker compose exec ollama ollama pull qwen3:14b
docker compose exec ollama ollama pull nomic-embed-text
```

Then point `config/litellm.yaml` at `http://ollama:11434/v1`. Nothing else changes — that indirection is the main payoff of running a gateway.

### `observability`

```yaml
# compose.observability.yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes: ["./config/otel.yaml:/etc/otelcol-contrib/config.yaml:ro"]
    networks: [asdlc]
  grafana:
    image: grafana/grafana-oss:latest
    ports: ["${GRAFANA_PORT:-3001}:3000"]
    volumes: [grafanadata:/var/lib/grafana, "./ops/dashboards:/etc/grafana/provisioning/dashboards:ro"]
    networks: [asdlc]
volumes: { grafanadata: {} }
```

Ship three dashboards from day one: **spend by stage/role**, **cache hit rates across L1–L3**, and **gate latency** (how long humans take). The third is the one that tells you whether the workflow is actually usable.

---

## .env.example

```bash
# ── core ──────────────────────────────────────────────────────────
ASDLC_PORT=8080
UI_PORT=3000
ASDLC_AUTH_MODE=token            # none | token | oidc
ASDLC_ALLOW_IDE_APPROVAL=false   # true for solo use
ASDLC_ATTEST_SIGN=none           # none | minisign | cosign
REPOS_PATH=./repos

# ── secrets (generate: openssl rand -hex 32) ──────────────────────
POSTGRES_PASSWORD=
QDRANT_API_KEY=
MINIO_ROOT_USER=asdlc
MINIO_ROOT_PASSWORD=
LITELLM_MASTER_KEY=sk-

# ── models ────────────────────────────────────────────────────────
EMBEDDING_MODEL=embed
EMBEDDING_HF_MODEL=BAAI/bge-m3
RERANK_ENABLED=false
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# ── integrations (all optional) ───────────────────────────────────
GIT_TOKEN=
CONTEXT7_API_KEY=
SLACK_WEBHOOK_URL=
OIDC_ISSUER=
```

---

## Makefile

```makefile
up:            ; docker compose up -d
down:          ; docker compose down
logs:          ; docker compose logs -f --tail=100
migrate:       ; docker compose run --rm core alembic upgrade head
seed:          ; ./ops/seed.sh
backup:        ; ./ops/backup.sh
restore:       ; ./ops/restore.sh $(SNAPSHOT)
doctor:        ; docker compose exec gateway python -m asdlc.doctor
local-llm:     ; docker compose -f compose.yaml -f compose.local-llm.yaml up -d
observability: ; docker compose -f compose.yaml -f compose.observability.yaml up -d
airgap-export: ; ./ops/airgap/export.sh
```

---

## First run

```bash
git clone https://github.com/namz/agentic-sdlc && cd agentic-sdlc
cp .env.example .env
./ops/gen-secrets.sh          # fills the blank secrets in .env
make up                       # ~3 min first time (TEI downloads bge-m3)
make migrate
make doctor                   # verifies every dependency

# register a project
docker compose exec core asdlc project create --id acme-api --name "Acme API"
docker compose exec core asdlc repo add --project acme-api --name api --path /repos/acme-api
docker compose exec core asdlc index sync --project acme-api --full
docker compose exec core asdlc project init --project acme-api   # bootstrap knowledge

# issue a token
docker compose exec core asdlc token create --project acme-api --role developer
```

---

## Air-gapped operation

Everything except Context7 passthrough and hosted model APIs runs offline.

**Preparation on a connected machine:**

```bash
./ops/airgap/export.sh --out asdlc-bundle.tar
# bundles: all service images, postgres/qdrant/minio/redis/litellm/TEI images,
#          the bge-m3 model weights, ollama model blobs, python/npm wheels,
#          tree-sitter grammars (pre-compiled), and optionally a
#          Context7 docs snapshot for the pinned libraries
```

**On the air-gapped host:**

```bash
./ops/airgap/import.sh asdlc-bundle.tar
docker compose -f compose.yaml -f compose.local-llm.yaml up -d
```

**Configuration deltas for air-gap:**

```bash
EMBEDDING_HF_MODEL=BAAI/bge-m3       # pre-baked into the image, no download
ASDLC_OFFLINE=true                   # disables all outbound fetchers
```
```yaml
# config/litellm.yaml
model_list:
  - model_name: default
    litellm_params: { model: openai/qwen3:14b, api_base: http://ollama:11434/v1, api_key: none }
  - model_name: embed
    litellm_params: { model: openai/bge-m3, api_base: http://embeddings:8081/v1, api_key: none }
external_docs: { provider: local_mirror }
```

`ASDLC_OFFLINE=true` makes URL ingestion, Context7 passthrough, git-host webhooks, and hosted-model routing hard-fail with clear messages rather than hanging on DNS. Fail fast beats mysterious timeouts.

---

## Resource sizing

| Scale | vCPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Solo, 1 repo ≤50k LOC | 4 | 8 GB | 40 GB | TEI on CPU is fine. |
| Small team, 3–5 repos ≤500k LOC | 8 | 16 GB | 200 GB | Comfortable. Consider `RERANK_ENABLED=true`. |
| Team + local LLM | 8+ | 32 GB | 400 GB | Ollama 14B needs ~10 GB; a GPU changes everything. |

Rough RAM split at the middle tier: Postgres 2 GB, Qdrant 2 GB, TEI 2 GB, services 3 GB, Redis 1 GB, headroom 6 GB.

---

## Backup & restore

```bash
# ops/backup.sh — atomic-ish snapshot
TS=$(date +%Y%m%dT%H%M%S); D=backups/$TS; mkdir -p "$D"
docker compose exec -T postgres pg_dump -U asdlc -Fc asdlc          > "$D/asdlc.dump"
docker compose exec -T postgres pg_dump -U asdlc -Fc litellm        > "$D/litellm.dump"
curl -sX POST "http://localhost:6333/collections/knowledge/snapshots" -H "api-key: $QDRANT_API_KEY"
curl -sX POST "http://localhost:6333/collections/code/snapshots"      -H "api-key: $QDRANT_API_KEY"
docker compose exec -T minio mc mirror --overwrite local/asdlc-artifacts "/backup/$TS/artifacts"
tar czf "backups/$TS.tar.gz" -C backups "$TS" && rm -rf "$D"
```

**What actually needs backing up:**

| Store | Critical? | Rebuildable? |
|---|---|---|
| Postgres | **Yes** | Only from the git mirror (`asdlc import`) — lossy for drafts. |
| MinIO | **Yes** | Approved artifacts are in git; drafts are not. |
| Qdrant | No | Fully rebuildable via `asdlc index sync --full` + knowledge re-embed. |
| Redis | No | Pure cache and queue. |

So: back up Postgres and MinIO on a schedule; treat Qdrant and Redis as regenerable. That halves your backup size and simplifies restore ordering.

**Restore drill.** Run `make restore SNAPSHOT=<ts>` against a scratch host quarterly. An untested backup is a hypothesis.

---

## Scaling beyond one host

When you outgrow Compose, the order to break things out is:

1. **Postgres → managed** (RDS/Cloud SQL). Highest operational value, least code change.
2. **Qdrant → its own host or Qdrant Cloud.** Vector memory is what fills the box first.
3. **MinIO → real S3.** Config-only change; the code already speaks S3.
4. **gateway + core → N replicas behind a load balancer.** Both are stateless by design (this is why the MCP transport being stateless matters).
5. **worker → scale horizontally by queue.** Split index/ingest/attest queues so a big reindex doesn't starve gate notifications.

The service boundaries in this design exist mostly so that this sequence is possible without a rewrite. Don't do any of it before you need to.
