# 2026-07-20-23 — project-d-user-hotfix

- **ModelID**: gpt-5
- **TurnCount**: 79
- **SessionDepth**: deep
- **Personas Active**: Project Engineer, QA Engineer, Code Reviewer, UX Designer, Project Manager, Database Engineer, Technical Writer
- **Beads Touched**: project-d-6lhmb, project-d-ffdx5, project-d-o8bzu, project-d-ftgqb, project-d-e0mbg, project-d-ybzmh, project-d-eb9ai, project-d-wsug0, project-d-8rp7e, project-d-gqkul

## User Value Delivered

This session shipped two user-facing improvements into the development branch.

First, we completed a scheduled-rule feature that lets users constrain automated behavior to inclusive date windows. The work covered persistence, imports and exports, unattended execution paths, editors, validation, backup and restore, documentation, and regression tests. The resulting behavior lets users schedule seasonal or temporary automation without manually toggling rules on the relevant dates. Follow-up work was recorded for automatic reversal behavior that was deliberately outside the chosen contract.

Second, we responded to a newly reported P1 usability defect in which a portaled dropdown opened beyond the browser viewport and made lower choices unreachable. The final hotfix made shared dropdown placement viewport-aware, preserved a usable scrolling region on very short screens, closed menus whose triggers left the viewport, coalesced scroll-driven repositioning, improved keyboard focus and accessibility relationships, and fixed the same confirmed behavior in a sibling select. Real-browser validation proved that the last option remained reachable and selectable at the reporter's viewport, a shorter viewport, and an extreme short-height case. The fix was reviewed, passed CI and multi-architecture builds, merged, and the user-facing issue was closed.

This was measurable user value: one requested capability became available, and one workflow-blocking interaction was removed for users with long option lists.

## What We Did Well Together

The strongest collaboration moment came when the PO explicitly required Playwright validation after the first component-level browser proof. That intervention changed the evidence standard from “the positioning calculation looks correct” to “the actual editor modal, CSS, portal, scrolling, and final selection work together.” QA then exercised the real flow at the reported and constrained viewport sizes, measured the rendered bounds and scroll area, selected the last option, verified the summary, and checked Escape/focus behavior. That concrete user-outcome proof exposed the distinction between layout-free DOM tests and real rendering and ultimately produced a much stronger hotfix.

## What the PO Could Improve

The PO surfaced two workflow expectations only after work had already reached the wrong stopping point. Earlier in the session, the agent stopped after implementation without opening a PR; the PO then correctly said that a PR was part of the workflow. Later, real-browser Playwright evidence became an explicit requirement only after the hotfix had already passed component-level checks.

At intake, one sentence such as “own this through a reviewed PR, require real-browser proof for user-facing layout defects, and stop before merge unless I authorize it” would have removed multiple correction loops. The terse follow-ups—asking for the PR number, asking why no PR existed, then separately requiring Playwright—were justified, but they made the session reconstruct the terminal state and evidence standard incrementally. The improvement is not “provide more context”; it is to state the expected delivery endpoint and verification layer at the first authorization to build.

## What the Agent Got Wrong

The agent initially treated local implementation as completion and failed to carry the repository's required publish-and-PR workflow through automatically. That was a direct workflow miss, not an ambiguity in the code task. The repository instructions already said work was not complete until pushed, and the agent should have translated that into an explicit state machine: claim issue, implement, verify, push, open PR, address review, and wait for merge authority.

The agent also accepted early synthetic browser/component evidence too readily for a viewport-and-overflow defect. The initial many-option DOM test manually supplied scroll dimensions and therefore proved selection wiring, not that a user could visually reach the last option. A computed-style assertion also ran where the shared stylesheet was not loaded, so it overstated what it proved. The agent should have recognized at triage that portals, CSS overflow, modal footers, and viewport geometry require a real-browser acceptance test. Waiting for the PO and later review to enforce that standard created avoidable rework and confidence inflation.

## What Would Make the Project Better

Add a delivery-and-evidence checklist to the repository workflow, with conditional gates by defect class:

- Every completed code bead ends in a pushed draft PR unless the PO explicitly says investigation only.
- Geometry, portal, overflow, focus, responsive, and CSS interaction defects require a real-browser test through the actual user flow at the reported viewport; screenshots alone are insufficient without assertions or measurements.
- Unit-test claims must state their layer honestly: pure placement math, scheduling/lifecycle behavior, component wiring, consumer integration, or rendered browser behavior.
- Reviews of shared UI primitives inventory sibling implementations and call sites before the first patch, so shared extraction is an intentional scope decision rather than a late review expansion.
- Merge readiness includes terminal CI, no in-flight verification, version/changelog consistency, base-branch freshness, and explicit merge authority.

This would shorten future hotfixes by preventing the exact sequence seen here: narrow patch, review-discovered shared behavior, rewritten tests, browser-proof escalation, base conflict, and another verification round.

## Persona Perspectives

### Project Engineer

- **User value assessment**: The implementation delivered requested scheduling control and removed a genuinely blocking UI failure. The modest expansion from one dropdown to a shared placement primitive prevented the same confirmed defect from remaining in a sibling control.
- **Session assessment**: Test-first correction was strong once review findings were consolidated. Failures were captured for short viewports, offscreen triggers, shared-control parity, and frame throttling before implementation. Full consumer suites, build gates, and real-browser proofs supported the final result.
- **What I'd flag**: The technical state machine should have been established before coding. Shared portaled controls should use one placement primitive with explicit coverage for layout-free test environments, tiny viewports, offscreen anchors, scroll bursts, cancellation, and cleanup.
- **Disagreement**: Speed alone was not the right success metric for the P1. A one-component patch would have shipped sooner but left a confirmed duplicate failure. The reviewer was right to require shared behavior, although unrelated accessibility or abstraction polish should not automatically become blocking without user-risk evidence.

