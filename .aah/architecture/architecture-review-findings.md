# Architecture Review Board — Findings
## Agentic Campaign Orchestrator

**Date:** 2026-08-02 · **Tier:** mvp · **Cycle:** 1  
**Verdict:** Rework-Required

---

## Findings Summary

| # | Lens | Finding | Severity | Resolution |
|---|------|---------|----------|------------|
| F-001 | 1 + 4 | HITL approval resumption mechanism is not designed anywhere | CRITICAL | decision-level |
| F-002 | 4 | Agent session model contradicts itself — per-breach vs long-lived | MAJOR | decision-level |
| F-003 | 2 + 3 | Aurora `action_log` write executed in MOD-003 flow but absent from MOD-003 db layer | MAJOR | design-level |
| F-004 | 2 | `memory_write_node` claimed by MOD-002 but executes inside MOD-003's context with no handoff spec | MAJOR | design-level |
| F-005 | 3 | DynamoDB action-audit attribute set inconsistent between `data-model.md` and `risk-security-compliance.md` | MAJOR | design-level |
| F-006 | 4 | Per-tenant token budget middleware has no enforcement path over AgentCore-originated LLM calls | MAJOR | decision-level |
| F-007 | 4 | OpenSearch tenant isolation is application-routing-only; no IAM/AOSS policy; threat model omits this vector | MAJOR | design-level |
| F-008 | 3 | `brand_rules` table has dual uncoordinated write paths from MOD-004 and MOD-005 | MAJOR | design-level |
| F-009 | 1 | MOD-000 designated non-demoable but `demo_criteria` includes Cognito login flow completion | MINOR | design-level |
| F-010 | 3 + 4 | `action-audit` immutability (SOC2) vs `approver_id` personal data (GDPR Art. 17) conflict unresolved | MAJOR | decision-level |

---

## Detailed Findings

### F-001 — HITL Approval Resumption Mechanism Is Not Designed

**Severity:** CRITICAL · **Resolution:** decision-level

**Gap:** `application-flow.md §5` states: *"→ act_node resumes with approved decision"*. The StateGraph in `agent-topology.md §2.2` has no such path. The queued branch terminates at `audit_write_node (status=queued)` and returns to END/poll. There is no waiting node, no DynamoDB polling loop, no webhook trigger, no new-session spawn, and no AgentCore API call in any design document that explains how `POST /api/v1/confirmations/{id}/approve` causes `act_node` to execute.

**Impact:** MOD-003 demo ("one action queued above $500, visible in action-audit") and MOD-004 demo ("approve a queued action as Co-Pilot user") both fail — the queued action never completes execution, the approval has no effect.

**Required action (decision-level — user must choose):**

Three candidate mechanisms:

1. **Spawn-new-session**: FastAPI spawns a new AgentCore session on approval, with the approved Decision object pre-loaded in `CampaignOrchestratorState`, bypassing monitor and reason nodes.
2. **Wait-node polling**: A `wait_for_approval_node` is added to the StateGraph that polls DynamoDB for status changes with a configurable timeout (e.g., 10 minutes).
3. **Event bridge**: SNS/SQS/EventBridge bridges the FastAPI approval event to an AgentCore session callback.

Chosen mechanism must appear in `agent-topology.md §2.2` and `application-flow.md Flow 4`.

---

### F-002 — Agent Session Model Contradicts Itself

**Severity:** MAJOR · **Resolution:** decision-level

**Gap:** `agent-topology.md §6.1` bullets 1–3 define a per-breach session model (each breach creates a session; session terminates after `memory_write_node`). Bullet 4 contradicts: *"Monitor's continuous polling loop runs as a long-lived session with checkpointing."* The LangGraph StateGraph shows monitor and reason/act nodes in the same graph, which implies a long-lived session, but "per-session microVM" billing and isolation in AgentCore applies per session.

**Impact:** Ambiguity cascades to AgentCore session cost modeling, state persistence between poll cycles, recovery after crash, and whether `cpc_baselines` dict persists across poll cycles.

**Required action:** Resolve as a decision-level slug — either (a) single long-lived LangGraph session per tenant (continuous polling + reasoning, checkpointed) or (b) separate lightweight monitor process outside AgentCore spawning per-breach sessions. Update `agent-topology.md §6.1` to use consistent terminology.

