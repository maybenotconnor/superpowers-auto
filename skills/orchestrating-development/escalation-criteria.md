# Escalation Criteria

The orchestrator decides autonomously by default. `AskUserQuestion` is reserved for the cases below. **No other use is permitted.**

## The four legitimate triggers

1. **Genuine intent ambiguity at the start.**
   `ORIGINAL_INTENT` is materially ambiguous between two interpretations that would produce different artifacts, AND the conflict cannot be resolved by reading the repo (e.g., "fix the login bug" with two unrelated login bugs visible in `git log`). Resolve this once, at Step 0 of `orchestrating-development`. Do not re-ask later.

2. **Same question bounced twice.**
   A subagent returned `STATUS: QUESTION`. The orchestrator answered from `ORIGINAL_INTENT` + artifacts. The orchestrator re-dispatched. The subagent returned `STATUS: QUESTION` again on the same point. Now escalate — the orchestrator's first answer was demonstrably insufficient.

3. **Blocked with no recovery.**
   A subagent returned `STATUS: BLOCKED`. The orchestrator tried ONE recovery (refined task framing, more context, or an adjacent skill). Still blocked. Escalate with the subagent's full diagnostic SUMMARY.

4. **Review loop did not converge.**
   The implementation → review loop ran 3 full cycles without the reviewer reaching "approved". Escalate with a summary of the open review items and what the implementer fixed each cycle.

## Calling AskUserQuestion correctly

When you do escalate, use `AskUserQuestion` with 2–4 concrete, mutually exclusive options. Never an open-ended text question. Example:

> Question header: "Login bug to fix"
> Question: "The repo has two recent commits mentioning 'login bug'. Which one should I fix?"
> Options:
>   - "OAuth callback returning 500 (commit a1b2c3)"
>   - "Password reset email not sent (commit d4e5f6)"

After the human answers, append their decision to `ORIGINAL_INTENT` and resume. Do not call `AskUserQuestion` again unless a different escalation trigger fires.

## What is NOT a legitimate trigger

- "I want to confirm the design with the user before writing code." → No. The autonomous loop writes the design and proceeds. The PR is the review surface.
- "The user said 'fix some things' so let me ask what." → If too vague, push back BEFORE invoking `orchestrating-development` (per CLAUDE.md, step 3: "Verify this is a real problem"). Inside the loop, do not re-ask.
- "I'm about to do something destructive, let me confirm." → Destructive actions are forbidden in autonomous mode. Return `STATUS: BLOCKED` from the relevant subagent; the orchestrator escalates only if Trigger 3 applies.
- "Should I create a PR or just push?" → Autonomous loop always pushes + creates a PR. No choice menu.
- "The reviewer disagrees with the planner." → Resolve in-loop; that's what the per-task review cycle is for. Only escalate at Trigger 4 (3 cycles, no convergence).

## Logging non-escalations

Every autonomous decision at a former human-prompt gate goes into `DECISIONS_LOG` and is surfaced in the final report to the human. The human can audit the decisions after the fact (in the final message and in the PR diff itself). That audit trail is the autonomous loop's accountability mechanism, not mid-flight check-ins.