### QA Engineer

- **User value assessment**: Testing focused on the user outcome: whether a person with a long list could reach and select the last item. It did not stop at coverage counts or position arithmetic.
- **Session assessment**: The final test layering was sound. Pure tests own geometry boundaries; hook tests own animation-frame coalescing and cleanup; component tests own accessibility, focus, and selection; consumer tests own integrations; Playwright owns rendered geometry and scrolling.
- **What I'd flag**: Two early tests gave false confidence. One manually manufactured overflow in a layout-free DOM and proved wiring only; another claimed effective CSS behavior when stylesheets were not loaded. Both needed honest names and narrower assertions.
- **Disagreement**: QA disagreed with treating the synthetic component proof as sufficient and rated the misleading regression claim more seriously than code review initially did. For a user-blocking layout bug, real modal evidence was necessary. QA supported the shared fix once duplicated exposure was confirmed, despite delivery pressure to keep the hotfix narrow.

### Code Reviewer

- **User value assessment**: Quality gates served users rather than aesthetics. Review caught an inherited CSS margin that made the rendered geometry differ from the JavaScript calculation and challenged tests that could pass without protecting the reported failure.
- **Session assessment**: The final code addressed placement, usable short-screen space, offscreen anchors, scheduling performance, cleanup, accessibility, shared behavior, release metadata, and base integration. The end state was maintainable and well evidenced.
- **What I'd flag**: Shared-dropdown parity and scroll lifecycle should have been identified in the initial design review. Late discovery forced two implementation rounds and increased conflict risk while the base branch moved.
- **Disagreement**: Mathematical top/max-height assertions were not sufficient because shared CSS participated in effective geometry. QA's real-browser evidence was the strongest proof of user value. Review favored centralization once duplicate behavior was confirmed, while delivery pressure initially favored an isolated patch.

### UX Designer

- **User value assessment**: The hotfix solved the reported inability to complete a task, including constrained-height cases that would otherwise leave only search and action chrome visible. This was accessibility and usability work, not visual polish.
- **Session assessment**: The final behavior respected viewport boundaries, retained scrollable choices, restored keyboard focus, and kept trigger/listbox relationships understandable to assistive technology.
- **What I'd flag**: Responsive acceptance criteria should include the reported viewport, a common shorter viewport, and an extreme lower bound. The product should define what happens when neither side of a trigger can fit the desired menu rather than leaving that decision implicit in component code.
- **Disagreement**: UX would not accept “users can resize the window or search” as an adequate workaround for a selector that visually hides options. It also supports closing an unanchored menu when its trigger leaves the viewport, even though engineering initially viewed that as a low-frequency edge.

### Project Manager

- **User value assessment**: Both workstreams reached merged outcomes rather than creating an unshipped backlog. The P1 was correctly prioritized and its issue and bead were closed after verification.
- **Session assessment**: Delivery was ultimately complete, but it was more serial and corrective than necessary. Missing early agreement on PR ownership, browser evidence, and shared-component scope produced extra review and CI cycles.
- **What I'd flag**: Define “done” and merge authority at intake, then consolidate review findings before redispatch. Repeated small correction loops increase elapsed time and expose branches to avoidable base/version conflicts.
- **Disagreement**: PM pressure should favor the smallest change that fully removes user risk, but not the smallest diff. Once the sibling control was confirmed to have the same defect, the shared fix was justified. Broader unrelated cleanup would not have been.

### Database Engineer

- **User value assessment**: Persistence and backup/restore work for the scheduling feature served the user's actual expectation that configured dates survive import, duplication, and disaster recovery. It was not schema work for its own sake.
- **Session assessment**: The session eventually verified round trips and malformed/range validation across storage boundaries. Review appropriately caught that a feature is incomplete if backup and restore silently discard its new fields.
- **What I'd flag**: New persisted fields should trigger a standard parity inventory across ORM, migrations, serializers, imports/exports, duplication, and backup/restore at design time. Discovering restore gaps in review is too late.
- **Disagreement**: A raw ORM-column parity assertion would be brittle because operational and transformed fields intentionally differ. A declarative restore-field contract is preferable to pretending every column should round-trip identically.

### Technical Writer

- **User value assessment**: Changelog and editor guidance helped users understand inclusive dates, time-zone semantics, and the absence of automatic reversal. For the hotfix, release metadata made the fix traceable to a build users can identify.
- **Session assessment**: Documentation was handled, but some explanations arrived through review rather than the original implementation brief.
- **What I'd flag**: User-visible scheduling semantics should be written before implementation so UI hints, validation, and changelog language share one contract. Review summaries should distinguish automated proof from manual/browser proof precisely.
- **Disagreement**: Documentation should not be used to excuse surprising behavior. Explaining that a clipped menu can be searched or that date expiry does not undo prior actions is not a substitute for either fixing the UI or making the behavioral contract explicit before users depend on it.

## Lessons

- **Keep**: Require independent code review and QA, and let real user artifacts—screenshots, viewport sizes, and reported task failures—shape acceptance criteria. The Playwright escalation materially improved the outcome.
- **Stop**: Stop calling local implementation “done” before the repository's publish-and-PR workflow is complete. Stop describing synthetic DOM tests as proof of rendered reachability.
- **Start**: At intake, state the terminal delivery state and map each acceptance criterion to the test layer capable of proving it. Inventory sibling shared components before patching a portal/geometry primitive.
- **Value learning**: Users needed reliable completion of ordinary tasks, not merely correct internal calculations. Scheduling value depended on persistence and clear semantics across every lifecycle path; dropdown value depended on the last option being reachable in the real editor. The user-visible boundary is the definition of done.
