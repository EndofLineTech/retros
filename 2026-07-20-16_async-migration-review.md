# 2026-07-20-16 — async-migration-review

- **ModelID**: gpt-5-codex
- **TurnCount**: 89
- **SessionDepth**: deep
- **Personas Active**: Project Manager, Project Engineer, IT Architect, Code Reviewer, Security Engineer, Database Engineer, QA Engineer, UX Designer, SRE, Technical Writer
- **Beads Touched**: project-d-feature-1 (pseudonymized; closed)

## User Value Delivered

This session shipped a preview-first migration workflow that lets operators safely translate assignments between two guide-source formats. The original implementation could partially change external state, exceed the request timeout, and return an opaque failure. The shipped version instead accepts long-running work asynchronously, exposes a batch identifier and live per-item progress, rejects concurrent runs immediately, preserves reconciliation evidence, and keeps partial results visible when polling is interrupted.

The work also improved the safety and trustworthiness of the feature: bounded and pinned network access remained intact; large parsing work moved off the event loop; preview signatures gained domain-separated keys; polling became principal-bound; audit-session cleanup was fixed; raw internal statuses became readable operator language; and focus recovery became test-backed. Documentation now states the real restart, cancellation, and external compare-and-swap limitations instead of implying stronger guarantees than the system can provide.

The feature was merged, its public issue and internal bead were closed, and the final public CI run completed without failures. This was measurable user value, not merely backlog movement.

## What We Did Well Together

The strongest collaboration moment was the PO asking for an action plan before authorizing implementation, then explicitly approving items 1–6. That separated diagnosis from mutation and made the architectural choice clear: reuse the established asynchronous job pattern, preserve per-item audit evidence, and document the residual external-system race rather than inventing a misleading atomicity claim. When independent reviewers found concrete blockers, the session continued through correction and re-review instead of treating the first green test suite as the finish line.

## What the PO Could Improve

Across the broader session, review work often arrived as a sequence of terse follow-ups such as another response being posted, followed by requests to fix the newly surfaced subset. That pattern made scope expand in waves and forced repeated context reconstruction. For this feature, asking for the action plan first was a major improvement; the next improvement would be to explicitly request one consolidated blocker/non-blocker pass before implementation begins. That would let the team agree on the whole pre-merge bar once instead of discovering it across successive correction rounds.

## What the Agent Got Wrong

I treated QA as a final validation step rather than an early design participant. Code, security, and data-integrity review ran first; QA then found the client-side one-hour cutoff, broken focus recovery, and later the hidden retained-progress panel. That sequencing produced multiple extra frontend commits and re-review cycles. These were not obscure polish issues: they were direct acceptance criteria for the promised operator recovery experience. I should have included QA in the first immutable-head review round, or at minimum written its slow-job, restart, retained-progress, and focus criteria into the engineer brief before the first implementation commit.

I also briefly used the PR-comment helper with the wrong invocation shape and received findings from an unrelated historical PR. The output itself clearly contradicted the requested PR number. I corrected course, but stricter premise validation before consuming tool output would have avoided the detour.

## What Would Make the Project Better

Add a standard failure-mode checklist for every multi-item external mutation workflow. Before implementation is called complete, the design and tests should explicitly cover request timeout ownership, cancellation, restart, concurrency, partial success, audit failure, idempotent retry, reconciliation, authorization of polling state, retention, and accessible error recovery. This checklist would turn the most valuable review findings from late discoveries into normal acceptance criteria.

The project would also benefit from one shared asynchronous-job contract for status vocabulary, retention, actor ownership, sanitized errors, cancellation semantics, and polling UX. Reusing code is useful, but reusing an explicit contract is what prevents each feature from rediscovering the same reliability boundary.

## Persona Perspectives

### Project Manager

- **User value assessment**: The session delivered and merged a complete user-facing workflow, then synchronized the public issue and internal tracker with detailed evidence.
- **Session assessment**: Delivery discipline was strong at the end, but successive review-and-fix cycles made completion less predictable than necessary.
- **What I'd flag**: Consolidate findings into pre-merge blockers and follow-ups before implementation begins.
- **Disagreement**: Not every worthwhile platform cleanup belongs in the current delivery; unrelated shared-tooling improvements should not delay user value.

### Project Engineer

- **User value assessment**: The final design prevents operators from receiving an ambiguous timeout after partial external changes and gives them usable recovery evidence.
- **Session assessment**: TDD, immutable commits, broad gates, and correction rounds were sound, but timeout and reconciliation should have been designed before the first review.
- **What I'd flag**: Multi-resource mutation work needs timeout, cancellation, concurrency, partial-success, and reconciliation criteria at design time.
- **Disagreement**: Obvious mechanical correctness bugs can be fixed while architectural findings are still being consolidated; not every correction needs to wait for the full panel.

