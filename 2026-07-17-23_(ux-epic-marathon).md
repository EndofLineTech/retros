# 2026-07-17-23 — ux-epic-marathon

- **ModelID**: claude-fable-5
- **TurnCount**: ~83 (33 user messages incl. mid-turn interjections, ~50 assistant turns; failure corrections clustered past turn 50)
- **SessionDepth**: deep — full-app UX walkthrough (3 passes, every tab), 16 PRs opened, 15 merged, ~14 subagent dispatches across 4 personas, live-instance work throughout
- **Personas Active**: ux-designer (3 walkthrough passes + re-check + menu design review), project-engineer (11 implementation dispatches + 1 read-only spike + 1 triage), technical-writer (tutorial + screenshot redaction), Explore (triage); code-reviewer and qa-engineer notably NOT invoked
- **Beads Touched**: UX epic children .2–.17 (created .2–.15 this session, .16–.17 late-session; closed .2–.15 except the unscheduled .17), tutorial bead .16-of-tutorial-epic (closed, shipped), 1 P1 task closed as working-as-designed, 3 new beads filed (rollback-409 bug, journal auto-purge policy, date-token validator bug — the last one fixed and merged same-session, held open for reporter confirmation)

## Section 1: User Value Delivered

Unusually high, and almost all of it directly operator-visible:

- **Shipped a complete end-to-end "first channels" tutorial** with 12 screenshots (privacy-redacted before publishing to the public repo), unblocking new-operator onboarding.
- **Shipped 14 UX improvements from the walkthrough epic in one day**: a dirty-guard on a modal that silently discarded batched toggles; a shared header-overflow policy fixing three tabs where buttons were literally unreachable at common laptop widths; Cancel buttons on 8 form modals plus a CI guard preventing recurrence; scroll-reset on settings navigation; three profile-concept naming collisions resolved; Streams-pane categorization over ~90 flat groups; sortable/filterable logo table at ~3k-logo scale (which also surfaced and fixed a silently-broken search); disambiguated destructive rollback actions; a rule editor converted to a true step wizard; a security-relevant setting moved out of a misnamed page and given explicit save semantics; a legacy/duplicate feature section folded away; merge candidates shown by name instead of raw id; and a 9-item polish batch.
- **Fixed a user-reported bug same-day**: a documented date-token feature was being rejected by the save-time validator — users literally could not save what the user guide told them to write. Triage → root cause → fix (which found a *second* broken gate triage had missed) → merged in a few hours.
- **Closed a P1** as working-as-designed with an evidence-backed reply to the reporter instead of speculative engineering.
- The only work-that-creates-work: one discarded PR (see Section 4).

## Section 2: What We Did Well Together

The **verification tripwires in engineer briefs** paid for themselves twice in an hour. Bead .8's brief required proving three "duplicate" fields were actually duplicates before deleting them — the engineer STOPPED, showed all three were distinct entities and two carried the operator's live config, and the PO reversed the decision on evidence. Immediately after, bead .4's brief required proving a migration was feasible before removing a "dead" feature — the engineer STOPPED again, showed the "dead" path was actually the one a shipped feature depends on, and the PO reversed that ruling too (options A/B/C, picked B, answered in one word mid-turn). Two wrong premises, zero wrong deletions, and the PO's fast decision turnaround meant neither stop cost more than minutes. This is the collaboration pattern at its best: agent verifies, PO decides, nobody guesses.

## Section 3: What the PO Could Improve

The legacy-vs-enhanced feature ruling was delivered as settled fact — "the legacy path is a thing, the enhanced path is dead, the opposite of what was thought" — when it was actually a hunch, and it was backwards. That confident framing shaped a bead description, a migration decision, and an opus-tier engineer dispatch before the feasibility gate caught it. The tripwire saved the outcome, but a "I *think* legacy is the supported one — verify before acting on that" would have routed this through a cheap read-only spike first and saved a full investigation cycle. Related, smaller: "I've gone ahead and added that and redeployed the container" — the redeploy hadn't actually taken effect (env var absent, container uptime unchanged), which cost a verification round-trip. Both are the same shape: statements of fact that were actually intentions or beliefs. The agent verifies either way — but flagged uncertainty gets verified *before* work is built on it, not after.

## Section 4: What the Agent Got Wrong

Bead .10 had the same user-facing design fork as .4 and .8 (wizard conversion vs. visual differentiation), and I surfaced those two to the PO before dispatch — but delegated .10's to the engineer with a "bias plus fallback" brief. The engineer chose the fallback, built it, PR went green, and only at the merge call did the PO point out: "You never asked me though, what I'd want to do with .10." They chose the other option; PR #692 was closed unmerged and a second opus run redid the work as a wizard. Entirely avoidable — the rule I then adopted mid-session should have been in force from the start: **a fork that changes what operators see gets surfaced before dispatch, even when the engineer is competent to choose.** Honorable mention: I let the screenshot-redaction agent grind through repeated full-image verification passes until the PO had to tell me it was burning too many tokens — I should have bounded that run's verification loop in the brief or killed it a cycle earlier myself.

