# Data Model
## Agentic Campaign Orchestrator

**Phase:** architecture · **Iteration:** 1 · **Authored:** 2026-08-02  
**Inputs:** `decision-registry.yaml`, `module-map.yaml`, `data-readiness.yaml`, `data-schema-snapshot.yaml`

---

## 1. Overview

The data layer is split across three store types, each chosen for its access pattern:

| Store | Service | Purpose |
|---|---|---|
| **Relational** | Aurora Serverless PostgreSQL | Campaign metrics (time-series), tenant configs, brand hard rules; SQL aggregation |
| **Vector** | OpenSearch Serverless | Brand guidelines RAG corpus; hybrid k-NN + BM25 retrieval |
| **Document** | DynamoDB (4 tables) | Session state, immutable action audit, human confirmations, PII audience |
| **Agent Memory** | Bedrock AgentCore Memory | EPISODIC + SEMANTIC cross-run campaign learnings |

All stores enforce **tenant isolation**. No cross-tenant data access is possible by design. Greenfield project — no migration from prior schema.

---

## 2. Aurora Serverless PostgreSQL Schema

### 2.1 tenants

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    subscription_tier TEXT NOT NULL CHECK (subscription_tier IN ('observer','copilot','autopilot')),
    cognito_group   TEXT NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT now(),
    deleted_at      TIMESTAMPTZ,          -- soft-delete (GDPR erasure)
    eu_data_residency BOOLEAN DEFAULT FALSE
);
```

### 2.2 campaigns

```sql
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    network         TEXT NOT NULL CHECK (network IN ('google_ads','meta')),
    external_id     TEXT NOT NULL,        -- ad network campaign ID
    name            TEXT NOT NULL,
    status          TEXT DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- RLS
ALTER TABLE campaigns ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON campaigns
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### 2.3 ad_sets

```sql
CREATE TABLE ad_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    campaign_id     UUID NOT NULL REFERENCES campaigns(id),
    external_id     TEXT NOT NULL,
    name            TEXT NOT NULL,
    network         TEXT NOT NULL,
    status          TEXT DEFAULT 'active',
    spend_cap_usd   DECIMAL(12,2),
    created_at      TIMESTAMPTZ DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

ALTER TABLE ad_sets ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON ad_sets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### 2.4 campaign_metrics_timeseries

High-write table — one row per (ad_set_id, poll_cycle). Partitioned by `ts` for efficient 7-day window queries.

```sql
CREATE TABLE campaign_metrics_timeseries (
    id              BIGSERIAL,
    tenant_id       UUID NOT NULL,
    ad_set_id       UUID NOT NULL REFERENCES ad_sets(id),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    cpc             DECIMAL(10,4),        -- cost per click (USD)
    ctr             DECIMAL(6,4),         -- click-through rate (0.0–1.0)
    roas            DECIMAL(10,4),        -- return on ad spend
    impressions     BIGINT,
    clicks          BIGINT,
    spend_usd       DECIMAL(12,4),
    PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

-- 7-day rolling baseline query (used by Monitor agent)
-- SELECT ad_set_id, AVG(cpc) AS baseline_cpc
-- FROM campaign_metrics_timeseries
-- WHERE tenant_id=$1 AND ts > now() - INTERVAL '7 days'
-- GROUP BY ad_set_id;

ALTER TABLE campaign_metrics_timeseries ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON campaign_metrics_timeseries
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### 2.5 brand_rules

Hard rules stored as structured rows alongside vector guidelines in OpenSearch.

```sql
CREATE TABLE brand_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    rule_type       TEXT NOT NULL CHECK (rule_type IN ('keyword_protect','spend_limit','cpc_ceiling','pause_threshold')),
    rule_key        TEXT,                 -- e.g. keyword value for keyword_protect
    rule_value      TEXT,                 -- e.g. "500" for spend_limit
    description     TEXT,
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE brand_rules ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON brand_rules
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Protected keyword lookup example:**
```sql
SELECT rule_key FROM brand_rules
WHERE tenant_id=$1 AND rule_type='keyword_protect' AND active=TRUE;
```

### 2.6 action_log (relational mirror)

Mirror of DynamoDB audit for SQL-queryable audit trail:

```sql
CREATE TABLE action_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    action_type     TEXT NOT NULL,
    ad_set_id       UUID,
    campaign_id     UUID,
    trigger_cpc     DECIMAL(10,4),
    baseline_cpc    DECIMAL(10,4),
    decision_json   JSONB,
    status          TEXT NOT NULL CHECK (status IN ('executed','queued','rejected','approved')),
    approver_id     TEXT,               -- Cognito user sub (if human approval)
    agent_version   TEXT,
    executed_at     TIMESTAMPTZ DEFAULT now(),
    deleted_at      TIMESTAMPTZ         -- GDPR soft-delete
);

