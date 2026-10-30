---
id: TASK-2
title: Prevent orphaned RLM settlement after child finalization errors
status: In Planning
assignee: []
created_date: '2026-10-30 21:41'
updated_date: '2026-10-30 21:54'
labels: []
dependencies: []
references:
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/src/core/agent-session.ts#L13075-L13116
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/src/core/agent-session.ts#L12365-L12373
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/src/core/agent-session.ts#L12669-L12718
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/src/core/agent-session.ts#L4522-L4526
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/src/core/session-manager.ts#L2152-L2175
  - >-
    https://github.com/SailorJoe6/prime-agent/blob/a1faacd53ac4473a75de1d434afaf50945c2f647/packages/coding-agent/test/agent-session-recursion.test.ts
  - 'https://github.com/PrimeIntellect-ai/prime-agent/pull/3195'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/pull/3209'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/pull/3244'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/pull/3386'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/pull/3754'
documentation:
  - 'https://github.com/PrimeIntellect-ai/prime-agent/releases/tag/v0.10.0'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/releases/tag/v0.10.1'
  - 'https://github.com/PrimeIntellect-ai/prime-agent/releases/tag/v0.10.2'
type: bug
ordinal: 2000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
## Outcome and scope
Prevent RLM quiescence from hanging indefinitely when unexpected non-deletion child-run finalization errors leave settlement unresolved. Preserve the October 27–28, 2026 daily-sweep wedge investigation as evidence, without claiming its historical cause is proven. This is one coding-agent bug specification, not authorization to implement or plan.

## Evidence and uncertainty
The recovered read-only investigation examined TypeScript baseline a1faacd53ac4473a75de1d434afaf50945c2f647. Its relevant source seams were checked against that unchanged baseline during specification recovery; no reproduction or tests were run during recovery.

Source-identified latent defect: in packages/coding-agent/src/core/agent-session.ts:13075–13116, finalization calls flushPendingChildUsageAttribution() and performs tracking cleanup before run.settled = true and run.settlement.resolve() at 13109–13111. An unexpected exception before that settlement skips it, and the detached IIFE catch at 13116 silently discards the error. A terminal run can therefore remain in _unsettledRlmChildRuns with unresolved settlement. waitForRlmQuiescence() waits on those settlement promises at 12365–12373 and can remain pending indefinitely unless cancelled. This is a source-identified failure path, not a fault-injection result from this recovery.

Historical observations: daily-sweep sessions stalled on October 27–28. Their saved evidence was consistent with this latent failure path, including a final durable child_usage_attributed row, but contained no exception or stack establishing that this path actually occurred. Normal attribution persistence errors are already caught at 12697–12702. A trailing attribution row also occurs in healthy sessions and is not a causal signature. Fixing this latent defect, or adding a finally block alone, must not be represented as proof that the historical wedges are resolved.

Rejected hypothesis: child_usage_attributed and an inbound steered message do not by themselves form a deadlock at this baseline. The attribution flush at 12669–12718 is synchronous, with no await. If the parent assistant is not yet indexed, it queues a retry on the parent event drain and returns at 12675–12688; child settlement does not wait for that retry. The parent event queue is promise-serialized at 4522–4526, and JavaScript cannot interleave steer delivery inside the synchronous attribution append. SessionManager.appendChildUsageAttribution() in packages/coding-agent/src/core/session-manager.ts:2152–2175 is also synchronous. No promise dependency cycle between attribution, _agentEventQueue, and steer delivery was identified. Do not introduce waits or locking around that queue to address the rejected hypothesis.

## Recovered engineering findings, not an approved implementation plan
The worker proposed focused deterministic coverage in the existing packages/coding-agent/test/agent-session-recursion.test.ts: create a child with pending usage; hold the parent event drain so attribution remains pending; inject a one-shot finalization fault after the attribution boundary (for example an appropriate throwing usage lookup or unsubscribe); then let the child reach terminal status. The reported old-path expectations were terminal run.status, run.settled still false, retention in _unsettledRlmChildRuns, and a quiescence wait that never resolves. The desired result is no orphaned settlement and removal from unsettled work, observable through quiescence completion or explicit failure reporting. Internal state observations are explanatory evidence, not sufficient acceptance by themselves.

The worker also proposed a separate gated fake-provider case that injects a steer exactly at child agent_end. It should complete on the baseline, showing that the suspected attribution/steer interleaving alone is not the deadlock. Preserve this negative control alongside the failure-path regression. Use explicit events/deferred promises or gates, not polling sleeps, paid providers, or larger timeouts as readiness signals; a timer may only bound failure. The failure-path regression must expose the old behavior when hardening is removed, rather than merely asserting a mock's return value.