## Section 5: What Would Make the Project Better

Two recurring process frictions, both fixable at the skill/brief level (captured here as retro payload per the hooks-are-not-beads rule):

1. **Engineer agents stall at the CI-watch step.** Five separate dispatches ended their run with "I'll wait for the background check-watch to notify me" — delivering no final report — because the agent's background `gh pr checks --watch` doesn't re-invoke a completed subagent. The orchestrator re-derived state from PR bodies each time (workable, but the .10 run also left the main checkout sitting on its feature branch, which later broke a branch-delete operation). The engineer brief template should say: *foreground the checks watch, deliver the full report after it resolves, and check out the default branch before finishing* — or the orchestrator should own the watch step entirely and the engineer should end at "PR open."
2. **Version-number collisions between overlapping PRs.** Two green PRs both claimed the same build number because the second branched before the first merged; the fix required a resume-rebase-rebump cycle. With a strictly-sequential merge cadence this is pure overhead — either enforce one-in-flight-PR, or make "rebase onto origin/dev and take the next free build number immediately before the version-bump commit" a standard clause in every implementation brief (it worked when included).

## Persona Perspectives

### UX Designer
- **User value assessment**: The 3-pass walkthrough converted into 14 shipped fixes — audit-to-ship in one session is the best possible outcome for heuristic review work. The menu redesign correctly stopped at a decided mockup instead of rushing implementation.
- **Session assessment**: My findings were treated as first-class work items, and my re-check pass (verify-before-filing) was honored. The v2 mockup round — where the PO surfaced the split-pane width constraint I'd missed — materially changed the recommendation (floating bar over in-pane buttons).
- **What I'd flag**: Two of my three major "consolidation" findings (duplicate fields, dead feature section) were wrong at the code level. My audits report what the UI *presents*; same-label-different-entity is exactly the trap surface-level review falls into. Future audit reports should carry an explicit "presentation-level claim — verify entity identity before acting" tag on consolidation findings.
- **Disagreement**: With the orchestrator — my pass-2 finding fed a PO decision (drop the fields) before anyone verified it. The tripwire caught it, but the verification should have preceded the decision option list, not the deletion.

### Project Engineer
- **User value assessment**: Every merged PR traced to an operator-visible pain point; no speculative infrastructure was built. The date-token fix restored a documented feature users were locked out of.
- **Session assessment**: TDD held across all 11 implementation runs (red-first proven in several), gates were green before every PR, and the stop-tripwires were respected rather than argued with. The rebase-rebump cycle on the colliding build numbers was clean.
- **What I'd flag**: The CI-watch stall pattern (Section 5) is ours to fix — five truncated final reports made the orchestrator do our reporting for us. And one run left the shared checkout on a feature branch, which is exactly the kind of state pollution the worktree rules exist to prevent.
- **Disagreement**: None of substance.

### Code Reviewer
- **User value assessment**: The shipped quality appears high — but that's inference from CI and orchestrator diff-reads, not review.
- **Session assessment**: **Fifteen PRs merged in one session with zero code-review persona passes.** CI (tests, CodeQL, Semgrep, a fake-test guard, visual regression) plus orchestrator verification of checks/file-lists substituted for human-lens review. For label renames and CSS that's defensible; for the wizard conversion (~68 tests rewritten), the pending-merges API contract change, and the validator fix touching the regex safety layer, a review pass was warranted and skipped.
- **What I'd flag**: The one place my absence nearly mattered: `window.confirm` for the dirty-guard was fine, but the audit-test idea in the Cancel-button PR is the kind of thing review normally has to demand — the engineer volunteering it was luck of a good brief, not process.
- **Disagreement**: With the PM's velocity framing below. Fifteen merges/day is only a win if the review debt is genuinely zero; we don't know that, we assume it.

### QA Engineer
- **User value assessment**: Live verification on the operator's real instance before every merge caught what unit tests can't (stale-cache false alarm, real data intact after every run) — that protected the user directly.
- **Session assessment**: Engineer self-verification was consistently thorough (the 422→200 live proof with residue checks was textbook). But self-verification is not independent verification; the orchestrator re-checked CI and file lists, not behavior.
- **What I'd flag**: The wizard conversion shipped with a strong round-trip test but no regression pass over the *analyzer/advisory* behaviors that observe all steps — if those interact with step gating in an unforeseen way, users find it before we do.
- **Disagreement**: With the engineer's "gates green = done" implicit stance on the larger PRs; the two biggest diffs deserved a dedicated QA dispatch.

