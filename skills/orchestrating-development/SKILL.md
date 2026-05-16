---
name: orchestrating-development
description: Use for ANY development request (build/fix/implement/add/refactor). Runs the full brainstorm → plan → implement → review → PR loop autonomously in one session. Dispatches every subagent itself. Only escalates to the human when genuinely stuck.
---

# Orchestrating Development

You are the **orchestrator**. The full brainstorm → plan → implement → review → PR loop runs in this single session. You dispatch every subagent yourself via the `Agent` tool. You do not prompt the human between phases. You only call `AskUserQuestion` when the escalation criteria below are met.

**Announce at start:** "I'm using the orchestrating-development skill to run this autonomously."

<EXTREMELY-IMPORTANT>
Every other Superpowers skill in this repo has been modified to respect autonomous mode. When you dispatch a subagent, you MUST prepend the autonomous-mode preamble (`./autonomous-mode-preamble.md`) to its prompt. Subagents themselves will NOT call `AskUserQuestion` — they return `STATUS: QUESTION` instead. You decide. Only you can escalate to the real human.
</EXTREMELY-IMPORTANT>

## Step 0: Capture Original Intent

Before dispatching anything:

1. Read the user's request verbatim. Save it as `ORIGINAL_INTENT`.
2. If the request is materially ambiguous between two interpretations that would produce different artifacts (e.g., "fix the login bug" with two unrelated login bugs in the repo), this is the ONE moment you may call `AskUserQuestion` to disambiguate. Otherwise, proceed.
3. Initialize `ARTIFACT_MAP` (empty): will hold `spec_path`, `plan_path`, `branch_name`, `pr_url` as phases produce them.
4. Initialize `DECISIONS_LOG` (empty): one-line records of every autonomous decision you make at a former human-prompt gate. You will surface this in the final summary.

## Step 1: Phase Sequence

Run these phases in order. Each phase is **one or more** `Agent` tool dispatches with `subagent_type: general-purpose`. Subagents cannot spawn further subagents (Claude Code docs: "No nested teams"), so YOU dispatch every per-task subagent yourself in Step 3.

| # | Phase             | Skill the subagent runs                  | Output                          |
|---|-------------------|------------------------------------------|---------------------------------|
| 1 | Brainstorm        | `superpowers:brainstorming`              | Spec file path                  |
| 2 | Worktree setup    | `superpowers:using-git-worktrees`        | Worktree path, branch name      |
| 3 | Plan              | `superpowers:writing-plans`              | Plan file path                  |
| 4 | Implementation    | (no skill — you run the per-task loop)   | Code on branch                  |
| 5 | Review            | `superpowers:requesting-code-review`     | Review verdict                  |
| 6 | Finishing         | `superpowers:finishing-a-development-branch` | PR URL                       |

Step 4 is the per-task loop documented in `superpowers:subagent-driven-development`. You execute it directly from this session — see Step 4 below.

## Step 2: Dispatch Protocol

For each phase subagent, build the prompt as:

```
<contents of ./autonomous-mode-preamble.md, with placeholders filled>

TASK: Run the `<skill name>` skill. <Any phase-specific guidance.>
```

Fill the preamble placeholders before dispatch:
- `ORIGINAL_INTENT`: the verbatim user request + any clarifications you've captured
- `PRIOR_ARTIFACTS`: the current `ARTIFACT_MAP` (paths and the branch name)
- `TASK`: the specific skill to run and what to produce

Always use `Agent` tool with `subagent_type: general-purpose`. Do not enable experimental agent teams. Subagents are one-shot: prompt in, structured result out.

## Step 3: Reading Subagent Results

Every phase subagent returns a trailer of the form:

```
STATUS: DONE | QUESTION | BLOCKED
ARTIFACT: <path or URL or "n/a">
SUMMARY: <≤200 words>
QUESTION (only if STATUS=QUESTION): <text + 2–4 options>
```

Handle the status:

- **DONE** → record `ARTIFACT` in `ARTIFACT_MAP`. Proceed to next phase.
- **QUESTION** → consult `ORIGINAL_INTENT` + `ARTIFACT_MAP`. If you can answer reasonably:
  - Log the decision in `DECISIONS_LOG`.
  - Re-dispatch the same phase with your answer appended to the preamble's `ORIGINAL_INTENT` field (so it persists to later phases too).
  - If the SAME question comes back a SECOND time → escalate via `AskUserQuestion`.
- **BLOCKED** → try ONE recovery: re-dispatch with refined task framing or an adjacent skill. If still blocked → escalate via `AskUserQuestion` with the subagent's diagnostic SUMMARY.

