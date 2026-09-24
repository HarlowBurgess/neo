# Plan: Make the Coding loop `[live]` in `neo-core`

> **Status: implemented. Nothing left to execute.**
> - All six commits are pushed to `claude/compassionate-darwin-b9rj8j` and open as [HarlowBurgess/neo#1](https://github.com/HarlowBurgess/neo/pull/1): no merge conflicts, no check runs reported yet.
> - Per your answer, I'm not watching the PR.
> - Two checks still have to run on your machine: the `copilot` install check (neo-core → "Installed 4 skills.") and the dry run.

## Context

Neo's Coding loop (Task → draft PR) takes one **Task** across **Boundary 1** and delivers one draft **PR** across **Boundary 2**. Its agents already ship in `plugins/neo-core/agents/`: technical-engineer, researcher, implementation-planner, code-writer and code-reviewer. But the loop was never specced, and the docs still mark it `[target]` (`glossary.md`, `architecture.md`, `process-flow.md`). `getting-started.md` contradicts itself: its table says `[live]` and its prose says `[target]`. The owning docs name the gaps that keep it from being live:

1. **No receiving side at Boundary 1.** `task-handoff-schema.md` §5 says the consumer must stop when a required field or the `be-approved` marker is missing. The Technical Engineer (TE) never checks for either.
2. **No task-level Validation gate.** Boundary 2 gate #1 requires every validation criterion to run to a deterministic pass. Today only per-unit build, lint and test checks run.
3. **The integration mode is ignored.** The TE always opens its PR against the default branch. Under Mode A (Neo's default) a task PR targets the parent feature's integration branch.
4. **No link to the parent feature.** Boundary 2 gate #4 requires one, and the TE's PR body doesn't carry it.
5. **The testing model is undecided.** `process-flow.md` § Boundary 2 says to choose one before speccing the internals.
6. **The gates don't match between agents.** Neo Business Engineer (BE) child sessions expect the BE to approve the TE's plan, but the TE waits for a human to run `/fleet` and `/rubber-duck`.

**Decisions you made:** interleaved labeled steps, delegable human gates, a new **Neo Validator** agent, and support for both integration modes with Mode A as the default.

**Outcome:**
- A be-approved Task runs intake → branch → research → plan → implement/review → **validate** → draft PR.
- The PR targets the correct base for the integration mode, and its body carries a criterion-by-criterion validation report and a review ledger.
- Every doc marks the loop `[live]`.

Work on branch `claude/compassionate-darwin-b9rj8j`. Use one Conventional Commit per logical change, push when done, and open no PR unless you ask for one.

---

## 1. New agent: `plugins/neo-core/agents/neo.validator.agent.md`

- Frontmatter:
  - `name: Neo Validator`
  - `model: Claude Sonnet 5`, `reasoningEffort: high` (a false pass is expensive to detect; see the model table in `agent-authoring-reference.md`)
  - `tools: [read, search, execute]`, `user-invocable: false`
- Model it on `neo.code-reviewer.agent.md`: the same read-only shell discipline and the same Output / Never sections.
- **Input** (from the TE):
  - the task id
  - the task's validation criteria, verbatim
  - the planner's criterion → step coverage map
  - the branch, base, and HEAD SHA
  - the integration mode (under Mode B: the flag name and how to enable it for the run)
- **Procedure:**
  1. Confirm HEAD matches the given SHA.
  2. Run the `AGENTS.md` build/lint/test gate for every layer touched by `git diff --name-only <base>...HEAD`.
  3. For each criterion, run the executable check that proves it: the test named in the coverage map, or an existing command.
  4. Record pass, fail or `unproven`. A criterion with no runnable check is `unproven`, and `unproven` counts as a failure. Never pass a criterion because the code looks right.
- **Output:**
  - a table: criterion | proof (test id / command, `path:line`) | result | evidence excerpt
  - `Verdict: validated | not validated`
  - gaps
- **Never:** edit files, commit, or mutate state; judge anything that needs a human (that is verification, and it belongs to the feature); invoke other agents.
- ⚠️ **CI trap:** don't write the phrases "child session", "base_branch" or "stacked PR" in this prompt, or in any other non-spawning prompt below. `_SPAWN_PROMPT_RE` in `scripts/validate-plugins.py` flags them, and CI fails because this agent has no `create_session`.

## 2. New skill: `plugins/neo-core/skills/neo-pr-authoring/SKILL.md`

This skill owns the artifact that crosses Boundary 2, the way `neo-task-authoring` owns the Task. The description opens with "Use when opening or checking the draft pull request that ends the Coding loop…". The skill defines:

- **Base branch:**
  - A kickoff `base_branch`, if given, wins (stacked dependency).
  - Otherwise Mode A targets the parent feature's integration branch, and Mode B targets the default branch.
  - The PR is always a draft.
- **Title:** in Conventional Commits form, naming the task.
- **Body sections, in this order:**
  1. `## Task`
  2. `## Parent feature`
  3. `## Integration mode` (A or B, and whether it was declared in `AGENTS.md` or defaulted)
  4. `## Summary`
  5. `## Validation` (the Validator's table, verbatim)
  6. `## Review ledger` (N/N steps approved: step id, `[feature|test]`, SHA(s), verdict)
  7. `## Checks` (the commands and results on HEAD)
- **Closing semantics:**
  - Mode B, or a PR that targets the default branch: `Closes #<task>`.
  - Mode A: `Refs #<task>`, because GitHub closing keywords act only when a PR merges into the default branch. That claim is **RECALL — UNVERIFIED**. The container's proxy blocked a fetch of the GitHub docs page (403); confirm it before relying on it. Otherwise the tasks close when the feature branch's squash commit lands. That commit is outside this loop; point to `process-flow.md` § Traceability under squash.
  - Azure DevOps: link the work item.

## 3. Rewrite `plugins/neo-core/agents/neo.technical-engineer.agent.md`

Keep its structure, voice, tool comment block and existing rules. Changes:

- `agents:` gains `'Neo Validator'`. Update `description` to list the phases: intake, branch, research, plan, implement, review, validate, draft PR.
- **New step 0 — Intake (Boundary 1 receiving gate):**
  - Read the task with `gh issue view` or `az boards work-item show`.
  - Check the fields in `task-handoff-schema.md` §2–§4: Parent feature, What, non-empty Validation criteria, Depends on, and the `be-approved` label or state.
  - If a field or the marker is missing, stop and route it back: a task gap goes to the Task Planner or BE, a feature gap to the BE. Never patch it in this loop.
  - Read the consuming repo's `AGENTS.md` for the integration mode. If none is declared, default to Mode A and say so. Under Mode B, if `AGENTS.md` names no flag convention, stop and ask: Mode B's entry conditions aren't met.
- **Step 1 — Branch:**
  - Resolve the integration target. Mode A: `feature/<feature-id>-<slug>`, created from the default branch and pushed if absent; this is never `main`. Mode B: the default branch.
  - A kickoff `base_branch` overrides the branch point.
  - The task branch stays `feat|fix/<task-id>-<slug>`, and the existing codename-branch rule stays.
- **Delegable gates** (steps 2 and 3): a run counts as *delegated* only when the kickoff prompt carries the explicit line `Delegated by: Neo Business Engineer`.
  - Interactive runs: unchanged. Ask the human for `/fleet` before research and `/rubber-duck` before implementation.
  - Delegated runs: `/fleet` is advisory. Fan out the researchers anyway and state whether they ran in parallel. For the plan gate, present the step list and stop. The orchestrator approves through its plan response or a session message. **Never self-approve.**
- **Step 3 — Plan:** pass the integration mode (and the flag convention) to the planner, and require the criterion → step coverage map back.
- **Steps 4–5:** rename "unit" to "step" throughout. The checklist, review loop and stall rule are otherwise unchanged. Under Mode B, feature steps carry their flag name to the writer and reviewer.
- **New step 6 — Validate:**
  - Once every step is `approved`, delegate to `Neo Validator` with the criteria verbatim, the coverage map, the branch, base and HEAD SHA, and the mode.
  - `not validated`: each failing or `unproven` criterion becomes a new fix or test step on the checklist, goes through writer → reviewer, and is then validated again.
  - The stall rule extends here: the same criterion failing twice with no progress escalates to the human.
- **Step 7 — Draft PR:**
  - Load `neo-pr-authoring`, push the branch, and run `gh pr create --draft --base <target>`.
  - The precondition becomes: every step `approved` **and** the verdict is `validated`.
- **Rules:** add Validate to "delegate every phase". Restate the two gates as delegable, but never skippable by the TE itself.

## 4. Align the workers

| File | Changes |
| --- | --- |
| `neo.implementation-planner.agent.md` | Rename unit → step. New input: integration mode, and the flag convention under Mode B. New coverage rule: every validation criterion maps to ≥1 `[test]` step whose test asserts it, or names an existing executable check. New output section: a **criterion → step(s) → proving test** map for the Validator. Under Mode B, feature steps state their flag gating. Done also means "the task is done only after validation". |
| `neo.code-writer.agent.md` | Rename unit → step. For a test step, prefer skills whose description names the test phase (`stack-plugin-contract.md` discovery rule 4). Under Mode B, implement feature steps behind the named flag. |
| `neo.code-reviewer.agent.md` | Rename unit → step. Under Mode B, check that feature steps are actually gated by the named flag. |
| `neo.business-engineer.agent.md` | Step 5, under Mode A: before spawning a feature's tasks, create or ensure that feature's integration branch and pass it in each kickoff. Each kickoff carries `Delegated by: Neo Business Engineer`, the integration mode, and `base_branch` for dependent tasks. Step 6 already answers each child's plan gate: tighten the wording to say the child waits for that answer, and that a child's `/fleet` doesn't block. |

## 5. Validator script: `scripts/validate-plugins.py`

Add `check_skill_refs(plugin)`. Every skill an agent tells itself to load must resolve to `skills/<name>/SKILL.md` in the same plugin. A reference is `` `neo-x` skill `` or `**neo-x** skill`. Plugins can't share files, and a missing skill fails silently, just like the #81 skills bug.

- Wire it into `check_plugin()` and extend the module docstring.
- First confirm it produces no false positives on today's tree: 13 references across both plugins, all of which currently resolve.

## 6. Docs (owning docs change; others only point to them)

| File | Changes |
| --- | --- |
| `docs/concepts/process-flow.md` | Chain table: Boundary 2 → `[live]`. Scope note: the Coding loop internals are now specced (point to the TE agent and `architecture.md`). Boundary 2: "What crosses" points to `neo-pr-authoring`; gate owner reads writer checks per step → reviewer per step → Validator per task → human on the PR; the Commits paragraph says "per step". Mark "Drift to reconcile" **Resolved: interleaved labeled steps**. Feedback edges: add **Validation → Implement** `[live]`. Boundary 1 drift note: intake is now enforced. Diagram 2's mislabel stays open. |
| `docs/concepts/architecture.md` | Loop 3 → `[live]`. Add a short "Coding loop in detail" section, parallel to the Specification loop one: the phases, delegable gates, interleaved steps, task-level validation, and PR targeting by mode. Update the Status section. |
| `docs/glossary.md` | Researcher, Implementation Planner, Step and Coding loop → `[live]`. Replace "Team Leader / Coder" with **Code Writer**, **Code Reviewer** and **Validator** entries. Step becomes "the Implementation Planner's labeled `[feature]`/`[test]` unit; one commit plus review fix-ups". Add the validate phase to the Neo Technical Engineer entry. |
| `docs/getting-started.md` | Fix the contradicting paragraph (lines 27–31). |
| `docs/guides/using-neo.md` | Status note. Step 3: drop "[target for full autonomy]" and describe intake, delegable gates, validation, and PR base by mode. The BE paragraph says it answers child plan gates. |
| `docs/guides/installing-neo.md` | Status note. Specify the declaration the TE reads (`Integration mode: A` or `B`, plus the flag convention for B), and that a missing declaration defaults to A. Commit conventions: per step. |
| `docs/contributing/reference/task-handoff-schema.md` | "Team Leader / orchestrator (#11)" → **Neo Technical Engineer**. §1: closing semantics point to `neo-pr-authoring`. §5 consumer: now enforced at TE intake. |
| `docs/contributing/reference/stack-plugin-contract.md` | "What ships" table gains a `validator` row and a `neo-pr-authoring` row. |
| `docs/contributing/guides/agent-authoring-reference.md` | Model table: add `neo.validator` to the Sonnet 5 / high row. |
| `AGENTS.md` | Shipped-agents table gains `validator`. "one commit per unit" → "per step". The install check changes to `neo-core` → **"Installed 4 skills."** |
| `README.md` (root), `plugins/neo-core/README.md` | Add the validator and the new skill ("two authoring skills" is stale). |
| `todo.md` | Coding loop live. The #10 interleaved-vs-phased question is resolved. Diagram 2 is still open. |

## 7. Version bump

- `plugins/neo-core/plugin.json`: 2.2.0 → **2.3.0**, and mention the validator in `description`.
- `.github/plugin/marketplace.json`: the neo-core entry `version` and `description`, plus `metadata.version`, → 2.3.0. `check_marketplace` requires the entry and `plugin.json` versions to match.

## Suggested commit order

1. `feat(neo-core): add Neo Validator agent`
2. `feat(neo-core): add neo-pr-authoring skill`
3. `feat(neo-core): wire intake, integration mode, and validation into the Technical Engineer` (includes the worker and BE alignment)
4. `feat(validate): require agent skill references to resolve in-plugin`
5. `docs: mark the Coding loop live and reconcile Boundary 2`
6. `chore(neo-core): release 2.3.0`

## Verification

- `python3 scripts/validate-plugins.py` passes. Then temporarily change the TE's skill reference to `neo-pr-authoringX` and confirm the new check fails; also confirm a spawn phrase in `neo.validator.agent.md` would fail. Revert both.
- Run the JSON sanity loop from `AGENTS.md` § Checks over the marketplace, `plugin.json` and `hooks.json`.
- `rg -n "Team Leader|one commit per unit|Coding loop\*\* \`\[target\]" docs AGENTS.md README.md plugins` returns nothing stale. `rg -n "Neo Validator" plugins` shows the agent's `name:` and the TE's allowlist.
- `copilot` isn't installed in this container, so the throwaway `COPILOT_HOME` install check has to run on your machine: `copilot plugin install neo-core@neo` must print **"Installed 4 skills."**, and Neo Validator must appear.
- Manual dry run (your machine, Copilot desktop app; see #23):
  1. Use a toy repo whose `AGENTS.md` declares `Integration mode: A`.
  2. File one issue **without** `be-approved` and invoke Neo Technical Engineer. Intake must stop and name the missing marker.
  3. Add the label and run again. The PR must target `feature/<feature-id>-…`, stay a draft, and carry `Refs #<task>`, a Parent feature link, a Validation table and a Review ledger.

## Out of scope (noted, not done)

- The `neo-stack-skill-authoring` skill that `stack-plugin-contract.md` promises.
- Stack plugins (#16 and others).
- The feature squash to main with task IDs, which happens at Boundary 3.
- Redrawing the Diagram 2 PDF.