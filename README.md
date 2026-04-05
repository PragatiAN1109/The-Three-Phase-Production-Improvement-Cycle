# Your AI System Isn't Broken — It Just Can't Fix Itself

## The three-phase architecture that separates AI systems that degrade from AI systems that improve — and why no model upgrade can substitute for it

---

### The Forty-Minute Fix That Took Six Weeks

On March 14, a production multi-agent system processing financial documents for a midsize brokerage started getting things subtly wrong. Not dramatically wrong — not the kind of wrong that triggers alarms. The kind of wrong that hides in spreadsheets.

A retrieval agent pulling regulatory filings from a vector store began returning documents from the wrong fiscal year about 4.2 percent of the time. Downstream, a summarization agent folded those mismatched filings into analyst reports. Further downstream, an orchestrator routed the reports to clients. Eleven compliance filings were submitted to the SEC based on flawed summaries.

No alarm fired. No log turned red. The error rate had crept from 1.1 percent in January upward by about 0.3 points per week. An engineer glanced at the number during a Monday standup, said "retrieval noise," and moved on.

The monitoring stack was expensive and well-integrated. It faithfully recorded every request, every latency spike, every token count. It did exactly what it was built to do: *observe.*

What it did not do was *decide.* It never declared that 4.2 percent was unacceptable. It never distinguished a 1.1 percent baseline from a 4.2 percent degradation. It never triggered a debugging workflow, isolated the retrieval agent, or prevented the summarizer from consuming poisoned inputs. For six weeks, it watched.

By the time a junior analyst noticed conflicting effective dates in two client reports, the root cause had migrated through three layers of infrastructure. The retrieval agent's embedding index had drifted because a nightly reindexing job had silently started timing out — a downstream consequence of a database migration three sprints earlier that increased average document length by 18 percent. The reindexing job didn't fail outright. It timed out on the largest documents and succeeded on the rest, producing an index that was complete enough to avoid hard errors but skewed enough to quietly degrade retrieval precision.

The team spent nine days debugging. They first suspected the summarization agent's prompt template, recently updated. They rolled it back. No change. They audited three hundred orchestrator routing decisions. The routing was correct. On day seven, an engineer grepping through timeout logs found the correlation between document length and reindexing failure.

**The fix took forty minutes. The investigation took nine days. The system had been wrong for six weeks. The monitoring had been perfect the entire time.**

---

### The Question That Changes Everything

If you are watching everything and improving nothing, what exactly is your monitoring for?

---

### Observation Is Not Intervention

Most teams building monitoring for AI systems share an unexamined assumption: *if you can see what is happening, you can fix what is happening.* Visibility produces understanding; understanding produces action; action produces improvement. This is the folk epistemology of observability, and it works tolerably well for stateless, deterministic systems where a CPU spike has a small number of possible causes and a well-understood fix.

Multi-agent systems violate every premise of that model.

They are **stateful** — one agent's output becomes another's context, and that context accumulates across turns, sessions, and workflows. They are **non-deterministic** — the same input to the same agent can produce meaningfully different outputs depending on prompt construction, retrieval results, and sampling randomness. They are **compositional** — the system's behavior is the product of agent interactions, not the sum of individual behaviors. And they are **opaque at the boundaries** — the interface between a retrieval agent and a summarization agent is a natural language string, not a typed API contract, so degradation in one agent's output can propagate downstream as subtly wrong answers rather than type errors.

In such systems, visibility without decision logic is not monitoring. It is surveillance footage — useful after the crime, useless during it.

What the financial document system lacked was not data. It was *architecture.* Specifically, the three-phase structure that converts raw signals into enforced interventions: a monitor that evaluates health against explicit thresholds, a debugger that activates on threshold breach and isolates root causes through structured diagnosis, and an improver that modifies components based on validated diagnoses rather than guesses.

