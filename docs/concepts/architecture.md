# Neo Agentic SDLC — Architecture

Terms in **bold** are defined in the [glossary](../glossary.md).

## What Neo is

Neo is spec-driven development with the spec unit shrunk from feature-sized to task-sized. Traditional spec-driven development treats a whole feature as one spec — too coarse a unit of work to reason about cleanly or to validate automatically. Neo keeps the business contract at the feature level and moves the _spec_ down to the **task**: bite-sized, technical, and machine-checkable.

Its second departure is one of emphasis. Spec-driven development centers the specification — but a specification is only instructions for humans and agents to follow while building code. The build is not the point; the proof is. **Specifications produce code; verification proves that code has value.** A spec that yields running code nobody needed has produced nothing. Neo treats verification — the business's judgment that a feature delivers what it promised — as the work that matters, and the specification as merely the means to it.

> Spec-driven development is dead. Long live specifications and verifications.

## The core rule

**Verify features, validate tasks. Humans verify, machines validate.**

Two axes, locked together:

| Level       | Proof            | Gatekeeper               | Against               |
| ----------- | ---------------- | ------------------------ | --------------------- |
| **Feature** | **Verification** | Human (**BE**) judgment  | the business contract |
| **Task**    | **Validation**   | Machine (tests + agents) | the spec              |

Proof is authored **when the unit is defined**, at every level:

- Feature defined → verification steps authored (BE)
- Task defined → validation criteria authored (BE + `task-planner`)
- Step defined (Coding loop) → commit

You can auto-validate a task-sized spec; you cannot reliably auto-validate a feature-sized one. The unit-of-work decision (task = spec) and the proof-mechanism decision (machine validates) are the same decision viewed twice.

## Hierarchy of work

**Problem / opportunity** → **PRD / Requirements** (segmented) → **Feature** (business, BE-signed) → **Task** (spec, ≈ 1 PR) → **Step** (≈ 1 commit).

The PRD is not an assumption. It is produced by the Product loop, which is where a problem
becomes a documented requirement.

## The four loops (Diagram 2, target end-state)