ALTER TABLE action_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON action_log
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### 2.7 Row-Level Security Enforcement

Every connection sets the tenant context at session start:
```sql
SET app.current_tenant_id = '<tenant_id_from_jwt>';
```
This is enforced by FastAPI dependency injection — no query runs without the session var being set.

---

## 3. OpenSearch Serverless — Vector Store

### 3.1 Index Structure

**One index per tenant:** `tenant-{tenant_id}-brand-guidelines`

```json
{
  "mappings": {
    "properties": {
      "tenant_id":    { "type": "keyword" },
      "chunk_id":     { "type": "keyword" },
      "doc_name":     { "type": "text" },
      "content":      { "type": "text", "analyzer": "standard" },
      "embedding":    {
        "type": "knn_vector",
        "dimension": 1536,
        "method": { "name": "hnsw", "engine": "faiss" }
      },
      "rule_type":    { "type": "keyword" },
      "ingested_at":  { "type": "date" }
    }
  }
}
```

### 3.2 Retrieval Strategy (Hybrid BM25 + k-NN)

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "knn": {
            "embedding": {
              "vector": "<query_embedding>",
              "k": 10
            }
          }
        },
        {
          "match": {
            "content": "<query_text>"
          }
        }
      ]
    }
  },
  "size": 5
}
```

### 3.3 Ingest Pipeline

```
[User uploads brand guidelines doc (PDF/DOCX/TXT)]
        │
        ▼
[AWS Comprehend PII scan] — blocks if PII detected
        │
        ▼
[Document chunking (512-token chunks, 64-token overlap)]
        │
        ▼
[Amazon Titan Text Embeddings v2 encode] → 1536-dim vector
        │
        ▼
[OpenSearch upsert — index: tenant-{id}-brand-guidelines]
```

---

## 4. DynamoDB Tables

### 4.1 action-audit (Immutable Audit Log)

**Partition key:** `tenant_id` | **Sort key:** `action_id` (KSUID — time-sortable)

| Attribute | Type | Description |
|---|---|---|
| `tenant_id` | S | Tenant partition |
| `action_id` | S | KSUID (time-sortable) |
| `action_type` | S | pause_ad_set / reallocate_budget / etc. |
| `ad_set_id` | S | Target ad set |
| `campaign_id` | S | Parent campaign |
| `trigger_cpc` | N | CPC that triggered breach |
| `baseline_cpc` | N | 7-day rolling avg at time of breach |
| `decision_json` | S | Full Decision object (JSON string) |
| `status` | S | executed / queued / approved / rejected |
| `approver_id` | S | Cognito user sub (if human approval) |
| `agent_version` | S | LangGraph state graph version hash |
| `executed_at` | S | ISO 8601 timestamp |

**Append-only enforced** via IAM condition: `"Condition": {"ForAllValues:StringEquals": {"dynamodb:Attributes": [...allowed attributes]}}` — no UpdateItem permission.

### 4.2 session-state

**Partition key:** `tenant_id#session_id`

| Attribute | Type | Description |
|---|---|---|
| `pk` | S | `tenant_id#session_id` |
| `state_json` | S | Serialized `CampaignOrchestratorState` |
| `updated_at` | S | ISO 8601 |
| TTL | N | Unix timestamp (auto-expire after 24h) |

### 4.3 human-confirmations

**Partition key:** `tenant_id` | **Sort key:** `confirmation_id` (KSUID)

