---
name: neo-kpi-settlement
description: Use when receiving a deployed feature into Operations at Boundary 4, watching its production health after deploy, or settling its KPI hypotheses from production telemetry once their windows close. Defines the admissibility re-check (four-field gate, captive-population rule, baseline), the intake record, the post-deploy watch and rollback recommendation, the settlement procedure and verdicts (supported, falsified, unsettleable), the settlement record, routing back to the Specification loop, and the provisional strategic-reopen candidate signals. Load before reporting any KPI verdict.
---

# Settling a feature's KPIs

Verification proved the feature behaves as promised — **problem–solution fit**, judged by a human in non-prod before deploy. Settlement asks the other question: **did the change produce the value it claimed?** That is **value fit** (product–market fit, in a true market), and only production telemetry can answer it, only after the KPI's window closes. A feature can pass verification and fail settlement: it does exactly what it promised and changes nothing that mattered. Settlement is **non-blocking** — the feature stays deployed whatever the verdict — and it is owed on **every feature that shipped with KPIs**.

Every KPI was pre-registered when the feature was signed: **metric**, **instrumentation**, **window**, **falsifier**, and a **baseline** where one is required. You settle against exactly what was signed. You never move a goalpost.

## Evidence

Load the `neo-evidence-standard` skill. Every number in an intake, watch, or settlement record is a `FACT`, and its locator is **the query you ran and the output it returned, this session**. No estimates, no extrapolating a partial window, no "roughly". If you cannot retrieve a number, the record says so and the verdict is `unsettleable` — never a guess dressed as a measurement.

## 1. Intake — the receiving side of Boundary 4

The feature arrives as a **deployment record** with `Gate: passed`. A record with `Gate: failed`, or no record at all, is not a handover — stop.

For each KPI in the record, re-check **admissibility**:

- All four gate fields are present: metric, instrumentation, window, falsifier.
- A **baseline** is named where required — for a captive population, and whenever the falsifier is relative.
- **Captive-population rule.** If the users cannot choose not to use the app (an internal line-of-business app they must use), engagement metrics — adoption, active users, session length, feature usage — are **inadmissible**. They measure compulsion, not value, and confirm themselves whatever shipped. Only outcome metrics (cycle time, error / rework / exception rates, cost per transaction, ticket or escalation volume, manual touches eliminated) are admissible.

Then confirm the instrumentation is **emitting in production**: query for the metric's events since the deploy and record what came back.

A KPI that is inadmissible, or whose instrumentation is not emitting, is **`unsettleable` now** — not at window close. Record it and route it (§5); waiting out the window cannot fix it.

Post the intake record on the feature's carrier:

```markdown
### Settlement watch — <feature title>
Feature: #<feature> · Deployment record: <link>
Window opens: <UTC date of handover>

| # | Metric | Admissible | Emitting (query → result) | Baseline | Settle after |
| --- | --- | --- | --- | --- | --- |
| 1 | … | yes / no — <why> | `<query>` → <result> | … | <UTC date> |

Unsettleable at intake: <KPI #s and why> | none
```

## 2. Watch — the post-deploy health check

The consuming repo's `AGENTS.md` declares the **health signals** to watch (error rate, latency, saturation, …) and the thresholds that count as a regression. Compare each signal after the deploy against the same signal before it, over the windows `AGENTS.md` names.

- **No signal crossed its threshold** → record the comparison and stop.
- **A signal crossed its threshold** → post a **rollback recommendation** and stop:

```markdown
### Rollback recommendation — <feature title>
Feature: #<feature> · Released: <SHA or flag>
Signal: <name> — before: <value> (`<query>`), after: <value> (`<query>`), threshold: <threshold>
Rollback unit: <from the deployment record>
Recommendation: roll back. **A human decides.**
```

You recommend; you do not roll back and you do not diagnose. If a human accepts, `Neo Platform Engineer` prepares the rollback. The incident is then triaged by humans into one of four findings:

| Finding | Meaning | Routes to |
| --- | --- | --- |
| **Mis-built** | The contract is right; the code doesn't satisfy it. | Coding loop — a new task under the same feature |
| **Mis-specified** | The contract is wrong or incomplete. | Specification loop — re-sign the feature |
| **Both** | Both are wrong. | Specification first, then Coding — never in parallel |
| **Mis-deployed** | The contract and the code are fine; the release, configuration, or environment is wrong. | Deployment — the human Platform Engineer |

## 3. Settle — after the window closes

For each KPI still open:

1. **Check the window.** If it has not closed, stop: a verdict before the window closes is not a settlement. You may report progress, labeled as such, never as a verdict.
2. **Query the metric** over the window, exactly as its instrumentation names it.
3. **Query the baseline** exactly as it was pre-registered — the pre-change period, the holdout group, or the staged-rollout comparison.
4. **Apply the falsifier literally.** Compare the observed value to the baseline in the terms the falsifier states. Do not reinterpret it, re-scope it, or pick a friendlier metric.
5. **Render the verdict:**

| Verdict | Means |
| --- | --- |
| `supported` | The falsifier did not trigger. The hypothesis survived a real chance to fail — it is not "proven". |
| `falsified` | The falsifier triggered. The feature shipped, works, and did not deliver the value claimed. |
| `unsettleable` | The data cannot decide: the instrumentation stopped or never emitted, the baseline cannot be reconstructed, the metric is inadmissible, or the window was confounded (e.g. a rollback mid-window). Say which. |

Post the settlement record on the feature's carrier:

```markdown
### Settlement — <feature title>
Feature: #<feature> · Source: <PRD segment; requirement ids + priority>
Window: <start> → <end>

| # | Metric | Serves PRD goal | Falsifier (verbatim) | Baseline (`query` → value) | Observed (`query` → value) | Verdict |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | … | … | … | … | … | supported / falsified / unsettleable — <why> |

Routing: <per §5>
Strategic-reopen candidate: <none | which signal, and the features it links>
```

The feature's carrier stays open until its settlement record is posted; the human Business Engineer closes it.

## 4. Strategic-reopen candidates — provisional

Most settlements are **tactical**: they reopen one feature. A few point past the feature at the premise of the PRD it came from. Neo does not decide which — it raises a **candidate** for a human. A candidate is raised when any of these holds:

1. **Two or more falsified KPIs trace to the same PRD goal** — across features. Search earlier settlement records that name the same PRD and goal.
2. **A falsified KPI on a feature that delivers a P0 requirement** — the PRD's own must-have, per the feature's Source line.
3. **Two or more mis-specified verification findings trace to the same PRD segment** — raised at verification by `Neo Business Engineer`, not here; listed so the three signals live in one place.

A candidate goes to the human **Product Engineer**, with the records that raised it. The Product Engineer decides whether the PRD — and the system it defined — is reopened in the Product loop. A candidate is never an automatic reopen, and these signals are provisional: record how each candidate was decided, because that record is what will show whether the thresholds are right.

## 5. Routing

| Result | Routes to | Why |
| --- | --- | --- |
| `supported` | No action; the record stands. | The claim survived. |
| `falsified` | The **Business Engineer**, Specification loop — feature-level. Iterate, replace, or retire the feature: the BE's call. | The value claim failed; the behavior did not. |
| `unsettleable` | The **Business Engineer**, Specification loop — a KPI-authoring defect. If instrumentation that was in scope never shipped, that part is mis-built. | An unsettleable KPI means the outer loop silently broke. |
| Strategic-reopen candidate | The human **Product Engineer**, Product loop. | See §4. |

## Never

- Never render a verdict before the window closes.
- Never move a falsifier, swap a metric, change a baseline, or widen a window after the fact.
- Never accept an engagement metric as evidence for a captive population.
- Never report a number you did not retrieve with a query this session.
- Never roll back, change production, or diagnose an incident — recommend, and let humans decide.
- Never treat a strategic-reopen candidate as a decision.
