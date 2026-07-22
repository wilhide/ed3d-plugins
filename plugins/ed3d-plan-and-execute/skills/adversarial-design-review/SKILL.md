---
name: adversarial-design-review
description: Use after a design document is committed in autonomous mode - dispatches the full design to an external harness for red-team review, fixes Critical/Important issues it finds, and loops up to three cycles before proceeding with any unresolved issues logged for human review
user-invocable: false
---

# Adversarial Design Review

## Overview

A second opinion from a genuinely different model, run before implementation planning ever begins. The same model that wrote a design is bad at finding its own blind spots; a harness with no stake in the design and an explicit instruction to find fault catches things brainstorming's incremental validation missed.

**Core principle:** Red-team, don't rubber-stamp. Fix what's fixable, log what isn't, never block the pipeline indefinitely on it.

**Announce at start:** "I'm using the adversarial-design-review skill to get a second opinion on this design before implementation planning starts."

## When This Applies

Only when `.ed3d/autonomous-mode.md` exists at the repo root, invoked by `starting-a-design-plan` immediately after the design document is committed (end of Phase 5, before the handoff to implementation planning). In interactive mode, the human's own incremental validation during brainstorming and the Acceptance Criteria approval step serve this role instead — this skill doesn't run.

## The Process

### 1. Read Configuration and Design

Read `.ed3d/autonomous-mode.md` for `HARNESS_CMD` (same convention as `ed3d-plan-and-execute:asking-questions-autonomously` — see that skill for the format and default).

Read the full design document that was just committed.

### 2. Dispatch the Review

Build a prompt instructing the harness to find problems, not confirm quality:

```
Review this design document as an adversarial reviewer. Your job is to find
problems, not to confirm it looks fine — assume there are issues and dig for
them.

Check for:
- Contradictions between the Definition of Done, Acceptance Criteria, and
  Implementation Phases
- Gaps: requirements with no phase that addresses them
- Infeasibility: phases that assume something the design doesn't establish
- Scope creep or missing scope boundaries
- Acceptance criteria that aren't actually observable/testable

For each issue found, report: severity (Critical/Important/Minor), the
specific section and text it applies to, and what's wrong.

If you find nothing wrong after genuinely looking, say so explicitly — but
default to skepticism, not approval.

--- DESIGN DOCUMENT ---
[full document text]
```

Run via the harness command, same mechanics as `asking-questions-autonomously` Step 2 (300s timeout, treat command failure as harness failure).

**If the harness fails to respond:** follow the same halt behavior as `asking-questions-autonomously` Step 5 (write `NEEDS_HUMAN_INPUT.md`, stop). Don't skip review silently just because the harness was unreachable.

### 3. Handle the Findings

**If zero issues reported:** log to `docs/autonomous-log.md` and proceed to the handoff.

**If issues reported:**

1. Log the full findings to `docs/autonomous-log.md`.
2. Fix Critical and Important issues directly by editing the design document (adjust Definition of Done, Acceptance Criteria, or Implementation Phases as needed to resolve the specific problem raised).
3. Minor issues: fix if the fix is small and unambiguous; otherwise carry them into step 4 below rather than guessing at scope changes.
4. Re-commit the design document with a `docs: address adversarial review findings for <topic>` commit.
5. Re-run the review (Step 2) against the updated document.

**Loop up to 3 cycles.** This mirrors the three-strike pattern used elsewhere in this plugin's review loops (see `executing-an-implementation-plan`).

### 4. After 3 Cycles (or Zero Issues)

If issues remain after 3 cycles, do not block the pipeline. Append the outstanding issues to the design document's "Additional Considerations" section under a `**Unresolved adversarial review findings:**` heading, log the full history to `docs/autonomous-log.md`, and proceed to the handoff. These are now visible to whoever reviews the eventual PR — the design doc is git-tracked and reversible, so proceeding with a documented gap is safer than stalling an otherwise-working pipeline over it.

**Never let this loop block on Minor-only findings.** Only unresolved Critical or Important issues are worth flagging prominently in Additional Considerations; note Minor leftovers in the same section but don't let them extend the cycle count.

## Common Rationalizations - STOP

| Excuse | Reality |
|--------|---------|
| "The harness said it's mostly fine, I'll count that as approved" | "Mostly fine" is not "zero issues." Treat any reported issue as something to fix or log. |
| "I already validated this during brainstorming, this is redundant" | Brainstorming validation happened with you as both author and reviewer. This is a different model with no stake in the design — that's the point. |
| "Fixing the design doc now feels like re-litigating an approved decision" | The whole point of autonomous mode is that decisions get revisited by machine review instead of a human. Fix real issues. |
| "3 cycles isn't converging, I'll keep going" | Stop at 3. Log what's left and move on — indefinite looping over the same document wastes the run. |

## Integration

**Called by:** `starting-a-design-plan`, after Phase 5 (Design Documentation) commits the design doc, only when `.ed3d/autonomous-mode.md` is present.

**Pairs with:** `.ed3d/autonomous-mode.md` (configuration, shared with `asking-questions-autonomously`), `docs/autonomous-log.md` (output trail).