The distinction between a system that records its own behavior and a system that acts on its own behavior is the central concern of this chapter. It is not a tooling problem — you can build the wrong architecture with excellent tools. It is not a model quality problem — the most capable foundation model available will still degrade if nothing around it detects and corrects degradation.

**The claim: the capacity to improve is determined by architecture, and the specific architecture that enables improvement has three phases with thresholds as the binding constraint between them.**

---

### The Core Thesis

> Production multi-agent systems do not improve through observation alone. They require a threshold-driven **Monitor → Debug → Improve** architecture that converts signals into enforced intervention decisions. In the absence of explicit thresholds, gradual degradation is normalized as noise, causing delayed debugging, incorrect root-cause analysis, and ineffective fixes. The threshold — not the model, not the dashboard, not the engineer's intuition — is the mechanism that separates systems that degrade from systems that recover.

---

### The Three-Phase Architecture

The improvement cycle is not a suggestion. It is a control loop with formally distinct phases, each with defined inputs, outputs, and transition conditions. You do not enter the Debug phase because someone feels concerned. You enter it because a quantitative threshold has been exceeded. You do not enter the Improve phase because someone has a theory. You enter it because the Debug phase has produced a validated root-cause analysis with sufficient evidence to justify a specific change.

Each transition is gated. Each gate has a criterion. The criteria are set before the system runs, not discovered after it fails.

**MONITOR → (threshold breach) → DEBUG → (root cause report, confidence ≥ 0.7) → IMPROVE → (metric validated) → MONITOR**

With rollback: IMPROVE → (metric not validated) → DEBUG

With hold: DEBUG → (confidence < 0.7) → DEBUG (continue trace collection)

The three phases are not equal in duration or cost. The Monitor phase runs continuously at negligible marginal cost — it computes metrics over data already being collected. The Debug phase is expensive: trace retrieval, comparison, analysis. The Improve phase is cheapest in analysis but most consequential in risk, because it modifies a production system.

This cost asymmetry is why thresholds matter so much. Too tight, and you activate the expensive Debug phase constantly. Too loose, and degradation compounds until debugging is harder and fixes are riskier.

---

#### Phase One: Monitor

The Monitor phase computes a bounded set of health metrics, compares them against pre-defined thresholds, and emits one of three signals: **nominal, degraded, or critical.** That is all it does. It does not diagnose. It does not suggest fixes. It evaluates and classifies.

The health metrics for a multi-agent system are not web-application metrics. A multi-agent system can return HTTP 200 on every request while producing materially wrong answers, because the failure mode is *semantic*, not structural. The metrics that matter operate at three levels:

**Agent level.** Task completion rate, latency *distribution* (not just mean — a bimodal profile indicates intermittent tool failure, not general slowness), and retry rate (an early indicator of upstream degradation that appears long before completion failures).

**Coordination level.** Handoff fidelity (inter-agent messages arriving structurally complete and semantically coherent), routing accuracy (requests reaching the correct downstream agent), and context propagation integrity (whether information established in turn *n* is accurately represented in turn *n + k*).

**Outcome level.** End-to-end correctness (final outputs meeting a defined quality bar), user-facing error rate (failures, refusals, nonsensical outputs), and business-metric alignment (whatever downstream metric the system exists to serve).

> **The Scale Consciousness Principle:** Agent-level metrics operate at the scale of a single function call (milliseconds). Coordination-level metrics operate at workflow scale (seconds to minutes). Outcome-level metrics operate at business-process scale (hours to days). Degradation propagates upward through these scales with a delay that makes single-scale monitoring blind to the signal's origin.

Each metric has a threshold — not a target, but a *trigger*. In the financial document system, the retrieval agent's error rate baseline was 1.1 percent with a standard deviation of 0.4 percent. A threshold at two standard deviations above baseline — 1.9 percent — would have triggered the Debug phase in week two of degradation, not week six.

The choice of where to set the threshold is not intuition. It is arithmetic:

```
threshold = μ_baseline + k · σ_baseline

where k is chosen such that:
P(false_positive) · C_debug < P(missed_detection) · C_degradation
```