---

### F-003 — Aurora action_log Write Absent from MOD-003 Module Scope

**Severity:** MAJOR · **Resolution:** design-level

**Gap:** `application-flow.md §5` places an Aurora INSERT into `action_log` inside `audit_write_node`. MOD-003's db layer in `module-map.yaml` lists only DynamoDB writes — the Aurora write is unacknowledged. Plan-phase feature decomposition will miss the Aurora audit write entirely.

**Fix:** Add `Aurora action_log INSERT (SQL mirror of DynamoDB action-audit; written by audit_write_node)` to MOD-003's db layer in `module-map.yaml`.

---

### F-004 — memory_write_node Ownership Boundary Undefined

**Severity:** MAJOR · **Resolution:** design-level

**Gap:** MOD-002 module-map claims `memory_write_node` in its memory scope ("triggered by MOD-003"). MOD-003 makes no mention of triggering it. The StateGraph places this node after `audit_write_node` (MOD-003 territory). Implementation responsibility is a blank — plan-phase produces an ownership gap.

**Fix:** Either (a) move `memory_write_node` from MOD-002's memory layer into MOD-003's agent layer, or (b) add an explicit "triggers memory_write_node after audit_write completes" entry to MOD-003's agent layer. Add a cross-reference note in MOD-002 distinguishing memory retrieval (pre-decision, MOD-002) from memory write (post-action, MOD-003).

---

### F-005 — DynamoDB action-audit Attribute Set Inconsistent Across Docs

**Severity:** MAJOR · **Resolution:** design-level

**Gap:** `data-model.md §4.1` defines 12 attributes for the action-audit table. `risk-security-compliance.md §3.2` defines the same table with 3 additional attributes not in the data model: `breach_pct: float`, `spend_delta: float`, `network: google_ads | meta`. `spend_delta` drives the approval-gate decision — it must be a queryable attribute, not only inside `decision_json`.

**Fix:** Update `data-model.md §4.1` to add `breach_pct`, `spend_delta`, and `network` attributes, matching the authoritative schema in `risk-security-compliance.md §3.2`. Data model is the single source of truth for table schemas.

---

### F-006 — Token Budget Middleware Has No Enforcement Path Into AgentCore LLM Calls

**Severity:** MAJOR · **Resolution:** decision-level

**Gap:** The per-tenant token budget middleware is a FastAPI component. LLM token consumption occurs almost entirely from within AgentCore Runtime (Monitor calls Haiku, Reason calls Claude/GPT). None of these calls pass through FastAPI middleware. There is no specification of how the FastAPI middleware throttles or blocks AgentCore-originated LLM calls. The `max-iteration guard` in the decision registry is not defined in any design document.

**Required action (decision-level — user must choose):**

- (a) AgentCore configured with per-tenant model invocation limits via IAM/service quotas; FastAPI middleware is advisory only.
- (b) Pre-call budget check inside the LangGraph node code itself (before each `bedrock.invoke_model` call), reading a counter from DynamoDB/ElastiCache, aborting with a graceful error if exceeded.

Add chosen mechanism to MOD-005 infra layer and `agent-topology.md §3.1` and `§3.3`.

---

### F-007 — OpenSearch Tenant Isolation Is Application-Routing-Only

**Severity:** MAJOR · **Resolution:** design-level

**Gap:** Every other store has a defence-in-depth isolation layer beneath the application (Aurora: RLS engine; DynamoDB: IAM condition keys; AgentCore: per-session microVM). OpenSearch isolation is exclusively application routing — the index name `tenant-{id}-brand-guidelines` is constructed by application code. A bug in FastAPI or the LangGraph retrieval node producing the wrong tenant_id queries another tenant's brand guidelines. There is no AOSS data access policy. The threat model in `risk-security-compliance.md §5.1` does not list any OpenSearch cross-tenant control.

**Fix:** Add an AOSS data access policy (IAM-enforced) restricting each execution role to its allowed index prefix (`tenant-{tenant_id}-*`). Document this in `risk-security-compliance.md §2.3` and `architecture-overview.md §4.5`. Update the threat model to include this mitigation.

---

### F-008 — brand_rules Table Has Dual Uncoordinated Write Paths

