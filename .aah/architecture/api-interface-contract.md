# API Interface Contract
## Agentic Campaign Orchestrator

**Phase:** architecture · **Iteration:** 1 · **Authored:** 2026-08-02  
**Inputs:** `decision-registry.yaml`, `module-map.yaml`, `application-flow.md`

---

## 1. Overview

All API endpoints are served by the FastAPI backend. Base URL: `https://api.<domain>/api/v1/`  
All endpoints require a valid Cognito JWT in the `Authorization: Bearer <token>` header unless marked `[PUBLIC]`.  
Tier enforcement: every endpoint declares its minimum required tier — the server enforces independently of client-side UX gating.

**Content types:** JSON for all REST endpoints. `text/event-stream` for SSE endpoints.  
**Error format:**
```json
{ "error": "string", "code": "TIER_INSUFFICIENT | NOT_FOUND | VALIDATION_ERROR | ...", "detail": "..." }
```

---

## 2. Auth

### POST /auth/token `[PUBLIC]`

Exchange Cognito authorization code for JWT.

**Request:**
```json
{ "code": "string", "redirect_uri": "string" }
```

**Response 200:**
```json
{
  "access_token": "string",
  "id_token": "string",
  "refresh_token": "string",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

**Response 401:** `{ "error": "Invalid authorization code" }`

---

## 3. Health

### GET /health `[PUBLIC]`

**Response 200:**
```json
{ "status": "ok", "version": "string", "ts": "ISO8601" }
```

---

## 4. Campaigns

### GET /campaigns

Returns all campaigns for the authenticated tenant.  
**Tier:** observer+

**Response 200:**
```json
{
  "campaigns": [
    {
      "id": "uuid",
      "name": "string",
      "network": "google_ads | meta",
      "status": "active | paused",
      "ad_sets": [
        { "id": "uuid", "name": "string", "status": "active | paused" }
      ]
    }
  ]
}
```

### GET /campaigns/{campaign_id}/metrics

Current and recent CPC/CTR/ROAS for a campaign.  
**Tier:** observer+

**Response 200:**
```json
{
  "campaign_id": "uuid",
  "ad_sets": [
    {
      "ad_set_id": "uuid",
      "current_cpc": 1.42,
      "baseline_cpc_7d": 1.18,
      "breach_active": true,
      "ctr": 0.032,
      "roas": 3.8,
      "spend_today_usd": 142.50
    }
  ],
  "as_of": "ISO8601"
}
```

---

## 5. Monitor Control

### POST /monitor/start

Start the campaign monitoring agent loop.  
**Tier:** copilot+

**Request:**
```json
{ "campaign_ids": ["uuid", "..."], "poll_interval_seconds": 60 }
```

**Response 200:**
```json
{ "session_id": "string", "status": "started" }
```

### POST /monitor/stop

**Tier:** copilot+

**Request:**
```json
{ "session_id": "string" }
```

**Response 200:**
```json
{ "session_id": "string", "status": "stopped" }
```

---

## 6. Action Confirmation Queue (Co-Pilot)

### GET /confirmations

List pending actions requiring human approval.  
**Tier:** copilot+

**Response 200:**
```json
{
  "confirmations": [
    {
      "confirmation_id": "string",
      "action_type": "pause_ad_set | reallocate_budget | update_bid_cap",
      "target_ad_set_id": "uuid",
      "source_ad_set_id": "uuid | null",
      "amount_usd": 200.00,
      "bid_cap": null,
      "rationale": "string",
      "spend_delta": 200.00,
      "trigger_reason": "spend_delta | campaign_pause | brand_keyword",
      "trigger_cpc": 1.89,
      "baseline_cpc": 1.18,
      "created_at": "ISO8601"
    }
  ]
}
```

### POST /confirmations/{confirmation_id}/approve

Approve a pending action. Approver identity recorded from JWT.  
**Tier:** copilot+

**Response 200:**
```json
{
  "confirmation_id": "string",
  "status": "approved",
  "approver_id": "cognito_sub",
  "executed_at": "ISO8601"
}
```

**Response 403:** Tier insufficient (observer cannot approve)  
**Response 404:** Confirmation not found or already decided

### POST /confirmations/{confirmation_id}/reject

**Tier:** copilot+

**Request:**
```json
{ "reason": "string (optional)" }
```

**Response 200:**
```json
{ "confirmation_id": "string", "status": "rejected" }
```

---

## 7. Audit Log

### GET /audit-log

Paginated action audit log. All tiers — content scope determined by tier.  
**Tier:** observer+

**Query params:** `page`, `page_size` (default 25), `ad_set_id`, `action_type`, `status`, `from_ts`, `to_ts`

**Response 200:**
```json
{
  "total": 142,
  "page": 1,
  "items": [
    {
      "action_id": "string",
      "action_type": "string",
      "ad_set_id": "uuid",
      "campaign_id": "uuid",
      "trigger_cpc": 1.89,
      "baseline_cpc": 1.18,
      "decision_summary": "string",
      "status": "executed | queued | approved | rejected",
      "approver_id": "cognito_sub | null",
      "agent_version": "string",
      "executed_at": "ISO8601"
    }
  ]
}
```

---

## 8. Guardrail Settings

### GET /guardrails

Fetch per-tenant guardrail configuration.  
**Tier:** observer+

**Response 200:**
```json
{
  "campaign_rules": {
    "cpc_ceiling_usd": 5.00,
    "daily_spend_cap_usd": 1000.00,
    "pause_threshold_pct": 30,
    "spend_delta_approval_threshold_usd": 500.00,
    "protected_keywords": ["brand-name", "trademark"]
  },
  "llm_safety": {
    "pii_redaction_enabled": true,
    "denied_topics": ["competitor_names"],
    "content_threshold": "medium"
  }
}
```

### PUT /guardrails

Update guardrail settings. Writes to Bedrock Guardrails API + Aurora brand_rules.  
**Tier:** copilot+

**Request:** same shape as GET response

**Response 200:**
```json
{ "status": "updated", "guardrail_version": "DRAFT" }
```

---

## 9. Brand Guidelines Ingest

### POST /brand-guidelines/ingest

Upload a brand guidelines document.  
**Tier:** copilot+

**Request:** `multipart/form-data`
- `file`: PDF/DOCX/TXT (max 10MB)
- `doc_name`: string

**Response 200:**
```json
{
  "doc_name": "string",
  "chunks_indexed": 42,
  "rules_extracted": 7,
  "pii_scan_result": "clean"
}
```

**Response 422:** PII detected — document blocked from ingest
```json
{ "error": "PII detected in document", "code": "PII_BLOCKED", "detail": "..." }
```

---

## 10. Real-Time SSE Streams

### GET /stream/metrics

Campaign metrics time-series stream. Emits every 30 seconds.  
**Tier:** observer+  
**Headers:** `Accept: text/event-stream`

**Event format:**
```
event: metrics
data: {"ad_set_id":"uuid","cpc":1.42,"ctr":0.032,"roas":3.8,"spend_usd":142.50,"breach_active":true,"ts":"ISO8601"}

