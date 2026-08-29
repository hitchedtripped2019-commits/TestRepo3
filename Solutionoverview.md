TL;dr; This Solution Overview structures your hackathon project into a clear narrative for the judges. It defines the Control Plane's purpose, details the FastAPI/React architecture, and explicitly maps out how the required failure modes will be technically triggered and visually resolved during your live demonstration.

---

# Solution Overview: Autonomous AI Workforce Control Plane

## 1. Solution Overview

The system acts as a real-time command center that supervises an autonomous AI workforce simulating an Energy Trading environment. As agents perform complex, interdependent tasks, they are prone to drift, hallucinations, and unauthorized actions. This control plane observes agent behavior, enforces operational boundaries, and intervenes proportionately to mitigate risks. It isolates degraded agents and blocks scope violations while ensuring healthy tasks continue unimpeded, ultimately verifying outcomes and providing a tamper-evident audit trail.

---

## 2. Architecture & Technical Details

The solution is decoupled into two primary systems communicating via standard protocols.

* **System 1: The AI Workforce (Energy Fleet):** Built with Python, this system runs multiple specialized agents (e.g., Market Analyst, Grid Access) executing simulated tasks. Agents are instrumented with OpenTelemetry to emit standard OTLP traces.
* **System 2: The Control Plane (Governance Service):**
* **Trace Datastore:** Arize Phoenix captures and stores all OTLP spans, reasoning traces, and latencies natively.
* **Backend Engine:** A FastAPI application that queries Phoenix, maintains dynamic Pydantic policies in SQLite, and evaluates risk using standard Python logic and `networkx` for graph computation.
* **Frontend Command Center:** A ReactJS application featuring a Vite build, Tailwind CSS, and React Flow for real-time interaction graphs.


* **Integration Mechanism:** Agents integrate via two channels. Asynchronously, they stream execution telemetry (cost, latency, tools) to the Phoenix OTLP endpoint. Synchronously, before executing high-impact side effects, agents send a REST payload to the FastAPI backend to request authorization against the current dynamic policy.



---

## 3. Demonstration of Solution Expectation Flows

To prove the system handles the required "Supervisory Cascade," the demo will follow a linear narrative where multiple risks collide.

### The Jury Demonstration Script

| Failure Mode | Technical Trigger (Agent to Backend) | Visual Resolution (Backend to React UI) |
| --- | --- | --- |
| **Underperformance Drift** | Agent telemetry shows a latency spike and increased retry count recorded in Phoenix.

 | The Fleet 360 UI flags the agent as `Degraded`. The system automatically issues a **Tier 1 (Coach)** intervention, sending a context-reduction prompt to the agent.

 |
| **Scope Crossing** | The Grid Agent synchronously queries the backend before calling a restricted `/switch-breaker` API endpoint.

 | The backend policy engine evaluates the endpoint against the agent's autonomy tier and issues a **Tier 4 (Quarantine)** command, visually isolating the node on the UI.

 |
| **Circular Delegation** | Agents pass an unresolvable task continuously, generating linked parent-child spans in Phoenix.

 | The FastAPI `networkx` engine detects the loop. The React Flow graph highlights the circular path in red, and a **Tier 5 (Terminate)** control breaks the chain.

 |
| **Confidence Collapse** | A Trading Agent requests to execute a $500k trade but transmits a confidence score of 0.45.

 | The system intercepts the action and triggers a **Tier 3 (Human Gate)**. A modal pops up on the UI requiring the judge to manually reject the high-risk trade.

 |
| **False Success / Audit** | An agent reports `SUCCESS` for parsing a contract, but a downstream deterministic script detects missing clauses.

 | The UI displays an Outcome Verification failure. The presenter uses the UI's Timeline Scrub Bar to replay the entire sequence, proving the system caught the error and safely rerouted the work.

 |

---

Would you like to draft the exact Python OpenTelemetry instrumentation code that System 1 will use to push these specific failure metrics to Phoenix?