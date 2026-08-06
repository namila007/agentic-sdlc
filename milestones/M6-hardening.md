# M6 — Hardening

**Week 20+** · [← Master plan](MILESTONES.md) · Implements
[`docs/11-security-and-multitenancy.md`](../docs/11-security-and-multitenancy.md) and the operational
half of [`docs/10-deployment.md`](../docs/10-deployment.md)

> **Goal:** survives a security review, a restore drill, and 10 concurrent runs.

---

## The threat this system actually has

Not a generic web-app threat model. The specific risk profile is:

> **Untrusted text — retrieved knowledge, code comments, ticket bodies — fed into an agent that has
> file-write and shell access on a developer's machine.**

Everything in M6 either narrows that, or narrows the blast radius when it fails.

```
┌──────────────────────────── TRUST BOUNDARY ─────────────────────────────┐
│                                                                          │
│  DEVELOPER MACHINE                          CONTROL PLANE                │
│  ┌────────────────────┐                    ┌────────────────────────┐   │
│  │ agent w/ file-write│◀── stage brief ────│ brief builder           │   │
│  │ + shell access     │                    │  · wraps retrieved      │   │
│  │                    │                    │    content in           │   │
│  │ TRUSTED: role      │                    │    <untrusted-context>  │   │
│  │   prompt, contract │                    │  · tools_allowed comes  │   │
│  │ UNTRUSTED: every-  │                    │    from role, NEVER     │   │
│  │   thing inside     │                    │    from retrieved text  │   │
│  │   <untrusted-      │                    └───────────┬─────────────┘   │
│  │   context>         │                                │                 │
│  └────────────────────┘                    ┌───────────▼─────────────┐   │
│           │                                │ knowledge · code index   │   │
│           │ artifact_put                   │ ← human-approved only    │   │
│           ▼                                │ ← secret-scanned         │   │
│  ┌────────────────────┐                    └──────────────────────────┘  │
│  │ secret scan        │                                                   │
│  │ contract validate  │           ┌──────────────────────┐               │
│  └────────────────────┘           │ egress proxy         │               │
│                                   │ allow-list only      │──▶ internet   │
└───────────────────────────────────┴──────────────────────┴───────────────┘
```

---

## Sub-milestones

