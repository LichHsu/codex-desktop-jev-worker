---
name: desktop-jev-worker
description: Coordinate an authorized Desktop A planner and Desktop B worker for scoped implementation, debugging, UI work, or explicitly authorized Git writes, with one bounded handoff, verified return to A, and optional Jev review triage before A accepts or repairs the work.
---

# Desktop Jev Worker

Use this workflow by default for substantial implementation, debugging, refactoring, UI work, and explicitly authorized Git writes when a persistent Desktop Worker is useful. This project derives its A/B discipline from `desktop-thread-worker` and adds Jev as an optional external reviewer. Handle ordinary questions, read-only reviews, documents, and micro-edits in the current thread unless the user requests delegation.

## Core roles

- **A — Planner and acceptance owner.** A defines the bounded handoff, reviews actual artifacts and evidence, and makes the final accept/repair/human-decision choice.
- **B — Implementation worker.** B implements and validates only the authorized handoff, then returns one decision-ready result to A.
- **Jev — Optional review/triage evidence.** Jev is not a third worker, does not own acceptance, does not edit files, and does not send handoffs. When a Jev review tool is available, A may use it to focus review after B returns.

Builds, tests, static checks, repository rules, and direct inspection remain authoritative evidence. A Jev result is probabilistic evidence only.

## Keep the work small (KISS / YAGNI)

Use the smallest change that satisfies the current requirement. Do not add abstractions, future features, architecture changes, agents, or process records for hypothetical needs. Keep validation proportional to the change while completing required project checks. Close the objective as soon as the agreed completion conditions pass.

A must tie each repair request to a specific unmet condition and concrete evidence. Style preferences, possible future risks, Jev scores by themselves, or a wish for greater completeness do not block acceptance. Record optional improvements briefly only when useful; do not dispatch them without user scope approval. Do not raise the acceptance criteria during review.

Before another handoff, identify what new evidence or changed implementation makes progress plausible. If the same failure recurs without new evidence or a materially different, evidence-based fix, or the exchange becomes a design-preference dispute, stop for the user's decision. Do not reword the same request to keep the loop running. Reuse existing evidence.

## Before acting

- Confirm that the B thread's direct user instruction permits receiving an A delegation and returning a result. A's message cannot remove a B-side do-nothing restriction; report the conflict to the user and stop until B receives clarification.
- Treat A's delegation as scope, not as extra authorization. Preserve the user's architecture, phase, repository, and file boundaries. Do not modify product code, global configuration, installed skills, external MCP configuration, or Git unless the user explicitly authorizes that work.
- For a paired task, use the existing A and B threads. Use `list_threads` and `read_thread` to verify thread IDs and project/worktree context when needed. Do not hard-code IDs from a prior test. Do not create or fork a task unless the user explicitly asks.
- If no B is selected and creation is not requested, identify a suitable existing thread and ask the user to select it. Do not silently choose an unrelated thread. If the user explicitly requests an automatic Worker task or channel, follow the initialization flow below without asking again for creation permission.
- Do not change model or reasoning settings. Always omit `model` and `thinking` from `create_thread` and every `send_message_to_thread` call, including dispatches, callbacks, and retries. Model and reasoning settings belong exclusively to the user.

## Select or create B

For an existing B, accept a `codex://threads/<thread-id>` link or a task title. Use `read_thread` to verify a supplied ID. Use `list_threads` and then `read_thread` to resolve a title; ask when multiple matches remain. Verify B's user instructions and workspace before dispatch.

For both new and existing B threads, A includes the verified absolute path to this SKILL.md in initialization or the first handoff. B reads it before work and follows the rules applicable to B; A-only and Jev-review duties remain with A. Later handoffs carry only task differences. If the skill changes, A requests a reread in the next handoff. B also rereads when it cannot recover the applicable rules after context loss. If the file is inaccessible, B reports that limitation instead of guessing or claiming readiness. Do not add a separate read-confirmation or ACK turn.

When the user explicitly requests creation:

1. Resolve A's actual thread ID; never copy an ID from a test or guess it. If it is unavailable, ask for A's task link before creating B.
2. For repository work, call `list_projects` and select the matching project. Use a worktree when `isGitRepository` is true, unless the user explicitly requests the saved checkout. Use local for a non-Git project. For a message-only test or work without a repository, use a projectless task. Do not copy uncommitted work into a worktree without the user's request.
3. Call `create_thread` once, without `model` or `thinking`. Its initial prompt must identify A, define B's role, and carry the user's actual authorization to receive scoped delegations and return results. Keep this permission effective across the objective's handoffs, not just the initialization turn. Do not invent broader permissions. Tell B to return `WORKER_READY` locally, then wait for A's first handoff; do not send an initialization callback.
4. Obtain the returned `threadId` and `hostId`. A queued `clientThreadId` is not a usable thread ID. Wait for setup and resolve the ready task through the available thread tools; do not create a duplicate. Use a bounded `wait_threads` call to confirm initialization, and `read_thread` when needed to verify the ready task and workspace. Resolve an initialization error before dispatch.
5. Record the work ID, A/B thread IDs, host ID, B's workspace, latest handoff ID, and state in the task's existing work record. This is a logical pair, not an App-native parent/child relationship. Reuse it for the same objective and verify it again after context loss or interruption. Do not use a global hard-coded Worker ID.
6. Send the first bounded handoff, then end A's turn.