The suggested minimal TypeScript hardening was an inner finally covering attribution flush and non-deletion tracking cleanup, with non-deletion settlement guaranteed even if those operations throw: mark settled, resolve settlement, remove from _unsettledRlmChildRuns, and retain the normal goal/autonomous continuation notifications. This is a candidate direction for later planning, not a prescribed patch or historical causal proof. The report also recommended replacing the silent terminal catch with observable diagnostics. A cleanup fault may leave stale tracking; hardening must not conceal that failure or falsely report cleanup success. Do not expand this task into wholesale cleanup redesign.

Detached deletion is deliberately excluded from unconditional non-deletion settlement: its stronger cleanup/settlement guarantees must remain intact. Preserve terminal result delivery, own/child usage accounting, ordinary steer and attribution behavior, goal/autonomous continuation, Orca integration, and extension behavior. No daemon wire/capability change or protocol-version bump was identified as necessary by the worker; this is not authorization for a wire change.

## Upstream context: adjacent Rust evidence, not a TypeScript patch
The original upstream researcher reported that PRs #3195, #3209, and #3244 entered Rust v0.10.0 and were inherited by v0.10.1 and v0.10.2. #3195 covered idle-passivation/stop races: adjacent race coverage, not attribution settlement. #3209 concerned whole-worker idle-passivation eligibility and was not relevant. #3244 woke passivated children for agent messages and explicitly did not solve parent usage accounting.

Later reported Rust behavior: v0.10.1 #3386 persisted child usage attribution rows; v0.10.2 #3754 corrected failed-child settlement; v0.10.2 also fixed late grandchild attribution and idle-parent spawn timing. These are preserved findings from the original upstream research, not independently revalidated release claims during this recovery. None is a directly applicable TypeScript patch or proof of the rejected attribution/steer deadlock. Do not require wholesale adoption of these Rust changes.

## Provenance
Original owner Conversation: 01a0e47b-77dd-755e-9f2b-883d4053c4d3 (Prime Agent Expert). Former read-only worker: wedge-root-cause; RLM child sub-27a10a1a; saved session 01a1842f-85d5-77ce-9f39-baed217e07b0. The worker completed at 2026-10-28 20:33:58 UTC and was deleted after findings retrieval at 20:34:25 UTC. Its complete final report was recovered from the original owner's retrieval record and supplied for this task. Foreign agent transcripts were not inspected during this recovery.

Original worker transcript: /Users/jlanders/.prime/agent/session-artifacts/01a0e47b-77dd-755e-9f2b-883d4053c4d3/sub-27a10a1a/01a1842f-85d5-77ce-9f39-baed217e07b0.jsonl. Nested upstream researcher transcript (reported relative artifact location): sub-7c80d5b2/01a1842f-eec2-75b1-be1b-e429fe210407.jsonl. These are provenance references only, not required runtime inputs or permission to inspect unrelated agents. Essential findings are preserved above so this task does not depend on transcript or chat survival.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [ ] #1 An unexpected non-deletion child finalization fault after terminal status cannot leave RLM quiescence waiting indefinitely on an orphaned settlement. The wait completes or reports an explicit failure, and later quiescence is not blocked by that orphaned run.
- [ ] #2 Unexpected finalization failures are observable through the supported error/diagnostic surface rather than silently discarded; reporting does not mask the original failure, falsely claim cleanup success, or introduce a new quiescence hang.
- [ ] #3 Deterministic failure-path coverage in the existing agent-session recursion suite injects a one-shot finalization fault with child usage pending and a controlled parent event drain. It exercises observable quiescence behavior, fails against the unhardened path, and does not rely only on private-state or mock-return assertions.
- [ ] #4 A gated fake-provider control delivers a steer at child agent_end and verifies normal completion and ordinary steer/usage attribution behavior. It distinguishes the rejected interleaving hypothesis from the injected finalization failure.
- [ ] #5 Detached-deletion settlement still requires its stronger cleanup guarantees, including failure handling; non-deletion hardening must not settle deletion early or report incomplete deletion as successful.
- [ ] #6 Terminal result delivery, child/parent usage accounting, goal/autonomous continuation, and Orca/extension behavior remain intact on ordinary and relevant failure paths; no wait or lock is added around the parent event queue to address the rejected hypothesis.
- [ ] #7 Coverage uses explicit completion events or deferred gates and self-contained fake providers, with no polling sleeps, paid provider calls, or readiness-by-timeout. Results retain the distinction between the source-identified latent defect and the unproven historical October 27–28 wedge cause; a finally change alone is not evidence of historical resolution.
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
1. Harden only child-run finalization in `packages/coding-agent/src/core/agent-session.ts` (`_startRlmChildRun`, baseline lines 13075–13116).
   - Keep the existing terminal result/error/cancellation delivery and retention decisions. Treat settlement as completion of the non-deletion run barrier, not proof that every bookkeeping operation succeeded; do not rewrite an already delivered terminal result as a cleanup success.
   - Give attribution flush and non-deletion tracking cleanup separate guarded attempts. A flush exception must not skip the cleanup attempt. Put the non-deletion settlement triplet (`run.settled = true`, resolve `run.settlement`, remove the run from `_unsettledRlmChildRuns`) in a guaranteed inner `finally` that covers both stages. Preserve same-run identity checks, retained-session subscription transfer, error-run retention, and usage memo invalidation. Do not erase retained tracking merely to make quiescence pass when cleanup failed.
   - After settlement, attempt `_maybeResumeGoalContinuationAfterRlmWork()` and `_maybeResumeAutonomousContinuationAfterRlmWork()` independently, once each. A synchronous exception in either notification must not skip the other, undo settlement, or replace an earlier failure. Leave existing pause/admission rollback and asynchronous autonomous-resume behavior in their owners; do not await continuation turns, replay notifications, or add retries in finalization.
   - Keep detached deletion outside this unconditional settlement guarantee. Preserve its shared cleanup promise/observer, `deletionRunFinished`, cleanup-failure reservation and explicit retry, and `_finishRlmRunDeletion()` ordering (notice and tracking cleanup before settlement). Attribution failure must not bypass that deletion cleanup path, and no generic catch/finally may settle deletion early. Do not redesign deletion recovery or relax its failure semantics.
   - Leave `waitForRlmQuiescence()` and the synchronous attribution retry on `_agentEventQueue` structurally unchanged. Add no waits, locks, or new queue dependencies for the rejected attribution/steer hypothesis.

