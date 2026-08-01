# Product Requirements Document
## Agentic Campaign Orchestrator

**Phase:** discuss · **Iteration:** 1 · **Status:** approved  
**Authored:** 2026-08-02 · **Registry:** `.aah/discuss/decision-registry.yaml`

---

## 1. Problem Statement

Marketing managers, media buyers, and agencies spend thousands of dollars monthly on Google Ads and Meta campaigns, but campaign performance degrades in real time — cost-per-click (CPC) spikes, budgets drain into underperforming ad sets, and human analysts only review dashboards once or twice a day. By the time a person intervenes, significant spend has been wasted. No autonomous system today continuously monitors campaigns, reasons about why performance degraded, and takes corrective action within minutes — safely, with full audit trails, and within user-defined brand guardrails.

---

## 2. Solution

The **Agentic Campaign Orchestrator** is a Python-based, AWS-native SaaS product that acts as an autonomous digital media buyer. It is deployed on **AWS Bedrock AgentCore Runtime** using a **3-agent supervisor topology** (Monitor → Reason → Act), orchestrated via LangGraph's event-triggered ReAct loop.

### How It Works

1. **Monitor Agent** (cost-optimised fast model, runs continuously): polls campaign performance every cycle from Google Ads and Meta via **FastMCP HTTP/SSE servers** (one per ad network). Detects when CPC breaches **20% above its 7-day rolling baseline**.

2. **Reason Agent** (Claude or GPT via provider-agnostic API, Bedrock-hosted): triggered on breach detection. Retrieves context from two stores simultaneously:
   - **SQL query against Aurora Serverless PostgreSQL** — exact CPC/CTR/ROAS aggregates for the affected campaign
   - **Hybrid RAG retrieval against OpenSearch Serverless** — semantic search for brand guidelines (Amazon Titan Text Embeddings v2) + deterministic rule lookup for hard constraints
   - Uses **Bedrock AgentCore Memory** (EPISODIC + SEMANTIC strategies) to recall past campaign interventions and their outcomes

3. **Act Agent** (MCP execution layer): executes the decided action via one of four MCP tools: `pause_ad_set`, `resume_ad_set`, `reallocate_budget`, `update_bid_cap`. Actions that exceed the **$500 spend-delta threshold** or target an entire campaign always route to the human confirmation queue, regardless of tier.

### Subscription Tiers

| Tier | Price | Capability |
|---|---|---|
| **Observer** | $99/mo | Monitor + alerts + read-only action log |
| **Co-Pilot** | $299/mo | Agent proposes actions → in-app approval queue + push notifications |
| **Autopilot** | $799/mo | Fully autonomous within guardrails; all actions logged |

---

## 3. User Stories

1. As a **marketing manager**, I want the system to detect when a campaign's CPC exceeds its 7-day rolling baseline by 20%, so that I am alerted before significant budget is wasted.

2. As a **Co-Pilot subscriber**, I want to see the agent's proposed actions (with the reasoning and projected effect) in an approval queue, so that I can approve or reject with one click, with my identity recorded against the decision.

3. As a **media buyer on Autopilot**, I want the agent to autonomously pause underperforming ad sets and reallocate budget within my guardrail settings, so that my campaigns self-optimise overnight.

4. As a **campaign analyst**, I want an audit log of every action taken — trigger condition, agent reasoning, before/after CPC, and the human approver ID where applicable — so that I can review and trust the agent's decision history.

5. As an **agency account manager**, I want to set brand-keyword protection lists and a per-client spend-delta approval threshold, so that the agent never takes actions that violate client brand guidelines.

6. As a **platform subscriber**, I want to choose my subscription tier and upgrade seamlessly, so that I can start with visibility and graduate to full automation without re-onboarding.

---

## 4. Technical Architecture

### 4.1 Agent Framework

- **Orchestration:** LangGraph `StateGraph` (TypedDict state)
- **Pattern:** Event-triggered ReAct loop (Monitor → Reason → Act)
- **Topology:** 3-agent supervisor in a single process; Monitor is always running, Reason+Act spawn on breach
- **Runtime:** AWS Bedrock AgentCore Runtime (managed serverless, per-session microVM isolation, MCP gateway)
- **LLM:** Provider-agnostic — Claude (Bedrock-hosted) or GPT API; model routing selectable at tenant level

### 4.2 MCP Layer

