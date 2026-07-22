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
- "Default: codex full-auto (Recommended)" — description: ⚠️ FULL AUTO: `codex exec --dangerously-bypass-approvals-and-sandbox "{{PROMPT}}"` — no sandbox, no approval prompts, unrestricted shell and write access. Only sane inside an isolated, disposable environment (e.g. a devcontainer). Requires the codex CLI installed and authenticated.
- "Custom command" — description: You provide a command template; use `{{PROMPT}}` where the question text goes. On a bare host, use a sandboxed variant like `codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"` — the harness answers questions, it doesn't need write access.

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

If the chosen harness command includes `--dangerously-bypass-approvals-and-sandbox`, insert this paragraph into the file between the intro paragraph and the `HARNESS_CMD:` line, so the file itself carries the warning:

```markdown
WARNING: the harness command below runs codex in full-auto bypass mode — no
sandbox, no approval prompts, unrestricted shell and write access. Run this
only inside an isolated, disposable environment such as a devcontainer.
```

The default harness command is:

```
codex exec --dangerously-bypass-approvals-and-sandbox "{{PROMPT}}"
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

### 4. Smoke-test the harness

A harness can "work" while being unable to read the repository. With sandboxed codex commands (e.g. `--sandbox read-only`), the sandbox needs unprivileged user namespaces, and where the kernel or AppArmor forbids them (common in containers and on Ubuntu 23.10+), codex silently answers from the prompt alone — exit code 0, no error. The full-auto default avoids that failure mode entirely but still depends on codex being installed and authenticated. Content, not exit codes, is the only reliable signal either way, so probe it:

1. Read the first line of `README.md` at the repo root yourself (if there is no README, pick another small tracked text file and use it in the probe below).
2. Build this probe prompt: "Read the file README.md in the current working directory and reply with its first line, verbatim." Bind it to a shell variable with a quoted heredoc (`PROMPT=$(cat <<'PROMPT_EOF' ... PROMPT_EOF\n)`), replace `{{PROMPT}}` in the chosen harness command with `$PROMPT`, and run it with stdin redirected from `/dev/null`, stdout+stderr captured to a scratch file, and the Bash tool's `timeout` parameter set to at least 300000 milliseconds — headless model calls routinely outlast the tool's 2-minute default.
3. Judge by content: the reply must contain the file's actual first line.

**If the reply contains the line:** report that the harness is reachable and can read the repository.

**If the command fails outright (not found, auth error):** the codex CLI needs installing or authenticating — an unreachable harness halts the pipeline at its first question.

**If the reply doesn't contain the line and the harness command uses a codex sandbox flag:** the sandbox likely failed to initialize. Autonomous mode will still run, but the harness will answer every question from prompt context alone — noticeably worse judgment on anything that needs repository knowledge. Quote the harness output to the user and present the remedies:

- **Environment fix:** allow unprivileged user namespaces. On Ubuntu 24.04-family hosts: `sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict` (from `apparmor-profiles`) or `sudo sysctl kernel.apparmor_restrict_unprivileged_userns=0`. Containers must be started with user namespaces permitted.
- **Codex backend switch (stopgap):** add `-c use_legacy_landlock=true` to the `HARNESS_CMD:` line — the Landlock backend needs no user namespaces and works in many containers where bwrap cannot. Codex marks this option deprecated and slated for removal, so use it to unblock now and plan the environment fix.
- **Switch to the full-auto default** (`--dangerously-bypass-approvals-and-sandbox`) where the environment itself is disposable and contained — it gives the harness shell and write access, so only there.

Leave the written file in place either way — the user decides whether to fix the environment or accept a repo-blind harness.

### 5. Confirm and explain

Tell the user:

- **⚠️ If the default harness command is in use, say this first and prominently:** the harness runs codex in FULL-AUTO bypass mode — no sandbox, no approval prompts, unrestricted shell and write access. That is only appropriate because this environment is assumed to be an isolated, disposable container; if this repo ever runs autonomous mode on a bare host, reconfigure `HARNESS_CMD:` with a sandboxed command first.
- Autonomous mode is now ON for this repo. The next `/start-design-plan` chains straight through design → plan → execute to a pushed PR without stopping; to turn it off, delete `.ed3d/autonomous-mode.md`.
- Decide deliberately whether to commit the file: committed, autonomous mode is on for everyone who clones the repo; left untracked (or gitignored), it's on for this machine only.
- Every harness exchange is logged to `docs/autonomous-log.md`. If the harness fails or its answer can't be resolved, the run halts and writes `NEEDS_HUMAN_INPUT.md` at the repo root instead of guessing.
- The smoke-test result from Step 4: a harness that can't read the repository answers from prompt context alone; an unreachable one halts the pipeline at its first question.