The difficulty is not the math. It is the organizational discipline to set the costs honestly and enforce the threshold they imply.

| Metric Level | Example Metric | Baseline (μ) | Threshold Method | Typical Trigger |
|---|---|---|---|---|
| Agent | Retry rate | 3.2% | μ + 2σ | ≥ 5.8% |
| Agent | P95 latency | 1.4s | μ + 2σ | ≥ 2.1s |
| Coordination | Routing accuracy | 94.0% | Percentile floor (P5) | ≤ 89.5% |
| Coordination | Handoff fidelity | 97.1% | Percentile floor (P5) | ≤ 93.8% |
| Outcome | End-to-end correctness | 91.3% | μ − 1.5σ | ≤ 87.0% |
| Outcome | CSAT | 4.1 / 5.0 | μ − 1.5σ | ≤ 3.85 |

*Table: Representative threshold configurations. Upward-bad metrics use upper thresholds; downward-bad metrics use lower thresholds. All values are illustrative and must be calibrated per system.*

Thresholds also require a **time window.** A single data point exceeding the threshold is a fluctuation, not a breach. A breach is the rolling average over a defined window exceeding the threshold value — typically 1–4 hours for agent-level metrics, 4–24 hours for coordination-level metrics, and 24–72 hours for outcome-level metrics. The window length matches the time scale at which degradation at that level is meaningful and distinguishable from noise.

One final point that is easy to undervalue: **thresholds must be versioned and stored as code**, not as dashboard configurations. When a threshold changes, that change must be tracked with the same rigor as a code change. A threshold silently loosened by an on-call engineer at 2 AM to suppress a noisy alert is a structural modification to the improvement architecture. If it is not reviewed and validated, it creates exactly the kind of blind spot that allowed the financial document system to degrade for six weeks.

---

#### Phase Two: Debug

The Debug phase is activated by a threshold breach, not by a meeting. Its purpose is root-cause isolation: given that metric *X* has exceeded its threshold at level *L*, identify the specific component, data flow, or interaction responsible.

This is the phase most teams skip or collapse into the Improve phase. It is the phase where incorrect root-cause analysis causes the most damage.

The fundamental problem: symptom and cause are usually at different levels, separated by time. The financial document system's symptom was at the outcome level (incorrect compliance reports). The cause was at the agent level (retrieval embedding drift), mediated by a coordination-level data flow (the retrieval-to-summarization handoff), and originating in an infrastructure event (the database migration) that preceded the symptom by three weeks.

A debugging process that starts at the symptom and works backward through the system's causal graph will find the root cause. A process that starts at the symptom and immediately hypothesizes about the most recently changed component — as the team did when they rolled back the prompt template — wastes time confirming the wrong hypothesis before finding the right one.

**Structured debugging follows a specific protocol:**

1. Identify the metric that breached its threshold and the level at which it operates.
2. Trace the data flow backward to upstream dependencies. For each, check whether its own agent-level metrics have also degraded. If yes, the cause is upstream — continue tracing. If no, the cause is at this level: the component receives healthy inputs and produces unhealthy outputs.
3. Once the component is isolated, examine its internal state — prompt template, tool configurations, data sources, recent execution traces. Compare degraded traces against baseline traces. The difference is the diagnosis.

```
// Debug Phase Protocol — Pseudocode

function debug(breached_metric, level):
    component = identify_owner(breached_metric)
    upstream_deps = get_upstream(component)

    for dep in upstream_deps:
        dep_health = check_agent_metrics(dep)
        if dep_health == DEGRADED:
            return debug(dep.breached_metric, dep.level)

    // All upstream deps healthy → fault is in this component
    baseline_traces = get_baseline_traces(component)
    degraded_traces = get_recent_traces(component, window="4h")
    diff = compare_traces(baseline_traces, degraded_traces)

    return RootCauseReport(
        component=component,
        fault_type=classify_diff(diff),
        evidence=diff,
        confidence=compute_confidence(diff)
    )
```

