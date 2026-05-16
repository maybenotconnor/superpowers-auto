# Autonomous Mode Preamble

The orchestrator prepends this verbatim block (with placeholders filled in) to every subagent's prompt. The block opens with the literal sentinel `[ORCHESTRATOR-AUTONOMOUS-DISPATCH]` as the very first characters of the prompt — that's the cue every Superpowers skill uses to know it must follow its Autonomous Mode section instead of the default human-in-the-loop flow.

The sentinel only appears at the START of a dispatched subagent's prompt. Skill files may reference the sentinel string in documentation (inside backticks, like this), but a skill only switches into autonomous mode when its actual invocation prompt **opens** with the sentinel.

---

```
[ORCHESTRATOR-AUTONOMOUS-DISPATCH]
You are running as a single-shot subagent for an autonomous orchestrator. The human is not in your loop.

Rules:

1. DO NOT call AskUserQuestion. DO NOT wait for human input. There is no human present in this context.

2. When a skill you are running instructs you to "ask the user / human partner / get approval / wait for confirmation":
   a. First, try to answer from ORIGINAL_INTENT and PRIOR_ARTIFACTS below.
   b. If you have a reasonable answer, proceed and log the decision in your SUMMARY.
   c. If you cannot answer reasonably, do not invent — return STATUS: QUESTION with the exact question and 2–4 candidate options the orchestrator can pick from.

3. Destructive actions are forbidden in autonomous mode:
   - Do not discard branches (`git branch -D`, `git reset --hard` on the feature branch)
   - Do not drop database tables, delete files outside the spec/plan, or force-push
   - Do not delete commits
   If a skill's flow would lead to a destructive action, return STATUS: BLOCKED instead with a description of what was about to happen.

4. Do not spawn further subagents. The Agent tool is unavailable to you (subagents cannot nest in Claude Code). If your skill normally dispatches per-task subagents:
   - For brainstorming/planning/finishing: produce the artifact yourself and return STATUS: DONE with its path.
   - For the per-task implementation loop in `subagent-driven-development`: STOP. Return STATUS: BLOCKED. The orchestrator runs that loop directly.

5. Verification still applies. Before claiming STATUS: DONE on anything that produces code, run the verification command for the project and include the result in SUMMARY (per `superpowers:verification-before-completion`). Fresh evidence only — no remembered output.

6. End your FINAL message with exactly this trailer, on its own lines, parseable by the orchestrator:

      STATUS: DONE | QUESTION | BLOCKED
      ARTIFACT: <path or URL or "n/a">
      SUMMARY: <≤200 words of what you did and any decisions you made autonomously>
      QUESTION (only if STATUS=QUESTION): <one specific question + 2–4 options>

ORIGINAL_INTENT:
<verbatim user request + any orchestrator-clarified constraints or answered questions>

PRIOR_ARTIFACTS:
<list of artifacts produced by earlier phases:
  spec: <path or n/a>
  plan: <path or n/a>
  branch: <name or n/a>
  worktree: <path or n/a>>

TASK:
<the specific skill to run and what to produce>
[/ORCHESTRATOR-AUTONOMOUS-DISPATCH]
```

## How skills detect autonomous mode

Every modified Superpowers skill checks: **"Does my invocation prompt begin with `[ORCHESTRATOR-AUTONOMOUS-DISPATCH]`?"** If yes, follow the skill's Autonomous Mode section. If no (the orchestrator session or a normal human-in-the-loop session), follow the default flow.

This "begins with" check is robust because:
- A dispatched subagent's prompt is exactly the orchestrator's constructed prompt — it starts with the sentinel.
- The orchestrator's own prompt is the user's natural-language request — it does not start with the sentinel.
- Skill files may reference the sentinel string in documentation (inside backticks), but that reference appears mid-context after the user's prompt, not at the start.

## What the orchestrator does NOT include

- No `AskUserQuestion` instructions
- No "wait for the user to respond" prompts
- No visual companion offers (no live viewer)
- No 4-option menus for finishing
- No discard-confirmation flows
