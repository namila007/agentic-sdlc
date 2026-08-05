# 03 — Artifact Library

Every durable output of every stage. Content-addressed, versioned, provenance-linked, attestable.

## Design principles

1. **Immutable content, mutable pointers.** Blobs are addressed by `sha256`. A "new version" is a new blob plus a new row; nothing is ever overwritten.
2. **Provenance is a first-class graph**, not a comment field. "What produced this, from what, approved by whom, under which prompt version" must be answerable in one query.
3. **Approved artifacts mirror to git.** The repo remains the durable, human-readable, offline-readable record. The database is the index and the workflow engine, not the only copy.
4. **Schema-validated on write.** A malformed artifact never enters the library.

---

## Storage layout

```
MinIO bucket: asdlc-artifacts
  <project_id>/blobs/<sha256[0:2]>/<sha256[2:4]>/<sha256>        ← content, immutable
  <project_id>/renders/<artifact_id>/<version>/preview.html      ← cached render for UI

Postgres:  metadata, versions, provenance edges, attestations
Git mirror (on approval):
  <repo>/docs/asdlc/<run_slug>/
    01-plan/prd.md
    01-plan/task-graph.json
    02-design/adr-014-oidc-session-store.md
    02-design/openapi.yaml
    ...
    manifest.json            ← artifact ids + hashes, for round-tripping
```

Content addressing gives free dedup: re-submitting an unchanged artifact writes no new blob, and identical ADRs across projects share storage.

---

## Schema

```sql
CREATE TYPE artifact_status AS ENUM
  ('draft','proposed','approved','changes_requested','rejected','superseded','archived');

CREATE TABLE artifacts (
  id                TEXT PRIMARY KEY,              -- art_01J8X... (ULID)
  project_id        TEXT NOT NULL REFERENCES projects(id),
  run_id            TEXT REFERENCES runs(id),
  stage             TEXT,                          -- plan|design|develop|review|test|document
  task_id           TEXT,                          -- set when produced by a fan-out task

  type              TEXT NOT NULL,                 -- prd|adr|task_graph|code_change|...
  name              TEXT NOT NULL,                 -- human label
  slug              TEXT NOT NULL,                 -- stable within (project, type)

  version           INT  NOT NULL DEFAULT 1,
  is_head           BOOLEAN NOT NULL DEFAULT TRUE, -- latest version of this slug
  supersedes_id     TEXT REFERENCES artifacts(id),

  status            artifact_status NOT NULL DEFAULT 'draft',

  -- content
  content_sha256    TEXT NOT NULL,
  content_uri       TEXT NOT NULL,                 -- s3://asdlc-artifacts/...
  mime              TEXT NOT NULL,
  size_bytes        BIGINT NOT NULL,
  format            TEXT,                          -- markdown|openapi-3.1|mermaid|json|diff
  schema_ref        TEXT,                          -- asdlc://schema/adr@1
  inline_preview    TEXT,                          -- first ~4KB, for cheap listing

  -- producer provenance
  produced_by_role     TEXT,                       -- architect
  produced_by_prompt   TEXT,                       -- architect@2.3.1
  produced_by_client   TEXT,                       -- claude-code/2.x | cursor/1.x | copilot
  produced_by_model    TEXT,                       -- self-reported, best-effort
  produced_by_user     TEXT,
  produced_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- approval
  approved_by       TEXT,
  approved_at       TIMESTAMPTZ,
  gate_event_id     TEXT REFERENCES gate_events(id),

  labels            JSONB NOT NULL DEFAULT '{}',
  metrics           JSONB NOT NULL DEFAULT '{}',   -- token_cost, latency_ms, retry_count
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX ON artifacts (project_id, type, slug, version);
CREATE INDEX ON artifacts (project_id, run_id, stage);
CREATE INDEX ON artifacts (project_id, type, status) WHERE is_head;
CREATE INDEX ON artifacts (content_sha256);

-- provenance DAG
CREATE TYPE edge_relation AS ENUM
  ('derived_from','supersedes','implements','tests','reviews','documents',
   'references','contradicts','blocked_by');

CREATE TABLE artifact_edges (
  from_artifact_id TEXT NOT NULL REFERENCES artifacts(id),
  to_artifact_id   TEXT NOT NULL REFERENCES artifacts(id),
  relation         edge_relation NOT NULL,
  confidence       REAL,          -- 1.0 when declared, <1 when inferred
  note             TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (from_artifact_id, to_artifact_id, relation)
);

CREATE TABLE artifact_attestations (
  id            TEXT PRIMARY KEY,
  artifact_id   TEXT NOT NULL REFERENCES artifacts(id),
  predicate     TEXT NOT NULL,     -- asdlc.dev/AgentProduced/v1
  statement     JSONB NOT NULL,    -- in-toto style
  signature     TEXT,              -- optional; cosign/minisign
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Versioning semantics

- `slug` is the stable identity ("adr-014-oidc-session-store"). `version` increments.
- `is_head` is maintained transactionally: writing v3 flips v2's `is_head` to false and sets v2 `status=superseded`.
- `artifact_get(slug)` returns head; `artifact_get(slug, version=2)` returns the pinned version.
- **Retrieval and stage briefs only ever use `is_head AND status='approved'`.** Draft artifacts are invisible to downstream stages by construction — this prevents a whole class of "agent built on unapproved work" bugs.

---

## Provenance graph

Edges are written automatically where the system knows the relationship, and declarable by the agent where it doesn't:

| Relation | Written by | Example |
|---|---|---|
| `derived_from` | automatic — every artifact links to its brief's `inputs` | ADR-014 derived_from PRD |
| `implements` | agent declares in `artifact_put` | code_change implements task T-7 / FR-3 |
| `tests` | agent declares | test_case TC-9 tests FR-3 |
| `reviews` | automatic | review_report reviews code_change[] |
| `supersedes` | automatic on version bump | ADR-021 supersedes ADR-014 |
| `contradicts` | reviewer declares | impl_note contradicts ADR-014 |
| `blocked_by` | agent declares | blocker blocked_by open question |

**Queries this makes cheap** (all one recursive CTE):

- *"Which requirement is this line of code traceable to?"* → walk `implements` backwards to `FR-*`.
- *"If we change ADR-014, what's affected?"* → walk `derived_from`/`implements` forward.
- *"Is every FR tested?"* → the coverage matrix, computed not eyeballed.
- *"Show the complete decision trail for this PR."* → the run's full subgraph, exportable for audit.

```sql
-- everything downstream of an artifact
WITH RECURSIVE downstream AS (
  SELECT to_artifact_id AS id, 1 AS depth
  FROM artifact_edges WHERE from_artifact_id = $1
  UNION
  SELECT e.to_artifact_id, d.depth + 1
  FROM artifact_edges e JOIN downstream d ON e.from_artifact_id = d.id
  WHERE d.depth < 12
)
SELECT a.* FROM artifacts a JOIN downstream d ON a.id = d.id;
```

---

## Attestations

Every approved artifact gets an in-toto-style statement. Not for supply-chain paranoia — for the practical question *"who steered this, and how was it checked?"*, which is where accountability lives when code is cheap to produce.

```jsonc
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{ "name": "adr-014-oidc-session-store.md",
                "digest": { "sha256": "9f2c..." } }],
  "predicateType": "https://asdlc.dev/AgentProduced/v1",
  "predicate": {
    "run": "run_01J8X...",
    "stage": "design",
    "agentRole": "architect",
    "promptVersion": "architect@2.3.1",
    "client": "cursor/1.7.3",
    "modelReported": "claude-opus-5",
    "inputs": [
      { "artifact": "art_01J8W...", "digest": "3ab1..." }
    ],
    "contextSources": {
      "knowledge": ["kn_0091", "kn_0104"],
      "code":      ["src/auth/session.ts@a91f3c"]
    },
    "automatedChecks": [
      { "id": "AC1", "name": "fr_traceability", "result": "pass" },
      { "id": "AC2", "name": "adr_schema",       "result": "pass" }
    ],
    "humanApproval": {
      "gateEvent": "gev_01J8Y...",
      "by": "namz", "at": "2026-08-05T09:14:22Z",
      "channel": "web", "comment": "Agreed on Redis; revisit TTL at scale."
    }
  }
}
```

Signing is optional (`ASDLC_ATTEST_SIGN=cosign|minisign|none`). Unsigned attestations are still valuable as an audit trail; signing matters when the artifact leaves your trust boundary.

---

## Git mirror

On gate approval, a worker job writes approved artifacts to a branch:

```
branch: asdlc/<run_slug>
  docs/asdlc/<run_slug>/<NN>-<stage>/<slug>.<ext>
  docs/asdlc/<run_slug>/manifest.json