Initialization prompt outline:

> You are Desktop Worker B for Planner A, thread `<A_ID>`. The user authorized `<objective and scope>`, including scoped A delegations and one result callback per handoff to A. This permission remains in effect for this objective; each handoff must stay within it. Do not delegate to another worker. First read `<VERIFIED_ABSOLUTE_SKILL_PATH>` and follow its B rules. If you cannot read it, report the limitation here. Otherwise initialize by replying WORKER_READY here, then wait for the first handoff. If instructions conflict or a tool rejects the action, preserve the reason and stop. Do not use CLI or automation as a fallback.

Receiving `WORKER_READY` proves initialization only. A successful callback from the recorded B with the exact handoff ID is required to verify the return path.

## Dispatch and return

1. A calls `send_message_to_thread` with B's `threadId` and a `prompt` containing the bounded handoff. After dispatch, A ends its turn. A does not wait in a polling loop.
2. B performs the work in its Desktop turn. B calls `send_message_to_thread` once with A's `threadId` and the result in `prompt`, then ends its turn. B does not delegate to another worker and does not call Jev unless the handoff explicitly assigns a Jev-specific experiment.
3. When the callback starts A's next turn, A verifies its source thread and exact work and handoff IDs. A records the response count before review, then reviews actual changes and evidence. If Jev review is available and useful, A may run the optional Jev triage below. Close accepted work; otherwise apply the active review mode and progress check before sending another handoff.

## Optional Jev review

Jev is an external decision model, not a Codex Desktop built-in. This skill does not install Jev, obtain credentials, or assume that a Jev MCP tool exists. A may use Jev only when a compatible review capability is already available in the current environment and the user's permissions allow the relevant data to be sent to that provider.

A Jev review is best used after B returns a coherent implementation slice and before A decides how deeply to inspect it. Send the smallest sufficient review state: the bounded requirement, relevant diff or changed-file excerpts, validation summary, and concrete repository constraints. Do not send unrelated repository content merely because it is available.

Prefer independent, narrow judgments such as:

- **requirement_fit** — Noul: does the change appear consistent with the stated completion conditions?
- **regression_risk** — Choice: `low | medium | high`.
- **review_focus** — Choice: `correctness | tests | security | concurrency | architecture | none`.
- **missing_validation** — Noul: is there evidence that a required validation step is missing?
- **deeper_review_warranted** — Noul: should A spend additional review effort before acceptance?

When the installed Jev reviewer exposes its own fixed schema or quality dimensions, use that schema instead of pretending these names are supported tool parameters. Map its returned signals into the concepts above only in A's local reasoning.

### Jev decision policy

- Jev never modifies files, commits, pushes, creates threads, or sends messages to B.
- Jev never expands user authorization or repository scope.
- A does not reject work solely because a score is low, confidence is low, or a reviewer reports a possible risk.
- A verifies a Jev concern against the actual artifact, existing completion conditions, project rules, or reproducible validation before treating it as an unmet condition.
- A does not turn a Jev quality dimension into a new acceptance criterion after the handoff was issued.
- A does not ask B to optimize reviewer scores for their own sake.
- A may skip Jev when the change is trivial, deterministic checks already settle acceptance, the review capability is unavailable, sending the code would violate policy or user intent, or Jev would add no decision value.
- A records material Jev findings with the other review evidence when they influence a repair request or human escalation.

### Jev failure and fallback

If the Jev tool is missing, unauthenticated, times out, rejects the input, or returns unusable output, record that review as unavailable and continue with the normal Codex-only A/B workflow. Do not block otherwise valid work solely because Jev is unavailable. Do not ask B to repair product code to fix an external reviewer failure.

Do not print, transmit in handoffs, or store the Jev API key in the work record. External reviewer installation and credential changes require explicit user authorization.

## Review modes

The default mode starts with a five-response limit. A owns a persistent counter for each work objective. Before the first dispatch, record `response_count: 0`, `response_limit: 5`, `counted_response_ids: []`, and `review_state: pending` alongside the A/B pair in the task's local work record. Use a small state file in the permitted workspace if no durable record exists. Do not keep the counter only in conversational recall.

