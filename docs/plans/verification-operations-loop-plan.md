# Plan: Make the Verification / Operations loop `[live]` in `neo-core`

> **Status: approved 2026-09-24.** Executed on this branch in the commits that follow this one.

## Context

Neo's fourth loop takes the draft **PRs** that leave the Coding loop at **Boundary 2** and carries their **Feature** to production and past it:

- task PRs are reviewed and merged by a human;
- they fan in to one verifiable feature;
- the feature is **verified** by the Business Engineer (BE) in non-prod;
- it squash-merges and deploys through the project's CD;
- its **KPIs** are settled from telemetry once their window closes.

`process-flow.md` already designs the Boundary 3 *gate*: BE judgment, the mis-built / mis-specified / both triage, the ordering rule, and the rejection record. Everything else is undesigned:

1. **Nobody runs the loop.** The glossary names "SRE Agent / Platform Engineering Agent" `[target]`, with no tools and no procedure (backlog #14).
2. **Boundary 3's internals are empty.** Nothing says who confirms fan-in, who deploys to non-prod, how a verification run is recorded, or where the rejection record lives.
3. **Nothing owns the Mode A squash.** `process-flow.md` § Traceability under squash says the squash commit must list every child task and the parent feature. The task PRs say `Refs #<task>` and wait for it. No agent or procedure executes it, so today Mode A tasks never close.
4. **Diagram 2 has no Deployment → Operations boundary.** Operations Space floats outside every loop, and the chain table's last row is an unnumbered `[target]`.
5. **The Operations → Specification edge exists only as prose.** The KPI hypotheses it would settle aren't falsifiable in practice: `neo-feature-authoring` still treats KPIs as optional and never asks for instrumentation, window, falsifier, or baseline (a `todo.md` loose end, and the back-door slice of G2).
6. **Two design gaps sit on this loop** (`framework-gap-analysis.md`):
   - **G3:** verification asks a confirmatory question.
   - **G4:** there is no signal for a strategic (system-level) reopen.

### Decisions you made

| Question | Decision |
| --- | --- |
| Verification tooling | **Neo Business Engineer assists; the human BE judges.** A Verify phase plus a new `neo-feature-verification` skill. The agent sequences, records, and runs the triage interview. It never renders a verdict. |
| G3 framing | **Lock + falsify pass.** Keep the word "verification" and the core rule. Verification steps are pre-registered and frozen at sign-off; a wish to change one mid-run is itself a *mis-specified* finding. Each step is run as written, then the BE makes at least one recorded attempt to break it. The default question becomes "try to make it fail." |
| Tier | **All in `neo-core`**, Process tier. The new agents are stack-agnostic. Deploy, smoke, and telemetry *mechanics* late-bind by description to stack skills (#67 `neo-azure-platform`, #68 `neo-ops`), which re-scopes #68 to skills only. |
| Squash | **Agent prepares, human merges.** After the BE's recorded pass, Neo Platform Engineer opens a **draft** feature PR (integration branch → default). The body carries `Closes #<task>` per child, `Refs #<feature>`, and the verification record link. A human marks it ready and squash-merges it with that title and body. The rule that agents never merge holds. |
| Deploy authority | **Non-prod yes, prod never.** The agent may trigger the non-prod deploy that `AGENTS.md` declares. Production changes only through the project's own CD after a human merge, or a human flag flip under Mode B. The agent watches, smoke-tests read-only, and records. |
| G4 | **Provisional signals; a human decides.** A *strategic-reopen candidate* is raised when any of these holds: ≥2 falsified KPIs trace to one PRD success criterion or assumption; one falsified KPI traces to a P0 success criterion; ≥2 *mis-specified* verification findings trace to one PRD segment. Candidates go to the human Product Engineer. A candidate is never an automatic reopen. G4 moves to **Partial**. *Refined during implementation:* the PRD template's success metrics are its "Goals & Success Metrics" rows, and they carry no priority; P0 tags requirements. So signal 1 keys off a shared **PRD goal**, and signal 2 off a falsified KPI on a feature that **delivers a P0 requirement**. Features now name both. |
| Ops scope | **Full loop now.** Neo SRE ships with intake, post-deploy watch, and KPI settlement. Loop 4 and the Operations → Specification edge go `[live]`. |
| Adjacent | **Fold in** the KPI falsifiability gate (`neo-feature-authoring`, Feature Agent, Task Planner) and a **Mermaid chain diagram** in `process-flow.md`. |

### Outcome

A BE-signed feature whose tasks have all merged goes through fan-in, a non-prod deploy and smoke test, BE verification (with the falsification pass), and then one of two paths:

- **Rejection:** a recorded triage routes it back.
- **Pass:** a draft feature PR is opened, a human squash-merges it, the project's CD deploys it, and a production smoke test runs. A deployment record hands it to Operations. Neo SRE confirms the instrumentation is emitting, watches the deploy, and settles each KPI when its window closes. The verdict goes back to the Specification loop, or to the Product loop as a strategic candidate.

Diagram 2's Deployment/Operations boundary gets a textual definition as Boundary 4, plus a Mermaid drawing.

Work on branch `claude/wizardly-meitner-ohu8yk`, created fresh from `main` at `e7ff77f`. Use one Conventional Commit per logical change, push when done, and open no PR unless you ask for one.

---

## The loop, end to end (what the docs will say)

| # | Phase | Space | Who | Output |
| --- | --- | --- | --- | --- |
| 1 | **PR review** — each task's draft PR reviewed and merged into the integration target | Verification | Human (Boundary 2's human half) | Merged task PRs |
| 2 | **Fan-in** — every task in the BE-approved set has landed | Verification | Neo Business Engineer | Fan-in report |
| 3 | **Non-prod deploy + smoke** of the integration target at a pinned SHA | Verification | Neo Platform Engineer | Env + SHA + smoke results |
| 4 | **Verify** — the frozen steps run as written, plus a falsification pass | Verification | Human BE judges; Neo Business Engineer records | Verification record, or rejection record + triage |
| 5 | **Release** — Mode A: draft feature PR with the traceability body. Mode B: a flag-release record | Deployment | Neo Platform Engineer prepares; human merges / flips | Squash commit on the default branch, or flag on |
| 6 | **CD + prod smoke** — the project's pipeline deploys; smoke checks run read-only | Deployment | Project CD; Neo Platform Engineer watches | Deployment record (Boundary 4) |
| 7 | **Intake** — every KPI's instrumentation is confirmed emitting in prod; the window opens | Operations | Neo SRE | Settlement-watch record |
| 8 | **Watch** — post-deploy health against the `AGENTS.md` health signals | Operations | Neo SRE; human decides any rollback | A rollback recommendation, or quiet |
| 9 | **Settle** — at window close, apply each KPI's falsifier to telemetry | Operations | Neo SRE | Settlement record → Spec loop / Product loop |

**Records live on the feature's carrier** (the issue each task names as `Parent feature`, or the ADO feature's discussion) as comments in the formats the skills define. The feature issue stays **open until its KPIs settle**, or until handover if it has none. The list of open, deployed features is therefore Operations' outstanding debt to the Specification loop. The human BE closes it.

