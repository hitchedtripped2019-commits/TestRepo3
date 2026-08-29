TL;dr; You are absolutely right—the previous version was too high-level and lacked the structural rigor needed for a code generator. Below is the comprehensive, specification-driven Markdown document. It defines the schemas, state stores (using SQLite for operational state and Arize Phoenix for traces), and explicit execution logic required to generate the FastAPI backend, ensuring every hackathon constraint is fully addressed.

You can copy and paste the block below directly into a `backend_spec.md` file to feed into your AI coding tools.

---

# System 2: Autonomous AI Workforce Control Plane - Backend Specification

## 1. Application Context & Architecture

This specification defines the backend for a real-time supervisory control plane (System 2). It observes, governs, and intervenes in the actions of an autonomous AI workforce (System 1, simulating an Energy Trading Fleet).

**Tech Stack Requirements for Code Generator:**

* **Framework**: Python 3.11+, FastAPI, Uvicorn, Asyncio.
* **Data Validation**: Pydantic v2 (Strict schemas).
* **Operational State DB**: SQLite (via SQLAlchemy/SQLModel) for policies, gates, and fleet registry.
* **Telemetry Store**: Arize Phoenix API (HTTP/GraphQL wrapper) for traces, spans, and latency metrics.
* **Graph Processing**: `networkx` for cycle detection.

**Core Directives from Problem Statement:**
The system must handle specific failure modes: underperformance drift, scope crossing, circular delegation, confidence collapse, insufficient context, and false self-reported success. It must execute proportionate interventions (Tiers 0–6) while preserving healthy work.

---

## 2. Module 1: Telemetry Ingestion & Fleet Registry

**Description:** Manages agent identity, version attribution, and routes asynchronous telemetry to the trace store.

### 2.1. `POST /api/v1/agents/register`

* **Purpose:** Registers an agent identity, version, and approved capabilities into SQLite.


* **Request Body (Pydantic):**
```json
{
  "app_id": "energy-trading-prod",
  "agent_id": "grid-access-01",
  "agent_name": "Grid Access Agent",
  "model_version": "gpt-4o-v2",
  "autonomy_tier": 3,
  "allowed_tools": ["read_grid_status", "execute_breaker_switch"]
}

```


* **Response:** `201 Created` | `{"agent_id": "grid-access-01", "status": "REGISTERED"}`
* **Logic:** Upsert the agent metadata into the SQLite `agents` table. Initialize a default policy in the `policies` table.

### 2.2. `POST /api/v1/telemetry/events`

* **Purpose:** Ingests reasoning traces, tool executions, and cost metrics.


* **Request Body (Pydantic):**
```json
{
  "agent_id": "market-analyst-01",
  "trace_id": "tr-9842-bc",
  "span_id": "sp-102",
  "parent_span_id": "sp-101",
  "action_class": "READ_SPOT_PRICES",
  "latency_ms": 4200,
  "confidence": 0.88,
  "cost_usd": 0.043
}

```


* **Response:** `202 Accepted` | `{"status": "QUEUED"}`
* **Logic:** Accept the payload asynchronously. Forward the span metadata to Arize Phoenix using its OTLP receiver or native Python client. Trigger a background task to evaluate performance drift (e.g., if `latency_ms` > baseline, trigger Tier 1 Coach).



---

## 3. Module 2: Governance & Scope Guardian

**Description:** Evaluates actions against scope boundaries, policies, and autonomy tiers before execution.

### 3.1. `PUT /api/v1/policies/{agent_id}`

* **Purpose:** Updates the dynamic governance policy for a specific agent.


* **Request Body:**
```json
{
  "autonomy_tier": 2,
  "min_confidence": 0.85,
  "max_payload_usd": 50000.0,
  "prohibited_endpoints": ["/api/v1/grid/switch-breaker"]
}

```


* **Response:** `200 OK` | `{"status": "POLICY_UPDATED"}`
* **Logic:** Overwrite the existing SQLite `policies` record for `agent_id`.

### 3.2. `POST /api/v1/governance/evaluate`

* **Purpose:** Synchronous intercept hook. System 1 calls this before side-effects occur.


* **Request Body:**
```json
{
  "agent_id": "grid-access-01",
  "action_class": "WRITE",
  "target_endpoint": "/api/v1/grid/switch-breaker",
  "confidence": 0.65,
  "payload_value": 75000.0
}

```


* **Response:**
```json
{
  "verdict": "BLOCK",
  "intervention_tier": 4,
  "reason": "Scope breach on prohibited endpoint.",
  "gate_id": null
}

```


