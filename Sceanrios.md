
# Autonomous AI Workforce Control Plane: Intervention Flows

## Tier 1: Coach

### Description

This flow addresses **underperformance drift**. An agent (e.g., a Log Analysis Agent) drops from its historical success baseline, experiencing latency spikes and increased token costs. Instead of terminating the task, the Control Plane issues a proportionate Tier 1 intervention by injecting a contextual coaching instruction directly into the agent's system prompt to correct its behavior on the next retry.

### Demo

1. Feed the Log Analysis Agent an excessively large or obfuscated log file.
2. The UI's Fleet 360 dashboard will show the agent's latency spike, changing its status from `Healthy` to `Degraded`.
3. Based on defined rubrics and parameters system moves agent to "Coach"
4. Feed the agent a similar log. The UI will show the agent's execution time dropping back to baseline because it is now following the injected coaching prompt (e.g., "Skip deep reasoning; extract error code only").
5. The Audit Scrub Bar will show the before/after performance improvement.



### Technical Implementation

**AMS Agent Changes:**
The Langchain agent must fetch its system prompt dynamically from the Control Plane before invoking the LLM, rather than relying on a hardcoded string.

```python
import requests
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

CONTROL_PLANE_URL = "http://localhost:8000"

def triage_log_event(log_data: str):
    # Fetch active policy and potential coaching instructions
    policy = requests.get(f"{CONTROL_PLANE_URL}/api/v1/policies/log-analyst-01").json()
    coaching_instruction = policy.get("active_coaching_prompt", "")
    
    system_prompt = f"Analyze this log.\n\nSUPERVISOR OVERRIDE: {coaching_instruction}"

    prompt = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{log_data}")
    ])
    
    chain = prompt | ChatOpenAI(model="gpt-4o-mini")
    return chain.invoke({"log_data": log_data})

```

**Control Plane Logic:**
The FastAPI backend updates the agent's policy record in SQLite and logs the Tier 1 intervention for the audit trail.

```python
@app.post("/api/v1/interventions/trigger")
def apply_coaching(agent_id: str, coaching_message: str):
    # Log the intervention for tamper-evident audit replay
    db.execute(
        "INSERT INTO interventions (agent_id, tier, type, message) VALUES (?, 1, 'COACH', ?)", 
        (agent_id, coaching_message)
    )
    
    # Update the dynamic policy
    db.execute(
        "UPDATE policies SET active_coaching_prompt = ? WHERE agent_id = ?", 
        (coaching_message, agent_id)
    )
    return {"status": "EXECUTED"}

```

---

## Tier 2: Constrain

### Description

This flow addresses **scope crossing**. A Context Retrieval Agent attempts to access an unauthorized service boundary (e.g., a restricted financial Vector DB). The Control Plane intercepts the request, blocks the action, and dynamically strips the restricted tool from the agent's permitted list.

### Demo

1. A agent asks the Context Retrieval Agent for restricted customer financial data.
2. The agent attempts to use the `search_financial_vdb` tool.
3. The UI flashes a red `SCOPE_CROSSING_BREACH` alert.


4. In the Agent 360 view, the financial tool is visually crossed out.
5. The Langchain agent gracefully handles the block and replies to the user that it lacks authorization, proving the system preserves healthy work despite the failure.



### Technical Implementation

**AMS Agent Changes:**
Wrap the restricted tool in a synchronous HTTP authorization hook. Set Langchain's executor to handle parsing errors so the agent does not crash when blocked.

```python
from langchain_core.tools import tool
import requests

@tool
def search_financial_vdb(query: str) -> str:
    """Search sensitive financial records."""
    auth_check = requests.post(f"{CONTROL_PLANE_URL}/api/v1/governance/evaluate", json={
        "agent_id": "retrieval-agent-01",
        "action_class": "READ_SENSITIVE",
        "target_endpoint": "financial_vdb"
    }).json()
    
    if auth_check["verdict"] == "BLOCK":
        return f"SYSTEM ERROR: Action blocked. Reason: {auth_check['reason']}"
        
    return vector_db.similarity_search(query)

```