event: metrics
data: {"ad_set_id":"uuid","cpc":0.98,...}
```

**Keepalive:** `: keep-alive` comment every 15s

### GET /stream/agent-events

AgentCore AG-UI SSE proxy. Forwards typed agent events to the browser.  
**Tier:** observer+  
**Headers:** `Accept: text/event-stream`

**Event types and payloads:**

```
event: RUN_STARTED
data: {"session_id":"string","tenant_id":"uuid","trigger_ad_set_id":"uuid","ts":"ISO8601"}

event: TOOL_CALL_START
data: {"session_id":"string","tool_name":"get_campaign_performance | pause_ad_set | ...","args":{...},"ts":"ISO8601"}

event: TOOL_CALL_END
data: {"session_id":"string","tool_name":"string","result_summary":"string","ts":"ISO8601"}

event: STATE_SNAPSHOT
data: {"session_id":"string","node":"reason_node | act_node | ...","state_keys":["decision","reasoning_trace"],"ts":"ISO8601"}

event: RUN_FINISHED
data: {"session_id":"string","action_type":"string","status":"executed | queued","confirmation_id":"string|null","ts":"ISO8601"}
```

---

## 11. Admin

### GET /admin/token-usage

Per-tenant token consumption.  
**Tier:** autopilot only

**Response 200:**
```json
{
  "tenant_id": "uuid",
  "current_period_start": "ISO8601",
  "tokens_used": 142500,
  "budget_limit": 500000,
  "budget_pct_used": 28.5
}
```

---

## 12. GDPR Erasure

### DELETE /tenants/{tenant_id}/gdpr-erasure

Initiate GDPR erasure for the tenant (admin/owner only).  
**Tier:** autopilot or admin claim required

**Response 200:**
```json
{
  "status": "erasure_initiated",
  "tenant_id": "uuid",
  "hard_delete_at": "ISO8601"
}
```

### GET /tenants/{tenant_id}/erasure-status

**Response 200:**
```json
{
  "status": "pending_soft_delete | soft_deleted | hard_deleted",
  "soft_deleted_at": "ISO8601",
  "hard_delete_scheduled_at": "ISO8601"
}
```

---

## 13. MCP Tool Interface Contract

Both FastMCP servers (Google Ads port 8001, Meta port 8002) expose identical tool interfaces:

### Tool: get_campaign_performance

**Input:**
```json
{ "campaign_id": "string", "date_range": "last_7_days | last_30_days" }
```

**Output:**
```json
{
  "campaign_id": "string",
  "ad_sets": [
    {
      "ad_set_id": "string",
      "cpc": 1.42,
      "ctr": 0.032,
      "roas": 3.8,
      "impressions": 50000,
      "clicks": 1600,
      "spend_usd": 2272.0
    }
  ]
}
```

### Tool: pause_ad_set / resume_ad_set

**Input:** `{ "ad_set_id": "string" }`  
**Output:** `{ "ad_set_id": "string", "status": "paused | active", "execution_id": "string" }`

### Tool: reallocate_budget

**Input:**
```json
{ "source_ad_set_id": "string", "target_ad_set_id": "string", "amount_usd": 200.00 }
```
**Output:** `{ "execution_id": "string", "new_source_budget": 800.00, "new_target_budget": 1200.00 }`

### Tool: update_bid_cap

**Input:** `{ "ad_set_id": "string", "new_bid_cap": 1.20 }`  
**Output:** `{ "ad_set_id": "string", "new_bid_cap": 1.20, "execution_id": "string" }`

### Mock Control Endpoint (test-only)

**POST /mock/inject-spike** (MCP server health port):
```json
{ "ad_set_id": "string", "cpc_multiplier": 1.5 }
```
Used by smoke tests to artificially trigger a CPC breach.

---

## 14. Provenance

| Section | Origin | Source |
|---|---|---|
| 1. Overview (auth, error format) | Inherited | `decision-registry.yaml → auth-strategy` |
| 2. Auth / token exchange | Inherited | `decision-registry.yaml → auth-strategy` (Cognito OIDC) |
| 4. Campaigns | Authored | Derived from `module-map.yaml (MOD-001 api layer)` |
| 5. Monitor Control | Authored | Derived from `module-map.yaml (MOD-001 api layer)` |
| 6. Confirmation Queue | Authored | `decision-registry.yaml → action-log-design`, `decision-registry.yaml → copilot-approval-ux`, `module-map.yaml (MOD-003 api layer)` |
| 7. Audit Log | Authored | `decision-registry.yaml → audit-logging`, `module-map.yaml (MOD-003 api layer)` |
| 8. Guardrail Settings | Authored | `decision-registry.yaml → guardrail-settings-ui`, `module-map.yaml (MOD-004 api layer)` |
| 9. Brand Guidelines Ingest | Authored | `decision-registry.yaml → brand-guidelines-ingestion`, `decision-registry.yaml → data-privacy-controls`, `module-map.yaml (MOD-005 api layer)` |
| 10. SSE Streams | Inherited + Authored | `decision-registry.yaml → notification-design`; AG-UI event types from agentcore-implement.md |
| 11. Admin / Token Usage | Authored | `decision-registry.yaml → cost-management`, `module-map.yaml (MOD-005 infra layer)` |
| 12. GDPR Erasure | Authored | `decision-registry.yaml → gdpr-erasure-strategy`, `application-flow.md §8 (Flow 7)` |
| 13. MCP Tool Interface | Inherited | `decision-registry.yaml → mcp-tool-definitions`, `agent-topology.md §4.2` |
