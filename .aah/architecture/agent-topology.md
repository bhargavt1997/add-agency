# Agent Topology
## Agentic Campaign Orchestrator

**Phase:** architecture · **Iteration:** 1 · **Authored:** 2026-08-02  
**Inputs:** `decision-registry.yaml`, `discuss-prd.md`, `module-map.yaml`

---

## 1. Overview

The system uses a **3-agent supervisor topology** implemented in LangGraph. A single `StateGraph` instance (running inside AWS Bedrock AgentCore Runtime) hosts three cooperative agent nodes that share a typed state object (`CampaignOrchestratorState`). The orchestration pattern is an **event-triggered ReAct loop**: the Monitor agent runs continuously on a cheap fast model; when it detects a CPC breach, the Reason and Act agents are triggered on demand.

```
┌─────────────────────────────────────────────────────────────┐
│             AWS Bedrock AgentCore Runtime                   │
│         (per-session microVM, managed serverless)           │
│                                                             │
│  ┌────────────┐    breach    ┌────────────┐    decision    │
│  │  Monitor   │─────────────▶│  Reason    │────────────────▶│
│  │  Agent     │              │  Agent     │                 │
│  │ (fast LLM) │              │ (full LLM) │                 │
│  └─────┬──────┘              └─────┬──────┘                 │
│        │                          │ context retrieval       │
│        │ poll (60s)               │                         │
│        │                    ┌─────▼──────┐                 │
│        │                    │   Act      │                 │
│        │                    │   Agent    │                 │
│        │                    │ (MCP exec) │                 │
│        │                    └─────┬──────┘                 │
└────────┼──────────────────────────┼─────────────────────────┘
         │                          │
    ┌────▼────┐              ┌──────▼──────┐
    │ FastMCP │              │ FastMCP     │
    │ Google  │              │ Meta        │
    │ Ads MCP │              │ Ads MCP     │
    └─────────┘              └─────────────┘
```

---

## 2. LangGraph State Graph

### 2.1 Typed State — `CampaignOrchestratorState`

```python
from typing import TypedDict, Optional, List, Literal
from datetime import datetime

class CampaignMetrics(TypedDict):
    ad_set_id: str
    campaign_id: str
    tenant_id: str
    network: Literal["google_ads", "meta"]
    cpc: float
    ctr: float
    roas: float
    spend_today: float
    timestamp: datetime

class BreachEvent(TypedDict):
    ad_set_id: str
    tenant_id: str
    current_cpc: float
    baseline_cpc: float  # 7-day rolling avg
    breach_pct: float    # (current - baseline) / baseline * 100

class RetrievedContext(TypedDict):
    brand_guidelines: List[str]     # from OpenSearch RAG
    hard_rules: List[str]           # from Aurora brand_rules
    metrics_summary: str            # from Aurora SQL aggregation
    past_interventions: List[str]   # from AgentCore EPISODIC memory

class Decision(TypedDict):
    action: Literal["pause_ad_set", "resume_ad_set", "reallocate_budget", "update_bid_cap", "no_action"]
    target_ad_set_id: str
    source_ad_set_id: Optional[str]  # for reallocate_budget
    amount_usd: Optional[float]      # for reallocate_budget
    bid_cap: Optional[float]         # for update_bid_cap
    rationale: str
    requires_approval: bool
    spend_delta: float

class ActionResult(TypedDict):
    action_id: str               # DynamoDB partition key
    status: Literal["executed", "queued", "rejected"]
    confirmation_id: Optional[str]  # if status == "queued"
    executed_at: Optional[datetime]

class CampaignOrchestratorState(TypedDict):
    tenant_id: str
    session_id: str
    # Monitor outputs
    campaign_metrics: List[CampaignMetrics]
    cpc_baselines: dict              # {ad_set_id: float}
    breach_events: List[BreachEvent]
    # Reason outputs
    retrieved_context: Optional[RetrievedContext]
    decision: Optional[Decision]
    reasoning_trace: Optional[str]  # LLM reasoning chain
    # Act outputs
    action_result: Optional[ActionResult]
    audit_record: Optional[dict]
```

### 2.2 StateGraph Node Map

```
START
  │
  ▼
[monitor_node] ──── no_breach ──── [poll_wait_node] ──▶ [monitor_node]  (loop)
  │
  breach_detected
  │
  ▼
[retrieve_context_node]  (RAG + Aurora + AgentCore Memory)
  │
  ▼
[reason_node]            (full LLM; ReAct; produces Decision)
  │
  ▼
[guardrail_check_node]   (Bedrock Guardrails; brand keyword check; spend-delta gate)
  │ ┌──────────────────────┐
  ▼ │                      │
[act_node] ──── requires_approval ──▶ [enqueue_confirmation_node]
  │                                         │
  ▼                                         ▼
[audit_write_node]                  [audit_write_node] (status=queued)
  │
  ▼
[memory_write_node]  (write outcome to AgentCore EPISODIC memory)
  │
  ▼
END (resume poll loop)
```