**Deployment Space vs Operations Space (the Diagram 2 fix).**

- **Deployment Space** is release mechanics: the non-prod deploys for verification, the squash / flag release, production CD, and executing a rollback. It ends when the feature is live in production, its CD run is green, and its prod smoke checks pass.
- **Operations Space** owns the *running* feature over time, from the deployment record onward: instrumentation intake, the post-deploy watch, and KPI settlement.

**Boundary 4 — Deployment → Operations:**

- **What crosses:** the deployment record.
- **Gate:** CD green and prod smoke pass. A machine gate owned by Neo Platform Engineer.
- **Receiving side:** Neo SRE's intake, like Technical Engineer (TE) intake at Boundary 1.
- **On failure:** a rollback recommendation to a human, then the triage.

**The triage gains one finding post-verification: `mis-deployed`.** The contract and the code are both fine; the release, configuration, or environment is wrong. It routes to the human Platform Engineer in Deployment Space. Without it, an environment fault defaults to "write more code," the exact failure the triage exists to prevent. Boundary 3's own triage stays three-way: a non-prod smoke failure stops *before* verification, so it never reaches the BE's triage.

---

## 1. Falsifiability gate (the adjacent loose end, and a prerequisite for settlement)

| File | Change |
| --- | --- |
| `plugins/neo-core/skills/neo-feature-authoring/SKILL.md` | **KPIs:** optional still, but a KPI is admissible only with all four fields: **metric**, **instrumentation** (exists today, or must ship with the feature, which puts it in scope), **window**, **falsifier**. Add a **baseline / counterfactual** line (pre-change measurement, holdout, or staged rollout), required for a captive population. Add the captive-population rule: for internal LOB, engagement metrics (adoption, DAU, session length, feature usage) are inadmissible; only outcome metrics are admissible. Each KPI names the PRD success criterion it serves, if any, for G4. **Verification steps:** add that they are **pre-registered**, frozen at sign-off; changing one later is a re-sign, not an edit; write each so an attempt to break it is meaningful. **Template:** gains `Source (PRD segment)` and the structured KPI block. |
| `plugins/neo-core/agents/neo.feature-agent.agent.md` | Step 3: propose a KPI only if all four fields can be named; drop it otherwise. Apply the captive-population check, and ask the BE whether the population is captive. Step 4: tell the BE the steps are frozen at sign-off. |
| `plugins/neo-core/agents/neo.task-planner.agent.md` | Procedure step 4: if a KPI's instrumentation must ship with the feature, the task set includes the task that emits it. That task's validation criterion asserts the event or metric is emitted. |

