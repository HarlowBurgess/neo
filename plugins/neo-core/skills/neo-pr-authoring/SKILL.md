---
name: neo-pr-authoring
description: Use when opening or checking the draft pull request that ends the Coding loop — the artifact that crosses Boundary 2 (Coding → Verification). Defines which base branch the PR targets under each integration mode, the title form, the required body sections (task and parent-feature links, validation report, review ledger, checks), and which closing keyword to use. Load before running `gh pr create`, and whenever checking a Coding-loop PR's shape.
---

# Authoring the Coding loop's draft PR

The draft PR is the one artifact that leaves the Coding loop. It closes exactly one **Task**, links through it to one **Feature**, and carries the proof that the task was validated and every step was reviewed — so a human can judge it without re-deriving any of that. Open it only when every planned step is review-approved **and** the Validator's verdict is `validated`.

It is always a **draft**. Never mark it ready for review, and never merge it.

## Base branch

Pick the first rule that applies:

1. **A `base_branch` was handed to you at kickoff** (the task depends on a sibling that hasn't merged yet) → target that branch, so the PR stacks on its predecessor.
2. **Mode A** (feature branch, squash to main — Neo's default) → target the parent feature's integration branch, `feature/<feature-id>-<short-name>`.
3. **Mode B** (tasks to main behind flags) → target the repository's default branch.

The integration mode comes from the consuming repo's `AGENTS.md`. If it declares none, the mode is **A**, and the PR says it was defaulted. The modes themselves are defined in `docs/concepts/process-flow.md` § Integration modes.

## Title

Conventional Commits form, naming the task's change: `<type>[optional scope]: <description>` — e.g. `feat(checkout): add checkout POST endpoint`. Use `feat` for new behavior, `fix` for a correction. Never `wip`, never the bare issue title.

## Closing keyword

GitHub acts on a closing keyword in a PR description **only when the PR targets the default branch**; on any other base the keyword is ignored.

| PR base | Task line |
| --- | --- |
| The default branch (Mode B, not stacked) | `Closes #<task>` |
| Anything else (Mode A, or stacked on a sibling) | `Refs #<task>` |

Under Mode A the tasks close later, when the verified feature branch squash-merges to the default branch and its commit body lists every child task — see `docs/concepts/process-flow.md` § Traceability under squash. That merge is outside the Coding loop.

On Azure DevOps, link the task's work item instead, using the project's configured linking convention.

## Body

Exactly these sections, in this order. Every one is required; write `None` rather than omit one.

```markdown
## Task
Closes #<task>        <!-- or: Refs #<task> — see Closing keyword -->

## Parent feature
#<feature>

## Integration mode
<A | B> — <declared in AGENTS.md | defaulted to A>. Base: `<base-branch>`.
<!-- Mode B only: Flag: `<flag-name>` gates every feature step. -->

## Summary
<what changed and why, in a few lines — the task's What, as built>

## Validation
<the Validator's criteria table, verbatim: # | criterion | proof | result | evidence>

Verdict: validated at `<head-sha>`.

## Review ledger
<N>/<N> steps approved by Neo Code Reviewer.

| Step | Type | Commit(s) | Verdict |
| --- | --- | --- | --- |
| 1 | feature | `<sha>` (+ fix-up `<sha>`) | approve |
| 2 | test | `<sha>` | approve |

## Checks
<each build / lint / test command run on the head commit, and its result>
```

Rules for the body:

- **Validation is pasted, not summarized.** The table is the Validator's output, unedited. A reader must be able to see which check proved which criterion.
- **The ledger counts every planned step.** `N/N` means every step on the plan was reviewed and approved — including any added late. If the counts differ, the PR is not ready.
- **Nothing aspirational.** No "should work", no untested claims, no follow-ups dressed as done. Open questions belong in a PR comment addressed to the human reviewer.

## Never

- Never open the PR as ready-for-review, and never merge it.
- Never target the default branch under Mode A.
- Never use `Closes` on a PR whose base is not the default branch — it silently does nothing.
- Never open the PR with a step unreviewed or a criterion not `pass`.