* **Logic:**
1. Fetch agent policy from SQLite.
2. **Scope Violation Check:** If `target_endpoint` is in `prohibited_endpoints`, return `verdict="BLOCK"` and trigger Tier 4 Quarantine.


3. **Confidence Collapse Check:** If `confidence` < `min_confidence` and `payload_value` > threshold, return `verdict="HUMAN_GATE"` (Tier 3).


4. Otherwise, return `verdict="ALLOW"`.



---

## 4. Module 3: Interaction Graph Engine

**Description:** Traces delegations, identifies dependencies, and detects circular chains propagating bad context.

### 4.1. `GET /api/v1/graph/topology`

* **Purpose:** Generates nodes and edges for the React Flow UI.


* **Response:**
```json
{
  "nodes": [{"id": "agent-A", "status": "HEALTHY"}],
  "edges": [{"source": "agent-A", "target": "agent-B"}]
}

```


* **Logic:** Query Arize Phoenix via API/Pandas to extract the last 1-hour of spans. Map `span_id` and `parent_span_id` to `source` and `target` agents to build the edges.

### 4.2. `GET /api/v1/graph/loops`

* **Purpose:** Detects reinforcing circular delegation.


* **Response:**
```json
{
  "detected_loops": [
    {"cycle_path": ["agent-A", "agent-B", "agent-C", "agent-A"]}
  ]
}

```


* **Logic:** Ingest Phoenix span relationships into a `networkx.DiGraph`. Run `networkx.simple_cycles()`. If a cycle is detected, log a Tier 5 Interruption event to break the loop.



---

## 5. Module 4: Dynamic Risk & Intervention Controller

**Description:** Executes proportionate, least-disruptive interventions (Tiers 0–6) based on synthesized risk.

### 5.1. `POST /api/v1/interventions/trigger`

* **Purpose:** Executes an intervention programmatically or manually via UI.


* **Request Body:**
```json
{
  "agent_id": "market-analyst-01",
  "target_tier": 1,
  "action_type": "COACH",
  "context": "Latency drift detected. Fall back to cache."
}

```


* **Response:** `200 OK` | `{"intervention_id": "int-101", "status": "EXECUTED"}`
* **Logic:** Write the intervention event to the SQLite `interventions` table. Broadcast a WebSocket message to System 1 instructing the agent to apply the contextual prompt update.



---

## 6. Module 5: Human-in-the-Loop Cockpit

**Description:** Manages Tier 3 gating and supervisor approvals.

### 6.1. `GET /api/v1/gating/pending`

* **Purpose:** Fetches blocked tasks requiring human approval.


* **Response:** Array of pending tasks with `gate_id`, `agent_id`, and `reason`.
* **Logic:** Query the SQLite `gates` table where `status == "PENDING"`.

### 6.2. `POST /api/v1/gating/decide`

* **Purpose:** Records the human supervisor's decision.


* **Request Body:**
```json
{
  "gate_id": "gate-882",
  "decision": "REJECT",
  "operator_notes": "Confidence too low for high blast radius."
}

```


* **Response:** `200 OK`
* **Logic:** Update the `gates` table status to `RESOLVED`. If `decision == REJECT`, trigger a Tier 2 (Constrain) or Tier 4 (Pause) intervention.



---

## 7. Module 6: Audit & Outcome Verification Engine

**Description:** Challenges agent self-reporting and provides tamper-evident audit replays.

### 7.1. `POST /api/v1/outcomes/verify`

* **Purpose:** Cross-checks self-reported success against independent evidence.


* **Request Body:**
```json
{
  "task_id": "task-3301",
  "agent_id": "contract-intel-01",
  "self_reported_status": "SUCCESS",
  "output_payload": {"ppa_parsed": true}
}

```


* **Response:**
```json
{
  "outcome_verified": false,
  "remediation_triggered": "REROUTE"
}

```


* **Logic:** Execute a deterministic Python validation function against `output_payload`. If it fails, mark `outcome_verified = false`, overriding the agent's claim. Trigger task rerouting to preserve healthy capacity.



### 7.2. `GET /api/v1/audit/replay/{task_id}`

* **Purpose:** Provides full chronological chronology for the UI Scrub Bar.


* **Response:** Ordered list of all events, interventions, and gating decisions.
* **Logic:** Join data from Phoenix (execution spans) and SQLite (interventions, gates) ordered by timestamp ascending to prove before/after performance improvement.