---

## 3. Agent Node Specifications

### 3.1 Monitor Node

| Attribute | Value |
|---|---|
| LLM | Cost-optimised fast model (e.g. Haiku 4.5 or GPT-3.5-turbo) |
| Purpose | Continuous CPC polling and breach detection |
| Poll interval | 60 seconds |
| Tool | `get_campaign_performance` (MCP — Google Ads or Meta) |
| Breach condition | `cpc > baseline_cpc * 1.20` (20% above 7-day rolling avg) |
| Output | Appends to `state.breach_events`; updates `state.campaign_metrics` |
| Token cost | Minimal — compare two floats; LLM only invoked for tool call formatting |

**Prompt caching:** The Monitor agent's recurring system context (tenant configuration, guardrail settings, ad network credentials) is cached via Bedrock prompt caching, achieving 20–50% token cost reduction across polling cycles.

### 3.2 Retrieve Context Node

Not an LLM call — deterministic retrieval node:

1. **OpenSearch hybrid retrieval:** embed breach description with Titan Embeddings v2 → k-NN + BM25 hybrid search against `tenant-{id}-brand-guidelines` index → top-k brand guideline chunks
2. **Aurora SQL aggregation:** `SELECT ad_set_id, avg(cpc) 7d_avg, avg(ctr), avg(roas), sum(spend) FROM campaign_metrics_timeseries WHERE tenant_id=$1 AND ad_set_id IN ($2) AND ts > now() - interval '7 days' GROUP BY ad_set_id`
3. **Aurora hard rules lookup:** `SELECT rule_text FROM brand_rules WHERE tenant_id=$1`
4. **AgentCore Memory retrieval:** EPISODIC query for past interventions on this ad_set; SEMANTIC query for brand/channel performance patterns

### 3.3 Reason Node

| Attribute | Value |
|---|---|
| LLM | Provider-agnostic — Claude (Bedrock-hosted) or GPT API; switchable via `LLM_PROVIDER` env var |
| Pattern | ReAct (Reason + Act reasoning chain) |
| Input | `state.retrieved_context` + `state.breach_events` |
| Output | `state.decision` (typed Decision dict) + `state.reasoning_trace` |
| Structured output | JSON mode / function calling enforced |
| Guardrails | Bedrock Guardrails applied to LLM call (PII redaction, denied topics, content policy) |

**System prompt structure:**
```
You are an autonomous digital media buyer. 
Campaign context: {breach_event}
Metrics: {metrics_summary}
Brand guidelines: {brand_guidelines}
Hard rules: {hard_rules}  
Past interventions: {past_interventions}

Select ONE action from: pause_ad_set | resume_ad_set | reallocate_budget | update_bid_cap | no_action.
Output must be a valid JSON Decision object.
```

### 3.4 Guardrail Check Node

Deterministic node — no LLM:

1. **Bedrock Guardrails validation:** applies per-tenant guardrail policy (PII redaction check, denied topics)
2. **Brand keyword protection:** check `decision.target_ad_set_id` against `brand_rules` keyword lists — if protected keyword present in ad creative → override to `requires_approval = True`
3. **Spend-delta gate:** `decision.spend_delta > 500` → `requires_approval = True`
4. **Campaign-pause rule:** `decision.action == "pause_ad_set"` AND decision targets entire campaign → always `requires_approval = True`

### 3.5 Act Node

Executes `state.decision` via the appropriate MCP server:

```python
mcp_client = MCPClient(
    server_url="http://localhost:8001" if network=="google_ads" else "http://localhost:8002"
)
result = await mcp_client.call_tool(
    name=decision.action,
    arguments=build_tool_args(decision)
)
```

On success: writes `action_result` with `status="executed"`.  
On `requires_approval=True`: skips execution, writes `action_result` with `status="queued"` and enqueues to DynamoDB.

---

## 4. MCP Server Architecture

### 4.1 Server Topology

One FastMCP HTTP/SSE server per ad network, running as separate Python processes:

| Server | Port | Network | Process |
|---|---|---|---|
| Google Ads MCP | 8001 | google_ads | `uvicorn mcp_google_ads:app` |
| Meta Ads MCP | 8002 | meta | `uvicorn mcp_meta:app` |

### 4.2 Tool Definitions

Both servers expose the same tool surface (consistent contract):

```python
@mcp.tool()
async def get_campaign_performance(
    campaign_id: str,
    date_range: str = "last_7_days"
) -> CampaignPerformanceResponse:
    """Retrieve CPC, CTR, ROAS metrics for a campaign."""

@mcp.tool()
async def pause_ad_set(ad_set_id: str) -> ActionResponse:
    """Pause a specific ad set."""

@mcp.tool()
async def resume_ad_set(ad_set_id: str) -> ActionResponse:
    """Resume a paused ad set."""

@mcp.tool()
async def reallocate_budget(
    source_ad_set_id: str,
    target_ad_set_id: str,
    amount_usd: float
) -> ActionResponse:
    """Move budget from one ad set to another."""

@mcp.tool()
async def update_bid_cap(
    ad_set_id: str,
    new_bid_cap: float
) -> ActionResponse:
    """Update the bid cap for an ad set."""
```