## 2. New agent: `plugins/neo-core/agents/neo.platform-engineer.agent.md`

Frontmatter:

- `name: Neo Platform Engineer`
- `model: Claude Sonnet 5`, `reasoningEffort: medium`: mechanical, checked immediately (see the model table)
- `tools: [read, execute]`
- `user-invocable: true`
- `argument-hint: <feature issue/story, and the operation>`

It supports the human **Platform Engineer**. It is invoked two ways: by Neo Business Engineer as a sub-agent for operations 1–2, or directly by a human for operations 3–4. Every run names one operation:

1. **`deploy-nonprod`**
   - Takes: feature id, integration target, mode, and the flag under Mode B.
   - Refuses if the integration branch is behind the default branch. Verification must run against what will land, and refreshing it is a human merge.
   - Runs the `AGENTS.md` non-prod deploy command at a pinned SHA; under Mode B, with the flag on in non-prod. Then runs the non-prod smoke checks.
   - Returns: environment, SHA, and smoke results.
   - If `AGENTS.md` declares no deploy command, it stops and names the gap. A human may deploy manually and hand over env + SHA; the agent then confirms and smoke-tests.
2. **`release`**
   - Precondition: a verification record with verdict `verified` at a SHA equal to the integration branch head. If the branch has moved, it refuses: re-verify.
   - Mode A: opens the **draft** feature PR in the `neo-release-authoring` shape.
   - Mode B: writes the flag-release record (flag, target env, "a human flips it", cleanup owed).
3. **`handover`**
   - Runs after a human merges or flips.
   - Finds the project's CD run for the squash SHA and watches it to completion (`gh run watch`, one blocking command, no poll loop).
   - Runs the prod smoke checks read-only.
   - Posts the **deployment record** (Boundary 4) on the feature carrier.
4. **`rollback`**
   - Runs only on an explicit human decision.
   - Mode A: a draft revert PR of the squash commit against the default branch (`revert:` type, `Refs #<feature>`).
   - Mode B: records the flag-off instruction for the human.

Never:

- deploy to production, or trigger production CD;
- merge, mark ready, or push to the default or integration branch;
- flip a production flag;
- diagnose a failure (report it; the triage is human);
- invoke other agents.

Late-binds deploy, smoke, and CD skills by description.

⚠️ **CI trap:** no spawn phrases in this prompt (`child session`, `base_branch`, `stacked PR`, `spawn … session`). It has no `create_session`.

## 3. New skill: `plugins/neo-core/skills/neo-release-authoring/SKILL.md`

It owns the artifacts on the Deployment side, the way `neo-pr-authoring` owns the task PR. The description opens with "Use when opening the feature PR that squash-merges a verified feature…". It defines:

- **Feature PR (Mode A):**
  - Base: the default branch. Head: `feature/<feature-id>-<short-name>`. Always a draft.
  - Title: Conventional Commits `feat(<scope>): <feature title>`. It becomes the squash subject.
  - Body sections:
    1. `## Feature` (`Refs #<feature>`: the feature closes at settlement, not here)
    2. `## Tasks` (one `Closes #<task>` per child in the BE-approved set)
    3. `## Verification` (record link + verified SHA)
    4. `## Traceability` (`Feature: #F` / `Tasks: #t1, #t2, …`, the lines the squash commit body must carry)
    5. `## Rollback` (revert this squash commit)
  - Instruction to the human: squash-merge with the PR title as subject and the body as commit message, pasting it if the repo's default differs.
