---
name: asking-questions-autonomously
description: Use when a plan-and-execute skill would call AskUserQuestion or otherwise stop to wait on the user, and .ed3d/autonomous-mode.md exists at the repo root (presence alone is the switch - an empty file means the default harness command and preamble) - answers the question via an external harness instead of blocking, logs the exchange, and halts safely if the harness can't resolve it
user-invocable: false
---

# Asking Questions Autonomously

## Overview

Replace a wait-on-human moment with a wait-on-harness moment. Every other skill in this plugin still writes its question exactly as it would for a person; this skill intercepts it, gets an answer from an external harness (a second AI, run headless), and hands that answer back as if the user had provided it.

**Core principle:** Same question, different respondent. Never invent an answer yourself — if the harness can't produce one, stop and leave a clear trail for a human to pick up.

**Announce at start:** "I'm using the asking-questions-autonomously skill to get an answer from the configured harness instead of waiting on you."

## When This Applies

Only when `.ed3d/autonomous-mode.md` exists at the repo root. Every other skill in this plugin checks for that file before any `AskUserQuestion` call or other stop; if it's absent, skills behave exactly as before and ask the user directly. This skill is never invoked otherwise.

**Never use this for irreversible actions.** Discarding a branch, force-pushing, merging to main, or deleting work are handled by hardcoded safe defaults elsewhere in this plugin (see `finishing-a-development-branch`), not by asking a harness. This skill answers judgment calls about content and approach — it does not authorize destructive operations.

## Configuration

Read `.ed3d/autonomous-mode.md`. Its presence is the entire switch — an empty file is valid and means "use both defaults below." Do not stop to ask whether autonomous mode is intended; the file existing is that answer. (`/auto-mode` generates this file, but a hand-created or empty one works the same.) Two things may be configured, both optional:

**1. `HARNESS_CMD:` line** — a command template using `{{PROMPT}}` as a placeholder — a literal, fixed token, always replaced with the 8 characters `$PROMPT` (a shell variable reference), never with the prompt text itself. See Step 2 for why.

```
HARNESS_CMD: codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"
```

If there is no `HARNESS_CMD:` line, use this default:

```
codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"
```

`--sandbox read-only` is deliberate: the harness is answering a question, not editing files. It may read the repository for context, but it should never need write access to do that.

**2. `## Preamble` section** — text prepended to every harness prompt (Step 1). Everything under that heading, up to the next heading or end of file, is the preamble. If there is no `## Preamble` section, use this default:

```
You are standing in for this project's human decision-maker on an unattended
automated run. Your answer will be treated as the human's final decision;
nobody is reviewing it live. Do not agree by default: verify the question's
claims against the repository (you have read access) and the context provided
before accepting them. Flagging a real gap is always better than
rubber-stamping one through.
```

## The Process

### 1. Build the Prompt