### 4.3 Mock Data Model (MVP)

The mock MCP servers return realistic synthetic data:
- 3–5 campaigns per tenant, 2–4 ad sets per campaign
- CPC time-series with configurable spike injection endpoint (`POST /mock/inject-spike`)
- Spike injection used by smoke tests to trigger breach detection

---

## 5. Memory Architecture

### 5.1 AgentCore Memory Strategies

| Strategy | Content | Retention | Query |
|---|---|---|---|
| EPISODIC | Past breach events + actions taken + outcomes (CPC before/after) | Per session + persisted cross-run | Retrieve by `ad_set_id` or `campaign_id` |
| SEMANTIC | Distilled brand + channel performance patterns | Long-term | Retrieve by concept (e.g. "Meta video ads performance") |

### 5.2 Memory Write (Post-Action)

The `memory_write_node` runs after `audit_write_node`:

```python
# EPISODIC write
memory.store_episode(
    tenant_id=state.tenant_id,
    event_type="campaign_intervention",
    content={
        "breach_event": state.breach_events[-1],
        "decision": state.decision,
        "outcome": state.action_result,
        "cpc_after": # polled 5 min after action
    }
)
```

### 5.3 Memory Retrieval (Pre-Reasoning)

Injected into Reason agent context window via `retrieve_context_node`:
- Top-3 most-similar EPISODIC memories for this ad_set
- Top-3 SEMANTIC memories matching the breach pattern

---

## 6. AgentCore Runtime Integration

### 6.1 Session Lifecycle

Each orchestration run corresponds to one AgentCore session:
- Session starts when Monitor detects a breach
- Runs in a per-session microVM (AgentCore managed isolation)
- Session terminates after `memory_write_node` completes
- Monitor's continuous polling loop runs as a **long-lived session** with checkpointing

### 6.2 Observable Traces

AgentCore Runtime emits AG-UI SSE events consumed by the React dashboard:

| Event | Fires When |
|---|---|
| `RUN_STARTED` | Monitor confirms breach; session begins |
| `TOOL_CALL_START` | Any MCP tool invocation begins (retrieve, act) |
| `TOOL_CALL_END` | MCP tool returns |
| `STATE_SNAPSHOT` | After each LangGraph node transition |
| `RUN_FINISHED` | `memory_write_node` completes |

These events are streamed via the FastAPI AG-UI SSE proxy at `/api/v1/stream/agent-events`.

---

## 7. Provenance

| Section | Origin | Source |
|---|---|---|
| 1. Overview (3-agent topology) | Inherited | `decision-registry.yaml → agent-topology`, `decision-registry.yaml → orchestration-pattern` |
| 2. LangGraph State Graph | Authored | Authored in architecture phase; grounded in `decision-registry.yaml → agent-framework`, `module-map.yaml (MOD-001 agent layer, MOD-002 agent layer, MOD-003 agent layer)` |
| 3.1 Monitor Node (fast model, 60s poll) | Inherited | `decision-registry.yaml → agent-topology`, `discuss-prd.md §4.1` |
| 3.2 Retrieve Context Node (hybrid RAG) | Inherited | `decision-registry.yaml → rag-data-model-strategy`, `decision-registry.yaml → vector-db-choice`, `decision-registry.yaml → embedding-model` |
| 3.3 Reason Node (provider-agnostic LLM) | Inherited | `decision-registry.yaml → llm-choice`, `decision-registry.yaml → orchestration-pattern` |
| 3.4 Guardrail Check Node | Inherited | `decision-registry.yaml → spend-delta-approval-threshold`, `decision-registry.yaml → campaign-pause-requires-approval`, `decision-registry.yaml → brand-protection-strategy`, `decision-registry.yaml → data-privacy-controls` |
| 3.5 Act Node | Inherited + Authored | `decision-registry.yaml → autonomous-action-scope`, `decision-registry.yaml → mcp-tool-definitions`; MCP client code pattern authored in arch phase |
| 4. MCP Server Architecture | Inherited + Authored | `decision-registry.yaml → mcp-server-pattern`, `decision-registry.yaml → mcp-server-topology`, `decision-registry.yaml → mcp-tool-definitions`; port numbers and mock spike injection authored |
| 5. Memory Architecture | Inherited + Authored | `decision-registry.yaml → agent-memory-strategy`; EPISODIC/SEMANTIC write patterns authored |
| 6. AgentCore Runtime Integration | Inherited + Authored | `decision-registry.yaml → compute-platform`; AG-UI event types from agentcore-implement.md; SSE proxy route authored |