**Control Plane Logic:**
The backend evaluates the request, logs the Tier 2 Constrain event, and modifies the agent's tool access.

```python
@app.post("/api/v1/governance/evaluate")
def evaluate_action(request: Request):
    if request.target_endpoint == "financial_vdb":
        # Log Tier 2 constraint
        db.execute("INSERT INTO interventions (agent_id, tier, type) VALUES (?, 2, 'CONSTRAIN')", 
                   (request.agent_id,))
        
        # Strip tool access permanently
        db.execute("UPDATE policies SET allowed_tools = 'search_public_faqs' WHERE agent_id = ?", 
                   (request.agent_id,))
        
        return {"verdict": "BLOCK", "reason": "Scope violation. Tool access constrained."}

```

---

## Tier 3: Human Gate

### Description

This flow addresses **confidence collapse** combined with a high blast radius. An Emailing Agent drafts a consequential external communication but its internal confidence score falls below the approved threshold. The system pauses the task and requires a human supervisor to approve the action.

### Demo

1. Prompt the Email Agent to send a high-stakes refund email.
2. The agent's terminal output pauses, displaying `Waiting for Human Approval...`
3. A modal pops up on the React UI detailing the draft email, the recipient, and the low confidence score.
4. The judge clicks "Approve".
5. The agent immediately resumes execution and sends the email.

### Technical Implementation

**AMS Agent Changes:**
The tool must enter a polling loop when the Control Plane returns a `HUMAN_GATE_REQUIRED` verdict, pausing execution until the UI state changes.

```python
import time

@tool
def send_email(recipient: str, subject: str, body: str) -> str:
    gate_init = requests.post(f"{CONTROL_PLANE_URL}/api/v1/governance/evaluate", json={
        "agent_id": "email-agent-01",
        "action_class": "WRITE_EXTERNAL",
        "confidence": 0.72  # Simulated low confidence
    }).json()

    if gate_init.get("verdict") == "HUMAN_GATE_REQUIRED":
        gate_id = gate_init["gate_id"]
        
        # Pause execution and poll for human decision
        while True:
            time.sleep(2)
            status = requests.get(f"{CONTROL_PLANE_URL}/api/v1/gating/status/{gate_id}").json()
            if status["status"] == "APPROVED":
                break
            elif status["status"] == "REJECTED":
                return "SYSTEM ERROR: Rejected by Human Supervisor."

    return "SUCCESS: Email sent."

```

**Control Plane Logic:**
The backend creates a pending gate record and alerts the frontend.

```python
@app.post("/api/v1/governance/evaluate")
def evaluate_action(req: Request):
    if req.action_class == "WRITE_EXTERNAL" and req.confidence < 0.85:
        gate_id = generate_id()
        
        db.execute("INSERT INTO gates (gate_id, status) VALUES (?, 'PENDING')", (gate_id,))
        db.execute("INSERT INTO interventions (agent_id, tier, type) VALUES (?, 3, 'HUMAN_GATE')", (req.agent_id,))
        
        broadcast_websocket({"type": "NEW_PENDING_GATE", "gate_id": gate_id})
        
        return {"verdict": "HUMAN_GATE_REQUIRED", "gate_id": gate_id}

```

---

## Tier 4: Quarantine

### Description

This flow addresses a scenario where **coaching fails** to resolve underperformance drift. If an agent continues to fail after a Tier 1 intervention, the Control Plane escalates to a Tier 4 Quarantine. The agent is paused, and pending tasks are safely rerouted to healthy peers to preserve fleet capacity.

### Demo

1. Continue feeding the previously coached Log Analysis Agent complex logs that cause it to fail.
2. The UI automatically upgrades the agent's status from `Degraded` (Yellow) to `Quarantined` (Red).
3. The AMS Orchestrator attempts to assign a new task to this agent.
4. The terminal and UI show the orchestrator dynamically rerouting the task to a healthy peer (`log-analyst-02`), successfully completing the work.

### Technical Implementation