### IT Architect

- **User value assessment**: Moving execution outside the request lifecycle gave users observable progress and explicit terminal behavior without inventing a second task framework.
- **Session assessment**: Reusing the established background-job pattern was correct, and the final separation among request handling, parsing, execution, and polling was materially stronger.
- **What I'd flag**: A request timeout is a consistency boundary when external state can be partially mutated.
- **Disagreement**: Shared concerns such as job lifecycle and transport security should converge on platform primitives, even when that cleanup is deferred from the feature PR.

### Code Reviewer

- **User value assessment**: Review prevented opaque partial migrations and caught visible defects in errors, borders, and status language.
- **Session assessment**: Large passing test counts and earlier approvals did not prove correctness; immutable-head review exposed claims that no longer matched reality.
- **What I'd flag**: Review evidence and PR claims must be tied to the exact head revision.
- **Disagreement**: Green CI did not make the synchronous timeout behavior acceptable; it was a correctness blocker, not polish.

### QA Engineer

- **User value assessment**: Testing ultimately focused on failures operators would experience: slow work, restart, partial progress, expiry, concurrent attempts, polling errors, and focus loss.
- **Session assessment**: The final tests were behavioral and valuable, but QA entered too late and exposed false confidence in aggregate suite totals.
- **What I'd flag**: Unit fixtures and broad suite counts must not be described as integration proof for operational failure paths.
- **Disagreement**: Successful backend and frontend suites are insufficient when the workflow's slow and interrupted paths have not been exercised directly.

### Security Engineer

- **User value assessment**: Signed preview integrity, bounded network access, principal-bound polling, domain-separated keys, and sanitized errors protected real operator actions.
- **Session assessment**: Security findings were addressed proportionately, and accepted residual races were documented honestly.
- **What I'd flag**: External systems without conditional update semantics leave a race that validation can narrow but not eliminate.
- **Disagreement**: Valid signatures alone did not make the token design complete; issuer hygiene and key separation still mattered, while unrelated repository-wide refactors did not need to block delivery.

### Database Engineer

- **User value assessment**: Exact matching, per-item revalidation, independent audit records, and reliable session cleanup made partial operations understandable without an unnecessary schema migration.
- **Session assessment**: The final model correctly avoided pretending that an external mutation and local audit record could be one transaction.
- **What I'd flag**: A failure between external acceptance and local journaling can still require direct reconciliation.
- **Disagreement**: A new schema would not automatically solve the external race; stronger durability claims would have been more dangerous than the documented limitation.

### UX Designer

- **User value assessment**: Preview, readable outcomes, retained partial progress, batch context, and focus recovery reduced destructive mistakes and operator uncertainty.
- **Session assessment**: Accessibility and failure-state clarity eventually became merge requirements, but the initial design underweighted the experience after an ambiguous timeout.
- **What I'd flag**: A batch identifier has little value unless the UI preserves it with progress, outcomes, and a clear next action.
- **Disagreement**: The synchronous timeout was not merely infrastructure debt; its most severe effect was loss of operator trust.

### SRE

- **User value assessment**: Async execution, immediate concurrency rejection, terminal retention, and partial-result visibility made failures observable and recoverable.
- **Session assessment**: The design followed a proven local pattern, but restart behavior and process-local ownership were discovered through review rather than defined up front.
- **What I'd flag**: Process-local jobs need explicit restart, stale-state, multi-worker, retention, and diagnostic contracts.
- **Disagreement**: Green CI and a clean merge state do not prove restart recovery or production observability.

### Technical Writer

- **User value assessment**: Accurate endpoint, status, retention, and reconciliation documentation gave operators and integrators guidance they can safely follow.
- **Session assessment**: Documentation improved substantially only after review found mismatches between claims and guarantees.
- **What I'd flag**: Maintain one conceptual state-transition table across API documentation and UI language.
- **Disagreement**: Writing the operator journey earlier would have exposed the timeout and reconciliation flaw; documentation should help shape the contract, not merely record it afterward.

## Lessons

- **Keep**: Use immutable-head, multi-discipline review and require each blocker to return through the implementing persona with a focused regression test.
- **Stop**: Treating aggregate green suites or earlier approvals as proof that slow, interrupted, and partially successful user journeys are correct.
- **Start**: Put QA, SRE, and documentation failure-mode criteria into the initial implementation brief for every long-running mutation workflow.
- **Value learning**: Operators need trustworthy reconciliation more than they need the mutation to appear instantaneous. A visible batch, last-known progress, readable per-row outcomes, and honest recovery guidance are core feature value, not ancillary operational detail.