- **Framework:** FastMCP (Python), HTTP/SSE transport
- **Topology:** One MCP server per ad network (Google Ads MCP, Meta MCP)
- **Tools defined:**
  - `get_campaign_performance(campaign_id, date_range)` → CPC/CTR/ROAS metrics
  - `pause_ad_set(ad_set_id)` / `resume_ad_set(ad_set_id)`
  - `reallocate_budget(source_ad_set_id, target_ad_set_id, amount_usd)`
  - `update_bid_cap(ad_set_id, new_bid_cap)`
- **MVP scope:** Mock ad network APIs (no live Google/Meta accounts required)

### 4.3 RAG Architecture

- **Vector store:** OpenSearch Serverless (k-NN + BM25 hybrid search)
- **Embedding model:** Amazon Titan Text Embeddings v2
- **Relational store:** Aurora Serverless PostgreSQL (CPC/CTR/ROAS time-series, brand hard rules)
- **RAG strategy:** Hybrid — semantic search (brand guidelines) + deterministic SQL lookup (campaign metrics + hard rules)
- **Ingestion:** Hybrid structured rules (SQL import) + unstructured guidelines (vector ingest)

### 4.4 Memory

- **Strategy:** Hybrid in-context + AgentCore Memory
- **EPISODIC memory:** past campaign interventions and outcomes (cross-run recall)
- **SEMANTIC memory:** distilled learnings about brand + channel performance patterns
- **Prompt caching:** Bedrock prompt caching on Monitor agent's recurring context (20–50% token reduction)

### 4.5 Data Architecture

| Store | Purpose | Isolation |
|---|---|---|
| Aurora Serverless PostgreSQL | Campaign metrics (CPC/CTR/ROAS), brand hard rules, tenant configs | Row-Level Security on `tenant_id` |
| OpenSearch Serverless | Brand guidelines vector index (RAG corpus) | Index-per-tenant |
| DynamoDB | Session state, action audit log (immutable), human confirmation records | Partition key = `tenant_id` |
| Bedrock AgentCore Memory | EPISODIC + SEMANTIC cross-run campaign learnings | AgentCore-managed isolation |

### 4.6 Frontend

- **Framework:** React 18 + TypeScript + Vite + Shadcn/UI
- **Hosting:** AWS Amplify Hosting (CDN, auto-HTTPS, static SPA zip-upload)
- **Real-time updates:**
  - Agent action events: AG-UI SSE (AgentCore native — typed `TOOL_CALL_START`, `TOOL_CALL_END`, `RUN_FINISHED` events) via FastAPI proxy
  - Campaign metric panels: plain FastAPI SSE endpoint (time-series data)
- **Key views:**
  - Campaign health dashboard (live CPC, CTR, ROAS per ad set)
  - Action queue (Co-Pilot: pending approve/reject with reasoning preview)
  - Audit log (all tiers: completed actions with actor + outcome)
  - Guardrail settings (unified form: campaign rules tab + LLM safety tab → Bedrock Guardrails)

### 4.7 Backend API

- **Framework:** Python FastAPI
- **Auth:** AWS Cognito OIDC/JWT; subscription tier encoded in user groups
- **Tier gating:** Hybrid — JWT claims for client UX; server enforces + logs every access decision

---

## 5. Guardrails & Safety

### Autonomous Action Scope (Autopilot)

| Action | Allowed | Approval Required |
|---|---|---|
| Pause / resume an ad set | ✅ | Only if spend delta > $500 |
| Reallocate budget between ad sets | ✅ | Only if spend delta > $500 |
| Adjust bid cap (≤20% decrease) | ✅ | Only if spend delta > $500 |
| Pause entire campaign | ❌ | Always requires human approval |

### Brand Protection

- User-defined **protected keyword list** per account — agent never modifies ad sets containing protected keywords without approval
- **Bedrock Guardrails** enforce PII redaction and content policy at both RAG ingest and LLM inference layers

---

## 6. Security & Compliance

### Authentication & Access Control

- **Auth:** AWS Cognito (OIDC/JWT) — subscription tiers as Cognito User Groups
- **Multi-tenant isolation:**
  - PostgreSQL Row-Level Security (RLS) on `tenant_id`
  - DynamoDB partition keys: `tenant_id#resource_id`
  - OpenSearch: index-per-tenant
  - Per-tool IAM scoping on AgentCore

### GDPR

