# Application Flow
## Agentic Campaign Orchestrator

**Phase:** architecture · **Iteration:** 1 · **Authored:** 2026-08-02  
**Inputs:** `decision-registry.yaml`, `discuss-prd.md`, `module-map.yaml`, `agent-topology.md`

---

## 1. Overview

This document describes the end-to-end event flows for all five demoable modules (MOD-001–MOD-005). Each flow is traced from the originating trigger through every system boundary to the final state change or user-visible outcome.

---

## 2. Flow 1 — CPC Breach Detection (MOD-001)

**Trigger:** Scheduled poll cycle (60-second interval) in Monitor agent  
**End state:** Breach event written to `CampaignOrchestratorState.breach_events`; Aurora metrics updated

```
Monitor Agent (LangGraph node — fast LLM)
│
│ 1. MCP call: get_campaign_performance(campaign_id, "last_7_days")
│    → FastMCP Google Ads MCP server (port 8001) OR Meta MCP server (port 8002)
│    ← CampaignPerformanceResponse { cpc, ctr, roas, impressions, clicks, spend }
│
│ 2. Aurora write: UPSERT campaign_metrics_timeseries
│    (tenant_id, ad_set_id, ts, cpc, ctr, roas, ...)
│
│ 3. Aurora query: 7-day rolling baseline
│    SELECT AVG(cpc) AS baseline FROM campaign_metrics_timeseries
│    WHERE tenant_id=$1 AND ad_set_id=$2 AND ts > now()-INTERVAL '7 days'
│
│ 4. Breach check: current_cpc > baseline_cpc * 1.20?
│    YES → append BreachEvent to state.breach_events
│         → AgentCore emits: RUN_STARTED event (AG-UI SSE)
│    NO  → poll_wait_node → sleep 60s → repeat
│
└─ State updated: breach_events, campaign_metrics, cpc_baselines
```

**Observability:** AgentCore `RUN_STARTED` event visible in React dashboard agent event stream.

---

## 3. Flow 2 — RAG Context Retrieval (MOD-002)

**Trigger:** Breach event in state (output of Flow 1)  
**End state:** `CampaignOrchestratorState.retrieved_context` populated

```
retrieve_context_node (deterministic — no LLM)
│
├─ A. OpenSearch hybrid retrieval
│   │
│   │ 1. Encode breach description with Titan Text Embeddings v2
│   │    Input: "CPC spike on ad_set {id}: current={cpc}, baseline={baseline}, delta={pct}%"
│   │    Output: 1536-dim float vector
│   │
│   │ 2. Hybrid BM25 + k-NN query → tenant-{id}-brand-guidelines index
│   │    Returns: top-5 brand guideline chunks (content, score)
│   │
│   └─ state.retrieved_context.brand_guidelines = [chunk_1, ..., chunk_5]
│
├─ B. Aurora SQL aggregation
│   │
│   │ SELECT ad_set_id, AVG(cpc) 7d_avg, AVG(ctr), AVG(roas), SUM(spend)
│   │ FROM campaign_metrics_timeseries
│   │ WHERE tenant_id=$1 AND ad_set_id IN ($breach_adsets)
│   │   AND ts > now() - INTERVAL '7 days'
│   │ GROUP BY ad_set_id
│   │
│   └─ state.retrieved_context.metrics_summary = formatted SQL result
│
├─ C. Aurora hard rules lookup
│   │
│   │ SELECT rule_type, rule_key, rule_value FROM brand_rules
│   │ WHERE tenant_id=$1 AND active=TRUE
│   │
│   └─ state.retrieved_context.hard_rules = [rule_1, ..., rule_N]
│
└─ D. AgentCore Memory retrieval
    │
    │ EPISODIC: query past interventions for this ad_set_id
    │ SEMANTIC: query patterns matching "CPC spike {network} {ad_format}"
    │
    └─ state.retrieved_context.past_interventions = [episode_1, ..., episode_3]
```

**AgentCore events emitted:** `TOOL_CALL_START` + `TOOL_CALL_END` for each retrieval tool.

---

## 4. Flow 3 — Autonomous Decision Generation (MOD-002)

**Trigger:** `retrieved_context` populated  
**End state:** `CampaignOrchestratorState.decision` (typed Decision JSON)

```
reason_node (full LLM — Claude or GPT)
│
│ 1. Build prompt:
│    System: "You are an autonomous digital media buyer..."
│    Context: breach_event + metrics_summary + brand_guidelines + hard_rules + past_interventions
│    Instruction: "Select ONE action. Output valid JSON Decision object."
│
│ 2. Bedrock Guardrails applied at LLM inference
│    (PII redaction, denied topics, content thresholds)
│    Bedrock prompt cache: system context cached (20-50% token savings)
│
│ 3. LLM response (JSON mode / function calling)
│    Returns: Decision { action, target_ad_set_id, amount_usd?, bid_cap?, rationale, spend_delta }
│
│ 4. Pydantic validation: enforce Decision TypedDict schema
│    Invalid → retry with clarification prompt (max 2 retries)
│
└─ state.decision = Decision object; state.reasoning_trace = chain-of-thought
```

