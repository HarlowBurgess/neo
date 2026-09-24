# Using Neo

For the **operator** — usually the **Business Engineer (BE)** — driving the crew through the
Specification loop. Assumes Neo is already installed (see
[installing-neo.md](./installing-neo.md)). Terms in **bold** are in the
[glossary](../glossary.md).

> **Status:** all four loops are `[live]` — the Product loop (problem → PRD), the Specification
> loop (PRD → Feature → Task), the Coding loop (Task → validated draft PR), and the Verification /
> Operations loop (merged PRs → verified feature → production → settled KPIs). The first two are
> interactive by design; the Coding loop runs on its own between two gates — plan approval and the
> draft PR; the last loop is human-judged at verification and human-performed at every merge and
> production change. See [getting-started.md](../getting-started.md#whats-live-vs-target).

## The agents you'll invoke

| Agent (picker name) | Role | You invoke it? |
| --- | --- | --- |
| **Neo Product Engineer** (`product.engineer`) | Orchestrator for the Product loop — drives research → lenses → synthesis → **PRD** | Yes — if you need a PRD |
| **Neo Product Researcher / Product Coach / Design Thinking Facilitator / Systems Thinking Facilitator** | Product-loop workers: research fan-out, then the viability, desirability, and feasibility lenses | No — the Product Engineer wires them |
| **Neo Business Engineer** (`business-engineer`) | Orchestrator for the Specification loop — segments the PRD, runs Feature Agent and Task Planner for each segment, files the approved tasks, then spawns one session per task | Yes — if you want the loop driven rather than driving it by hand |
| **Neo Feature Agent** (`feature-agent`) | Drafts a **Feature** — What/Why/KPIs/verification — from a PRD segment, with you | Yes |
| **Neo Task Planner** (`task-planner`) | Splits a signed feature into **Tasks**, with you | Yes |
| **Neo Technical Engineer** (`technical-engineer`) | Orchestrator for the Coding loop — takes a Task (issue/story) and drives research → plan → implement → review → validate → draft PR | Yes |
| **Neo Researcher / Implementation Planner / Code Writer / Code Reviewer / Validator** | Coding-loop workers the orchestrator delegates to | No — the orchestrator wires them |
| **Neo Platform Engineer** (`platform-engineer`) | Deployment Space — non-prod deploys for verification, the feature's release PR, watching CD, production smoke checks, the deployment record, and rollbacks | Yes — for `handover` and `rollback`; the Business Engineer delegates the rest |
| **Neo SRE** (`sre`) | Operations Space — KPI intake, the post-deploy watch, and KPI settlement | Yes |

The workers are deliberately sharp and single-purpose; they don't know each other or the whole
spec. The orchestrator (and you) wire them together.

> **Neo Business Engineer needs the Copilot desktop app.** Spawning sessions is a desktop-app
> capability; in a bare terminal those tools don't exist and the agent will tell you so and stop
> after filing the tasks. Also **select it as your session agent** rather than asking another agent
> to delegate to it — session tools aren't passed down to a sub-agent two levels deep
> ([copilot-cli#3293](https://github.com/github/copilot-cli/issues/3293)).

The Product agents ship in the optional `neo-product` plugin. If it isn't installed they won't
appear in the picker — see [installing-neo.md](./installing-neo.md).

## Step 0 — Produce a PRD

The Specification loop starts from a **PRD/requirements segment**. If you already have one, skip
to step 1.

If you don't, that's the Product loop's job. Invoke **Neo Product Engineer** with the problem or
opportunity — not a solution. It fans out Product Researchers, runs the **viability**,
**desirability**, and **feasibility** lenses, synthesizes, and produces a PRD.

It stops for you twice: once at synthesis, to decide whether the problem is worth pursuing at all,
and once at the end, where you **accept the PRD**. That acceptance is
[Boundary 0](../concepts/process-flow.md#boundary-0--product--specification) — the PRD is the
artifact that crosses it, and it's what you segment in step 1.

The loop is **upstream of**, not a replacement for, `feature-agent` and `task-planner`. It answers
*what should exist, and why*; the Specification loop answers *what to build*.

## The Specification loop, step by step

This is the part that's `[live]`. It is **interactive and human-gated** — never autonomous — because
a bad split poisons everything downstream.

You can run the three steps below yourself, invoking each agent in turn, or hand the whole loop to
**Neo Business Engineer**, which sequences them for you. Either way the gates are identical: the
orchestrator drafts, sequences, and spawns, but **you sign the feature and you approve the task
set**. It cannot sign for you.

1. **PRD → Feature.** Invoke **Neo Feature Agent** with a PRD/requirements segment — from step 0,
   or one you already had. It drafts
   **What**, **Why**, optional **KPIs**, and **verification steps**, working with you. It stops at a
   **BE-signed feature** — it does not decompose tasks. Entry to *ready-to-work* requires What + Why
   + verification steps **and** your sign-off. If you can't verify it, it can't ship.

2. **Feature → Tasks.** Invoke **Neo Task Planner** on the signed feature. It proposes a breakdown,
   surfaces its uncertainty, and converges with you. Default to **logical chunks (vertical slices)**,
   not stack layers. Each Task is sized to **≈ one PR** and carries **validation criteria** that are
   machine-checkable. "Done" is a **BE-approved** task set, not an agent-emitted one.

3. **Task → draft PR** — the Coding loop. Invoke **Neo Technical Engineer** with a Task (filed as a
   GitHub Issue or Azure DevOps story). It first checks the Task at **intake**: every field of the
   [task-handoff schema](../contributing/reference/task-handoff-schema.md) and the `be-approved`
   label must be there, or it stops and tells you what's missing — it never fills the gap itself.
   It then cuts a task branch (`feat/<issue-id>-<short-name>`) from your project's integration
   target — under Mode A, the parent feature's `feature/<feature-id>-…` branch — and runs research →
   plan → implement → review → **validate**. It pauses twice for you: **`/fleet`** before research
   and planning, and **`/rubber-duck`** on the plan before implementation. Review findings loop back
   to the writer until the reviewer approves each step; then the Validator runs the check behind
   every validation criterion, and anything that fails becomes new work. Only a fully validated
   branch becomes a **draft** PR, carrying the validation report and review ledger. It never
   commits to `main` and never merges.

   Occasionally a Task turns out not to fit in one reviewable PR. When that happens the Technical
   Engineer will tell you and **stack** it: one child session per layer, built bottom-to-top, each
   PR based on the one below. That's the exception — the default contract is still one Task, one
   draft PR — and like the Business Engineer's fan-out it needs the desktop app.

**Running the loop with Neo Business Engineer.** Invoke it with a PRD (or a single segment). It
proposes a segmentation, then for each segment runs the Feature Agent and stops for your sign-off,
runs the Task Planner and stops for your approval of the *set*, files each approved task as its
carrier issue, and spawns one session per task running the Technical Engineer — in parallel where
tasks are independent, stacked where one depends on another. Under Mode A it creates each feature's
integration branch first. Each child session stops at its plan gate for the Business Engineer
rather than for `/rubber-duck`. The Business Engineer approves or rejects each plan, steers the
sessions, and reports back every task, issue, branch, and draft PR.

## After the PRs — verify, release, settle

A merged task PR is not a shipped feature. **Verification is per-feature**, so this part starts when
every task under a feature has merged. Each task's draft PR is reviewed and merged by a human; no
agent merges. Under Mode A the PRs merge into the feature's integration branch.

1. **Verify.** Invoke **Neo Business Engineer** with the feature. It confirms every task has
   landed, has the **Neo Platform Engineer** deploy the feature to non-prod and smoke-test it, then
   walks you — the BE — through each verification step **exactly as you signed it**. For each step
   you run it as written, then **try to break it**: a boundary input, the wrong user, an
   interrupted flow. The agent suggests variations; you choose and judge. The verdict is yours, and
   it's recorded on the feature's issue. If a step now seems wrong, that isn't an edit — it's a
   finding that the feature was mis-specified. See
   [process-flow.md § Boundary 3](../concepts/process-flow.md#boundary-3--verification--deployment).
2. **Triage a rejection.** If anything fails, the agent interviews you to decide the finding —
   **mis-built** (new tasks under the same feature), **mis-specified** (re-sign the feature), or
   **both** (spec first). You decide; it records and routes.
3. **Release.** On a pass, the Neo Platform Engineer opens a **draft feature PR** (Mode A) whose
   body closes every task and carries the traceability lines. A human marks it ready and
   **squash-merges it with the PR body as the commit message**. Under Mode B, a human turns the
   flag on in production instead.
4. **Hand over.** After the merge, invoke **Neo Platform Engineer** with `handover`. It watches
   your project's CD — it never deploys to production itself — runs read-only production smoke
   checks, and posts the **deployment record**. If CD or a smoke check fails, it recommends a
   rollback; a human decides, and `rollback` prepares it.
5. **Operate and settle.** Invoke **Neo SRE** with `intake`: it confirms each KPI's instrumentation
   is emitting in production. Use `watch` for the post-deploy health check. When a KPI's window
   closes, `settle` applies its falsifier to production telemetry and posts `supported`,
   `falsified`, or `unsettleable`. The last two come back to the BE. A pattern that implicates the
   PRD itself goes to the Product Engineer as a **strategic-reopen candidate**, and the Product
   Engineer decides what it means.

The feature's issue stays open until its KPIs settle, so your open, shipped features are exactly
the verdicts Operations still owes you.

## Your job at the gates

Neo puts the human where judgment is irreplaceable and lets machines handle the rest:

- **Author proof when you define the unit.** Verification steps at feature time; validation criteria
  at task time. Don't retrofit them.
- **Own the decomposition.** The task split is a conversation, not a hand-off. Push back when the
  planner is unsure.
- **Verify features yourself — and try to break them.** Verification is human judgment against the
  business contract; validation (tests + agents) is the machine's job against the spec. See
  [architecture.md § The core rule](../concepts/architecture.md#the-core-rule).
- **Make KPIs falsifiable, or leave them out.** Name the metric, the instrumentation, the window,
  and the result that would disprove it. For an internal app your users must use, only outcome
  metrics count — never adoption or usage.

## What "well-formed work" looks like

Before you file a Task for the orchestrator, make sure it meets the handoff bar — one feature of
origin, one-PR sizing, machine-checkable validation criteria. That's its own page:
[filing-work.md](./filing-work.md).

## Tuning the crew

If runs feel off, log them and read the per-agent stats to fix the weakest prompt — see
[../contributing/guides/observability.md](../contributing/guides/observability.md). That's a
contributor-flavored task, but operators running many features find it worth it.