## Step 4: Per-Task Implementation Loop

After the planner returns the plan path, you run the per-task loop yourself. This is the work `superpowers:subagent-driven-development` documents — but you execute it from this orchestrator session, because subagents cannot spawn further subagents.

```
1. Read the plan file. Extract all tasks with full text + scene-setting context.
2. Create a TodoWrite item per task.
3. For each task, in order:
   a. Dispatch an implementer subagent with preamble + the implementer-prompt.md template
      + full task text + context.
   b. Read its trailer.
      - STATUS: QUESTION → answer from ORIGINAL_INTENT/plan; re-dispatch. Escalate on repeat.
      - STATUS: BLOCKED → recover once; escalate if still blocked.
      - STATUS: DONE → continue.
   c. Dispatch a spec-compliance reviewer subagent (spec-reviewer-prompt.md).
      If it reports issues → dispatch implementer again to fix; re-review.
   d. Dispatch a code-quality reviewer subagent (code-quality-reviewer-prompt.md).
      If it reports issues → dispatch implementer again to fix; re-review.
   e. Mark the task complete in TodoWrite.
4. After all tasks are DONE, proceed to Step 5 (Review phase).
```

The implementer/spec-reviewer/code-quality-reviewer prompt templates already live in `skills/subagent-driven-development/` — use them verbatim and prepend the autonomous-mode preamble.

## Step 5: Review and Finishing

1. Dispatch a full code-review subagent (`superpowers:requesting-code-review` skill, with the `code-reviewer.md` template). It reviews the entire branch diff.
2. If the reviewer reports fixable issues → loop back into the per-task implementer pattern (Step 4) for the specific fixes. **Cap this loop at 3 cycles.** If still not converged after 3 cycles → escalate via `AskUserQuestion`.
3. When the review verdict is "approved", dispatch the Finishing phase subagent: run `superpowers:finishing-a-development-branch`. In autonomous mode it skips the 4-option menu and goes directly to push + PR.
4. Record the PR URL in `ARTIFACT_MAP`.

## Step 6: Stop Condition and Final Report

The orchestrator stops **only** when:
- `pr_url` is in `ARTIFACT_MAP`, OR
- A blocking escalation has been raised to the human and the human's answer redirects work.

There is no "should I continue?" check between phases. Do not pause for status updates. The human asked you to build this; you build it.

Final message to the human, when done:

```
Done. PR: <pr_url>

Artifacts:
- Spec: <spec_path>
- Plan: <plan_path>
- Branch: <branch_name>

Autonomous decisions made:
<DECISIONS_LOG, one bullet per decision>
```

## Escalation Criteria — when (and only when) to call AskUserQuestion

See `./escalation-criteria.md` for the canonical list. Summary:

1. Original intent is materially ambiguous and cannot be resolved from context (Step 0).
2. A subagent returned the same `STATUS: QUESTION` twice; your first answer didn't resolve it.
3. A subagent returned `STATUS: BLOCKED` and one recovery attempt also failed.
4. Implementation → review loop exceeded 3 cycles without convergence.

All other former human-prompt gates are decided autonomously and logged in `DECISIONS_LOG`.

## Forbidden in Autonomous Mode

- Discarding work (`git branch -D` of the feature branch, `git reset --hard`, etc.). The Finishing phase will not offer this option in autonomous mode.
- Force-pushing without explicit user request.
- Skipping the per-task review subagents (spec compliance + code quality).
- Skipping `superpowers:verification-before-completion` evidence before declaring DONE.
- Dispatching parallel implementer subagents (they would conflict on the worktree).

## Integration

- **`./autonomous-mode-preamble.md`** — the verbatim preamble you prepend to every subagent prompt
- **`./escalation-criteria.md`** — the four legitimate `AskUserQuestion` triggers
- **`superpowers:using-git-worktrees`** — Phase 2; in autonomous mode, always creates a worktree
- **`superpowers:brainstorming`** — Phase 1; in autonomous mode, returns the spec path without human approval gate
- **`superpowers:writing-plans`** — Phase 3; in autonomous mode, omits the execution-choice prompt (you always do subagent-driven)
- **`superpowers:subagent-driven-development`** — the per-task pattern you execute in Step 4
- **`superpowers:requesting-code-review`** — Phase 5 full-branch review
- **`superpowers:finishing-a-development-branch`** — Phase 6; in autonomous mode, push + PR, no menu
- **`superpowers:verification-before-completion`** — unchanged; enforce evidence before declaring DONE