The output is not a fix. It is a **Root Cause Report** — a structured document identifying the component, describing the fault, presenting evidence as trace comparisons and metric deltas, and assigning a confidence score. If the confidence score is below 0.7, the Debug phase continues with additional trace collection rather than advancing to Improve with an uncertain diagnosis.

This is where discipline matters most. The pressure to "just fix something" is enormous when a system is degraded in production. But an Improve phase based on a wrong diagnosis does not fix the system — it introduces a second change into an already-degraded environment, making the next debugging cycle harder.

---

#### Phase Three: Improve

The Improve phase consumes a Root Cause Report and produces a specific, scoped change. Not a model upgrade. Not a general prompt rewrite. A targeted modification to the component identified in the Debug phase, designed to address the specific fault described in the report.

In the financial document case, the correct improvement was not "upgrade the retrieval model." It was: increase the reindexing job's timeout from 120 to 300 seconds, add a completeness check comparing documents in the new index against the old, and set a threshold on that completeness ratio at 0.98 below which the reindexing job fails and the old index is retained.

**The scoping is critical.** An Improve action broader than the diagnosis introduces uncontrolled variables. If you change the prompt template and the reindexing timeout simultaneously and the error rate drops, you do not know which change was responsible. If the error rate rises, you do not know which change made it worse.

The Improve phase operates under the same logic as a controlled experiment: one variable changes at a time, the effect is measured against the same metrics that triggered the Debug phase. If the metric returns to within baseline bounds, the improvement is validated. If not, the improvement is rolled back and the Debug phase resumes.

---

### Where the Clean Model Gets Messy

If the three-phase cycle were as clean in practice as described above, this chapter would be half as long. The model above assumes metrics are independent, degradation is monotonic, root causes are singular, and improvements are additive. Every one of these assumptions fails in production, and the failures reveal where the architecture must be *extended* rather than abandoned.

**Metric coupling.** Agent-level retry rate and coordination-level handoff fidelity are not independent. When a retrieval agent retries, the retry takes longer, shifting its latency distribution. The orchestrator's timeouts increase. Those timeouts register as coordination failures, even though the root cause is at the agent level. The Monitor phase, treating these as independent breaches, reports two problems when there is one. The solution: a **correlation layer** between Monitor and Debug that groups co-occurring breaches by temporal proximity and causal graph adjacency before passing them to the debugger.

**Intermittent degradation.** Not all degradation is a steady climb. Some is spiky — a retrieval agent fails badly on a specific query class while performing normally on others. The aggregate metric may not breach its threshold because the problematic class is a small fraction of traffic. But the users who issue those queries experience severe failure. The principle: **thresholds must be set at the granularity of the failure mode.** If the system can fail in ways invisible at the aggregate level, monitoring must operate at finer granularity.

**Multi-cause degradation.** In systems older than six months, two or more components commonly degrade simultaneously from independent causes. The Debug phase's trace-backward protocol must fork, and multiple root-cause analyses must proceed independently. This requires a queue of open investigations, not a single linear trace.

**Improvement interference.** This is the most pernicious complication. Fix one component, and the changed behavior alters data flowing through a second degraded component. The second component's pre-fix measurements may no longer be valid. The Improve phase must therefore re-validate *all* open Debug reports after each change — not only the metric associated with the change it just made. Skipping this step produces oscillatory behavior (fix A, break B, fix B, re-break A) that teams describe as "flaky" when the actual cause is an Improve phase that ignores cross-component coupling.

---

### Failure Case: The System That Optimized Itself Into Incoherence

> **Documented Failure — Normalization of Gradual Degradation**

A five-agent pipeline handled automated customer support at a SaaS company. An intake classifier routed tickets to domain-specific agents (billing, technical, account management), which drafted responses that a tone-adjustment agent polished before sending. The system processed 2,400 tickets per day with CSAT stable at 4.1 out of 5.0 for three months.

