# 2026-04-26-22 — dnd-kit-observer-sweep

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: 12 (6 user messages + 6 assistant responses, plus /retro turn currently in flight)
- **SessionDepth**: moderate — single bug from issue read through ship + reporter follow-up; full lifecycle in one session, ~20 frontend files touched via agents, three persona dispatches, one PR shipped to merge
- **Personas Active**: project-engineer (×3 — investigation, implementation, shipping), qa-engineer (verification strategy)
- **Beads Touched**: project-d-5xg17 (description updated twice, closed)

## Section 1: User Value Delivered

A real user reporting unbounded memory growth (up to 5 GB on Edge) got a fix shipped to `dev` (v0.16.0-0071, PR #210, merge `9df46db0`) and a personalized retest request posted to gh #207 within ~1 hour of the PO opening the conversation. Heap leak rate on Channel Normalization went from ~2,400 KB/s to a small negative slope (GC working normally) per local Chromium verification; idle MutationObserver count went from 19 to 0. The fix swept all 9 `<DndContext>` mount sites in the codebase, not just the two reported, so future leaks of the same family on project-b Manager, Channels pane, ChannelDetail, and SubstitutionPairsEditor are pre-empted.

But: the fix introduces a UX cost. Drag handles are now hidden until users click a "Reorder" button per section. That's a real cost borne by every user — including the majority on Chromium who never experienced the leak — to fix a problem only Edge-on-busy-instance users hit. We did not validate that cost with users; we ratified it because the leak severity (5 GB) overruled UX preservation. That's a defensible call but not a free one.

Authoritative verification is still pending. Reporter reverification on Edge 147 is the only conclusive signal; PO machine couldn't reproduce locally. We shipped on a high-confidence proxy (zero observers idle = nothing for childList records to bind to), not on a verified end-to-end repro. If the reporter reports back negative, the work was directionally right but the gate was wrong.

## Section 2: What We Did Well Together

The decision-prompt compression actually worked end-to-end this session. After the engineers reported, I gave the PO a tight `## DECISIONS NEEDED` block with two questions and three options for the first one. PO answered "1. a. 2. Sweep all." — six characters, both decisions made. Same pattern again at ship time ("ship it") and at the reporter-followup question ("b"). Three decisions in three single-token replies. The format isn't theoretical — it shipped a fix in a session.

The specific moment that worked: the second decision prompt (after engineer reported clean local verification). I led with one line of state, separated UX trade-off from scope as two distinct questions, gave three options for the first with one-line trade-offs each. PO read it once, decided once. No follow-up clarifying questions. No "what does (b) actually mean?" That's the cache-warm decision-making the orchestration discipline doc is trying to produce.

## Section 3: What the PO Could Improve

When I dispatched the read-only investigation, the PO's instruction was "Have the engineers dig into it." Three words, no scope clarification. I had to interpret: investigate-only? investigate-and-propose? investigate-propose-and-fix? I chose investigate-and-propose (right call this time), but a tighter brief would have saved one round of "do we have authorization to fix this if we find it?" implicit guesswork.

A single extra clause — "have the engineers investigate and report back before fixing" or "have the engineers find and fix it" — would have removed ambiguity. The PO is right to be terse, but at the dispatch boundary, terse instructions multiply across N agents. One extra word at the PO turn saves N words of agent confusion.

The same pattern at "ship it." The fix was verified on Chromium-Playwright with the explicit caveat that the affected platform (Edge 147) couldn't be reproduced locally. PO didn't push back on that gap before authorizing ship. Saying "ship it but make sure we've planned the reverification path" or "ship to dev — but I want eyes on the reporter response" would have been worth one extra word. Instead the reverification flow had to be raised by me as a follow-up question after the merge had already happened. That's reverification-as-afterthought, not reverification-as-gate.

## Section 4: What the Agent Got Wrong

When I framed the three options for the fix shape (edit-mode gate / shared observer / dedupe toasts), I presented option (b) "shared single observer at higher level" as "more invasive, no UX change." I had not actually investigated (b)'s feasibility before presenting it. I described it the way I imagined it, not the way the codebase or @dnd-kit's API would actually permit it. The PO chose (a) on a partially-informed framing. If (b) was actually achievable cheaply — which I don't know because I didn't have the project-engineer evaluate it — we shipped a UX cost that wasn't necessary.

The fix is to push the project-engineer to evaluate ALL the options before I compress them into a decision prompt for the PO, not to invent option labels and have the engineer execute on whichever one wins. I shortcut the investigation phase to make the decision prompt look clean. That's option-tax: I made the PO pick from options I hadn't priced.

A second smaller miss: when I dispatched the shipping engineer, I over-specified the brief. I told them the exact version bump (0070→0071), the commit message format, the PR title format, the CHANGELOG entry skeleton. That's not orchestration, that's dictation. The engineer is competent to follow `docs/shipping.md` and pick the version off the current package.json. Over-specifying erodes the engineer's judgment over time and signals that I don't trust persona competence. "Follow `docs/shipping.md`" + the bead ID + the local verification numbers is enough. I padded ~600 words of unnecessary scaffolding.

## Section 5: What Would Make the Project Better

We have no path to authoritative verification of frontend memory fixes that depend on browser-scheduler differences. PO machine couldn't reproduce; Chromium-Playwright couldn't reproduce; the reporter is our only verification surface. That's fragile — if the reporter goes quiet, the next leak in this family ships unverified. Two concrete improvements would change this:

1. **Edge headed in CI or local dev environment**, with a documented seed for "active notification stream" (the conditions under which the leak surfaces). The reporter's heap delta is driven by toast traffic from the 2-second poll tier, not by the page itself. Once we can seed that condition reliably, any browser can become a verification surface — the bug isn't actually Edge-specific in cause, only in observed symptom.

2. **A regression test for the bug we just fixed.** The QA engineer's strategy explicitly called out a unit test asserting `MutationObserver` disconnect on `<DndContext>` unmount. The shipping engineer did not add it. We shipped the fix without a test that would catch this exact pattern regressing. That's the bug-becomes-regression-test rule getting silently violated. A follow-up bead for the test gap would close it.

Both are eligible for follow-up beads and would prevent the next round of "ship blind, hope the reporter is patient."

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: Fix delivers clear value — measurable heap reduction on Chromium proxy, 0 observers idle vs 19 pre-fix. But the user value is partially speculative because we couldn't reproduce the absolute bug locally. Static analysis is high-confidence for the leak class; symptom-level confirmation requires reporter reverification.
- **Session assessment**: Sound implementation. Read-only investigation phase identified the @dnd-kit `useRect` pattern as root cause via direct node_modules grep. Write-mode phase swept all 9 mount sites cleanly. Lint, typecheck, and 1231 tests passed.
- **What I'd flag**: 8 files repeat the same `isReorderMode` pattern. No abstraction extracted. That's intentional under "three similar lines is better than premature abstraction," but at 9 sites we are past the threshold where a `<GatedDndContext>` wrapper would be defensible. Worth a future cleanup bead, not this PR.
- **Disagreement**: Disagrees with UX Designer's read that the "Reorder/Done" toggle is an anti-pattern. For lists with 30+ items where reordering is rare and memory cost is real, gating is the correct trade-off — disagreement is on principle, not on this fix.

### QA Engineer
- **User value assessment**: Verification strategy was sound and actually got used — the MutationObserver instrumentation snippet shipped verbatim into the reporter follow-up comment. The reporter has the right tool to verify themselves.
- **Session assessment**: Strategy was thorough (repro recipe, measurement plan, four verification gates, risk callouts). Most of the strategy was honored.
- **What I'd flag**: Verification gate #3 (unit/integration test asserting `disconnect()` on unmount, `MutationObserver` mock-spy) was NOT added in PR #210. The shipping engineer skipped it and I didn't catch it. We have shipped the fix without the regression test that would catch the next instance of this exact pattern. That's the rule violation that surfaces as "we fixed this once already" in 3 months.
- **Disagreement**: Disagrees with Project Manager's "ship fast, scope honored" — scope was honored on the implementation but NOT on the verification (the test gate was dropped silently).

### Code Reviewer
- **User value assessment**: Quality served the user — fix targets the actual root cause, not a symptom. No band-aids.
- **Session assessment**: Diff is well-scoped (+484/-114 across 8 files). Naming is consistent (`isReorderMode`/`isEditMode`). Lint and typecheck clean.
- **What I'd flag**: No review-checkpoint between "engineer reports clean local verification" and "PO authorizes ship." The shipping engineer rolled review and ship into one dispatch. For an 8-file frontend change, an explicit review pass — even a self-review against the style guide — would have caught the missing regression test that QA called out.
- **Disagreement**: Disagrees with Project Manager that 5/5 checks green on first run validates quality. CI checks validate compilation and high-level correctness, not the bug-becomes-regression-test discipline. Green CI on a missing regression test is an artifact of CI not knowing what test is missing.

### UX Designer
- **User value assessment**: We made a UX decision (drag handles hidden behind a Reorder button) under memory pressure, without user research. The cost is paid by every user who reorders frequently. We have no data on whether that population is large or small in the user base. We should have asked.
- **Session assessment**: UX implications were briefly mentioned in the decision prompt ("UX changes — drag handles invisible/disabled until the user clicks an Edit/Reorder button") but not weighted. The PO authorized in 6 characters; UX cost was a footnote.
- **What I'd flag**: The "Reorder mode" pattern is well-known anti-pattern territory for frequent-reorder workflows. Channel Normalization in particular gets reordered when users tune their normalization rule priority — likely a routine task for ECM operators. We may have just slowed down a daily workflow to fix a leak that affects a subset of operators. A "Reorder" toggle is acceptable for one-time setup; it is a real cost on routine reordering.
- **Disagreement**: Disagrees with Project Engineer's "memory cost is real, gating is correct trade-off" — agrees the trade-off is defensible, disagrees that we made it on enough information.

### Security Engineer
- **User value assessment**: No security implications. Memory leak doesn't expose data; the fix doesn't change auth, validation, or data flow.
- **Session assessment**: No concerns relevant to security domain. CodeQL clean.
- **What I'd flag**: Nothing.
- **Disagreement**: None.

### SRE
- **User value assessment**: Reliability work was reactive — we fixed a leak after a user reported it, and we have no telemetry to detect the next instance. User experience was protected, but in the slowest possible feedback loop (user reports, we investigate, we ship, they verify).
- **Session assessment**: No instrumentation added. No metric for "MutationObserver instance count," "MutationRecord retained-size," or "toast emission rate." If this regresses we will learn from another reporter, not from telemetry.
- **What I'd flag**: We also have no metric for "session memory growth slope" client-side. The reporter's `performance.memory` probe is exactly the kind of signal we could ship as background telemetry to detect leaks proactively. That's a meaningful platform investment, not a tactical fix.
- **Disagreement**: Disagrees with Project Manager's "shipped fast" framing — fast-without-instrumentation buys us speed today and slow detection of the next leak. Net velocity is mediocre.

### IT Architect
- **User value assessment**: Architectural concern at the dependency boundary — `@dnd-kit/core` attaches a MutationObserver to `document.body` with `subtree: true` for every sortable element. That's a vendor-imposed footgun. We mitigated by gating mounts; we didn't address the underlying pattern.
- **Session assessment**: No architectural decisions surfaced this session. The fix was tactical.
- **What I'd flag**: Does the project keep using `@dnd-kit`? We have 9 mount sites; the underlying observer pattern is unchanged. If usage grows, the leak surface grows. An ADR-class decision: stay on @dnd-kit and live with the gating pattern, or evaluate alternatives (e.g., `react-beautiful-dnd-context-replacements` with different observer strategy). Not urgent, but worth a future technical-spike bead.
- **Disagreement**: None.

### Project Manager
- **User value assessment**: User got a fix in <1 hour from PO opening the conversation to PR merged. Reporter got a personalized retest request with their own probe technique. That's strong delivery on user-perceived responsiveness.
- **Session assessment**: Scope was tight (no opportunistic cleanup), authorization was explicit (1.a, sweep all, ship it, b), 5/5 checks green first try. PR #210 closed cleanly.
- **What I'd flag**: No follow-up beads were filed for the gaps surfaced in this retro: regression-test gap (QA's unsatisfied gate #3), Edge verification infrastructure (SRE/QA cross-cutting), @dnd-kit architectural review (IT Architect). Those are bead candidates that should not stay buried in a retro.
- **Disagreement**: Disagrees with QA Engineer's "shipped without regression test" framing — the test gap is real but does not invalidate the ship. Disagrees with Code Reviewer that ship-without-review-checkpoint was a quality miss; the engineer self-reviewed implicitly via lint + typecheck + 1231 tests.

### Technical Writer
- **User value assessment**: CHANGELOG entry is detailed and grep-friendly. Bead description is updated. No user-facing documentation update for the new "Reorder" button paradigm.
- **Session assessment**: Documentation discipline was honored at the engineering level (CHANGELOG, bead). Not at the user level.
- **What I'd flag**: ECM operators who use Channel Normalization daily may notice drag handles are gone after the v0.16.0-0071 update and not know about the Reorder button. A user-guide note or release-notes blurb under `docs/user_guide/` would be ~5 minutes of work and would prevent "where did the drag handles go?" support questions. We have a `docs/user_guide/` per CLAUDE.md but did not update it.
- **Disagreement**: None — this is a gap, not a contested decision.

### Database Engineer
- **User value assessment**: Not relevant to this session. No schema changes, no query work.
- **Session assessment**: N/A.
- **What I'd flag**: Nothing.
- **Disagreement**: None.

## Section 7: Lessons for Future Sessions

- **Keep**: Compressed `## DECISIONS NEEDED` blocks separated from state/findings. The PO answered three decisions in three single-token replies this session. Format works.
- **Keep**: Two-stage dispatch — read-only investigation first (project-engineer + qa-engineer in parallel, no worktree), write-mode implementation second (single project-engineer in worktree, scope already determined). The investigation/implementation firewall held.
- **Stop**: Inventing decision-prompt options I haven't priced. The "(b) shared single observer" framing was a description of what I imagined, not what the codebase or `@dnd-kit` API permits. Have the engineer evaluate options BEFORE compressing them.
- **Stop**: Over-specifying shipping briefs. `docs/shipping.md` exists; the engineer is competent to follow it. Brief should be: bead ID, branch, worktree path, local verification numbers, "follow `docs/shipping.md`." Not 600 words of step-by-step mechanics.
- **Start**: Filing follow-up beads at retro time — the gaps surfaced here (regression-test, Edge verification, @dnd-kit architectural review, user-guide update) should become beads, not retro footnotes. Three of those (test gap, user-guide note) are <30-minute work units.
- **Start**: Surfacing UX cost as a weighted dimension in decision prompts, not a footnote. "Option (a) — fixes leak, hides drag handles behind a Reorder button (UX cost: routine reorderers add one click)" is more honest than "UX changes."
- **Value learning**: Browser-scheduler differences mean local Chromium verification is necessary but not sufficient for memory leaks driven by event delivery rates. The reporter is our authoritative verification surface; we should treat them as a partner in the verification gate, not as a recipient of the ship. Posting the reverification request immediately (option b) was the right move precisely because of this.