### Security Engineer
- **User value assessment**: The screenshot-redaction flow protected the operator's real provider relationships and internal addresses from a public repo — genuine harm prevented, including the orchestrator's enumerative image review before ship and catching an internal IP the writer missed.
- **Session assessment**: Good instincts throughout (fleet-wide grandfathering on the legacy fold, no data deletion defaults). But the SSRF-allowlist relocation — moving a security control, changing its save semantics, deleting its page — merged with no security-persona review. Low-risk as executed (endpoint untouched), wrong as precedent.
- **What I'd flag**: The deprecated admin router with the only user-creation endpoint ("live, unreached attack surface") had its deletion step lost when its tracking bead closed — the PO declined to re-file when offered. Noted here so the corpus remembers: unreached attack surface with no deletion tracker.
- **Disagreement**: With the orchestrator's "routine UI work" classification of the SSRF relocation; anything touching a security control's UX should get my read-only pass by default.

### Project Manager
- **User value assessment**: 13 beads closed, an epic taken from triage to done-minus-one-optional in a single session, and every merge explicitly PO-authorized. Board hygiene was maintained in real time (decisions recorded on beads as they happened).
- **Session assessment**: The decision-block discipline worked — the PO answered most forks in under a sentence because they were presented as clean options. The one process wobble (.10) was named, corrected, and the correction held for the rest of the session.
- **What I'd flag**: Session length. By the final third we were stacking parked decisions (h2oxl fix shape asked three times, still unanswered at session end) — parked items should get a single end-of-session summary rather than repeated re-asks inside working turns.
- **Disagreement**: With the code reviewer: the merge cadence was appropriate *for this batch* — small scopes, hard CI, live verification. I'd concede the two large PRs as exceptions that deserved review.

### Technical Writer
- **User value assessment**: The tutorial is real operator value, and the redaction protected the operator while keeping instructional fidelity. Docs were updated in-line with every behavior change (style guide overflow policy, deprecation notes, backup chooser copy).
- **Session assessment**: Good — docs shipped *inside* the feature PRs rather than as follow-up debt.
- **What I'd flag**: The tutorial's two M3U screenshots now show a pre-redesign header (kebab didn't exist at capture time); the retake is queued but unscheduled. Every week it slips, new operators see a UI that doesn't match their screen — the exact failure the tutorial epic exists to prevent.
- **Disagreement**: None.

### SRE
- **User value assessment**: The MCP health-badge fix (a deployment env-var gap, root-caused to DNS search-domain resolution from inside the container) removed a standing false alarm on the operator's console — small but real trust repair.
- **Session assessment**: Live-instance discipline was strong: single-writer sequencing across agents, read-only fences enforced, zero unintended mutations across ~10 live-touching runs, and the one container-recreate surprise (Portainer pull reverting docker-cp'd builds) was detected and reasoned about rather than blindly re-deployed.
- **What I'd flag**: The session repeatedly deployed unmerged builds to the operator's production container as part of verification. Standard practice for this project, and it worked — but the fleet-pull event mid-session shows how easily container state and merged state drift. A "container state vs dev HEAD" check at session start would make that drift visible instead of discovered.
- **Disagreement**: None.

### IT Architect / Database Engineer
- **User value assessment** (joint, minimal footprint): The shared-module extraction for date placeholders and the shared OverflowMenu/overflow policy are the right shape — single source of truth replacing drift-prone duplicates. No schema work this session.
- **What I'd flag** (architect): The pending-merges API took the router's existing "first 1000 entities" batch convention, which degrades silently on large installs; acceptable inherited debt, but it's now in two more places than it was.
- **Disagreement**: None.

## Lessons

- **Keep**: Mandatory verify-before-destroy tripwires in every brief that deletes, migrates, or deprecates — two of two wrong premises were caught by them this session, at the cost of minutes each.
- **Keep**: Ranked-option decision blocks with one-word answerability; the PO resolved ~15 forks this session with near-zero latency because every ask was pre-structured.
- **Stop**: Delegating user-facing design forks to engineer judgment via "bias plus fallback" briefs. If an operator will see the difference, the PO picks first — the .10 rework was the direct cost of skipping this.
- **Start**: A standard closing contract in engineer briefs — foreground the CI watch, deliver the final report after resolution, return the checkout to the default branch — or move the watch step to the orchestrator entirely. Five stalled final reports in one session is a pattern, not an accident.
- **Value learning**: UI-level "duplication" findings are hypotheses about *labels*, not facts about *entities*. Both major consolidation premises this session (duplicate fields, dead feature) inverted on code contact — while both survived every earlier discussion untouched. The cheap read-only code trace, done before the PO decision instead of after, is the correction.