**Week one:** The billing agent's response latency climbed from 1.8 seconds to 2.3 seconds. The dashboard displayed a yellow dot — a cosmetic color rule someone had set during initial deployment. No threshold existed. No action was taken.

**Week two:** The intake classifier's routing accuracy dropped from 94 percent to 89 percent. Five percent of billing queries began arriving at the technical support agent, which lacked context to answer them correctly. It did not refuse — it attempted to answer, producing responses that were grammatically correct, structurally complete, and factually wrong. The tone-adjustment agent polished these wrong answers into professional-sounding wrong answers. Billing-ticket CSAT dropped from 4.1 to 3.6. Aggregate CSAT, diluted across all ticket types, dropped from 4.1 to 3.95. The weekly report read: "CSAT stable at ~4.0."

**Week three:** Follow-up tickets spiked from 8 percent to 14 percent. The team attributed this to a recent billing policy change. A reasonable hypothesis. Completely wrong. The spike came from misrouted billing tickets receiving incorrect technical answers. But nothing in the system's monitoring had told them *the system* was broken — so they looked for the explanation in the business.

**Week four:** The team "improved" the system by updating the tone-adjustment agent's prompt to be "more empathetic." This did not address the routing error or the latency increase. It made the wrong answers warmer. CSAT dropped to 3.7. Follow-up rate hit 19 percent.

**Week five:** The CTO intervened. A dedicated engineer discovered the routing degradation on day two of investigation by comparing the classifier's distribution against the three-month baseline. The classifier had been retrained on data containing 340 mislabeled billing tickets tagged as technical support. The fix: remove the mislabeled examples and retrain. Routing accuracy returned to 93 percent within 24 hours. The billing agent's latency issue was separately traced to a rate limit change by a third-party payment processor. A caching layer resolved it.

**Total time from first signal to resolution: 35 days. Estimated time with proper thresholds: 3–5 days.**

The thirty-day difference — degraded customer experience, one unnecessary prompt change to revert, and a CTO's attention diverted — is the cost of an architecture that observes without deciding.

**Four structural failures, each mapping to a missing element of the three-phase architecture:**

**First,** the latency threshold was cosmetic. A yellow dot is not a threshold breach — it is a color. A threshold breach is an event that activates a downstream process. The yellow dot activated nothing.

**Second,** no coordination-level metric existed for routing accuracy. The classifier's distribution was not compared against a baseline. The 5 percent shift was invisible because no one had defined "correct routing" quantitatively.

**Third,** the Debug phase was skipped entirely. The team went from observation ("CSAT is declining") directly to improvement ("make the tone agent more empathetic") without isolating the root cause. This is the most common failure pattern in production AI systems: the **Observe → Improve shortcut** that bypasses diagnosis.

**Fourth,** the improvement was broader than the diagnosis. "Make the tone agent more empathetic" is a change to a component that no diagnosis had identified as faulty. A properly scoped Improve phase would have required a Root Cause Report pointing at the classifier, not the tone agent.

---

### How This Connects to Structure–Property–Processing–Performance

The relationship between the Monitor → Debug → Improve architecture and the SPPP tetrahedron is direct, and making it explicit reveals why this architecture is a structural necessity rather than a best practice.

**Structure** is the agent graph: number of agents, roles, data flows, and contracts. Structure determines what kinds of failure are possible. A linear pipeline has different failure modes than a fan-out topology. The Monitor phase must match the structure — and when you change the structure, you must re-derive the metrics and recalibrate thresholds.

**Properties** are the system's observable behaviors — latency, accuracy, coherence, fidelity. These are what the Monitor phase measures. Properties are emergent: they arise from structure and processing interacting, not from any single component. An agent can be individually healthy while contributing to a systemically unhealthy property, just as a correctly manufactured rivet can be part of a structure that fails because the rivet pattern is wrong.