| ID | Name | Depends on | Days |
|---|---|---|---|
| [M6.1](#m61--oidc-and-row-level-security) | OIDC & row-level security | M1.8 | 3 |
| [M6.2](#m62--secret-scanning-on-all-three-paths) | Secret scanning on all three paths | M1.4, M2.2, M3.2 | 2.5 |
| [M6.3](#m63--egress-allow-list-and-network-posture) | Egress allow-list & network posture | M4.1 | 2 |
| [M6.4](#m64--attestation-signing-and-verification) | Attestation signing & verification | M1.3 | 2 |
| [M6.5](#m65--air-gap-export-and-import) | Air-gap export & import | M4.1 | 3 |
| [M6.6](#m66--backup-restore-drill-and-project-purge) | Backup, restore drill & project purge | M1.2 | 2.5 |
| [M6.7](#m67--observability-profile) | Observability profile | M4.6 | 2 |
| [M6.8](#m68--load-test) | Load test | all above | 2 |

---

## M6.1 — OIDC and row-level security

### Auth modes complete

| Mode | Behaviour |
|---|---|
| `none` | No auth. Gateway hard-fails if bound to a non-loopback address. |
| `token` | M1.8 — Bearer tokens, project+role scoped, argon2id at rest |
| `oidc` | **New.** OAuth2/OIDC to an IdP; roles from claims/groups; short-lived tokens. |

OIDC config: `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`. Claims mapping: `sub → subject`;
`groups`/`roles` claim → system roles; optional `project` claim → project assignment. The ID token
establishes the session, the access token is refreshed server-side, and logout revokes the refresh
token.

### RLS — the backstop, not the primary defence

```sql
ALTER TABLE artifacts ENABLE ROW LEVEL SECURITY;
CREATE POLICY artifacts_tenant ON artifacts
  USING (project_id = current_setting('asdlc.project_id', true));
```

Applied to every multi-tenant table: `artifacts`, `runs`, `stages`, `gates`, `gate_events`,
`knowledge_entries`, `knowledge_chunks`, `symbols`, `symbol_refs`, `audit_log`, `tokens`.

Each request sets `SET LOCAL asdlc.project_id = '<id>'` **inside the transaction** before any query.

**RLS catches the query that forgot its `WHERE project_id = ?`.** The `ScopedRepo` pattern (M0.2)
remains the primary defence — RLS is defence in depth, and it is why the invariant was established in
M0 rather than here.

### Testing tenancy properly

Not "does filtering work" but "can I break out":

- Two projects with overlapping slugs and titles; assert zero cross-visibility on every endpoint
- A token for project A with `Mcp-Project: B` → `PROJECT_SCOPE_DENIED`
- Deliberately write a repository method missing the project filter → assert RLS blocks it anyway
- Qdrant search with a hand-crafted payload filter for another project → assert the service layer overrides it
- A presigned URL for project A's blob used with project B's session → denied

### Acceptance

- [ ] All five tenancy break-out tests pass
- [ ] RLS enabled on every table carrying `project_id` — asserted by an `information_schema` test
- [ ] OIDC login maps groups to roles correctly
- [ ] A missing `SET LOCAL` causes queries to return zero rows rather than all rows

---

## M6.2 — Secret scanning on all three paths

Three entry points, all of which must scan:

| Path | When | On hit |
|---|---|---|
| **`artifact_put`** | Every write | Reject + `fix` message + `secret_blocked` audit event. **Content never stored.** |
| **Indexer** | Every file considered for indexing | Skip the file, log the path only |
| **Ingestion** | Every knowledge source | Reject, log the source ref |

### Deny-list (path-based, not overridable per project)

```
.env*        *.pem         *.key      *.p12
id_rsa*      credentials*  *.kdbx     secrets.*
(anything gitignored AND matching a secret pattern)
```

**Not overridable per project**, on purpose. "We need to index our .env for context" is never the
right answer, and making it configurable guarantees someone eventually configures it.

### Content patterns

Gitleaks-style rules for provider keys, tokens and private-key headers. Plus an entropy heuristic:
**> 4.0 bits/char over ≥20 chars** in non-binary files — **warn-only**, because false positives on
base64 assets and test fixtures are common and hard-failing on them would make the system unusable.

### Logging discipline

The audit event records that a secret was blocked, its type, and the artifact or file — **never the
matched content**. Logging the secret you just blocked is the classic own-goal here.

### Acceptance

- [ ] A PRD containing an AWS key is rejected; the key appears in no log or database row
- [ ] A `.env` in the repo is never indexed
- [ ] Entropy warnings do not block
- [ ] `secret_blocked` events are queryable and dashboarded

---

## M6.3 — Egress allow-list and network posture

### Ports

```
Exposed externally:      8080 gateway · 3000 UI
Bound to 127.0.0.1:      4000 LiteLLM · 9001 MinIO console
Compose network only:    6333 Qdrant · 5432 Postgres · 6379 Redis · 9000 MinIO API · 8081 TEI
```

Production: a TLS-terminating reverse proxy (Caddy or Traefik) in front of gateway and UI.

Headers on every response: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`,
`X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`,
`Permissions-Policy: camera=(), microphone=(), geolocation=()`. Plus a nonce-based CSP on the UI —
which matters precisely because the UI renders agent-generated markdown.

### Egress proxy

```yaml
HTTP_PROXY:  http://egress-proxy:3128
HTTPS_PROXY: http://egress-proxy:3128
```

Allow-list: model provider APIs, the git host, Context7 (if enabled). **Everything else hard-fails
with a clear error.**

This is the containment for agent hijacking. If a poisoned knowledge entry convinces an agent to
exfiltrate, there is nowhere to send it. It also makes accidental dependency-driven callouts visible
rather than silent.

### Acceptance

- [ ] A service attempting a non-allow-listed host fails with a clear error naming the host
- [ ] Only 8080 and 3000 are reachable from outside the host
- [ ] CSP blocks inline script in a rendered artifact
- [ ] Security headers present on every response

---

## M6.4 — Attestation signing and verification

`ASDLC_ATTEST_SIGN`:

| Value | Algorithm | Key management |
|---|---|---|
| `none` | — | Content hash only (M1.3 default) |
| `minisign` | Ed25519 | Public key in policy; private key in artifact-svc env |
| `cosign` | ECDSA P-256 | KMS or local key; Sigstore optional |

```bash
asdlc verify --artifact <id>     # recomputes hash, checks against DB, validates signature
asdlc verify --run <id>          # whole run
```

### The threat this addresses

An attacker with repo write access edits `docs/asdlc/**` to forge an approval record. The mitigation
is that **the git mirror is a copy, never the authority** — attestations record hashes, and
`asdlc verify` recomputes them against the database.

Signing matters specifically when an artifact leaves the trust boundary: shipped to an auditor,
handed to a customer, used as compliance evidence. For purely internal use, unsigned attestations
plus the append-only audit log are already a real trail.

### Acceptance

- [ ] `asdlc verify` detects a tampered artifact in the git mirror
- [ ] Signature verification passes for signed attestations and fails on a modified statement
- [ ] Key rotation does not invalidate previously signed attestations

---

## M6.5 — Air-gap export and import

**Q7 decides how much this milestone matters.** If air-gap is a real requirement rather than a
nice-to-have, this moves much earlier — and the honest consequence is that local models become the
*primary* configuration, which changes what the system can promise about artifact quality.

```bash
./ops/airgap/export.sh --out asdlc-bundle.tar     # on a connected machine
./ops/airgap/import.sh asdlc-bundle.tar           # on the air-gapped host
docker compose -f compose.yaml -f compose.local-llm.yaml up -d
```

### Bundle contents

Service images · base images (postgres, qdrant, minio, redis, TEI, ollama) · bge-m3 weights ·
Ollama model blobs · Python wheels and `node_modules` · pre-compiled tree-sitter grammars ·
optionally a Context7 docs snapshot for pinned libraries.

### `ASDLC_OFFLINE=true`

Hard-fails **fast** on every outbound path — URL ingestion, Context7 passthrough, git webhooks,
hosted model routing. Fast failure with a clear message, not a DNS timeout. A 30-second hang with no
explanation is the worst possible air-gap experience.

### Config deltas

```yaml
# config/litellm.yaml
model_list:
  - { model_name: default, litellm_params: { model: openai/qwen3:14b, api_base: http://ollama:11434/v1, api_key: none } }
  - { model_name: embed,   litellm_params: { model: openai/bge-m3,    api_base: http://embeddings:8081/v1, api_key: none } }
external_docs: { provider: local_mirror }
```

**Test on a genuinely disconnected host.** Not a host with a firewall rule — one with no network
interface. Every air-gap bug found any other way is a bug found in production.

### Acceptance

- [ ] Bundle imports and reaches a working stack on a truly disconnected machine
- [ ] `ASDLC_OFFLINE=true` fails fast with named endpoints on every outbound path
- [ ] A complete run finishes end to end with no network
- [ ] Bundle size and import time documented

---

## M6.6 — Backup, restore drill and project purge

### What actually needs backing up

| Store | Critical? | Rebuildable? | Frequency |
|---|---|---|---|
| Postgres (`asdlc`, `litellm`) | **Yes** | Only from the git mirror, lossily | Daily |
| MinIO (`asdlc-artifacts/*`) | **Yes** | Approved artifacts are in git; drafts are not | Daily |
| Qdrant | No | `asdlc index sync --full` + re-embed | Never — regenerate on restore |
| Redis | No | Pure cache and queue | Never |

Backing up Qdrant is a common instinct and a waste — it is derived data, and restoring a stale vector
index alongside fresh Postgres state produces subtle inconsistency.

### The drill

```bash
make backup
# then, on a scratch host:
make restore SNAPSHOT=<ts>
asdlc index sync --full
make doctor
```

**Run this quarterly on a scratch host.** An untested backup is a hypothesis. Record the wall-clock
restore time — that number is your actual RTO, and it is usually longer than anyone guesses.

### Project purge — the right-to-deletion path

```bash
asdlc project purge --project <id>
```

Removes: all rows by `project_id` · all S3 blobs under the prefix · all vectors by payload filter ·
all cache keys by prefix · all audit entries for the project.

**Test this before you need it.** A purge that half-works leaves orphaned vectors that still surface
in retrieval — both a correctness bug and a compliance failure.

### Acceptance

- [ ] Full restore drill completed on a scratch host; RTO recorded
- [ ] Qdrant rebuilds from Postgres + repos with no data loss
- [ ] `asdlc project purge` leaves zero traces — verified by direct queries against every store
- [ ] Backup script runs unattended and alerts on failure

---

## M6.7 — Observability profile

```bash
docker compose -f compose.yaml -f compose.observability.yaml up -d
```

Adds `otel-collector` and `grafana` (:3001).

### The three dashboards

| Dashboard | Panels |
|---|---|
| **Spend by stage/role** | Cost per run · per stage · per role · budget vs. actual |
| **Cache hit rates L0–L3** | Per-tier hit rate · **L2 false-positive rate** · latency correlated with hits |
| **Gate latency** | Submit → decision, per stage, per approver · SLA breaches |

**Gate latency is the usability metric.** If it degrades, adoption degrades, regardless of how well
everything else works. It deserves a dashboard of its own rather than a panel on someone else's.

### Log hygiene

Never log artifact bodies, retrieved chunks, prompts or completions at INFO. Hashes and IDs only.

Prompt-level debugging sits behind `ASDLC_DEBUG_PROMPTS=true`, with a loud startup warning and a
**24-hour auto-disable**. The auto-disable exists because debug flags left on are how prompt content
ends up in a log aggregator six months later.

### Audit log

```sql
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL DEFAULT now(),
  project_id TEXT,
  actor TEXT NOT NULL, actor_type TEXT NOT NULL,   -- human|agent|system
  token_prefix TEXT,                                -- sk-asdlc-a1b2, never the full token
  action TEXT NOT NULL,                             -- artifact.put|gate.decide|knowledge.approve|
                                                    -- token.create|secret_blocked|index.sync
  resource TEXT, outcome TEXT NOT NULL,             -- ok|denied|error
  detail JSONB, ip INET, client TEXT
);
CREATE INDEX ON audit_log (project_id, ts DESC);
CREATE INDEX ON audit_log (action, ts DESC);
```

Append-only. Retention 2 years by default.

### Acceptance

- [ ] Three dashboards provisioned and populated
- [ ] No prompt or artifact content in logs at INFO — asserted by a log-scanning test
- [ ] `ASDLC_DEBUG_PROMPTS` auto-disables after 24 hours
- [ ] Every denial (auth, rate limit, RLS) produces an audit row

---

## M6.8 — Load test

**Target: 10 concurrent runs · 5 repos · 1M LOC.**

| Scenario | Measures |
|---|---|
| 10 concurrent runs across 3 projects | Gateway throughput, DB contention, gate correctness under concurrency |
| Full reindex of 1M LOC while runs are active | Worker queue starvation, embedding backpressure |
| 50 concurrent `code_search` calls | p95 latency under load, L3 effectiveness |
| Gateway replica failover mid-run | Whether statelessness actually holds |
| Budget exhaustion during a run | Graceful `BUDGET_EXCEEDED`, not a crash |

### Watch for

- **Worker starvation** — a full reindex must not block git-mirror jobs. Split the queues:
  index / ingest / attest / git.
- **Connection-pool exhaustion** under concurrent runs
- **Qdrant memory** at 1M LOC across 5 repos
- **Task-claim lease expiry** during a slow develop fan-out

### Scaling path, in order of operational value

1. Postgres → managed (RDS / Cloud SQL) — highest value, least code change
2. Qdrant → separate host or Qdrant Cloud — vector memory fills the box first
3. MinIO → real S3 — config only; the code already speaks S3
4. gateway + core → N replicas behind a load balancer — both stateless by design
5. worker → horizontal, with queues split by job type

### Acceptance

- [ ] All five scenarios pass without data loss or incorrect gate state
- [ ] Resource ceilings documented for each sizing tier
- [ ] Replica failover mid-run is transparent to the client
- [ ] Split queues prevent a reindex from starving git-mirror jobs

---

## M6 exit criteria

- [ ] OIDC works; RLS applied and **break-out tested**, not merely enabled
- [ ] Secret scanning active on all three paths, with no secret ever stored or logged
- [ ] Egress allow-list enforced; a non-allow-listed host fails clearly
- [ ] Attestation signing works; `asdlc verify` detects tampering
- [ ] Air-gap bundle tested on a genuinely disconnected host
- [ ] Backup/restore drill completed with RTO recorded; `project purge` verified clean
- [ ] Three observability dashboards live
- [ ] Load test passes at 10 concurrent runs / 5 repos / 1M LOC

## Security review checklist

- [ ] `ASDLC_AUTH_MODE` is not `none` in team mode
- [ ] Gateway not bound to `0.0.0.0` without auth (hard-fails if attempted)
- [ ] Reverse proxy with TLS in front
- [ ] Egress proxy allow-list configured
- [ ] `ops/gen-secrets.sh` run; no default passwords anywhere in the repo
- [ ] `.env` not committed; `chmod 600`
- [ ] Two tokens per developer — `agent` in `.mcp.json`, `approver` in the browser only
- [ ] Token rotation scheduled (quarterly)
- [ ] Provider API keys exist only in the LiteLLM container
- [ ] `secret_blocked` events reviewed regularly
- [ ] Audit log exported to SIEM if required
