# Neo Glossary

Canonical vocabulary for the Neo Agentic SDLC. Define a term once here and link to it from other docs, agents, and skills rather than restating it.

**Status key:** `[live]` designed and drafted this cycle · `[target]` part of the end-state design (Diagram 2), not yet specced.

## Roles

**Agents work for humans.** Every seat below comes in a pair: an unprefixed name is a real
human being, and its **Neo**-prefixed counterpart is the agent that supports that human in the
role. The human owns the judgment calls and the sign-offs; the agent does the legwork and holds
the gates open. Never the reverse, and never conflate the two — "Business Engineer" is always a
person, "Neo Business Engineer" is always software.

### Human roles

**Product Engineer** `[live]` — The human who stares down an open problem and finds the shape of something worth building. Same seat as "product manager" or "product lead" — the one who turns a hunch, a complaint, or a raw opportunity into a product other people can build a future on.

**Business Engineer** `[live]` — The human who turns intent into action. You take what the business actually needs and shape it into something an engineer can build — without losing the *why* in the handoff. Same seat as "the business," "business analyst," and the scrum "product owner" — but never the transcription-and-toss-it-over-the-wall BA. You author the feature contract, you sign it, and you stay in the room through Feature→Task decomposition, because the person who understands the intent is the person who should decide what "done" means.

**Technical Engineer** `[live]` — The human who turns a plan into working software and stands behind it. Same seat as "the developer" or "the engineer" — the craftsperson who carries a task from spec to draft PR, making the calls no spec can fully anticipate.

**Platform Engineer** `[live]` — The human who owns the road to production. Same seat as "DevOps engineer" or "release engineer" — the one who decides what reaches production and when, owns the pipelines and environments, and makes the rollback call. Owns **Deployment Space**.

**Site Reliability Engineer (SRE)** `[live]` — The human who owns the running system. Watches production health, calls regressions, and settles whether a shipped feature delivered the value it claimed. Owns **Operations Space**.

### Agent roles

**Neo Business Engineer** `[live]` — The *agent* (`neo.business-engineer`, `neo-core`) that supports the human Business Engineer in this task. It may be handed a PRD, a subset of a PRD, or a raw feature to elaborate, and runs the **Specification loop** on the Business Engineer's behalf: segments the PRD, sequences the **Feature Agent** and **Task Planner** for each segment, files the approved tasks as their carrier issues, and spawns one session per task running the **Neo Technical Engineer**. Once a feature's tasks have landed, it carries the feature through **verification**: confirms the fan-in, has the **Neo Platform Engineer** deploy it to non-prod, walks the Business Engineer through the verification steps and any rejection triage, records the result, and has the release prepared. It is **not** the Business Engineer — it holds the gates open, it does not pass through them. Feature sign-off, task-set approval, and the verification verdict remain the human's, always. Canonical `name:` is **Neo Business Engineer**.

**Neo Product Engineer** `[live]` — The agent (`neo.product.engineer`, `neo-product`) that supports the human Product Engineer by running the **Product loop**: fans out **Product Researchers**, sequences the three lenses (**Product Coach**, **Design Thinking Facilitator**, **Systems Thinking Facilitator**), and drives the result to a **PRD**. It orchestrates rather than authors — the analysis belongs to the lenses, the PRD drafting to the Product Coach. Canonical `name:` is **Neo Product Engineer**.

**Neo Technical Engineer** `[live]` — The agent (`neo.technical-engineer`, `neo-core`) that supports the human Technical Engineer by running the **Coding loop**: it takes a **Task** (filed as a GitHub Issue or Azure DevOps story), checks it at intake, and drives it through research → plan → implement → review → validate to a draft PR, delegating implementation to **Code Writer**, review to **Code Reviewer**, and validation to **Validator**. Canonical `name:` is **Neo Technical Engineer**.

**Product Researcher** `[live]` — Agent (`neo-product`) that answers one scoped product-discovery question — existing code and docs, prior decisions, users, market context. Fanned out in parallel, one question each. Distinct from **Researcher** below, which investigates *how the code works* for an already-specified task.

**Product Coach** `[live]` — Agent (`neo-product`) for the **viability** lens: should we build this? Also drafts the PRD in Phase 5.

**Design Thinking Facilitator** `[live]` — Agent (`neo-product`) for the **desirability** lens: do people need this?

**Systems Thinking Facilitator** `[live]` — Agent (`neo-product`) for the **feasibility & dynamics** lens: how does this behave in the real world?

**Researcher** `[live]` — Agent (`neo-core`) that answers one scoped question about the codebase for a task — affected code, existing patterns, constraints, risks. Fanned out in parallel by the Technical Engineer, one question each.

