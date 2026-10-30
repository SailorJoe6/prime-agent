---
id: TASK-1
title: Establish v0.9.x bug-fix intake and audit upstream releases
status: In Design
assignee: []
created_date: '2026-10-28 20:14'
updated_date: '2026-10-30 21:52'
labels: []
dependencies: []
references:
  - 'https://github.com/PrimeIntellect-ai/prime-agent/releases'
  - >-
    https://github.com/SailorJoe6/prime-agent/commit/a1faacd53ac4473a75de1d434afaf50945c2f647
ordinal: 1000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
## Context

Prime Agent upstream moved from the TypeScript/pi-based implementation after v0.9.8 to a Rust rewrite. The Rust line cannot be adopted because it breaks required Orca integration. Joe will maintain the pinned TypeScript downstream line at `cwd-fix-v0.9.8-r1` / `a1faacd53ac4473a75de1d434afaf50945c2f647` as a long-lived stability fork that remains in the `0.9.x` version series.

Rust fixes cannot be mechanically cherry-picked into this codebase. They can reveal language-independent defects or invariants that may also apply to the TypeScript implementation. Locally discovered defects must enter the same evidence-backed maintenance flow.

## Outcome

Establish a durable, repeatable bug-fix intake process for the TypeScript `0.9.x` fork and use it to audit upstream stable releases `v0.10.0`, `v0.10.1`, and `v0.10.2`. Produce a complete applicability ledger that identifies which upstream fixes are irrelevant, already safe, potentially applicable, or proven to require a TypeScript fix. Define how confirmed upstream-applicable and locally discovered bugs enter the next point-release queue.

This task establishes the intake process and performs the initial audit. It does not port a candidate fix or publish a `0.9.x` release; confirmed fixes become separately reviewable bug tasks.

## Required behavior

- Verify and record upstream stable releases `v0.10.0`, `v0.10.1`, and `v0.10.2`, including their peeled commits, publication dates, and audit cutoff.
- Inspect release notes, commits, linked issues or pull requests, and regression tests so the inventory is not limited to changelog wording.
- Trigger recurring intake only from stable releases. Upstream `main`, nightlies, and release candidates may be inspected as supporting provenance for a stable fix but do not independently enter the ledger or release queue.
- Record every bug fix from each release in a canonical version-controlled Markdown ledger under `docs/maintenance/`, with its immutable upstream provenance, observable symptom, intended invariant, affected Rust subsystem, analogous TypeScript/pi surface, applicability classification, supporting evidence, and next action.
- Use these classifications: Rust-only or architecturally irrelevant; already safe in TypeScript; potentially applicable and needing reproduction; reproduced and requiring a port; ported and verified. The initial audit may not claim the final classification unless evidence supports it.
- For every potentially applicable fix, identify the exact TypeScript paths and a concrete reproduction or regression-test strategy. For every reproduced defect, create a separate provider-native bug task before implementation.
- Document the same regression-first flow for defects found locally: capture reproduction and root cause, prove a focused regression test fails without the behavior, implement the smallest stability fix, run required validation, and queue it for the next `0.9.x` point release.
- Keep the authoritative maintenance line on `maint/0.9.x`, created from exact commit `a1faacd53ac4473a75de1d434afaf50945c2f647`, separate from the immutable v0.9.8 release provenance and from the upstream-mirroring Rust `main`. Changing the repository default branch is outside this task.
- Preserve lockstep package versioning, changelog-fragment rules, immutable release tags, rollback capability, and Orca compatibility gates for future point releases.
- Make the ledger and process easy to update when another stable upstream release appears, without automatically changing code, publishing releases, or deciding applicability.

## Constraints

- Stability bug fixes only. Do not add features or adopt Rust architecture.
- Port behavior and invariants, never Rust code mechanically.
- Do not treat an upstream fix as applicable solely because it sounds similar.
- Preserve daemon protocol compatibility. Any wire change must follow the repository capability/version rules.
- A future code fix must follow the repository test policy, run each modified focused test directly, prove the regression test can fail, and pass `npm run check`.
- Orca integration is a hard release requirement, not optional follow-up validation.
- Do not modify, disable, or repurpose the existing Rust maintenance automation in this task.
- Do not install, stage, activate, or restart into a Rust Prime Agent build.

## Non-goals

- Implementing any candidate bug fix.
- Cutting or publishing `0.9.9`.
- Repointing the fork default branch or rewriting existing release provenance.
- Broad dependency upgrades, refactors, feature backports, or speculative compatibility work.
- Treating upstream `main`, nightlies, or release candidates as independent intake releases.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [ ] #1 The audit records stable releases `v0.10.0`, `v0.10.1`, and `v0.10.2`, their peeled commits and publication dates, and a clear cutoff, with immutable references.
- [ ] #2 Every bug fix in those releases appears exactly once in a canonical version-controlled Markdown ledger under `docs/maintenance/` and is traceable to release notes plus its strongest available commit, issue, pull-request, or regression-test evidence.
- [ ] #3 Each ledger item states the language-independent symptom and invariant, identifies the analogous TypeScript/pi code surface, and carries one defined applicability classification with evidence.
- [ ] #4 Rust-only and already-safe conclusions cite concrete architectural or TypeScript evidence; potentially applicable items include a reproducible investigation or regression-test strategy.
- [ ] #5 Any defect proven to affect the TypeScript line has its own provider-native bug task and is not implemented as part of this audit task.
- [ ] #6 The documented local-defect workflow requires reproduction, root-cause evidence, a regression test proven red without the fix, the smallest stability change, focused validation, `npm run check`, and Orca compatibility verification.
- [ ] #7 The authoritative `maint/0.9.x` branch starts at `a1faacd53ac4473a75de1d434afaf50945c2f647`, the fork remains in `0.9.x`, immutable v0.9.8 provenance is preserved, and the upstream-mirroring Rust `main` and default branch remain unchanged.
- [ ] #8 The future point-release gate requires lockstep package versions, changelog fragments, immutable tags, rollback evidence, and explicit operator authority before publication.
- [ ] #9 The intake triggers only for stable upstream releases and can append a later stable release without duplicating prior entries or automatically modifying code, automation, branches, tags, installations, or releases; main, nightly, and candidate builds are supporting provenance only.
- [ ] #10 No existing Rust maintenance automation is modified, disabled, or repurposed, and no Rust build is installed, staged, activated, or restarted.
<!-- AC:END -->
