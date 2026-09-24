---
name: neo-feature-authoring
description: Use when writing or reviewing a Feature during PRD-segment→Feature definition. Defines the required fields (What, Why, optional KPIs, verification steps), the falsifiability gate every KPI must pass (metric, instrumentation, window, falsifier — plus a baseline), the captive-population rule for internal line-of-business apps, the human-executable verification-steps format, and the BE sign-off gate that freezes them. Load whenever proposing, editing, or checking a feature's shape.
---

# Authoring a clean feature

A feature is the business-level unit of work. It derives from a PRD/requirements segment and is **the contract** the whole pipeline must satisfy to deploy — if the BE cannot verify it, it cannot ship.

## Required fields

- **Title** — business-level, imperative and specific.
- **Source** — the PRD/requirements segment this feature derives from. Downstream, a pattern of failures that traces back to one segment is how a problem with the PRD itself gets noticed.
- **What** — a brief description of the business-level change. No implementation detail, no stack layers, no task-sized breakdown — that's the Task Planner's job, one step downstream.
- **Why** — justification for building it _now_.
- **KPIs** (optional) — hypotheses about the value the feature will produce. Optional, but every KPI you do write must pass the falsifiability gate below.
- **Verification steps** — the contract. See below.

Do not author tasks or validation criteria here. A feature is decomposed into tasks by the Task Planner, collaboratively with the BE — that decomposition is a separate step, governed by the `neo-task-authoring` skill.

## KPIs — the falsifiability gate

A KPI is a hypothesis, and a hypothesis that cannot be falsified is an aspiration. It is settled in production, from telemetry, after its window closes — long after this feature was signed. Nothing written later can repair a KPI that was unsettleable from the start, so the gate is here, at definition time.

A KPI is **admissible only if all four are named**:

1. **Metric** — the specific quantity that moves.
2. **Instrumentation** — what emits it, and whether that telemetry exists today or must ship with the feature. If it must ship, **emitting it is in scope for this feature** — say so, so the Task Planner carves the task that emits it. A KPI whose instrumentation never ships can never be settled.
3. **Window** — how long after deploy until the hypothesis is settleable.
4. **Falsifier** — the observed result that would count as *disproved*, written before the build. If no observable outcome would falsify it, drop the KPI.

Two more lines travel with every KPI:

- **Baseline** — what the result is compared against: a pre-change measurement, a holdout group, or a staged rollout. **Required** for a captive population (below) and whenever the falsifier is relative ("15% below…"). Design it now — it usually cannot be reconstructed after the fact — and choose the window to accommodate it.
- **PRD success criterion** — the one the KPI serves, if the feature came from a PRD (or `none`).

Omit a KPI rather than invent one with no credible basis. A feature with no KPIs is legitimate; a feature with an unfalsifiable KPI is not.

### Captive populations — internal line-of-business apps

Ask whether the users can choose not to use this. Internal users of a mandated line-of-business app cannot: they have no alternative to defect to. Under that captivity, **engagement metrics are inadmissible** — adoption, active users, session length, and feature usage measure compulsion, not value. A mandatory workflow reaches full adoption whether it is excellent or miserable, so a KPI built on them confirms itself no matter what shipped.

Admissible KPIs for a captive population are **outcome** metrics — quantities that move only if the feature actually helped:

- cycle time / time-to-complete a task
- error, rework, and exception rates
- cost per transaction
- support ticket or escalation volume
- manual touches eliminated per case

A captive-population KPI **must** name a deliberate baseline — there is no churn signal and no competitor to compare against by default.

Before / after:

- ✗ `Increase adoption of the new claims screen to 90% in 30 days.` (engagement metric on a captive population; no falsifier; no baseline)
- ✓ `Metric: median time from claim intake to adjudication. Instrumentation: the existing claim_status_changed event. Window: 60 days. Falsifier: median intake-to-adjudication is not at least 15% below baseline. Baseline: the 60 days before deploy. Serves PRD success criterion SC-2.`

## Verification steps — the rules

Verification is **human judgment**, executed by the BE in a non-prod environment. This is the opposite proof mechanism from a task's validation criteria (machine-checked, no human judgment) — don't write a verification step that's really a validation criterion in disguise.

Each step must be:

- **BE-executable** — something a human can actually do and observe in a non-prod environment.
- **Tied to the business outcome** — proves the feature delivers what it promised, not an implementation detail.
- **A pass/fail judgment the BE renders** — not "run this test", which belongs at the task level.
- **Breakable** — specific enough that an attempt to make it fail means something. At verification the BE runs each step as written, then deliberately tries to break it; a step so vague that nothing could fail it proves nothing.

Before / after:

- ✗ `POST /api/checkout returns 201.` (machine-checkable — a task validation criterion, not a business verification)
- ✓ `As a shopper, complete checkout with a valid card in the staging environment and confirm the order appears in the order history with a paid status.` (a human runs this and judges it)

If every step you can write is really machine-checkable, the "feature" may already be task-sized — flag it back to the BE rather than forcing a verification step that doesn't fit.

## Sign-off gate — and what it freezes

A feature is **not** ready-to-work until it has What + Why + verification steps **and** explicit BE sign-off. No sign-off, no downstream decomposition — the Task Planner refuses an unsigned feature.

**Sign-off pre-registers the contract.** Once signed, the verification steps and each KPI's falsifier are frozen: they are what the feature is judged against, and they are not edited to fit what got built. Changing one later is a **re-sign** of the feature — a new contract — never a quiet edit. At verification, a step the BE wants to change is itself a finding that the feature was mis-specified.

## Template

```
Title:
Source (PRD segment):
What:
Why:
KPIs (optional — each must name all four gate fields, or be dropped):
  - Metric:
    Instrumentation: <what emits it; exists today | ships with this feature>
    Window:
    Falsifier: <the result that would count as disproved>
    Baseline: <pre-change measurement | holdout | staged rollout | none — only if not captive and the falsifier is absolute>
    Serves PRD success criterion: <id | none>
Captive population: <yes | no>
Verification steps:
  - <BE-executable, non-prod, pass/fail judgment>
  - <BE-executable, non-prod, pass/fail judgment>
Signed off by (BE):
```
