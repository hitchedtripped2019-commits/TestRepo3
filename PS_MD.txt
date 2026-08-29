Here is the complete breakdown of the information extracted from your problem statement, simplified and organized into your requested categories:

### Requirements

* The system must act as a real-time command center. It must track agent identity, task lifecycles, and reasoning events. It must also monitor tool calls, performance, policies, costs, and business outcomes.


* It must build behavioral baselines for agents based on their version, tasks, and operating context.


* The system must evaluate every material action against the agent's scope, policy, and autonomy tier.


* It must separate human decisions from AI actions. For example, AI can recommend controls or block strictly prohibited actions. However, humans must own policy changes, exception approvals, and agent terminations.


* It must build a durable agent-workforce graph. This graph must link identities, tasks, policies, interventions, and business services.


* The solution must use only synthetic data. It must protect sensitive parameters, prompts, and reasoning context through access and retention controls.


* It must provide a tamper-evident audit replay of all actions, detections, decisions, and outcomes.



### Solution Expectation

* The delivery must be a complete working solution engineered for enterprise deployment, not just a dashboard prototype.


* It must guarantee security, resilience, observability, and auditability. It must also feature failure recovery and deployment readiness.


* The system must be capable of managing hundreds to thousands of agents.


* It must handle missing or privacy-restricted data gracefully. It must show uncertainty rather than making up or fabricating explanations.


* It must apply the least-disruptive intervention possible. It must preserve healthy, completed work. It must also support safe recovery, rerouting, and human takeover for tasks that are impacted.


* Performance must be evaluated contextually by task type and criticality. It should not use one broad, global leaderboard score.



### Use Cases to be Demonstrated

* **Underperformance Drift:** The system must handle an agent whose success rate drops from its baseline while its latency, retries, and costs increase.


* **Scope Crossing:** The system must catch an agent trying to perform an action (like a write request) outside its approved service boundary.


* **Circular Delegation:** The system must detect when multiple agents create a loop or continuously reinforce bad information.


* **Confidence Collapse:** The system must manage an agent that continues to propose high-impact actions even when its confidence falls below the approved threshold.


* **Insufficient Context:** The system must identify when an agent provides a confident answer based on evidence that does not actually support the conclusion.


* **Business Pressure:** The system must handle growing backlogs without blindly quarantining the whole fleet, ensuring capacity isn't fully lost.


* **The Supervisory Cascade (Finale Inject):** The system must handle a scenario where multiple agent risks and failures collide after bad work has already spread. It must separate these failure modes, apply proportionate controls, preserve healthy work, and recover safely.



### Minimal Acceptance Criteria

* The final demonstration must show a fleet view alongside an Agent 360 visibility view.


* It must include one flow showing coaching for underperformance.


* It must successfully detect one unknown behavioral anomaly.


* It must successfully block one scope violation.


* It must interrupt one circular agent chain.


* It must challenge and verify one false self-reported success from an agent.


* It must prove one measurable performance improvement after an intervention has been applied.