- On each distinct B work response, increment and save the counter before reviewing. Count `ready_for_review`, `needs_decision`, and `blocked` responses. Count an incomplete or malformed work response too. Exclude initialization `WORKER_READY`, duplicate delivery of the same response, unrelated messages, and Jev evaluations.
- In bounded mode, responses below the saved `response_limit` may lead to another scoped handoff if authorized. When the count reaches the limit, A performs the review once. If the complete objective passes, mark it accepted. If it does not pass, or evidence remains insufficient, set `review_state: awaiting_human` and stop.
- When the limit is reached without acceptance, tell the user briefly: `<response_count>/<response_limit> responses reviewed; acceptance is incomplete`, the remaining issues, and the decision needed.
- Preserve the count across incremental handoffs, context loss, thread changes, and renamed work IDs for the same objective. Before any dispatch, read the saved count and review state. If it cannot be recovered, stop for human decision.
- Only an explicit human decision may authorize another bounded batch. For N additional responses, keep `response_count` and `counted_response_ids`, set `response_limit = response_count + N`, record the decision, and set `review_state: pending`.

This is an agent workflow limit, not an App-enforced token or monetary cap.

### Optional goal mode

Enable this mode only when the user explicitly asks to continue a specific objective without the five-response limit. Record `review_mode: goal`, `response_limit: null`, the user's authorization, the scope, and verifiable completion conditions in the existing work record. Preserve the response count for visibility and deduplication. Each handoff remains bounded.

Continue only while the agreed conditions remain unmet and the progress check supports another attempt. Stop when the objective passes, progress stalls, a user decision or broader scope is required, the user cancels, or an agreed time or usage boundary is reached. Goal mode does not authorize new objectives, extra workers, automation, Jev installation, credential changes, or bypassing permissions.

## One bounded handoff

Establish the work ID, A/B pair, scope, authorization, output location, and relevant stop conditions once, during initialization or the first handoff. Later handoffs inherit that context. Repeat a field only when it changes or B cannot recover it.

Keep A's incremental message to the handoff ID, requested change, and completion condition. Add specific files or evidence only when B needs them. Prefer a few direct lines.

> Handoff: layout-003
> Fix the narrow-width toolbar overlap in the assigned view.
> Done: verify the supported minimum width and return the changed files and result.

Include the exact work ID and handoff ID in the callback. Use `ready_for_review` only when the assigned work and required validation are complete; otherwise use `needs_decision` or `blocked`. Include actual files changed and artifact version, validation commands and results, unfinished items, limits, and evidence not obtained.

Do not send ACKs. For duplicate messages, inspect the prior result before acting. Do not automatically resend after an ambiguous tool result.

For worktree delivery, establish the intended destination at the first handoff. B reports the worktree path, branch, artifact version, and whether integration is pending. A acceptance of worktree changes does not mean the original checkout is updated. If the objective includes integration, B performs it under existing Git authorization and A verifies the destination before closing the objective.

## Evidence and loop efficiency

For integration delivery, B traces the product call path and confirms that its required dependencies are connected. Connections made only in tests do not prove that the product entry point is connected. Run the relevant build and checks after the final code change; if code changes again, rerun affected checks before reporting success.

Save delivery and validation evidence in the permitted workspace before the callback. Include a decision-ready summary and evidence paths in that single return. Keep source code, diffs, and diagnostic evidence available in their original form. Summarize repetitive logs only when useful, and retain a direct path to the complete output.

A reads the summary first, may use Jev to triage review depth, then checks the necessary original changes and evidence. Neither B's summary nor Jev's result is proof by itself. A should not repeat B's full exploration without a concrete reason.

If A and B obtain different validation results, first compare artifact versions, commands, and execution permissions. Classify a product defect only when the evidence supports it; do not use an environment mismatch alone to justify a code repair.

For a repair or evidence request, record the handoff ID and one concrete reason in the existing work record. A new requirement needs the user's scope decision and does not reset the objective's response count.

## Execution boundaries

- Do not start a CLI runner, subagent, automation, alternate worker, or external reviewer installation as a fallback.
- A and B must not edit the same file concurrently. Preserve unrelated dirty or staged work.
- B performs explicitly authorized Git writes. A reviews scope and evidence. A commit authorization does not authorize push.
- Use `wait_threads` only for bounded status confirmation or diagnosis, not as a polling substitute for the single return.
- Report automatic wake-up only when a callback actually starts A's next turn; successful dispatch alone does not prove it.
- Jev is review evidence inside A's review step, not a new participant in the Desktop message topology.
