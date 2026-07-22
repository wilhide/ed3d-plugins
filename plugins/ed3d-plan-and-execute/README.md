# ed3d-plan-and-execute

A workflow plugin for Claude Code that guides you from rough idea to working implementation through structured design, planning, and execution phases.

## The Problem

Claude Code is excellent at implementing specific, well-defined tasks. But when you have a feature idea that's still forming - where you don't yet know exactly what you want, let alone how to build it - you need more structure. You need to explore alternatives, ground your design in the actual codebase, and break work into verifiable steps.

This plugin provides that structure through three connected commands.

## The Workflow

```
Rough Idea
    │
    ▼
/start-design-plan  ──────► Design Document (committed to git)
    │
    ▼
/start-implementation-plan ──► Implementation Plan (phase files)
    │
    ▼
/execute-implementation-plan ──► Working Code (reviewed & committed)
```

Each phase produces artifacts that feed the next. By default you clear context between phases to ensure fresh, focused work — or run the whole thing unattended in [Autonomous Mode](#autonomous-mode), which chains straight through instead.

---

## Philosophy: What Each Phase Produces

**Design (archival)** — Can be checked into git and referenced months later. Describes WHAT to build and WHY at module/component level. Fully specifies contracts (APIs, interfaces) because other designs may depend on them. Does NOT include implementation code — that's intentional.

**Implementation Plan (just-in-time)** — Created immediately before execution. Verifies current codebase state matches design assumptions. Generates fresh, executable code based on actual state. May diverge from design if codebase has changed. Tasks are 2-5 minutes of work each.

**Execution** — Follows implementation plan exactly. Code review at every step ensures quality.

The key insight: **design plans tell you where you're going; implementation plans tell you how to get there from where you are now.** A design plan written two weeks ago is still valid direction. But the implementation plan must be generated fresh because the codebase may have changed.

---

## Phase 1: Design (`/start-design-plan`)

**What you provide:** A rough idea, some constraints, maybe URLs to relevant docs.

**What happens:**

1. **Context Gathering** - Claude asks what you're building, what constraints exist, what you've already decided.

2. **Clarification** - Ambiguous terms get disambiguated. "OAuth2" becomes "client credentials flow for service accounts." "Users" becomes "internal services, not humans." This prevents building the wrong thing.

3. **Brainstorming** - Claude proposes 2-3 architectural approaches with trade-offs. A codebase investigator subagent finds existing patterns your design should follow. You pick an approach through incremental validation - small sections presented for feedback.

4. **Design Documentation** - The validated design gets written to `docs/design-plans/YYYY-MM-DD-<topic>.md` with explicit implementation phases (≤8).

4. **Acceptance Criteria** - Definition of Done gets translated into specific, verifiable criteria. You validate these before documentation completes.

**Output:** A committed design document with clear phases, explicit file paths, "done when" criteria, and validated acceptance criteria.

---

## Phase 2: Planning (`/start-implementation-plan @design-doc.md`)

**What you provide:** Path to the design document from Phase 1.

**What happens:**

1. **Branch Setup** - Claude asks if you want a git worktree (isolated workspace) or standard branch. Creates the branch from main/master.

2. **Codebase Verification** - For each design phase, a codebase investigator verifies that assumptions about existing files, patterns, and dependencies are accurate. If the design says "auth is in src/services/auth.ts," the investigator confirms it exists and has the expected structure.

3. **Task Creation** - Each design phase becomes detailed tasks with:
   - Exact file paths (confirmed by investigator)
   - Complete code examples (no TODOs or placeholders)
   - Specific verification commands and expected output
   - Clear commit points

4. **Plan Validation** - A code reviewer validates that the implementation plan fully covers the design before you start.

5. **Test Requirements** - Acceptance criteria are mapped to specific automated tests (with expected file paths) or documented as requiring human verification. This becomes `test-requirements.md` in the plan directory.

**Output:** Implementation plan files in `docs/implementation-plans/YYYY-MM-DD-<feature>/` with one file per phase plus `test-requirements.md`.

---

## Phase 3: Execution (`/execute-implementation-plan @plan-directory/`)

**What you provide:** Path to the implementation plan directory (not a single phase file — pass the directory so all phases execute).

**What happens:**

For each task (or group of related tasks that complete a subcomponent):

1. **Dispatch Implementor** - A task-implementor subagent implements exactly what the task specifies, using TDD (test first, then code), and commits.

2. **Code Review** - A code-reviewer subagent verifies:
   - Tests pass, build succeeds, linter clean
   - Implementation matches plan requirements
   - Code quality standards met (FCIS pattern, type safety, error handling)
   - No shortcuts or missing coverage

3. **Fix Loop** - If issues are found (Critical, Important, or Minor), a bug-fixer subagent resolves them. Re-review continues until zero issues.

4. **Progress** - Task marked complete, move to next task.

After all tasks:

5. **Project Context Update** - A librarian subagent checks if CLAUDE.md files need updating based on what changed.

6. **Final Review** - Full implementation reviewed against all requirements.

7. **Test Analysis** - A test-analyst validates that automated tests cover all acceptance criteria. Missing coverage triggers a fix loop. Once coverage passes, a human test plan is generated.

8. **Completion Options** - Merge to main, create PR, keep branch, or discard.

**Output:** Working, reviewed code on your feature branch with clean commits, plus a human test plan in `docs/test-plans/`.

---

## Why This Structure?

**Design before code.** Brainstorming surfaces constraints and alternatives you'd otherwise discover mid-implementation. The design document becomes a contract between "what we decided" and "what we'll build."

**Plans grounded in reality.** Codebase investigation confirms assumptions. You won't write a plan that references files that don't exist or patterns that aren't followed.

**Bite-sized, verifiable tasks.** Each task is 2-5 minutes of work with explicit verification. No task depends on "this will exist somehow" - dependencies are explicit.

**Code review at every step.** Issues caught early are cheaper than issues caught at PR review. The review-fix loop runs until zero issues, not until "good enough."

**Fresh context between phases (by default).** You /clear between design → plan and plan → execute. Each phase gets full context for its specific job. [Autonomous Mode](#autonomous-mode) trades this for chaining straight through and relying on auto-compaction instead.

---

## Working with Larger Problems

For larger efforts, we've found success in first decomposing the problem before starting the design phase.

**Identify independent and dependent parts.** Before running `/start-design-plan`, sketch out which parts of the system can be built independently and which have dependencies. A service that other services call should be built first. A UI that consumes an API depends on that API existing.

**Build blocks of specifications.** Each independent block becomes its own input to the design planner. Rather than one massive design, you get several focused designs that can be planned and executed separately.

**Chain the designs.** Run `/start-design-plan` for the foundational blocks first. Their completed implementations become context for dependent blocks. This prevents the "design assumes X exists but it doesn't" problem.

This decomposition happens before you touch the plugin - it's thinking work you do to scope what goes into each design cycle.

---

## Required Plugins

This plugin uses subagents and skills from other plugins. Install these for full functionality:

| Plugin | What It Provides | Required For |
|--------|------------------|--------------|
| **ed3d-research-agents** | `codebase-investigator`, `internet-researcher` | Codebase verification, external research during design |
| **ed3d-house-style** | `coding-effectively` and sub-skills | Code quality standards during implementation and review |
| **ed3d-extending-claude** | `project-claude-librarian` | Updating CLAUDE.md files after implementation |

Without these plugins, the workflow will still run but will skip the corresponding subagent dispatches (with a warning).

---

## Subagents

The plugin uses specialized subagents for different roles:

| Agent | Plugin | Role |
|-------|--------|------|
| **codebase-investigator** | ed3d-research-agents | Verifies file paths, finds patterns, confirms assumptions |
| **internet-researcher** | ed3d-research-agents | Finds current API docs, library patterns, best practices |
| **task-implementor-fast** | ed3d-plan-and-execute | Implements tasks with TDD, runs verification, commits |
| **code-reviewer** | ed3d-plan-and-execute | Enforces quality standards, blocks on issues |
| **task-bug-fixer** | ed3d-plan-and-execute | Fixes issues identified by code reviewer |
| **test-analyst** | ed3d-plan-and-execute | Validates test coverage against acceptance criteria, generates human test plans |
| **project-claude-librarian** | ed3d-extending-claude | Updates CLAUDE.md files when contracts change |

You interact with the main orchestrating agent. It dispatches subagents and shows you their full responses.

---

## Getting Started

```bash
# Start with an idea
/start-design-plan
```

Claude will guide you through context gathering, brainstorming, and design documentation.

When design is complete, you'll get instructions to copy the next command, then /clear:

```bash
# Copy this command first, then run /clear, then paste it
/start-implementation-plan @docs/design-plans/2025-01-14-your-feature.md .
```

After planning, same pattern:

```bash
# Copy this command first, then run /clear, then paste it
/execute-implementation-plan @docs/implementation-plans/2025-01-14-your-feature .
```

**Or, run it unattended:** run `/auto-mode` first (see [Autonomous Mode](#autonomous-mode)) and `/start-design-plan` chains straight through to a pushed PR without stopping for you.

---

## Utility Command: `/flesh-it-out`

Not every idea needs the full design-plan-execute workflow. Sometimes you have a rough concept - a feature description, a technical approach, a document draft - that just needs to be made more specific and coherent.

`/flesh-it-out` uses the clarifying-questions skill in standalone mode. It focuses on understanding what you actually mean, not just what you said:

- **Surfaces contradictions** - "Real-time updates" and "batch processing is fine" pull in different directions. Which do you actually need?
- **Disambiguates terminology** - "OAuth2" could mean authorization code flow, client credentials, or both. Which one?
- **Clarifies scope boundaries** - "Users" might mean human customers, service accounts, internal employees, or all of the above.
- **Verifies assumptions** - "Must use library X" might be a hard requirement, team preference, or outdated guideline.

The goal is to resolve unacknowledged trade-offs and turn vague intentions into concrete requirements. The output might become input to `/start-design-plan` later, or it might just be clearer thinking about a problem you're not ready to solve yet.

---

## Customization

Provide project-specific guidance by creating files in a `.ed3d/` directory:

- `.ed3d/design-plan-guidance.md` — Loaded before clarification in `/start-design-plan`. Define domain terminology, architectural constraints, technology preferences, and scope boundaries.
- `.ed3d/implementation-plan-guidance.md` — Loaded when creating implementation plans and during final code review. Specify coding standards, testing requirements, and review criteria.
- `.ed3d/autonomous-mode.md` — Turns on [Autonomous Mode](#autonomous-mode). Presence alone changes control flow, not just prompt content.

Run `/how-to-customize` for details and example files.

---

## Autonomous Mode

Run `/auto-mode` (or create `.ed3d/autonomous-mode.md` yourself — presence alone is the switch, an empty file works) and the whole pipeline runs unattended, from `/start-design-plan` through a pushed PR, without stopping for you at any point along the way.

**What changes:**

- **Every `AskUserQuestion` gets shelled out instead.** Anywhere a skill would normally stop and ask you something — clarifying questions, Definition of Done confirmation, "does this look right so far," blocked implementation decisions, three-strike review escalations — it instead runs a configured external harness (a second AI, headless) with the same question and treats the answer as yours.
- **A second model red-teams the design.** After the design document is committed, `adversarial-design-review` sends it to the harness with instructions to find gaps and contradictions, not rubber-stamp it. Fixes loop up to 3 cycles before moving on.
- **Mechanical choices get hardcoded, not asked.** Implementation planning always uses a worktree and always writes all phases to disk before review — these were never real judgment calls, so nothing gets shelled out for them either.
- **No `/clear` handoffs.** Design → plan → execute chains directly in the same session via the Skill tool, relying on auto-compaction instead of fresh context per phase.
- **Finishing always opens a PR.** Merging to main and discarding work stay human-only decisions — autonomous mode never picks either, regardless of what a harness might say.
- **Every decision is logged.** `docs/autonomous-log.md` records each shelled question, the harness's reasoning and answer, and the decision taken — the audit trail for a run nobody watched live.
- **Failure halts, it doesn't guess.** If the harness can't be reached or its answer can't be resolved, the workflow writes `NEEDS_HUMAN_INPUT.md` at the repo root with full context and stops rather than picking a default on a real judgment call.

Every question is sent with a preamble that casts the harness as the project's decision-maker on an unattended run — instructed to verify claims against the repository rather than agree by default — and the harness must reply with brief reasoning plus a final `ANSWER:` line, so the log captures why, not just what.

Configure with a `HARNESS_CMD:` line (defaults to `codex exec` if omitted) and an optional `## Preamble` section (defaults to the framing above). `/auto-mode` sets the file up interactively; run `/how-to-customize` for the file format and more detail.

---

## What This Is Not

- **Not for simple tasks.** If you know exactly what to change and it's a few files, just do it. This workflow adds overhead that pays off for larger features.

- **Not infinitely scoped.** Design phases are capped at 8 to keep implementations tractable. Larger efforts split into multiple implementation plans.

---

## Attribution

This plugin is derived from [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.

## License

The original [obra/superpowers](https://github.com/obra/superpowers) code is licensed under the MIT License, copyright Jesse Vincent. See `LICENSE.superpowers`.

All modifications and additions are licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/), copyright Ed Ropple.