Include, in this order:
1. The preamble (see Configuration). Every call, even for trivial-looking questions — models agree by default, and a bare "yes" is the cheapest reply there is; the preamble is what pushes the harness to scrutinize instead.
2. The exact question text the calling skill would have shown the user
3. Options, if any, each with its description (same as you'd pass to `AskUserQuestion`)
4. Relevant context the calling skill has on hand (design doc excerpt, Definition of Done, file paths, prior decisions) — enough that the harness isn't guessing blind
5. A closing instruction requiring reasoning plus a machine-readable final line:
   - With options: "State your reasoning in 1-3 sentences, checking the claims above against the repository and the context given rather than accepting them. Then end your reply with a final line of exactly: `ANSWER: <option label>`. If the label contains a placeholder like `<gap>`, replace the placeholder with your specifics."
   - Without options (open-ended): "State your reasoning in 1-3 sentences, then end your reply with a final line: `ANSWER: <short direct answer>`."

Never ask for the bare option label alone. A label-only reply makes the audit trail unauditable — it can't show whether the harness scrutinized anything or rubber-stamped — and it makes fill-in options like "Needs adjustment: \<gap\>" unanswerable, because the label without the gap is the one part that doesn't matter.

### 2. Bind the Prompt to a Shell Variable — Never Splice It Directly

**This step is not optional, even for short-looking prompts.** The text from Step 1 is content this skill doesn't control — design-doc excerpts and question text can contain `!`, embedded `"`, backticks, or `$(...)`. Spliced directly into a double-quoted command string, any of these can misfire: a literal `!` can trigger history expansion in a real interactive shell (`event not found`, or worse, silent substitution of an unrelated prior command), and a stray `` ` ``, `$(...)`, or `"` can break the quoting outright and run something other than what was intended. This is the same class of bug as SQL injection — untrusted text landing in a place that mixes data and code.

Bind it to a shell variable via a quoted heredoc instead. The quoted delimiter (`'PROMPT_EOF'`) disables all expansion of the body — no variable expansion, no command substitution, no history expansion — so the content passes through exactly as written, whatever it contains:

```bash
PROMPT=$(cat <<'PROMPT_EOF'
[the exact prompt text from Step 1, verbatim, unescaped]
PROMPT_EOF
)
```

### 3. Run the Harness

Take `HARNESS_CMD` and replace its `{{PROMPT}}` placeholder with the literal text `$PROMPT` (the variable reference from Step 2, not the prompt content). Run it with three defenses against hanging or losing output — headless model calls are slow, and a Bash call that produces nothing is otherwise impossible to diagnose:

1. **Set the Bash tool's own `timeout` parameter to at least 300000 (milliseconds), every time.** Its default is 120000ms (2 minutes) — well under what a headless model call needs. A 300-second harness call gets killed by the *tool* before the command finishes if you don't override this. It does not carry over from a previous call; set it explicitly on each one.
2. **Redirect stdin from `/dev/null`.** `codex exec` reads from stdin even when a prompt argument is given — without a closed stdin it prints "Reading additional input from stdin..." and hangs waiting for input that will never arrive. That hang, not the model call itself, is what actually eats the timeout budget. Verified: omitting `< /dev/null` produces that exact message and never returns; adding it returns cleanly.
3. **Capture combined stdout+stderr to a scratch file**, not just inline output, so a hung or failed process still leaves something to inspect.

**Do not wrap the command in a shell-level `timeout` utility.** It is GNU coreutils, not a POSIX/BSD standard tool — it doesn't exist by default on macOS (no `timeout`, no `gtimeout` without an extra Homebrew install), so a command built around it fails immediately with "command not found" on exactly the kind of machine this skill needs to run on. The Bash tool's own `timeout` parameter from step 1 is the only timeout enforcement needed; it doesn't depend on anything being installed in the target shell.

```bash
OUTPUT_FILE=$(mktemp)
codex exec --sandbox read-only -c approval_policy=never "$PROMPT" < /dev/null > "$OUTPUT_FILE" 2>&1
EXIT_CODE=$?
cat "$OUTPUT_FILE"
```

Treat a nonzero exit code, empty output, or a command-not-found error as harness failure (see Step 6).

### 4. Resolve the Answer

The decision is the last line of the reply that starts with `ANSWER:` (case-insensitive). Everything before that line is the harness's reasoning — keep it for the log (Step 5).

**With options:** Match the text after `ANSWER:` against the option labels (case-insensitive, tolerate minor wording drift). For a label with a placeholder (e.g. "Needs adjustment: \<gap\>"), match on its fixed prefix ("Needs adjustment:") — everything after the prefix is the placeholder content and travels with the decision; it's the substance the calling skill acts on.

**Without options:** Use the text after `ANSWER:` as the answer.

**If there is no `ANSWER:` line, or it matches no option:** retry once with a sharper closing instruction: "End your reply with a final line of exactly `ANSWER: <label>` where <label> is one of: [labels]." If the retry still doesn't resolve, treat as harness failure (Step 6). Don't infer a decision from reasoning prose that lacks an `ANSWER:` line — that line is what keeps the log unambiguous.

### 5. Log the Exchange

Append to `docs/autonomous-log.md` at the repo root (create it with a `# Autonomous Decision Log` header if it doesn't exist):

```markdown
## [calling skill/phase] — [one-line question summary]
**Asked:** [full question + options]
**Harness:** `[HARNESS_CMD with prompt substituted, truncated if long]`
**Reasoning:** [the harness's reasoning lines, verbatim — condense only if very long]
**Answered:** [the harness's ANSWER line, verbatim]
**Decision:** [resolved option label or answer text used, including any placeholder content]
```

This is the audit trail for a run nobody watched live. Log every exchange, not just the ones that mattered.

### 6. On Harness Failure — Stop, Don't Guess

If the harness command fails, times out, or its reply can't be resolved to an answer after one retry:

1. Write (or append to) `NEEDS_HUMAN_INPUT.md` at the repo root:
   ```markdown
   ## Blocked: [calling skill/phase]
   **Question:** [full question + options]
   **Why the harness failed:** [error text / timeout / unparseable reply]
   **To resume:** Answer the question above, then re-run the command that was in progress.
   ```
2. Stop the entire workflow — do not proceed past this point, do not pick a default yourself, and do not retry beyond the one retry in Step 4.

**Why stop instead of picking a reasonable-looking default:** a wrong guess here compounds silently through every phase downstream, and nobody is watching the run to catch it. A clear halt with full context costs the human two minutes to resolve; a silent wrong guess can cost hours of misdirected work.

## Common Rationalizations - STOP

| Excuse | Reality |
|--------|---------|
| "The harness reply is close enough to an option, I'll just pick the nearest one" | Retry with a sharper prompt first. If still ambiguous, that's a harness failure — log it as one. |
| "I can guess what the human would have picked" | You're not the human. Shell to the harness or halt. Never substitute your own judgment for a question the calling skill deliberately routed here. |
| "Logging is slowing this down" | The log is the only record of what happened. Always write it, even for trivial answers. |
| "This one destructive-adjacent choice is probably fine to shell out" | If an action can't be undone by editing a file, it doesn't go through this skill. Check `finishing-a-development-branch`'s hardcoded defaults instead. |
| "Harness failed once, I'll just proceed without an answer" | One retry, then halt via `NEEDS_HUMAN_INPUT.md`. Never proceed on an unresolved question. |
| "This prompt is short/simple, I'll splice it into the command directly" | Length and apparent simplicity don't matter — you don't control what the calling skill put in the prompt. Always bind via the Step 2 heredoc, every time. |
| "The reply has no ANSWER line but its intent is obvious" | Retry with the sharper instruction from Step 4. Inferring decisions from prose breaks the one-line audit trail the log depends on. |
| "The preamble bloats this simple question, I'll skip it" | The preamble is the only counterweight to the harness's agree-by-default bias — and simple-looking confirmations are exactly where rubber-stamps slip through. Include it on every call. |
| "The file is empty, so no harness is really configured" | Presence alone is the switch. An empty file means the default command and default preamble, not "not configured" — proceed autonomously. |

## Integration

**Called by:** Any plan-and-execute skill that would otherwise call `AskUserQuestion` or stop to ask the user, when `.ed3d/autonomous-mode.md` is present. See each skill's "Autonomous Mode" section for its specific call sites.

**Pairs with:** `.ed3d/autonomous-mode.md` (configuration — generated by `/auto-mode`, but hand-created or empty files work too), `docs/autonomous-log.md` (output trail), `NEEDS_HUMAN_INPUT.md` (failure escalation).