**Processing** is everything that transforms the system: training data, prompts, tool configurations, retrieval indices, deployment cadences. The Improve phase operates here. Every improvement is a processing change. The critical insight: processing changes modify the system's effective structure (a new prompt changes an agent's behavior, changing the data it sends downstream), which means the Monitor phase must re-validate properties after every change.

**Performance** is the business outcome the system exists to serve. Performance degrades → Monitor detects the property change → Debug traces it to a structural or processing cause → Improve applies a processing fix. Without this cycle, performance degradation is simply a fact the team observes and reacts to ad hoc.

> **The Threshold as a Structural Joint:** The threshold occupies the same position in this architecture that a grain boundary occupies in a polycrystalline material. It is the interface between two phases. It determines whether a signal propagates or is absorbed. A grain boundary that is too weak allows crack propagation; a threshold that is too loose allows degradation propagation. A grain boundary that is too strong creates brittle failure; a threshold that is too tight creates alert fatigue. The threshold is not an accessory to the architecture. It is the load-bearing joint.

---

### Threshold Design: The Quantitative Core

Everything in this architecture rises or falls on threshold quality.

Begin with a **baseline period**: run the system under normal conditions for at least two weeks to capture daily and weekly cycles. Compute mean and standard deviation for each metric at each level.

For approximately normally distributed metrics (latency, token counts, rate-based metrics over long windows), set the threshold at μ + kσ, where k balances sensitivity against false positives. In practice: *k* = 2 for most agent-level metrics, *k* = 1.5 for outcome-level metrics where degradation costs more.

For non-normally distributed metrics (many coordination-level metrics are bounded between 0 and 1 with skewed distributions), use percentile-based thresholds — the 95th or 99th percentile of the baseline distribution.

---

### Why Architecture, Not Model Quality, Determines Improvability

This is the chapter's most counterintuitive claim.

The prevailing instinct when a system produces bad outputs is that the model is the problem. Upgrade to a newer version. Fine-tune on more data. Switch to a larger context window. This instinct is wrong — not because model quality is irrelevant, but because model quality is a property of a single component, and the failure modes of multi-agent systems are properties of the architecture.

Consider two systems:

**System A** uses a state-of-the-art model in every agent slot, costs $2.40 per execution, and has no threshold-driven improvement cycle.

**System B** uses a model one generation older, costs $0.80 per execution, and implements the full Monitor → Debug → Improve architecture with calibrated thresholds at all three metric levels.

At deployment, System A is more accurate: 94 percent end-to-end correctness versus System B's 89 percent.

Six months later, System A has drifted to 88 percent. No one noticed until a quarterly business review.

System B has improved to 93 percent, because the improvement cycle caught and corrected four degradation events, two prompt regressions, and one retrieval index drift.

**The crossover happened at month four.** System B, with its inferior model, is now the more accurate system, and the gap is widening. System A is degrading because no architecture exists to detect and correct degradation. The model did not get worse — the world around the model changed. Upstream data sources shifted. User query distributions evolved. APIs modified their behavior. The model, frozen at deployment quality, could not adapt because adaptation requires detection, diagnosis, and targeted modification — the three phases — and System A has none of them.

```
Correctness(t) = Correctness(t₀) − δ_drift · t + Σ improvements(tᵢ)

System A: C(t) = 0.94 − 0.01t + 0
System B: C(t) = 0.89 − 0.01t + 0.01 · n_fixes(t)
```

Both systems experience the same drift rate (one percentage point per month, consistent with observed degradation in production RAG systems). System A has no improvement term. System B catches and fixes roughly one degradation event per month — a conservative estimate for calibrated thresholds — keeping its correctness stable or improving while System A monotonically declines.

**Model quality determines the starting point. Architecture determines the slope. Given enough time, slope dominates.**

You cannot buy your way to improvement with a better model. You can only build your way to improvement with a better loop.

---

### Chapter Summary

The difference between a multi-agent system that degrades and one that improves is not the quality of the models inside it. It is the presence or absence of a threshold-driven control loop that converts observability signals into enforced intervention decisions.

