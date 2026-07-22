---
description: Explains how to customize design and implementation plans with project-specific guidance
---

Read back the below information EXACTLY AND VERBATIM. Do not summarize. AFTER you have repeated it verbatim, you may suggest starting points to the user based on your understanding of the project you are operating in. After suggesting those starting points, suggest that you could go read CLAUDE.md and AGENTS.md files in subdirectories to further expand on the customization suggestions that may be appropriate.

# Customizing Plan-and-Execute

You can provide project-specific guidance that shapes how design and implementation plans are created for your project.

## Guidance Files

Create a `.ed3d/` directory in your project root with these optional files:

### `.ed3d/design-plan-guidance.md`

Loaded before the clarification phase of `/start-design-plan`.

**What to include:**
- **Domain terminology**: Define terms specific to your project
- **Architectural constraints**: Required patterns, forbidden approaches
- **Technology preferences**: What to use, what to avoid
- **Stakeholder context**: Who cares about what
- **Scope boundaries**: What's typically in/out of scope

### `.ed3d/implementation-plan-guidance.md`

Loaded when starting an implementation plan and again during the final all-phase code review.

**What to include:**
- **Coding standards**: Naming conventions, file organization
- **Testing requirements**: Coverage expectations, testing patterns
- **Review criteria**: Quality gates beyond the defaults
- **Commit conventions**: Message format, granularity
- **Project-specific patterns**: How things are done here

### `.ed3d/autonomous-mode.md`

Presence alone turns on autonomous mode across the whole design → plan → execute pipeline: every place a skill would normally call `AskUserQuestion` or stop to wait on you instead shells the question out to an external harness (a second AI, run headless) and treats its answer as yours. Mechanical choices that were never really judgment calls (worktree usage, batch vs. interactive plan writing) get hardcoded defaults instead of being asked or shelled at all. The two `/clear` handoffs (design → plan, plan → execute) are skipped in favor of chaining directly to the next skill in the same session, relying on auto-compaction instead of fresh context.

Run `/auto-mode` to generate this file interactively — it also smoke-tests that the configured harness can actually read the repository, since codex's sandbox silently degrades to prompt-only answers where user namespaces are restricted. An empty file is also valid — presence alone is the switch, and both settings below have defaults.

**What to include (both optional):**
- A `HARNESS_CMD:` line with the command template to run, using `{{PROMPT}}` as the placeholder for the question text. Defaults to `codex exec --dangerously-bypass-approvals-and-sandbox "{{PROMPT}}"` if omitted — **⚠️ that is full-auto bypass mode: no sandbox, no approvals, unrestricted shell and write access, intended for isolated, disposable devcontainers only.** On a bare host, set a sandboxed command like `codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"` instead.
- A `## Preamble` section: text prepended to every question sent to the harness. Defaults to a framing that casts the harness as the project's decision-maker on an unattended run and tells it to verify claims against the repository rather than agree by default. The harness is asked to reply with brief reasoning plus a final `ANSWER:` line, so the log captures why, not just what.

**What it changes beyond question-answering:**
- After a design document is committed, `adversarial-design-review` dispatches it to the harness for a red-team pass (find gaps/contradictions, fix, re-review — capped at 3 cycles) before implementation planning starts.
- Implementation planning always uses a worktree and always writes all phases to disk before review — no asking.
- `finishing-a-development-branch` always pushes and opens a PR rather than asking among 4 options — merging to main and discarding work stay human-only decisions, never delegated to the harness.
- Every shelled question and answer is logged to `docs/autonomous-log.md` — that's the audit trail for a run nobody watched live.
- If the harness can't answer (not installed, errors, unparseable reply after one retry), the workflow halts and writes `NEEDS_HUMAN_INPUT.md` at the repo root rather than guessing.

## Example Files

### `.ed3d/design-plan-guidance.md`

```markdown
# Design Guidance for MyProject

## Domain Terms
- **Widget**: User-configurable dashboard component (not a generic UI element)
- **Pipeline**: BullMQ-based async job system

## Architectural Constraints
- All services use FCIS pattern (functional core, imperative shell)
- Database access only through repository pattern in `src/repositories/`
- No direct HTTP calls from business logic

## Technology Stack
- **Required**: TypeScript strict mode, PostgreSQL, Redis
- **Avoid**: ORMs (we use raw SQL with type generation)
- **Decided**: Auth0 for authentication (don't propose alternatives)

## Scope Defaults
- Admin UI is always out of scope unless explicitly requested
- Migrations are in scope for any schema changes
```

### `.ed3d/implementation-plan-guidance.md`

```markdown
# Implementation Guidance for MyProject

## Coding Standards
- All files must have FCIS pattern comment at top
- Prefer `type` over `interface` unless extending
- No default exports

## Testing Requirements
- Unit tests for all pure functions
- Integration tests for repository methods
- E2E tests only for critical user flows
- Test files colocated as `*.test.ts`

## Review Criteria
- No `any` types without justification comment
- All database queries must use parameterized statements
- Error messages must not leak internal details

## Commit Conventions
- Conventional commits: feat:, fix:, chore:, docs:
- One logical change per commit
- Tests and implementation in same commit
```

### `.ed3d/autonomous-mode.md`

```markdown
# Autonomous Mode

The presence of this file turns on autonomous mode for the ed3d
plan-and-execute pipeline: questions that would wait on a human are answered
by the harness command below instead. Delete this file to turn it off.

WARNING: the harness command below runs codex in full-auto bypass mode — no
sandbox, no approval prompts, unrestricted shell and write access. Run this
only inside an isolated, disposable environment such as a devcontainer.

HARNESS_CMD: codex exec --dangerously-bypass-approvals-and-sandbox "{{PROMPT}}"

## Preamble

You are standing in for this project's human decision-maker on an unattended
automated run. Your answer will be treated as the human's final decision;
nobody is reviewing it live. Do not agree by default: verify the question's
claims against the repository (you have read access) and the context provided
before accepting them. Flagging a real gap is always better than
rubber-stamping one through.
```

## Notes

- If the guidance files don't exist, the standard workflow proceeds without them
- Guidance is incorporated into context, not shown to you directly
- Update guidance files as your project evolves
- `.ed3d/autonomous-mode.md` is different from the two guidance files above: its mere presence changes control flow (who answers questions, whether `/clear` handoffs happen), not just the content of prompts