- **Flag release (Mode B):** the release record.
- **Deployment record (Boundary 4):**
  - feature, and the released SHA or flag state;
  - CD run link + result;
  - each prod smoke command + result;
  - the rollback unit;
  - the feature's KPIs, each with its named instrumentation, for SRE intake;
  - `Mode B cleanup owed: <flag>` where applicable.
- **Revert PR:** the rollback artifact.
- **Never:**
  - `Closes` the feature;
  - omit a child task;
  - open a feature PR without a matching verified SHA;
  - target anything but the default branch.

## 4. New agent: `plugins/neo-core/agents/neo.sre.agent.md`

Frontmatter:

- `name: Neo SRE`
- `model: Claude Sonnet 5`, `reasoningEffort: high`: a wrong verdict is expensive to detect, the same row as the reviewer and validator
- `tools: [read, execute]`
- `user-invocable: true`

It supports the human **SRE**. It loads `neo-kpi-settlement` and `neo-evidence-standard`: every number it reports is `FACT`, with the query and its output as the locator. Every run names one operation:

1. **`intake`** (Boundary 4 receiving side)
   - Reads the deployment record.
   - For each KPI, re-checks admissibility (the four fields, captive-population, baseline), then queries production to confirm its instrumentation is **emitting**.
   - Records the window start and settle-by date.
   - A KPI not emitting, or inadmissible, is recorded **unsettleable now**, not at window close, and routed to the Specification loop.
2. **`watch`**
   - Compares the `AGENTS.md` health signals before and after the deploy.
   - On a regression: a **rollback recommendation** to the humans, with evidence. A human decides, Neo Platform Engineer executes, and the incident then goes to the four-way triage (mis-built / mis-specified / both / mis-deployed).
3. **`settle`**
   - Refuses before the window closes.
   - Queries the metric and compares it to the pre-registered baseline.
   - Applies the falsifier **literally**.
   - Verdict: `supported`, `falsified`, or `unsettleable`.
   - Posts the settlement record. Routes:
     - `falsified` → BE / Specification loop (feature-level);
     - `unsettleable` → Specification loop (KPI-authoring defect);
     - strategic-reopen candidates → the human Product Engineer.

Never:

- change production;
- settle early;
- move a falsifier or substitute a metric;
- treat engagement metrics as evidence under captivity;
- invoke other agents.

Late-binds telemetry skills by description.

⚠️ **CI trap:** the same one as §2.

## 5. New skill: `plugins/neo-core/skills/neo-kpi-settlement/SKILL.md`

The description opens with "Use when receiving a deployed feature into Operations, watching it after deploy, or settling its KPI hypotheses…". It defines:

- the admissibility check;
- the intake record;
- the watch and its rollback recommendation;
- the settlement procedure and record: per KPI, the metric, query, window, baseline, observed value, falsifier, verdict, and evidence;
- routing;
- the three G4 candidate signals, each stated as a *candidate*;
- the `mis-deployed` finding;
- evidence rules, pointing at `neo-evidence-standard`.

## 6. New skill + Business Engineer extension (Boundary 3)

**`plugins/neo-core/skills/neo-feature-verification/SKILL.md`** owns the Boundary 3 run and its records. The description opens with "Use when running, recording, or triaging the verification of a Feature…". It defines:

- **Entry conditions:** fan-in complete; deployed SHA pinned; non-prod smoke green.
- **The frozen contract:** steps quoted verbatim from the signed feature.
- **Per-step procedure:**
  1. Execute as written.
  2. Observe.
  3. Make a **falsification attempt**: the agent may *suggest* disconfirming variations; the BE chooses and runs them.
  4. The BE's verdict, in the BE's words.
- **Verdict rule:** `verified` only if every step passes, including its attempt. A variation the contract never addressed is a mis-specified candidate, and the BE judges it.
- **Pinning:** if the integration branch moves mid-run, the record is void.
- **The verification record.**
- **The triage interview:** questions that separate mis-built from mis-specified.
- **The rejection record:** the four fields `process-flow.md` requires, plus the route.
- **The ordering rule.**
- **Learning signals:** a feature mis-specified twice goes to the Spec loop. The third G4 signal: ≥2 mis-specified findings from one PRD segment.
- **Never:** the agent renders a verdict, edits a step, or defaults a failure to mis-built.

**`plugins/neo-core/agents/neo.business-engineer.agent.md`:**