**Implementation Planner** `[live]` — The Coding-loop agent that breaks a **task** into **steps**, and maps every validation criterion to the test or check that proves it. Named for what it produces, matching **Task Planner** below.

**Code Writer** `[live]` — Coding-loop agent that implements and commits exactly one **step**, using whatever stack skills match the work. Diagram 2's "Coder".

**Code Reviewer** `[live]` — Coding-loop agent that reviews one step's commit — feature/fix code or test code — and approves it or returns findings to the Code Writer.

**Validator** `[live]` — Coding-loop agent that runs **validation** for a whole task: once every step is approved, it runs the check behind each validation criterion on the branch head and reports pass, fail, or unproven. The draft PR opens only on a clean result.

**Neo Platform Engineer** `[live]` — The agent (`neo.platform-engineer`, `neo-core`) that supports the human Platform Engineer in **Deployment Space**: deploys a feature's integration target to non-prod and smoke-tests it for verification, opens the draft feature PR that squash-merges a verified feature (Mode A) or records its flag release (Mode B), watches the project's own CD after a human merges, smoke-tests production, and posts the **deployment record** that crosses Boundary 4. Prepares rollbacks for a human to perform. It never deploys to production, flips a production flag, or merges. Canonical `name:` is **Neo Platform Engineer**.

**Neo SRE** `[live]` — The agent (`neo.sre`, `neo-core`) that supports the human SRE in **Operations Space**: receives the deployment record, confirms each KPI is admissible and its instrumentation is emitting, watches post-deploy health and recommends rollback on a regression, and performs **KPI settlement** once each window closes. It reads production and never changes it. Canonical `name:` is **Neo SRE**.

## Units of work

