---
name: asking-questions-autonomously
description: Use when a plan-and-execute skill would call AskUserQuestion or otherwise stop to wait on the user, and .ed3d/autonomous-mode.md exists at the repo root - answers the question via a configured external harness instead of blocking, logs the exchange, and halts safely if the harness can't resolve it
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

Read `.ed3d/autonomous-mode.md`. It contains a `HARNESS_CMD:` line with a command template using `{{PROMPT}}` as a placeholder — a literal, fixed token, always replaced with the 8 characters `$PROMPT` (a shell variable reference), never with the prompt text itself. See Step 2 for why.

```
HARNESS_CMD: codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"
```

If the file exists but has no `HARNESS_CMD:` line, use this default:

```
codex exec --sandbox read-only -c approval_policy=never "{{PROMPT}}"
```

`--sandbox read-only` is deliberate: the harness is answering a question, not editing files. It may read the repository for context, but it should never need write access to do that.

## The Process

### 1. Build the Prompt

Include, in this order:
1. The exact question text the calling skill would have shown the user
2. Options, if any, each with its description (same as you'd pass to `AskUserQuestion`)
3. Relevant context the calling skill has on hand (design doc excerpt, Definition of Done, file paths, prior decisions) — enough that the harness isn't guessing blind
4. A closing instruction:
   - With options: "Reply with ONLY the option label that best answers this, nothing else."
   - Without options (open-ended): "Reply with a short, direct answer."

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

Take `HARNESS_CMD` and replace its `{{PROMPT}}` placeholder with the literal text `$PROMPT` (the variable reference from Step 2, not the prompt content), then run the result via Bash with a generous timeout (300s) — headless model calls can be slow:

```bash
codex exec --sandbox read-only -c approval_policy=never "$PROMPT"
```

Capture stdout. Treat a nonzero exit code, empty stdout, or a command-not-found error as harness failure (see Step 6).

### 4. Resolve the Answer

**With options:** Match the harness's reply against the option labels (case-insensitive, allow partial/fuzzy match — the harness may add a stray word). If exactly one option matches, that's the decision. If none match clearly, retry once with a sharper prompt: "Your answer must be exactly one of: [labels]. Reply with only the label." If the retry still doesn't match, treat as harness failure.

**Without options:** Use the reply text directly as the answer.

### 5. Log the Exchange

Append to `docs/autonomous-log.md` at the repo root (create it with a `# Autonomous Decision Log` header if it doesn't exist):

```markdown
## [calling skill/phase] — [one-line question summary]
**Asked:** [full question + options]
**Harness:** `[HARNESS_CMD with prompt substituted, truncated if long]`
**Answered:** [raw harness reply]
**Decision:** [resolved option label or answer text used]
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

## Integration

**Called by:** Any plan-and-execute skill that would otherwise call `AskUserQuestion` or stop to ask the user, when `.ed3d/autonomous-mode.md` is present. See each skill's "Autonomous Mode" section for its specific call sites.

**Pairs with:** `.ed3d/autonomous-mode.md` (configuration), `docs/autonomous-log.md` (output trail), `NEEDS_HUMAN_INPUT.md` (failure escalation).