- `description` adds verification. `agents:` gains `'Neo Platform Engineer'`.
- The argument hint allows a feature reference.
- **New entry point:** given a feature whose tasks have landed, start at step 7.
- **Step 7 — Fan-in:**
  - Every task in the approved set has a merged PR into the integration target.
  - Mode A: match `gh pr list --base <feature-branch> --state merged` against `Refs #<task>`.
  - Mode B: every task issue is closed by a merged PR.
  - Otherwise stop and report which tasks are outstanding.
- **Step 8 — Verify:**
  - Delegate `deploy-nonprod` to Neo Platform Engineer.
  - Load `neo-feature-verification` and walk the human BE through it.
  - Record the result on the feature carrier.
  - On rejection, run the triage interview and route. **Mis-built:** new task(s) under the unchanged feature, back through steps 3–6. **Mis-specified:** step 2 (re-sign), then step 3 (re-decompose). **Both:** spec first, never in parallel.
- **Step 9 — Release:** delegate `release` to Neo Platform Engineer, then tell the human what to merge and what comes next (a human invokes Neo Platform Engineer `handover` after the merge, and Neo SRE after that).
- **Rules:** the verification verdict and the triage finding are the human's; never default a failure to mis-built.

## 7. Docs (owning docs change; others only point to them)

| File | Changes |
| --- | --- |
| `docs/concepts/process-flow.md` | **Scope note:** loop 4 internals are owned by the three agents and summarized in `architecture.md`. **Chain table:** Boundary 3 → `[live]`; Deployment → Operations becomes **Boundary 4** (gate: CD green + prod smoke pass; owner: Neo Platform Engineer; received by Neo SRE) `[live]`. **Boundary 3:** `[live]`. Adds the **falsification framing** (frozen steps, falsification pass, "try to make it fail"), the entry conditions (fan-in, pinned SHA, smoke), who carries it, and where the records live. The **squash** paragraph says who prepares it and who merges it. **New § Boundary 4** covers the Deployment Space / Operations Space definition, the `mis-deployed` finding, rollback, and a **Mermaid diagram** of the whole chain. **Feedback edges:** Verification → Coding/Spec `[live]`; Operations → Specification `[live]`; a new **Operations/Verification → Product** (strategic-reopen candidate) `[live]`, provisional. New § **Strategic-reopen candidates** owns the three signals. **Mode A / Traceability:** who executes the squash. **Mode B:** flag release is a human flip; cleanup is recorded as owed (mechanism left open). **Related open items:** the Operations boundary is now specified; the PDF remains stale and needs its owner to redraw it; Mode B flag-cleanup mechanism. |
| `docs/concepts/architecture.md` | Loop 4 → `[live]`, and "all four loops are built". New **"Verification / Operations loop in detail"** section, modeled on the Coding loop section: the nine phases, the three spaces, the records. Update the Status section. |
| `docs/glossary.md` | Human roles: **Platform Engineer**, **Site Reliability Engineer (SRE)**. Agent roles: **Neo Platform Engineer**, **Neo SRE** replace "SRE Agent / Platform Engineering Agent". Neo Business Engineer gains verification. Loops & spaces: the loop → `[live]`, plus **Verification / Deployment / Operations Space** entries. Proof: **Verification** notes the falsification pass; new **KPI settlement** entry. Artifacts: the three new skills and three records. |
| `docs/contributing/design/framework-gap-analysis.md` | Dated update, not a re-baseline. **G3 → Partial:** word kept by design, goalposts frozen, falsification pass added, with the reasoning. **G4 → Partial:** three provisional candidate signals, the human decides; open question 1 narrowed, not closed. Coverage table and notes updated. |
| `docs/contributing/reference/stack-plugin-contract.md` | "What ships" gains `platform-engineer`, `sre`, and the three skills. Discovery rule 4 ("state the phase it serves") extends to **deploy** and **operate**. A note that #68-style ops plugins ship skills that these agents late-bind. |
| `docs/contributing/guides/agent-authoring-reference.md` | Model table: `neo.sre` → the Sonnet 5 / high row; `neo.platform-engineer` → the Sonnet 5 / medium row. |
| `docs/getting-started.md` | The table's loop 4 row → `[live]`; update the prose. |
| `docs/guides/using-neo.md` | Status note. New step: **verify, release, settle**, covering the BE's role in the falsification pass and triage, the human merge, and when to invoke the Platform Engineer and SRE. |
| `docs/guides/installing-neo.md` | Status note. New `AGENTS.md` items the loop reads: non-prod deploy command, smoke checks per environment, the CD pipeline, telemetry query access, and health signals. Missing items stop the agent, which names them. Mode A: squash-merge with the PR body. |
| `docs/guides/filing-work.md` | Feature KPIs: the four fields and captive-population; steps are frozen at sign-off; two new common mistakes. |
| `docs/contributing/reference/task-handoff-schema.md` | §1: under Mode A, tasks close when the feature PR squash-merges (`neo-release-authoring`). |
| `AGENTS.md` | Shipped-agents table gains `platform-engineer` and `sre`. Install check: `neo-core` → **"Installed 7 skills."** |
| `README.md`, `plugins/neo-core/README.md` | The two agents and three skills; skill counts. |
| `todo.md` | Loop 4 is live. #14 is built by this change (close it upstream). #68 is re-scoped to skills. The falsifiability loose end is done. Diagram 2 is specified in text and Mermaid; the PDF redraw is still owed. G3/G4 → Partial. Mode B flag cleanup is still open. |