All four loops are built. Diagram 2 is the original drawing of them; where it and the text disagree, the text wins (see [process-flow.md § Related open items](./process-flow.md#related-open-items)).

1. **Product loop** `[live]` — problem/opportunity → **PRD**. Research fan-out, then the
   viability / desirability / feasibility lenses, then synthesis. Human-gated twice: the decision
   to proceed past synthesis, and the BE's acceptance of the PRD. Shipped by the optional
   `neo-product` plugin; the crew and its internals are documented in that plugin's README.
2. **Specification loop** `[live]` — problem space → solution space. Human-gated at both
   ends.
3. **Coding loop** `[live]` — one **Task** → intake → research → plan → implement and review
   (interleaved feature and test **steps**) → validate → draft **PR**. Machine-validated, human-gated
   at plan approval and at the PR. See [Coding loop in detail](#coding-loop-in-detail).
4. **Verification / Operations** `[live]` — the draft PRs → fan-in → non-prod deploy and smoke
   test → **verification** by the BE (user test) → release → CD and production smoke test →
   telemetry and KPI settlement. Human-judged at verification, human-performed at every merge and
   every change to production, machine-checked everywhere else. It runs through three spaces —
   Verification, Deployment, Operations — with the **Neo Business Engineer**, **Neo Platform
   Engineer**, and **Neo SRE** supporting their humans. See
   [Verification / Operations loop in detail](#verification--operations-loop-in-detail).

The **artifact that crosses each boundary** — including Boundary 0, where the PRD leaves the
Product loop — is owned by [process-flow.md](./process-flow.md).

## Specification loop in detail

### Feature definition

A feature is business-level and contains:

- **What** — a brief description.
- **Why** — justification for building it _now_.
- **KPIs** (optional) — falsifiable hypotheses, each naming its metric, instrumentation, window, and falsifier up front (see [process-flow.md § Falsifiability is a gate on KPI authoring](./process-flow.md#falsifiability-is-a-gate-on-kpi-authoring)). Settled from production telemetry after the window closes.
- **Verification steps** — business-executable in non-prod. **This is the contract.**

Entry to _ready-to-work_ requires What + Why + verification steps **and** BE sign-off. If the BE cannot verify it, it cannot deploy. Sign-off freezes the verification steps and KPI falsifiers: they are pre-registered, and changing one later is a re-sign.

### Feature → Task decomposition

Interactive and collaborative between the **BE** and the `task-planner` agent — never autonomous. A bad split poisons everything downstream, so the agent proposes a breakdown and surfaces its uncertainty; the BE converges and approves. "Done" is a BE-approved task set, not an agent-emitted one.

- **Strategy is chosen per feature, with the BE.** Default to logical chunks (vertical slices), not stack layers. Layer-based splits are legitimate only when the change is genuinely one layer. The stack skills (React, Web API, Bicep, …) serve the Coder _inside_ a task — they are not a decomposition template.
- **Sizing: one task ≈ one PR.** Too big → split; too small (can't stand as its own PR) → fold.
- **Validation criteria are authored at task creation** and must be machine-checkable — an assertion a test or agent can run to a deterministic pass/fail.

Governed by the `neo-task-authoring` skill (what a clean task _is_) and run by the `task-planner` agent (how to _carve_). Both ship for GitHub Copilot (`.github/`).

### PRD → Feature

One step upstream of Feature→Task, and interactive with the BE in the same way: the `feature-agent` drafts What, Why, optional KPIs, and verification steps from a PRD/requirements segment, governed by the `neo-feature-authoring` skill. It stops at a BE-signed feature and hands off to `task-planner` for decomposition — it does not decompose tasks itself.

## Coding loop in detail

One **Task** in, one draft **PR** out — the Specification loop's output is this loop's input. The
**Neo Technical Engineer** orchestrates; it delegates every phase and does none of the work itself.
The procedure is owned by `neo.technical-engineer.agent.md`; the boundaries it sits between are
owned by [process-flow.md](./process-flow.md).

1. **Intake.** The task must arrive whole — every field of the
   [task-handoff schema](../contributing/reference/task-handoff-schema.md) plus the `be-approved`
   marker. A task that fails intake goes back to the Specification loop; it is never patched here.
2. **Branch.** A task branch named from the task, cut from the project's **integration target**:
   the parent feature's integration branch under Mode A (the default), the default branch under
   Mode B. See [process-flow.md § Integration modes](./process-flow.md#integration-modes).
3. **Research.** `researcher`s fanned out in parallel, one question each, held to the
   `neo-evidence-standard`.
4. **Plan.** The `implementation-planner` breaks the task into **steps**, each labeled `[feature]` or
   `[test]` and one commit, plus a coverage map from every validation criterion to the test that
   proves it. **Human gate:** the plan is approved before any code is written — by the human, or by
   the Neo Business Engineer when it spawned the session. The Technical Engineer never approves its
   own plan.
5. **Implement and review.** The `code-writer` implements and commits one step at a time; the
   `code-reviewer` approves each step or returns findings to the writer. Testing is not a separate
   phase — test steps are interleaved with the feature steps they cover.
6. **Validate.** Once every step is approved, the `validator` runs the check behind each validation
   criterion on the branch head. Anything `fail` or `unproven` becomes a new step and loops back.
   This is the machine half of the core rule: **the task is validated, not reviewed into done**.
7. **Draft PR.** Opened against the integration target in the shape the `neo-pr-authoring` skill
   defines, carrying the validation report and the review ledger. A human takes it from there.

## Verification / Operations loop in detail

Many draft **PRs** in, one **Feature** proven twice out — once by a human before it ships
(**verification**, problem–solution fit), once by telemetry after (**KPI settlement**, value fit).
The two checks are different and stay separate; see
[process-flow.md § The two fits](./process-flow.md#the-two-fits). The boundaries the loop crosses —
3 and 4 — are owned by [process-flow.md](./process-flow.md); each space's procedure is owned by its
agent.

**Verification Space** — the Business Engineer, supported by the **Neo Business Engineer**:

1. **PR review.** A human reviews each task's draft PR and merges it into the integration target.
   No agent merges.
2. **Fan-in.** The Neo Business Engineer confirms every task in the feature's BE-approved set has
   landed — per the [integration mode](./process-flow.md#integration-modes).
3. **Non-prod deploy and smoke test.** The **Neo Platform Engineer** deploys the integration target
   at one pinned SHA, using the consuming repo's declared deploy command, and runs the smoke checks.
   A stale branch or a failed smoke check stops the loop before the BE is involved.
4. **Verify.** The BE runs each frozen verification step as written, then tries to break it; the
   verdict on every step is the BE's. On rejection, a recorded triage — mis-built, mis-specified,
   or both — routes the feature back, specification first. Owned by the `neo-feature-verification`
   skill.

**Deployment Space** — the Platform Engineer, supported by the **Neo Platform Engineer**:

5. **Release.** On a `verified` record: under Mode A, a draft feature PR whose body carries every
   child task and the traceability the squash commit needs; a human squash-merges it. Under Mode B,
   a human flips the flag. Owned by the `neo-release-authoring` skill.
6. **CD and production smoke test.** The project's own CD deploys; the Neo Platform Engineer watches
   it, runs read-only production smoke checks, and posts the **deployment record** — the artifact
   that crosses Boundary 4. It never deploys to production itself.

**Operations Space** — the Site Reliability Engineer, supported by the **Neo SRE**:

7. **Intake.** Each KPI is re-checked for admissibility and its instrumentation confirmed emitting
   in production; one that fails is unsettleable now, not later.
8. **Watch.** Post-deploy health against the consuming repo's declared signals. A regression becomes
   a rollback recommendation; a human decides, and the failure is triaged — with **mis-deployed**
   added to the three findings.
9. **Settle.** After each window closes, the pre-registered falsifier is applied literally to
   production telemetry: `supported`, `falsified`, or `unsettleable`. The verdict goes to the BE;
   a pattern that implicates the PRD itself goes to the Product Engineer as a strategic-reopen
   candidate. Owned by the `neo-kpi-settlement` skill.

The records the loop writes — verification, rejection, deployment, intake, settlement — all live on
the feature's carrier issue, which stays open until its KPIs settle.

## Key decisions

- **Task = spec.** The framework's central bet: a smaller spec unit is a machine-validatable one.
- **No hand-off BA.** The BE owns intent from PRD segment through decomposition — no transcribe-and-throw-over-the-wall.
- **Proof at definition.** Verification and validation are authored when the unit is created, not retrofitted later.
- **Logical chunks over layers.** Default decomposition is vertical slices that validate independently.

## Status

- **Live:** Product loop (`neo-product` — the Product Engineer, Product Researchers, and the three
  lenses); Specification-loop design; `neo-task-authoring` skill + `task-planner` agent, and
  `neo-feature-authoring` skill + `feature-agent`; Coding loop (`technical-engineer`, `researcher`,
  `implementation-planner`, `code-writer`, `code-reviewer`, `validator`, and the
  `neo-pr-authoring` skill); Verification / Operations loop (`business-engineer` for verification,
  `platform-engineer`, `sre`, and the `neo-feature-verification`, `neo-release-authoring`, and
  `neo-kpi-settlement` skills) (GitHub Copilot).
- **Still `[target]`:** redrawing the Diagram 2 PDF to match the text; stack plugins that supply
  platform-specific deploy and telemetry skills.

## Open threads

- **Root `AGENTS.md`** — the portable project backbone now exists and describes Neo itself (layout, the single-harness rule, checks). See the repo-root `AGENTS.md`.