The **Monitor** phase computes bounded health metrics at three levels — agent, coordination, and outcome — and evaluates them against quantitative thresholds calibrated during a baseline period.

The **Debug** phase, activated only by a threshold breach, traces degradation backward through the system's causal graph to isolate the root cause with structured evidence rather than intuition.

The **Improve** phase applies a scoped, single-variable change based on a validated Root Cause Report and re-validates system health after the change.

Each transition is gated by a quantitative criterion: the threshold for Monitor → Debug, the confidence score for Debug → Improve, and metric recovery for Improve → Monitor. Without these gates, teams observe without deciding, guess without diagnosing, and change without validating — the pattern that turned a forty-minute fix into a six-week degradation in the financial document system and a three-day diagnosis into a thirty-five-day decline in the customer support pipeline.

**Architecture determines whether improvement is possible. Thresholds are the mechanism that makes the architecture work. Everything else — dashboards, models, trace logs, engineering talent — is necessary infrastructure. But infrastructure without architecture is surveillance without control.**

---

### Student Activities

**Problem 32.1 — Threshold Calibration Under Asymmetric Costs.** A four-agent pipeline processes insurance claims. The retrieval agent's baseline error rate is 2.3% with σ = 0.6%. An unnecessary Debug cycle costs approximately 8 engineering-hours. A week of undetected degradation costs approximately $45,000. Using the threshold equation, derive the optimal *k*, compute the threshold value, show your work and distributional assumptions, and explain what happens to optimal *k* if the cost of a false positive doubles.

**Problem 32.2 — Root-Cause Isolation Protocol.** Given: outcome-level correctness dropped from 91% to 84% over two weeks. Agent C (retrieval) retry rates increased from 5% to 11%. Handoff fidelity between Agent C and Agent D (summarization) dropped from 96% to 88%. Agent D's own metrics are nominal in isolation with known-good inputs. Apply the Debug protocol. Write the structured root-cause report. Identify at least one additional piece of evidence needed before advancing to Improve, and explain why.

**Problem 32.3 — Designing the Monitor Phase for a New Topology.** Design monitoring for an automated code review system with five agents: Intake (parses PRs), Static Analysis (linting/type-checking), Semantic Analysis (LLM-based logic/style evaluation), Conflict Detection (compares against open PRs), and Orchestrator (aggregates results). Draw the agent graph. For each agent and data flow, define a health metric, specify the threshold method, and justify the time window. Identify a failure mode invisible to agent-level monitoring alone, and explain which coordination- or outcome-level metric would catch it.

**Problem 32.4 — Improvement Interference Analysis.** Agent A (intent classifier, routing accuracy 95% → 88%) and Agent C (response generator, coherence 4.2 → 3.5/5.0) are simultaneously degraded. Fixing Agent A (retrain, accuracy restored to 94%) causes Agent C's coherence to drop from 3.5 to 3.3. Explain mechanistically why fixing A could degrade C. Describe whether to revert A's fix, fix C independently, or take a different approach. Justify quantitatively.

**Problem 32.5 — Open Design: The Self-Monitoring Agent.** Design a "meta-agent" implementing the Monitor phase autonomously within a multi-agent system. Define its prompt template, tool set, input schema, output schema (breach report format), and escalation criteria for when to hand off to a human operator. Identify at least two failure modes of the meta-agent itself and describe how you would monitor the monitor.

**Problem 32.6 — LLM-Assisted Debugging.** Design a prompt sequence for an LLM debugging assistant with access to 200 degraded traces and 200 baseline traces. The LLM must: (a) identify statistical differences, (b) generate three ranked root-cause hypotheses, and (c) propose a diagnostic test for each. Specify: what cognitive load the LLM reduces, what judgment the human must still perform, and the explicit decision point where the human evaluates LLM output before any production change. Produce prompt templates, expected output schemas, and justify where the human decision node is placed and why it cannot be removed.