- **Data residency:** EU tenants default to `eu-west-1` / `eu-central-1`; non-EU with SCCs as legal basis
- **Erasure:** Soft-delete on request → hard-delete after 30 days
- **PII schema segregation:** Audience identifiers stored in a separate IAM-gated DynamoDB table; never included in the RAG corpus
- **Ingest-layer PII detection:** AWS Comprehend scans brand guidelines before vector ingest; Bedrock Guardrails redact at LLM layer

### SOC2

- **Audit trail:**
  - CloudWatch + AgentCore OTEL (primary operational logs)
  - DynamoDB immutable action audit table (action type, trigger condition, agent version, timestamp — append-only)
  - Human confirmation log (action ID, approver Cognito user ID, timestamp, decision)
- **Access logging:** FastAPI middleware logs every tier-gated API call to CloudWatch

---

## 7. Deployment & Infrastructure

- **Agent runtime:** AWS Bedrock AgentCore Runtime (serverless, auto-scaling, per-session microVM)
- **Frontend:** AWS Amplify Hosting (global CDN, static SPA zip-upload)
- **Deployment strategy:** Canary release for agent version updates (gradual traffic shift, automatic rollback on error rate threshold)
- **Observability:** CloudWatch dashboards, AgentCore built-in observability, per-tenant token-budget alarms
- **Cost controls:**
  - Per-tenant token budgets (enforced at FastAPI middleware)
  - Tiered model routing (cheaper model for Monitor; full model for Reason)
  - Bedrock prompt caching on Monitor agent's recurring system context

---

## 8. Success Metrics

| Metric | Target (MVP) |
|---|---|
| CPC reduction (Autopilot vs control) | ≥15% reduction within 7 days |
| ROAS improvement | ≥10% improvement vs baseline |
| Time-to-detect-and-act (breach → action) | <5 minutes end-to-end |
| Actions taken + outcome accuracy | ≥80% of autonomous actions validated correct in audit review |
| CPC breach false-positive rate | <10% |

---

## 9. Integration Surface

| System | Type | MVP |
|---|---|---|
| Google Ads API | External ad network | Mock (FastMCP server) |
| Meta (Facebook) Ads API | External ad network | Mock (FastMCP server) |
| OpenSearch Serverless | Vector store | AWS managed |
| Aurora Serverless PostgreSQL | Relational store | AWS managed |
| DynamoDB | Session/audit store | AWS managed |
| Bedrock AgentCore Runtime | Agent runtime | AWS managed |
| Bedrock AgentCore Memory | Cross-run memory | AWS managed |
| Bedrock Guardrails | Safety/PII layer | AWS managed |
| AWS Cognito | Auth | AWS managed |
| AWS Amplify Hosting | Frontend CDN | AWS managed |

---

## 10. Decision Traceability

All decisions in this PRD are traceable to `decision-registry.yaml`. Key traceability links:

| PRD Section | Registry Slugs |
|---|---|
| Agent topology (§4.1) | `agent-framework`, `agent-topology`, `orchestration-pattern`, `llm-choice` |
| MCP layer (§4.2) | `mcp-server-pattern`, `mcp-server-topology`, `mcp-tool-definitions` |
| RAG architecture (§4.3) | `vector-db-choice`, `rag-data-model-strategy`, `embedding-model`, `brand-guidelines-ingestion` |
| Memory (§4.4) | `agent-memory-strategy` |
| Data stores (§4.5) | `campaign-data-model` |
| Frontend (§4.6) | `frontend-framework`, `notification-design`, `action-log-design` |
| Backend API (§4.7) | `auth-strategy`, `tier-gating-pattern` |
| Guardrails (§5) | `autonomous-action-scope`, `campaign-pause-requires-approval`, `spend-delta-approval-threshold`, `brand-protection-strategy`, `guardrail-settings-ui` |
| Security/GDPR (§6) | `multi-tenant-isolation`, `gdpr-data-residency`, `gdpr-erasure-strategy`, `pii-handling-strategy`, `data-privacy-controls`, `audit-logging` |
| Deployment (§7) | `compute-platform`, `frontend-compute-platform`, `deployment-pattern`, `monitoring-strategy`, `cost-management` |
| Success metrics (§8) | `cpc-threshold-method`, `cpc-spike-threshold-pct`, `success-metrics` |
| Subscriptions | `monetisation-model` |

---

*This document is the authoritative PRD for iteration 1. It was produced by `/aah-discuss` and is consumed by `/aah-arch` (module decomposition) and `/aah-plan` (feature decomposition into GitHub issues).*