```

`manifest.json` maps file paths → artifact IDs + hashes, so the repo can be re-imported into a fresh control plane (`asdlc import <repo>`). This is the disaster-recovery and portability story: **the database can be rebuilt from git.**

Configurable per project:

```yaml
git_mirror:
  enabled: true
  mode: branch            # branch | pr | commit_to_main
  path: docs/asdlc
  include_stages: [plan, design, review, test, document]   # 'develop' diffs usually excluded
  open_pr_on: [document]  # open the PR once docs are approved
```

---

## MCP tools

Full schemas in [07 — MCP tool surface](07-mcp-tool-surface.md). Summary:

| Tool | Purpose |
|---|---|
| `artifact_put` | Create or version an artifact. Validates against the active `output_contract` and `schema_ref`. Returns `artifact_id`, `version`, and validation errors if rejected. |
| `artifact_get` | Fetch by id or `(type, slug[, version])`. `inline` for text, presigned URL for binaries. |
| `artifact_list` | Filter by run/stage/type/status/label. Returns previews, not full content. |
| `artifact_diff` | Unified diff between two versions. Used by the approvals UI and by agents on `changes_requested`. |
| `artifact_link` | Declare a provenance edge. |
| `artifact_trace` | Walk the provenance graph up or down N hops. |
| `artifact_search` | Text/semantic search across the library, scoped to project. |

### Write-time validation

`artifact_put` rejects, with actionable messages, when:

1. The type isn't in the current stage's `output_contract`.
2. `max` for that type is already reached.
3. Content fails `schema_ref` validation (JSON Schema for structured, section-presence + front-matter checks for markdown, `openapi-spec-validator` for API contracts, mermaid parse for diagrams).
4. A declared `implements`/`tests` target doesn't exist or isn't approved.
5. Size exceeds `max_artifact_bytes` (default 8 MB; diffs above this should be a git ref instead).

Rejections return `{"ok": false, "errors": [{"path": "...", "message": "...", "fix": "..."}]}` — the `fix` field exists because agents repair far more reliably from a suggested correction than from a bare error.

---

## Retention

| Class | Default | Rationale |
|---|---|---|
| Approved artifacts | forever | Audit trail. |
| Draft artifacts, superseded ≥90d | archive to cold prefix, drop `inline_preview` | The bulk of storage; rarely read. |
| Rejected artifacts | 180 days | Useful for prompt improvement; not forever. |
| Renders/previews | 30 days, regenerable | Pure cache. |
| Attestations | forever | Small, and the point of the exercise. |

`asdlc gc --dry-run` reports what would be removed. Never run destructively by default.