---

## 5. Flow 4 — Guardrail Gate + Action Execution (MOD-003)

**Trigger:** `decision` populated  
**End state:** Action executed (MCP) or queued (DynamoDB); audit record written

```
guardrail_check_node (deterministic)
│
├─ spend_delta > $500 → requires_approval = True
├─ action == "pause_ad_set" + targets entire campaign → requires_approval = True
├─ brand keyword in target ad_set creative → requires_approval = True
└─ Bedrock Guardrails content check on decision rationale
│
│
┌──────────────┴──────────────┐
│ requires_approval = False   │ requires_approval = True
▼                             ▼
act_node                      enqueue_confirmation_node
│                             │
│ MCP tool execution:         │ DynamoDB PUT: human-confirmations
│ await mcp_client.call_tool( │   {confirmation_id, action_payload,
│   name=decision.action,     │    spend_delta, trigger_reason, status=pending}
│   arguments={...}           │
│ )                           │ → FastAPI pushes AG-UI STATE_SNAPSHOT event
│                             │ → Co-Pilot queue card appears in React UI
│ ActionResponse {success,    │
│   execution_id}             │ [Human approves/rejects in UI]
│                             │   POST /api/v1/confirmations/{id}/approve
│                             │   → DynamoDB UPDATE: approver_id, status, decision_at
│                             │   → act_node resumes with approved decision
│                             │
▼                             ▼
audit_write_node              audit_write_node
│                             │
│ DynamoDB PUT (append-only): action-audit table
│ {action_id, action_type, ad_set_id, trigger_cpc, baseline_cpc, decision_json,
│  status, approver_id (if applicable), agent_version, executed_at}
│
│ Aurora INSERT: action_log (SQL mirror)
│
▼
memory_write_node
│
│ AgentCore EPISODIC write:
│   {breach_event, decision, outcome, cpc_before, cpc_after (polled 5min later)}
│
└─ AgentCore emits: RUN_FINISHED event (AG-UI SSE)
```

---

## 6. Flow 5 — Dashboard Real-Time Updates (MOD-004)

**Trigger:** Browser opens dashboard or agent run starts  
**End state:** UI shows live metrics + agent events

```
Browser (React EventSource)
│
├─ Campaign Metrics Stream
│   GET /api/v1/stream/metrics
│   FastAPI SSE endpoint (text/event-stream)
│   │
│   │ Every 30s: Aurora SELECT latest metrics per ad_set for tenant
│   │ Emits: data: {"ad_set_id": ..., "cpc": ..., "ctr": ..., "ts": ...}
│   │
│   └─ React: updates CPC/CTR/ROAS panels; highlights breach ad_sets
│
└─ Agent Event Stream
    GET /api/v1/stream/agent-events
    FastAPI SSE proxy (bridges AgentCore AG-UI SSE → browser)
    │
    │ AgentCore → FastAPI proxy → browser EventSource
    │ Events: RUN_STARTED, TOOL_CALL_START, TOOL_CALL_END, STATE_SNAPSHOT, RUN_FINISHED
    │
    └─ React: shows live agent trace in event panel
```

**Co-Pilot Approval Flow (MOD-004 — Co-Pilot tier only):**
```
1. enqueue_confirmation_node writes to DynamoDB
2. AgentCore emits STATE_SNAPSHOT (includes confirmation_id)
3. React picks up STATE_SNAPSHOT from agent event stream
4. React fetches GET /api/v1/confirmations → pending queue
5. Action queue card appears (action type, reasoning, projected spend delta)
6. User clicks Approve/Reject
7. POST /api/v1/confirmations/{id}/approve
   FastAPI: Cognito JWT auth → extract cognito_sub → tier check (copilot or autopilot)
   DynamoDB UPDATE: status=approved, approver_id=cognito_sub, decision_at=now()
   FastAPI: re-enqueues approved decision to act_node
8. act_node executes → audit_write_node logs approver_id
9. React: confirmation card moves to completed; appears in audit log
```

---

## 7. Flow 6 — Brand Guidelines Ingest (MOD-005)

**Trigger:** User uploads brand guidelines document via dashboard  
**End state:** Chunks indexed in OpenSearch; PII-clean guarantee

```
Browser
│ POST /api/v1/brand-guidelines/ingest (multipart/form-data)
│ Cognito JWT auth: copilot or autopilot tier required
│
FastAPI
│
│ 1. AWS Comprehend PII detection on document text
│    If PII detected → 422 Unprocessable Entity (block ingest)
│
│ 2. Document chunking (512-token chunks, 64-token overlap)
│
│ 3. For each chunk:
│    a. Amazon Titan Text Embeddings v2 → 1536-dim vector
│    b. OpenSearch upsert → tenant-{id}-brand-guidelines index
│
│ 4. Aurora INSERT: brand_rules table (structured rule extraction from doc)
│
└─ 200 OK: {chunks_indexed: N, rules_extracted: M}
```

