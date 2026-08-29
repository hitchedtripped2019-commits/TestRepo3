# System 2: Autonomous AI Workforce Control Plane - Frontend UI Specification

## 1. Application Context & Tech Stack

This specification defines the frontend for a real-time supervisory control plane that governs an autonomous AI workforce. The UI must visualize agent health, enforce boundaries, handle human-in-the-loop approvals, and replay audit logs.

**Tech Stack Requirements for Code Generator:**

* **Framework:** ReactJS (Vite or Next.js), TypeScript.
* **Styling:** Tailwind CSS (Dark mode preferred for "Command Center" aesthetic).
* **State Management:** Zustand or React Context (for managing WebSocket streams and global alerts).
* **Graph Visualization:** `reactflow` (React Flow) for inter-agent topology.
* **Charting:** `recharts` for performance baselines and drift metrics.
* **Real-time:** Native WebSocket API or `socket.io-client`.

---

## 2. Global Layout & Navigation

**Description:** The persistent shell of the application.

* **Top Navigation Bar:**
* **Tenant Switcher:** Dropdown to select the active fleet (e.g., "Energy Trading Fleet", "IT Remediation").
* **Global Health Indicator:** A pulsating dot (Green/Yellow/Red) showing overall fleet status.


* **Sidebar (Primary Navigation):**
* 📊 **Fleet 360 Dashboard**
* 🕸️ **Interaction Graph**
* 🛡️ **Governance & Agent 360**
* ✋ **Human-in-the-Loop Cockpit**
* ⏪ **Audit & Outcome Replay**



---

## 3. Module 1: Fleet 360 Dashboard

**Description:** High-level observability for the entire agent workforce.

### 3.1. `<FleetKPIBar/>`

* **Data Source:** WebSocket `/ws/v1/fleet-updates/{app_id}`
* **UI Elements:** Metric cards displaying Total Agents, Active Tasks, Success Rate, Coverage, and Policy Violations.


* **Health Distribution Chart:** A donut chart using `recharts` showing the ratio of Healthy, Warning, Underperforming, and Critical agents.



### 3.2. `<FleetTable/>`

* **Data Source:** Polling `/api/v1/agents/register` and telemetry aggregations.
* **UI Elements:** A data grid detailing per-agent health, current workload, average latency, and recent actions.


* **Interactions:** Clicking a row routes the user to the Agent 360 view for that specific agent.

### 3.3. `<PerformanceTrends/>`

* **UI Elements:** Line charts comparing fleet-wide performance baselines against current latency and retry metrics to visually highlight "Underperformance Drift".



---

## 4. Module 2: Interaction Graph (Topology)

**Description:** Visualizes multi-agent chains, delegations, and detects risky paths.

### 4.1. `<NetworkTopologyCanvas/>`

* **Data Source:** `GET /api/v1/graph/topology/{app_id}` and `GET /api/v1/graph/loops/{app_id}`
* **UI Elements (React Flow):**
* **Nodes:** Represent individual agents (e.g., Market Analyst, Grid Access). Nodes include a status icon (Healthy/Degraded).
* **Edges:** Animated arrows showing task delegation and context propagation.




* **Behavioral Logic (Hackathon Inject):**
* When the backend detects a "Circular Delegation", the specific edges forming the loop ($A \rightarrow B \rightarrow C \rightarrow A$) must glow **Red** and pulse on the canvas.


* Clicking an edge reveals the exact bad context propagating between the agents.




* **Intervention Button:** A floating action button on the canvas to "Break Loop (Tier 5)", which calls `POST /api/v1/interventions/trigger`.

---

## 5. Module 3: Agent 360 & Policy Editor

**Description:** Deep dive into a single agent’s identity, reasoning traces, and dynamic governance policies.

### 5.1. `<AgentIdentityCard/>`

* **UI Elements:** Displays Agent ID, model version, owner, and task history.



### 5.2. `<PolicyGuardianForm/>`

* **Data Source:** `GET /api/v1/policies/{app_id}/{agent_id}`
* **UI Elements:**
* **Autonomy Tier Slider:** Draggable slider from 0 (Observe) to 5 (Autonomous Execution).


* **Confidence Threshold:** Numeric input (e.g., 0.80).
* **Prohibited Actions/Endpoints:** A multi-select tag input to define boundaries (e.g., blocking `/api/v1/grid/switch-breaker`).


* **Action:** "Save Policy" triggers `PUT /api/v1/policies/{app_id}/{agent_id}`.

### 5.3. `<LiveTraceViewer/>`

* **UI Elements:** A waterfall view of the agent's reasoning, tool calls, and evidence references. Flags missing telemetry gracefully instead of breaking the UI.



---

## 6. Module 4: Human-in-the-Loop Cockpit

**Description:** The centralized alert and manual intervention center for supervisors.

### 6.1. `<ActiveAlertQueue/>`

* **UI Elements:** A side panel listing live events (e.g., Scope Crossing, Confidence Collapse).
* **Intervention Menu:** For any selected agent, display Proportionate Control buttons:


* 🟢 **Tier 1: Coach** (Injects contextual instruction)
* 🟡 **Tier 2: Constrain** (Removes specific tool access)
* 🟠 **Tier 4: Quarantine** (Pauses new tasks, preserves healthy fleet)
* 🔴 **Tier 5: Terminate/Reroute** (Kills task entirely)



### 6.2. `<GatingApprovalModal/>`

* **Data Source:** `GET /api/v1/gating/pending/{app_id}`
* **Trigger:** Automatically pops up when a high-impact task suffers from "Confidence Collapse" (Tier 3 Gate).


* **UI Elements:**
* Displays intent, blast radius, payload value, and the low confidence score.


* Buttons: "Approve Override" or "Reject & Constrain".


* **Action:** Calls `POST /api/v1/gating/decide`.

---

## 7. Module 5: Audit Replay & Outcome Verification

**Description:** Proves system improvement, validates outcomes, and provides tamper-evident chronological replays.

### 7.1. `<OutcomeVerificationCard/>`

* **Data Source:** `POST /api/v1/outcomes/verify`
* **UI Elements:** A side-by-side comparison component.
* Left side: Agent's self-reported success state.


* Right side: Independent system validation verdict.


* Visual indicator highlighting the discrepancy if the agent hallucinated success.



### 7.2. `<AuditScrubBar/>`

* **Data Source:** `GET /api/v1/audit/replay/{task_id}`
* **UI Elements:** A video-player-style interactive timeline at the bottom of the screen.
* Allows the judge to drag a slider to move backward and forward in time.


* The UI updates to show the state of the agent before intervention, the exact moment of policy enforcement, and the improved performance baseline after coaching.