2. Report unexpected finalization failures through the existing structured diagnostic surface.
   - Add a standard top-level `getLogger` import from `@earendil-works/pi-ai` and the scoped logger `getLogger("coding-agent.rlm")`. The verified API is `Logger.error(message: string, fields?: Record<string, unknown>): void` (`packages/ai/src/log.ts` and installed `dist/log.d.ts`). Production already routes it through `src/core/logging.ts` to `logs/agent.jsonl`; without a sink, or when a sink throws, the logger falls back to structured `console.error` and does not throw.
   - Replace the detached IIFE's silent terminal catch with guarded reporting. Retain the original error message/stack plus parent session ID, child ID/name, terminal status, detached-deletion flag, and failing stage (attribution, tracking cleanup, goal/autonomous notification, or escaped detached task). Report secondary errors separately without masking the first. Do not log prompts, credentials, or full transcripts, invent a session event, or emit an extra model-facing terminal notice.
   - Capture finalization errors without allowing diagnostic formatting or delivery to control settlement. Complete the non-deletion barrier and attempt both notifications even if cleanup or diagnostics fail. Keep reporting synchronous, bounded, and no-throw, including unknown thrown values; rely on the existing logger's sink-failure fallback rather than adding a new logging system. Diagnostics must state failed/incomplete cleanup truthfully, not certify successful cleanup. Do not change ordinary recoverable `appendChildUsageAttribution()` persistence handling into a new task failure.

