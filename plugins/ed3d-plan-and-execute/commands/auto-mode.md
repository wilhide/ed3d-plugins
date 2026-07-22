---
description: Set up autonomous mode by configuring .ed3d/autonomous-mode.md
---

You are setting up autonomous mode for the plan-and-execute pipeline. The presence of `.ed3d/autonomous-mode.md` at the repo root makes the whole design → plan → execute pipeline run unattended: every question that would wait on a human is answered by an external harness command (a second AI, run headless) instead. This command creates that file deliberately, with the user choosing the harness command and the preamble.

## Steps

### 1. Check for an existing file

If `.ed3d/autonomous-mode.md` already exists, autonomous mode is already on — presence alone is the switch. Show the user its current contents and use AskUserQuestion to ask whether to reconfigure it (continue to Step 2, overwriting) or leave it as is (stop here).

### 2. Ask what to configure

Use AskUserQuestion with two questions:

**Question 1 — header "Harness cmd":** "Which harness command should answer questions during autonomous runs?"
- "Default: codex exec (Recommended)" — description: `codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"`. Read-only sandbox; requires the codex CLI installed and authenticated.
- "Custom command" — description: You provide a command template. Use `{{PROMPT}}` where the question text goes; keep the harness read-only if you can — it answers questions, it doesn't edit files.

**Question 2 — header "Preamble":** "What preamble should precede every question sent to the harness?"
- "Default preamble (Recommended)" — description: Frames the harness as the project's human decision-maker on an unattended run and instructs it to verify claims against the repository instead of agreeing by default. This is what pushes it to scrutinize rather than rubber-stamp.
- "Custom preamble" — description: You provide the text prepended to every harness prompt.

If the user picks a custom option and hasn't already supplied the text (e.g. via "Other"), ask for it in conversation before proceeding.

### 3. Write the file

Create the `.ed3d/` directory if it doesn't exist. Write `.ed3d/autonomous-mode.md` with exactly this structure, substituting the chosen harness command and preamble:

```markdown
# Autonomous Mode

The presence of this file turns on autonomous mode for the ed3d
plan-and-execute pipeline: questions that would wait on a human are answered
by the harness command below instead. Delete this file to turn it off.

HARNESS_CMD: [chosen harness command]

## Preamble

[chosen preamble]
```

The default harness command is:

```
codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"
```

The default preamble is (keep in sync with the default in ed3d-plan-and-execute:asking-questions-autonomously):

```
You are standing in for this project's human decision-maker on an unattended
automated run. Your answer will be treated as the human's final decision;
nobody is reviewing it live. Do not agree by default: verify the question's
claims against the repository (you have read access) and the context provided
before accepting them. Flagging a real gap is always better than
rubber-stamping one through.
```

Write both values into the file even when the defaults are chosen — the file should show what will actually run, so the user can read and tweak it later without consulting the skill. Write `{{PROMPT}}` literally: it is a placeholder the asking-questions-autonomously skill substitutes at call time; never replace it with anything yourself.

### 4. Confirm and explain

Tell the user:

- Autonomous mode is now ON for this repo. The next `/start-design-plan` chains straight through design → plan → execute to a pushed PR without stopping; to turn it off, delete `.ed3d/autonomous-mode.md`.
- Decide deliberately whether to commit the file: committed, autonomous mode is on for everyone who clones the repo; left untracked (or gitignored), it's on for this machine only.
- Every harness exchange is logged to `docs/autonomous-log.md`. If the harness fails or its answer can't be resolved, the run halts and writes `NEEDS_HUMAN_INPUT.md` at the repo root instead of guessing.
- If the default harness command is in use, verify the `codex` CLI is installed and authenticated before kicking off a run — an unreachable harness halts the pipeline at its first question.
