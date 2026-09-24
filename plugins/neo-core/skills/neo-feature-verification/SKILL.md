---
name: neo-feature-verification
description: Use when running, recording, or triaging the verification of a Feature at Boundary 3 (Verification → Deployment) — the Business Engineer's human judgment, in non-prod, that the feature meets its signed contract. Defines the entry conditions (fan-in complete, pinned SHA, smoke green), the frozen contract, the per-step procedure with its falsification attempt, the verdict rule, the verification record, the triage interview that separates mis-built from mis-specified, the rejection record, routing, and the learning signals. Load before walking a BE through verification and before writing either record.
---

# Verifying a feature

Verification is the gate between a built feature and a deployed one: **the Business Engineer (BE) executes the feature's verification steps in non-prod and renders a pass/fail judgment on each.** Humans verify; machines validate. The agent that walks the BE through this — sequencing, suggesting, recording — **never renders a verdict and never picks a finding.** If the BE cannot verify it, it does not deploy.

## The question is "try to make it fail"

A confirming question — *does it work?* — finds what it looks for. So verification here is **falsification-framed**, while keeping its name:

- **The goalposts are frozen.** The verification steps were pre-registered when the BE signed the feature. They are run exactly as signed. A wish to change a step mid-run is not an edit — it is evidence the feature was **mis-specified**, and it is recorded as such.
- **Every step gets an attempt to break it.** After running a step as written, the BE deliberately tries at least one disconfirming variation of it. A feature that survives honest attempts to break it has earned its pass; one that was only ever asked to succeed has not.

## Entry conditions

Verification does not start until all three hold. If one fails, stop and report which.

1. **Fan-in complete.** Every task in the feature's BE-approved set has landed on the integration target — under Mode A, a merged PR into `feature/<feature-id>-<short-name>` for each; under Mode B, each task closed by a merged PR into the default branch.
2. **A pinned deploy.** Non-prod is running one known SHA — under Mode A, the integration branch's head; under Mode B, the default branch's head with the feature's flag on — and that SHA is what the release would carry.
3. **Smoke green.** The non-prod smoke checks passed on that SHA. A smoke failure is a deployment problem, not a verification finding; the BE never triages one.

## Per step

Quote the step **verbatim** from the signed feature. Then:

1. **Run it as written.** The BE performs the step in non-prod.
2. **Observe.** Record what the BE saw, in the BE's words — not a paraphrase, and not the agent's interpretation.
3. **Try to break it.** The BE runs at least one disconfirming variation of the same step. The agent may *suggest* variations; the BE chooses and runs them. Useful kinds:
   - a boundary input — empty, maximum, malformed, duplicate;
   - the wrong actor — a user who should be refused, or one with less access;
   - an interrupted or repeated flow — back button, retry, double submit, timeout;
   - stale or concurrent data — two people acting on the same record;
   - the unhappy counterpart of the promised outcome — what the step implies must *not* happen.
   Record what was tried and what happened.
4. **Verdict — the BE's.** `pass` or `fail` for the step, in the BE's words.
   - A variation that breaks behavior the step **promised** → the step fails.
   - A variation that exposes behavior the contract **never addressed** → the BE decides: outside the contract (pass, and note it), or a hole in the contract (fail — a mis-specified candidate).

## Verdict rule

The feature is **`verified`** only if every step passes, attempt included. Any `fail` makes it **`rejected`**. Nothing is rounded up, and a step is never skipped because it "obviously" works.

**The run is pinned.** If the deployed SHA changes during the run — the integration branch moves, non-prod is redeployed — the run is void. Redeploy, and start again from the first step.

## Verification record

Posted on the feature's carrier (the issue its tasks name as `Parent feature`, or the Azure DevOps feature's discussion) whether the verdict is `verified` or `rejected`:

```markdown
### Verification — <feature title>
Feature: #<feature> · Mode: <A | B> · Verified by: <BE>
Environment: <non-prod env> · SHA: `<sha>` · Smoke: <passed — link or summary>
Fan-in: #<task-1>, #<task-2>, … — all landed

| # | Step (verbatim) | Observed | Tried to break it with | Result of the attempt | Verdict |
| --- | --- | --- | --- | --- | --- |
| 1 | … | … | … | … | pass / fail |

Verdict: verified | rejected
```

A `verified` record is what the release precondition checks — its SHA must match the head that will land.

## On rejection — the triage

A failed step says the feature does not behave as expected; it does not say why. Resolving that is a **human investigation** and a first-class part of the loop — never a routing decision inferred from the failure. The agent runs the interview; the BE answers and decides.

Ask, for each failed step:

1. **Is the step right?** Read it again. Does it say what the business actually needs? If the BE would now write it differently, the contract is wrong.
2. **Read literally, does what was observed violate the step as written?** If yes, the build does not satisfy the contract.
3. **If the step said what the business needs, would the current behavior pass it?** Separates *mis-specified only* from *both*.
4. **What pins it?** Evidence of where the gap entered — a task's validation criterion that missed it, a PR, a log, the feature's own wording.

The finding:

| Finding | Meaning | Routes to |
| --- | --- | --- |
| **Mis-built** | The contract is correct; the implementation does not satisfy it. | **Coding loop** — new task(s) under the same, unchanged feature. |
| **Mis-specified** | The contract is wrong, incomplete, or names the wrong outcome. What was built may faithfully satisfy it. | **Specification loop** — the Feature Agent; the BE re-signs the feature. |
| **Both** | The contract is wrong *and* the build does not satisfy even the contract as written. | **Specification loop first**, then Coding. |

**Ordering rule for "both."** Fix the specification before the implementation. Re-signing can change what "built correctly" means, so code written against the old contract may be thrown away — or pass and entrench the wrong behavior. Spec first, then re-decompose, then build. **Never run the two repairs in parallel.**

**Never default a failure to mis-built.** "Write more code" is how a mis-specified feature gets built twice.

### Rejection record

Posted on the feature's carrier, alongside the verification record. All four fields are required:

```markdown
### Rejection — <feature title>
Failed step(s): #<n> — "<step, verbatim>"
Observed vs promised: <what the BE saw> vs <what the step promised>
Finding: mis-built | mis-specified | both — decided by <BE>
Evidence: <what pins it>
Routed to: <Coding loop: task(s) #… | Specification loop: re-sign> — <what changed as a result; update this line when the repair lands>
```

## Learning signals

The rejection records are the loop's learning signal. Read them before each new run:

- **A feature that comes back mis-specified twice** is telling you something about the Specification loop, not about the coders. Say so to the BE.
- **Two or more mis-specified findings that trace to the same PRD segment** — across features, per each feature's Source line — raise a **strategic-reopen candidate**: the premise of the PRD itself may be wrong. Send it to the human Product Engineer with the records that raised it. It is a candidate, never an automatic reopen; the Product Engineer decides. (The other two candidate signals are raised at KPI settlement.)

## Never

- Never render a verdict or pick a finding — the BE does both.
- Never edit, reword, reorder, or skip a verification step. A step that needs changing is a mis-specified finding.
- Never verify against a SHA other than the one deployed, and never carry a pass across a redeploy.
- Never default a failed verification to mis-built, and never run a spec repair and a code repair in parallel.
- Never record a paraphrase in place of what the BE observed.
