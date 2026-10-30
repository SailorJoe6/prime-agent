---
id: TASK-2
title: Prevent orphaned RLM settlement after child finalization errors
status: In Design
assignee: []
created_date: '2026-10-30 21:41'
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