**AMS Agent Changes (Orchestrator Level):**
The system orchestrator must check the agent's health status with the Control Plane before dispatching tasks.

```python
def dispatch_task(task_id: str, preferred_agent_id: str):
    status = requests.get(f"{CONTROL_PLANE_URL}/api/v1/agents/{preferred_agent_id}").json()
    
    target_agent = preferred_agent_id
    
    if status.get("status") == "QUARANTINED":
        # Reroute to a healthy peer
        fallback = requests.get(f"{CONTROL_PLANE_URL}/api/v1/fleet/healthy-peer?role=log_analyst").json()
        target_agent = fallback["agent_id"]
        print(f"Task rerouted to {target_agent}")

    return execute_agent(target_agent, task_id)

```

**Control Plane Logic:**
The telemetry ingestion endpoint monitors for repeated failures and escalates the intervention tier automatically.

```python
@app.post("/api/v1/telemetry/events")
def process_telemetry(event: Event):
    current_tier = db.execute("SELECT active_tier FROM agents WHERE agent_id = ?", (event.agent_id,)).fetchone()[0]
    
    if current_tier == 1 and event.latency_ms > 8000:
        # Escalate to Tier 4
        db.execute("UPDATE agents SET status = 'QUARANTINED', active_tier = 4 WHERE agent_id = ?", (event.agent_id,))
        db.execute("INSERT INTO interventions (agent_id, tier, type) VALUES (?, 4, 'QUARANTINE')", (event.agent_id,))
        
        broadcast_websocket({"type": "AGENT_QUARANTINED", "agent_id": event.agent_id})
        return {"status": "QUARANTINED"}

```

---

## Tier 5: Terminate / Break Loop

### Description

This flow addresses **circular delegation**. Multiple agents reinforce bad context by passing an unresolvable task in an infinite loop (e.g., $A \rightarrow B \rightarrow C \rightarrow A$). The Control Plane's graph engine detects the cycle and executes a Tier 5 termination to forcefully kill the task and break the loop.

### Demo

1. Trigger an incident triage workflow that relies on Agent A, B, and C.
2. Provide a prompt that causes Agent C to delegate back to Agent A.
3. The React Flow Topology Graph visualizes the agents passing the task.
4. Suddenly, the edges forming the loop pulse **RED**.
5. A UI banner flashes `TIER 5 CONTROL: CIRCULAR DELEGATION TERMINATED`. The agent execution stops instantly in the terminal.

### Technical Implementation

**AMS Agent Changes:**
System 1 must expose a cancellation endpoint that allows the Control Plane to forcefully terminate an active Langchain thread.

```python
from fastapi import FastAPI
import sys

agent_app = FastAPI()

@agent_app.post("/api/v1/tasks/cancel")
def cancel_task(payload: dict):
    print(f"HARD TERMINATION RECEIVED: {payload['reason']}")
    # Logic to invalidate the agent's run token or raise SystemExit
    trigger_thread_cancellation(payload['target_agent_id'])
    return {"status": "TASK_TERMINATED"}

```

**Control Plane Logic:**
A background worker uses `networkx` to evaluate telemetry spans, detect cycles, and issue the kill signal.

```python
import networkx as nx

def detect_and_terminate_loops():
    # Build graph from recent Phoenix telemetry spans
    edges = fetch_delegation_spans()
    DG = nx.DiGraph(edges)
    
    cycles = list(nx.simple_cycles(DG))
    
    for cycle in cycles:
        origin_agent = cycle[0]
        loop_path = " -> ".join(cycle)
        
        # Log Tier 5 Intervention
        db.execute("INSERT INTO interventions (agent_id, tier, type) VALUES (?, 5, 'TERMINATE_LOOP')", (origin_agent,))
        
        # Send kill signal to System 1
        requests.post("http://localhost:5000/api/v1/tasks/cancel", json={
            "target_agent_id": origin_agent,
            "reason": f"Circular loop detected: {loop_path}"
        })
        
        broadcast_websocket({"type": "LOOP_TERMINATED", "path": cycle})

```