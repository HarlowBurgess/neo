---
name: neo-release-authoring
description: Use when opening the feature PR that squash-merges a verified feature to the default branch (Mode A), preparing a feature-flag release (Mode B), writing the deployment record that hands a deployed feature to Operations at Boundary 4, or preparing a rollback. Defines each artifact's base branch, title, required sections, and the traceability lines the squash commit must carry. Load before running `gh pr create` for a feature or a revert, and before posting a deployment record.
---

# Authoring a feature's release

These are the artifacts that leave **Deployment Space**: the feature PR (or, under Mode B, the flag release) that carries a verified feature to the default branch, the **deployment record** that hands it to Operations, and the revert that takes it back out. Each one is prepared by an agent and acted on by a human. **An agent never merges, never marks a PR ready, and never changes production.**

The integration mode comes from the consuming repo's `AGENTS.md` (`Integration mode: A` or `B`; none declared means **A**). The modes are defined in `docs/concepts/process-flow.md` § Integration modes.

## Precondition — a verified feature, at the head that will land

Prepare a release only when the feature's carrier carries a **verification record** with verdict `verified`, and the SHA it names equals the head of what will land:

- **Mode A** — the head of the feature's integration branch, `feature/<feature-id>-<short-name>`.
- **Mode B** — the default-branch SHA verified with the flag on.

If the head has moved since verification, the record no longer describes it. Stop: the feature is re-verified, not released.

## Feature PR (Mode A)

The integration branch squash-merges to the default branch as **one commit, one feature**. The PR you open is that commit's draft.

- **Base:** the repository's default branch. **Head:** the feature's integration branch.
- **Always a draft.** A human marks it ready and merges it.
- **Title:** Conventional Commits, naming the feature — `feat(<scope>): <feature title>`; `fix` only when the feature corrects existing behavior. It becomes the squash commit's subject.

Body — exactly these sections, in this order; write `None` rather than omit one:

```markdown
## Feature
Refs #<feature>        <!-- never Closes: the feature closes when its KPIs settle -->

## Tasks
Closes #<task-1>
Closes #<task-2>
<!-- one line per task in the BE-approved set — every child, none omitted -->

## Verification
Verified by <BE> at `<verified-sha>` — <link to the verification record>.
<!-- If the default branch has moved since that SHA, say by how many commits here; the human decides whether to re-verify. -->

## Traceability
Feature: #<feature>
Tasks: #<task-1>, #<task-2>

## Rollback
Revert the squash commit this PR produces. Feature-level revert is one commit.
```

**The squash commit body is the PR body.** Squash-merging collapses the per-task commits; the traceability lines above are how the commit → task → feature chain survives on the default branch (`docs/concepts/process-flow.md` § Traceability under squash). Tell the human, in your report: *squash-merge with the PR title as the subject and the PR body as the commit message — paste the body if the repository's default squash message differs.* Because the PR targets the default branch, its `Closes` lines also close every task when it merges; the task PRs could not, because their base was the integration branch.

## Flag release (Mode B)

Under Mode B the code is already on the default branch, dark. The release is turning the flag on in production, and that is a **human** action. Post a release record on the feature's carrier:

```markdown
### Release — <feature title>
Feature: #<feature> · Mode: B
Verified at: `<verified-sha>` with flag `<flag>` on — <link to the verification record>
Release: turn flag `<flag>` on in production. **A human flips it**, per the flag convention in AGENTS.md.
Cleanup owed: flag `<flag>` — the code path stays conditional until it is removed.
```

## Deployment record — Boundary 4

Posted on the feature's carrier once production has the feature. It is the artifact that crosses **Boundary 4 (Deployment → Operations)**: Operations receives the feature through this record and nothing else.

```markdown
### Deployment record — <feature title>
Feature: #<feature> · Mode: <A | B>
Released: `<squash-sha>` on `<default-branch>` | flag `<flag>` on in production
Verification record: <link>
CD: <run link> — <success | failure>        <!-- Mode B: n/a — flag flip confirmed by <human> -->

Production smoke checks:
| Check | Command | Result |
| --- | --- | --- |
| … | `…` | pass / fail |

Rollback unit: revert `<squash-sha>` | turn flag `<flag>` off
KPIs handed to Operations:
| Metric | Instrumentation | Window | Falsifier | Baseline |
| --- | --- | --- | --- | --- |
| … | … | … | … | … |      <!-- verbatim from the signed feature; "None" if it has no KPIs -->

Mode B cleanup owed: `<flag>` | n/a
Handed over: <UTC date>
Gate: passed | failed — <what failed>
```

- **The gate passes** only when the CD run succeeded and every production smoke check passed. Then Operations takes the feature from here.
- **The gate fails** otherwise. Post the record anyway, with `Gate: failed`, and recommend a rollback to the humans — do not hand the feature to Operations. What went wrong is diagnosed by humans, not by you.
- **KPIs are copied verbatim** from the signed feature. Never restate, round, or "clarify" a falsifier — Operations settles against exactly what was signed.

## Revert (rollback)

Prepared only on an explicit human decision to roll back.

- **Mode A** — a branch `revert/<feature-id>-<short-name>` from the default branch, carrying `git revert` of the feature's squash commit, opened as a **draft** PR against the default branch. Title: `revert: <the feature PR's title>`. Body: `Refs #<feature>`, the squash SHA being reverted, and a link to the evidence behind the decision.
- **Mode B** — no code change. Post the instruction on the feature's carrier: turn flag `<flag>` off in production — **a human flips it**.

A rollback is not a diagnosis. After it lands, the failure is triaged by humans — mis-built, mis-specified, both, or **mis-deployed** (the contract and code are fine; the release, configuration, or environment is wrong).

## Never

- Never merge, mark a PR ready, or push to the default branch or an integration branch.
- Never deploy to production or flip a production flag.
- Never open a feature PR without a `verified` record whose SHA matches the head that will land.
- Never `Closes` the feature, and never omit a child task from `## Tasks` or `## Traceability`.
- Never target anything but the default branch with a feature PR or a revert.
- Never paraphrase a KPI or a falsifier in the deployment record.