**PRD / Requirements** `[live]` — High-level product or system requirements. Produced by the **Product loop** (`neo-product`), then segmented by the Business Engineer as the input to the Specification loop. A PRD must be **segmentable** — each segment carrying its own business justification — which is the gate at [Boundary 0](./concepts/process-flow.md#boundary-0--product--specification). Its format is owned by the `neo-product-requirements` skill.

**Feature** `[live]` — The business-level unit. Carries What, Why, optional KPIs, and verification steps; signed off by the Business Engineer. A feature is **not** the spec.

**Task** `[live]` — The spec-level unit; the spec analog. Derives from exactly one feature, sized to ≈ one pull request, and carries machine-checkable validation criteria. Shrinking the spec to task grain is Neo's central move.

**Step** `[live]` — The Coding loop's unit of work: one change labeled `[feature]` or `[test]`, landing as one commit (plus review fix-ups). Authored by the **Implementation Planner** inside the Coding loop, not during Feature→Task decomposition. Test steps are interleaved with the feature steps they cover; testing is not a separate phase.

## Proof

**Verification** `[live]` — Human judgment proving a **feature** meets its business contract, executed by the Business Engineer in a non-prod environment. Falsification-framed: the steps are frozen at sign-off, each is run as written, and each then gets at least one deliberate attempt to break it. The gate at [Boundary 3](./concepts/process-flow.md#boundary-3--verification--deployment).

**KPI settlement** `[live]` — Telemetry proving (or disproving) that a deployed **feature** delivered the value its KPIs claimed: each KPI's pre-registered falsifier applied literally to production data after its window closes, with a verdict of `supported`, `falsified`, or `unsettleable`. Establishes **value fit**, where verification establishes problem–solution fit; non-blocking, and never collapsed into verification. Performed by the **Neo SRE**.

**Triage finding** `[live]` — The human diagnosis recorded when verification fails or a deployed feature is rolled back: **mis-built** (the contract is right; the code doesn't satisfy it), **mis-specified** (the contract is wrong), **both** (spec is fixed first), or — after verification only — **mis-deployed** (contract and code are fine; the release or environment is wrong).

**Validation** `[live]` — Machine execution (unit tests, system tests, autonomous agents) proving a **task** meets its spec. No human judgment.

> **Verify features, validate tasks. Humans verify, machines validate.**

**The contract** `[live]` — A feature's verification steps, authored at feature-definition time. The gate the whole pipeline must satisfy to deploy: if the Business Engineer cannot verify it, it cannot ship.

## Loops & spaces (Diagram 2)

**Product loop** `[live]` — The loop *upstream* of the Specification loop, shipped by the `neo-product` plugin: research fan-out → viability/desirability/feasibility lenses → synthesis → **PRD**. It answers "what should exist, and why" and is the origin of the PRD that Neo previously assumed into being. Human-gated at two points: the decision to proceed past synthesis, and the Business Engineer's acceptance of the PRD at [Boundary 0](./concepts/process-flow.md#boundary-0--product--specification). It does **not** absorb or replace `feature-agent`/`task-planner`.

**Specification loop** `[live]` — PRD→Feature and Feature→Task; problem space into solution space. Human-gated: *Start Human, Finish Human; Critical Thinking required.*

**Coding loop** `[live]` — One **Task** → intake → research → plan → implement and review (interleaved steps) → validate → draft **PR**. Run by the **Neo Technical Engineer**; ends at [Boundary 2](./concepts/process-flow.md#boundary-2--coding--verification).

**Verification / Operations loop** `[live]` — Draft PRs → PR review → fan-in → non-prod deploy and smoke test → **verification** (the user test) → release → CD and production smoke test → telemetry and **KPI settlement**. *Human Judgement Required.* Runs through three spaces; crosses [Boundary 3](./concepts/process-flow.md#boundary-3--verification--deployment) and [Boundary 4](./concepts/process-flow.md#boundary-4--deployment--operations).

**Verification Space** `[live]` — Where a feature is proven before it ships: task PR review and merge, fan-in, the non-prod deploy, and the Business Engineer's verification and triage. Ends at Boundary 3.

**Deployment Space** `[live]` — Release mechanics: the feature PR and squash (Mode A) or flag release (Mode B), production CD, production smoke checks, and executing a rollback. Owned by the Platform Engineer. Ends at Boundary 4, when the feature is live, its CD run is green, and its production smoke checks pass.

**Operations Space** `[live]` — The running feature over time, from its deployment record onward: instrumentation intake, the post-deploy watch, and KPI settlement. Owned by the SRE. Ends when every KPI the feature shipped with has settled.

**Strategic-reopen candidate** `[live]`, provisional — A signal that a failure points past one feature at the premise of the PRD it came from, raised to the human Product Engineer, who decides whether the Product loop reopens. Never an automatic reopen. The signals are owned by [process-flow.md](./concepts/process-flow.md#strategic-reopen-candidates-provisional).

## Artifacts

**neo-task-authoring** `[live]` — The skill defining what a clean task is: fields, validation-criteria format, one-PR sizing rule.

**neo-pr-authoring** `[live]` — The skill defining the draft **PR** that crosses Boundary 2 (Coding → Verification): which branch it targets under each integration mode, its required body sections — including the validation report and review ledger — and its closing keyword.

**neo-feature-verification** `[live]` — The skill defining a feature's verification run at Boundary 3: entry conditions, the frozen contract, the falsification attempt on every step, the verdict rule, the **verification record**, the triage interview, and the **rejection record**.

**neo-release-authoring** `[live]` — The skill defining the Deployment-side artifacts: the Mode A feature PR and the traceability its squash commit must carry, the Mode B flag release, the **deployment record** that crosses Boundary 4, and the revert.

**neo-kpi-settlement** `[live]` — The skill defining Operations' work on a deployed feature: the admissibility re-check, the intake record, the post-deploy watch and rollback recommendation, the settlement procedure, verdicts and record, routing, and the strategic-reopen signals.

**Deployment record** `[live]` — The artifact that crosses Boundary 4: what was released, the CD run, the production smoke results, the rollback unit, and the feature's KPIs verbatim. Posted on the feature's carrier issue, like the verification, rejection, intake, and settlement records.

**Task handoff schema** `[live]` — The normative definition of the **Task** artifact that crosses Boundary 1 (Specification → Coding): its carrier (a Task *is* the GitHub Issue / Azure DevOps story it is filed as), fields, and on-harness format. See [`task-handoff-schema.md`](./contributing/reference/task-handoff-schema.md).

**Task Planner** `[live]` — The agent (`task-planner`) that runs interactive Feature→Task decomposition with the Business Engineer. Named for what it produces (tasks), matching **Implementation Planner**.

**Feature Skill / Feature Agent** `[live]` — The level above `neo-task-authoring` / `task-planner`: PRD-segment → Feature. The `neo-feature-authoring` skill defines what a clean feature is; the `feature-agent` runs the interactive drafting with the Business Engineer.

---

**Neo, stylized.** The system is always written **Neo** in prose — never "neo", never "NEO".
Lowercase `neo` appears only as a literal identifier: plugin names (`neo-core`, `neo-product`),
the marketplace (`neo`) and install targets (`neo-core@neo`), agent filenames
(`neo.<role>.agent.md`), skill directories (`neo-feature-authoring`), and repository paths. Inside
code fences and inline code, reproduce the identifier exactly — do not "correct" it.

**Planner naming.** The two planners are named by output, never by level: **Task Planner** (Feature → Tasks, spec loop) and **Implementation Planner** (Task → Steps, coding loop). The PRD→Feature agent is deliberately **not** a "Feature Planner" — it is the **Feature Agent**, to avoid colliding with the two planners one level down.