**Severity:** MAJOR · **Resolution:** design-level

**Gap:** MOD-004 (`PUT /api/v1/guardrails`) and MOD-005 (`POST /api/v1/brand-guidelines/ingest`) both write rows to `brand_rules` with overlapping `rule_type` values. `data-model.md` does not acknowledge dual ownership, specify whether ingest-extracted rules are additive to or override guardrail-configured rules, or define conflict resolution. The `guardrail_check_node` (MOD-003) reads `brand_rules` during every breach response — inconsistent state produces unpredictable guardrail behavior.

**Fix:** Update `data-model.md §2.5` to document dual write ownership and precedence. Add a `source` column (`enum: guardrails_api | ingest_pipeline`) to the `brand_rules` table. Specify in `application-flow.md Flow 6` whether extracted rules are merged with or replace existing guardrail settings.

---

### F-009 — MOD-000 demo_criteria Includes Demoable Auth Flow

**Severity:** MINOR · **Resolution:** design-level

**Gap:** MOD-000 is designated "non-demoable infra" in the module-map header. Its `demo_criteria` includes "Cognito login flow completes" — a demoable user interaction requiring a working Cognito User Pool, authorization code exchange, and returned JWT. This contradicts the non-demoable designation.

**Fix:** Replace "Cognito login flow completes" with "Cognito User Pool exists; `/auth/token` endpoint responds to valid code exchange" (smoke-test assertion, not a feature demo).

---

### F-010 — action-audit approver_id SOC2/GDPR Conflict Unresolved

**Severity:** MAJOR · **Resolution:** decision-level

**Gap:** `approver_id` stores a Cognito user sub. Under GDPR Recital 26, a pseudonymous identifier that can be re-linked to a natural person through the identity provider (Cognito) constitutes personal data. The design's proposed resolution ("legitimate interest / legal obligation overrides") is a legally contested position that varies across EU member state interpretations. No DPA guidance is cited. No pseudonymization alternative is offered.

**Required action (decision-level — user must choose):**

- (a) **Legal opinion path**: Obtain and document DPA guidance confirming legitimate-interest override for Cognito sub retention in SOC2 audit logs.
- (b) **Pseudonymization path**: At `audit_write` time, replace `approver_id` with a one-way HMAC of the Cognito sub keyed by a tenant-specific secret (stored in Secrets Manager). GDPR erasure erases the key, making all past HMACs irreversible. `approver_id` becomes non-personal under GDPR.

Update `data-model.md §4.1` and `risk-security-compliance.md §4.4` with the chosen path.

---

## Rework Roadmap

### Step 1 — User Decisions Required (4 decision-level findings)

Before any doc patching, the following require user input:

| Finding | Decision Needed |
|---------|----------------|
| F-001 | HITL approval resumption mechanism (spawn-new-session / wait-node / event-bridge) |
| F-002 | Session model (long-lived per-tenant / per-breach spawn) |
| F-006 | Token budget enforcement path (IAM quotas / in-node pre-call check) |
| F-010 | approver_id handling (DPA legal opinion / pseudonymization at write time) |

### Step 2 — Design-Level Patches (can run concurrently after decisions)

| Finding | Document(s) to Update |
|---------|----------------------|
| F-003 | `module-map.yaml` — add Aurora action_log write to MOD-003 db layer |
| F-004 | `module-map.yaml` — clarify memory_write_node ownership MOD-002/MOD-003 |
| F-005 | `data-model.md §4.1` — add breach_pct, spend_delta, network attributes |
| F-007 | `risk-security-compliance.md §2.3 + §5.1`, `architecture-overview.md §4.5` — AOSS policy |
| F-008 | `data-model.md §2.5`, `application-flow.md Flow 6` — brand_rules dual write |
| F-009 | `module-map.yaml` — fix MOD-000 demo_criteria |

---

## Provenance

| Section | Origin |
|---------|--------|
| All findings | Adversarial review by `aah-arch-review-board` agent, 2026-08-02, Cycle 1 |
| Evidence citations | `module-map.yaml`, `agent-topology.md`, `architecture-overview.md`, `data-model.md`, `application-flow.md`, `api-interface-contract.md`, `risk-security-compliance.md` |