---

## 8. Flow 7 — GDPR Erasure (MOD-005)

**Trigger:** User or API request to erase tenant data  
**End state:** All tenant data soft-deleted; hard-delete scheduled at 30 days

```
DELETE /api/v1/tenants/{id}/gdpr-erasure
Cognito JWT auth: tenant admin role required
│
FastAPI
│
│ 1. Aurora: UPDATE tenants SET deleted_at=now() WHERE id=$tenant_id
│            UPDATE campaigns SET deleted_at=now() WHERE tenant_id=$1
│            UPDATE ad_sets SET deleted_at=now() WHERE tenant_id=$1
│            UPDATE action_log SET deleted_at=now() WHERE tenant_id=$1
│            (brand_rules, campaign_metrics_timeseries: soft-delete similarly)
│
│ 2. DynamoDB: mark all items in human-confirmations + pii-audience with deleted_at
│    TTL attribute set to now() + 30 days → DynamoDB auto-hard-deletes
│
│ 3. DynamoDB action-audit: immutable — cannot delete. Log erasure request itself.
│    (SOC2 requirement: audit log is permanent)
│
│ 4. OpenSearch: DELETE by query on tenant-{id}-brand-guidelines index
│
│ 5. AgentCore Memory: invalidate all sessions/memories for tenant_id
│
└─ 200 OK: {status: "erasure_initiated", hard_delete_at: "<ISO timestamp + 30d>"}
```

---

## 9. Flow 8 — Subscription Tier Gating (all tiers, MOD-005)

**Trigger:** Any API request  
**End state:** Request allowed (correct tier) or 403 Forbidden (wrong tier)

```
Browser request + Cognito JWT
│
FastAPI Cognito JWT middleware
│ 1. Verify JWT signature (Cognito JWKS)
│ 2. Extract: cognito:groups claim → tier (observer/copilot/autopilot)
│ 3. Set: request.state.tenant_id, request.state.tier
│
Tier enforcement dependency (FastAPI Depends)
│ 4. Check endpoint's required tier >= request.state.tier
│    Example: POST /confirmations/{id}/approve requires_tier = "copilot"
│
│ IF tier insufficient: return 403 Forbidden
│ 5. Log access decision to CloudWatch:
│    {endpoint, method, tenant_id, tier, allowed: bool, ts}
│
└─ Proceed to endpoint handler (if allowed)

Client UX (parallel, client-side):
│ JWT decoded in React → group claim → client-side tier check
│ Approve button hidden for observer tier (UX only — server is authoritative)
```

---

## 10. Provenance

| Section | Origin | Source |
|---|---|---|
| 1. Overview | Inherited | `discuss-prd.md §4`, `module-map.yaml` |
| 2. Flow 1 — CPC Breach Detection | Inherited + Authored | `decision-registry.yaml → cpc-threshold-method`, `decision-registry.yaml → cpc-spike-threshold-pct`, `decision-registry.yaml → mcp-tool-definitions`; sequence authored |
| 3. Flow 2 — RAG Retrieval | Inherited + Authored | `decision-registry.yaml → rag-data-model-strategy`, `decision-registry.yaml → vector-db-choice`, `decision-registry.yaml → embedding-model`, `agent-topology.md §3.2`; sequence authored |
| 4. Flow 3 — Decision Generation | Inherited + Authored | `decision-registry.yaml → llm-choice`, `decision-registry.yaml → orchestration-pattern`, `agent-topology.md §3.3` |
| 5. Flow 4 — Guardrail Gate + Execution | Inherited + Authored | `decision-registry.yaml → spend-delta-approval-threshold`, `decision-registry.yaml → campaign-pause-requires-approval`, `decision-registry.yaml → autonomous-action-scope`, `agent-topology.md §3.4`, `agent-topology.md §3.5` |
| 6. Flow 5 — Dashboard Real-Time | Inherited + Authored | `decision-registry.yaml → notification-design`, `decision-registry.yaml → action-log-design`, `decision-registry.yaml → copilot-approval-ux`; sequence authored |
| 7. Flow 6 — Brand Guidelines Ingest | Inherited + Authored | `decision-registry.yaml → brand-guidelines-ingestion`, `decision-registry.yaml → data-privacy-controls`; Comprehend gate authored |
| 8. Flow 7 — GDPR Erasure | Inherited + Authored | `decision-registry.yaml → gdpr-erasure-strategy`, `decision-registry.yaml → pii-handling-strategy`; store-by-store erasure sequence authored |
| 9. Flow 8 — Tier Gating | Inherited + Authored | `decision-registry.yaml → tier-gating-pattern`, `decision-registry.yaml → auth-strategy`; JWT flow authored |