| Attribute | Type | Description |
|---|---|---|
| `tenant_id` | S | Tenant partition |
| `confirmation_id` | S | KSUID |
| `action_payload` | S | Decision JSON requiring approval |
| `spend_delta` | N | Spend impact |
| `trigger_reason` | S | `spend_delta` / `campaign_pause` / `brand_keyword` |
| `status` | S | pending / approved / rejected |
| `approver_id` | S | Cognito user sub (set on decision) |
| `decision_at` | S | ISO 8601 timestamp |
| `created_at` | S | ISO 8601 |
| `deleted_at` | S | GDPR soft-delete timestamp |
| TTL | N | Auto-expire after 30 days (GDPR hard-delete) |

### 4.4 pii-audience (IAM-Gated)

**Partition key:** `tenant_id` | **Sort key:** `audience_id`

Contains audience targeting identifiers (hashed email lists, custom audience IDs) — never included in the RAG corpus. Accessible only to the `pii-service` IAM role.

| Attribute | Type | Description |
|---|---|---|
| `tenant_id` | S | Tenant partition |
| `audience_id` | S | Audience identifier |
| `audience_type` | S | hashed_email_list / custom_audience / lookalike |
| `network` | S | google_ads / meta |
| `deleted_at` | S | GDPR soft-delete timestamp |
| TTL | N | 30-day hard-delete |

---

## 5. AgentCore Memory

| Memory Strategy | Content Stored | Retention |
|---|---|---|
| EPISODIC | Past breach events + decisions + CPC outcomes per ad_set | Cross-run; managed by AgentCore |
| SEMANTIC | Distilled brand + channel performance patterns | Long-term; managed by AgentCore |

Memory is opaque (no schema control) — queried via AgentCore Memory SDK by `tenant_id` + retrieval query.

---

## 6. Entity Relationship Overview

```
tenants (1) ──────< campaigns (1) ──────< ad_sets
    │                                        │
    │                                        │
    │< brand_rules                           │< campaign_metrics_timeseries
    │                                        │
    │< action_log ──────────────────────────>│
    │
    ├── DynamoDB: action-audit (tenant_id partition)
    ├── DynamoDB: session-state (tenant_id#session_id)
    ├── DynamoDB: human-confirmations (tenant_id partition)
    ├── DynamoDB: pii-audience (tenant_id partition, IAM-gated)
    └── OpenSearch: tenant-{id}-brand-guidelines (per-tenant index)
```

---

## 7. GDPR Data Residency

| Store | EU Tenant Residency | Non-EU Residency |
|---|---|---|
| Aurora Serverless | `eu-west-1` / `eu-central-1` cluster | `us-east-1` cluster |
| DynamoDB | EU region tables | US region tables |
| OpenSearch | EU region collection | US region collection |
| AgentCore Memory | EU AgentCore endpoint | US AgentCore endpoint |

EU tenant routing enforced at FastAPI middleware layer using `tenant.eu_data_residency` flag.

---

## 8. Provenance

| Section | Origin | Source |
|---|---|---|
| 1. Overview (store selection) | Inherited | `decision-registry.yaml → campaign-data-model`, `decision-registry.yaml → vector-db-choice`, `decision-registry.yaml → rag-data-model-strategy` |
| 2. Aurora Schema | Authored | Schema designed in architecture phase; grounded in `module-map.yaml (MOD-000 db layer, MOD-001 db layer)`; RLS pattern from `decision-registry.yaml → multi-tenant-isolation` |
| 3. OpenSearch Index + Ingest | Authored | Index schema authored; embedding model from `decision-registry.yaml → embedding-model`; ingest pipeline from `decision-registry.yaml → brand-guidelines-ingestion` |
| 4. DynamoDB Tables | Authored | Table designs authored; attribute set from `decision-registry.yaml → audit-logging`; GDPR TTL from `decision-registry.yaml → gdpr-erasure-strategy`; PII segregation from `decision-registry.yaml → pii-handling-strategy` |
| 5. AgentCore Memory | Inherited | `decision-registry.yaml → agent-memory-strategy` |
| 6. Entity Relationships | Authored | Derived from Aurora schema + DynamoDB tables in architecture phase |
| 7. GDPR Data Residency | Inherited | `decision-registry.yaml → gdpr-data-residency` |