3. Add compact, deterministic TASK-2 coverage in the existing `packages/coding-agent/test/agent-session-recursion.test.ts`.
   - Reuse its `deferred`, typed inspection seams, `createHarness`/faux-provider terminal-notice setup, and existing usage/deletion assertions. Use `FauxResponseFactory` async gates and real extension `message_end` handlers: hold the parent assistant before persistence, signal that the parent drain is held, then allow the child's nonzero-usage response. This leaves attribution pending without making child settlement await the parent queue. Register child lifecycle listeners before releasing the provider gate.
   - Exercise one-shot faults at actual finalization boundaries, not normal append errors already swallowed by design. Use the existing `SubagentRuntimeHost.completeRlmSubagentRuntime` retention callback to arm a single throwing `_findAssistantEntryForMessage` lookup and return false; the subsequent finalization flush hits it with usage pending. A companion table row wraps the real child subscription disposer and throws once when non-deletion tracking cleanup invokes it. The wrapper must still release the real listener. Keep these fault controls test-local; add no production fault-injection API.
   - Signal fault entry explicitly, release the held parent drain, and await public `waitForRlmQuiescence()` completion and a second quiescence call. Verify the parent's consumed terminal notice is not lost/duplicated and pending usage is attributed after the drain, including durable attribution rows and parent-own versus child totals after reload. Private `settled`/map observations may explain the path but are not the acceptance result. Include attribution-fault and tracking-fault diagnostics with original failure identity; do not assert only that a spy was called.
   - Add a separate no-fault negative control using the same gated provider/drain fixture. From the child's first `agent_end` callback, enqueue exactly one parent steer with the existing `queueAgentMessagePrompt(..., "steer", customMessage)` surface. Signal callback entry and enqueue start, release the parent gate, then await enqueue and consumption; verify the steer is consumed, the ordinary terminal notice and usage attribution survive, and public quiescence completes. This control must pass on the unchanged baseline; it is not a reproduction of the historical wedge.
   - Cover diagnostic-sink failure with real `setLogSink` and its structured console fallback, preserving the original finalization error and completed barrier. Cover notification isolation with one-shot throws at the goal/autonomous notification seams and observable continuation delivery by the unaffected real path, not only invocation counts. Retain existing goal/autonomous deferral, accounting, and terminal-notice ordering checks; no new continuation policy.
   - Preserve the existing active-tool deletion, deleted-live-run, and settled-error cleanup-retry cases. For any touched deletion case, replace sleep/poll readiness with deferred cleanup-start, unwind, failure-notice, and retry gates. Assert no cancelled-completion notice or quiescence completion before both unwind and successful cleanup, failure remains retryable, and explicit successful retry releases the barrier. Do not apply non-deletion fault expectations to deletion.
   - Use unique temporary paths, fake providers only, explicit gate/event ordering, and `finally` to open gates, cancel outstanding waits, unsubscribe, restore spies/log sinks/fake timers/environment, and dispose every parent/child before removing fixtures. Do not copy this suite's legacy `waitFor`, `vi.waitFor`, sleeps, or timing races into new/touched cases. A timer or runner deadline may only fail a stuck test, never establish readiness or make it pass.

4. Validate during implementation, not during this planning pass.
   - From `packages/coding-agent`, run each modified file directly with `npx tsx ../../node_modules/vitest/dist/cli.js --run test/agent-session-recursion.test.ts` (substitute the exact modified file for any additional file). Also run the existing focused continuation suites with the same package-root command: `test/goal-continuation-quiescence.test.ts` and `test/autonomous-continuation-subagent-gate.test.ts`.
   - Temporarily remove only the production hardening and run the targeted TASK-2 finalization regression: it must fail because observable quiescence/diagnostics are broken, while the no-fault steer control still passes. Restore the hardening and run green. Record the intentional failing run; do not retry it to green or claim the fault caused the October 27–28 incidents.
   - Run the focused recursion/continuation files with deterministic ordering and multiple explicit shuffle seeds, e.g. append `--sequence.shuffle --sequence.seed=17` and then `--sequence.shuffle --sequence.seed=73` to separate focused invocations. Stop at the first unexpected failure and fix it; repeated runs are evidence, not retries. No real providers, paid tokens, daemon/client launches, readiness sleeps, or larger timeouts.
   - From the repository root run `npm run check` with full output and `npm run check:test-policy`; fix errors, warnings, and infos. Never use `npm test`, `npm run dev`, or `npm run build`. Measure test/source growth before finalizing: use compact tables and reuse/consolidate existing setup while preserving every distinct valid assertion; keep added test lines within the actual source-change allowance and report added/net counts. Do not pad production, weaken test policy, delete valid coverage, or create another test file to evade the line budget. If a compliant compact change cannot fit, report that concrete constraint rather than bypassing it.

5. Finish the same bounded task.
   - Add one user-visible fragment, `packages/coding-agent/.changes/task-2-rlm-finalization-settlement.md`, describing the non-deletion quiescence fix and failure diagnostics. No new documentation tree, daemon wire/capability/schema change, runtime installation, Orca API change, extension API change, or Rust adoption is needed. Keep existing Orca/event contracts untouched; the gated extension case checks that extension/persistence ordering remains usable.
   - Record actual changed files, direct-test results, mutation-test failure/restoration, shuffle results, checks, and remaining limits in TASK-2's native implementation notes/acceptance state during the later Episode. Preserve the specification/provenance and the distinction between a source-identified latent defect and an unproven historical cause. Mark criteria only against actual evidence, not a finally block alone.

This plan introduces no external dependency and authorizes no implementation. Keep TASK-2 In Planning for operator review. A separate explicit `/implement-task TASK-2` is required before implementation resources or code changes.
<!-- SECTION:PLAN:END -->