## 8. Version bump

- `plugins/neo-core/plugin.json`: 2.3.0 → **2.4.0**, and the description names the new agents.
- `.github/plugin/marketplace.json`: `neo-core` entry `version` + `description`, and `metadata.version` → 2.4.0.
- `neo-product` is unchanged. Its Product Engineer already takes "a problem or opportunity", and a strategic-reopen candidate is one.

## Validator

No new check. The existing ones cover everything new:

- skill refs resolve in-plugin;
- `agents:` resolves (`'Neo Platform Engineer'` in the BE);
- `tools:` are CLI-effective;
- no spawn phrases without `create_session`.

## Suggested commit order

1. `docs(plans): add Verification/Operations loop plan` (this file)
2. `feat(neo-core): add KPI falsifiability gate to feature authoring`
3. `feat(neo-core): add Neo Platform Engineer and neo-release-authoring`
4. `feat(neo-core): add Neo SRE and neo-kpi-settlement`
5. `feat(neo-core): wire Boundary 3 verification into the Business Engineer`
6. `docs: mark the Verification/Operations loop live and define Boundary 4`
7. `docs: reconcile G3 and G4 in the framework gap analysis`
8. `chore(neo-core): release 2.4.0`

## Verification

- `python3 scripts/validate-plugins.py` passes after every commit.
- Negative checks, each reverted afterwards:
  - a spawn phrase in `neo.sre.agent.md` must fail CI;
  - a `neo-kpi-settlementX` skill ref must fail CI;
  - a misspelled `'Neo Platform EngineerX'` in the BE allowlist must fail CI.
- The JSON sanity loop from `AGENTS.md` § Checks.
- `rg` for stale text returns nothing: `SRE Agent / Platform`, `Verification / Operations\*\* \`\[target\]`, `Installed 4 skills`, `the rest is the end-state map`.
- On your machine (`copilot` isn't in this container):
  - Throwaway-`COPILOT_HOME` install: `neo-core@neo` must print **"Installed 7 skills."**, and Neo Platform Engineer and Neo SRE must appear.
  - Dry run in a toy repo (Mode A):
    - Neo Business Engineer on a feature with one unmerged task: fan-in must stop and name it.
    - Merge it and run again: `deploy-nonprod` must stop if `AGENTS.md` names no deploy command.
    - After a pass: the draft feature PR must carry `Closes #<task>` for each child, and `Refs #<feature>`.

## Backlog reconciliation (from `todo.md`; the upstream issue API was unreachable this session)

- **#14 SRE / Platform Eng agents:** built by §2 and §4. Close upstream after merge.
- **#68 `neo-ops`:** stays a stack plugin, re-scoped to deploy, smoke, and telemetry *skills* these agents late-bind. It must ship no agent.
- **#67 `neo-azure-platform`:** unchanged. Its deploy skills gain a consumer.
- **The `todo.md` falsifiability loose end:** closed by §1.
- **G2's front-door question:** untouched, still open.

## Out of scope (noted, not done)

- Stack plugins (#16, #65–68) and `neo-stack-skill-authoring`.
- Redrawing the Diagram 2 PDF. The boundary is specified in text and Mermaid instead.
- A Mode B flag-cleanup mechanism, recorded as an open item.
- The front-door testability gate (G2) and the single-BE gate (G5).
