# 2026-07-02-12 — onboard-review-pr18

- **ModelID**: claude-fable-5
- **TurnCount**: ~48 (7 substantive PO messages; the remainder assistant turns and 18 subagent completion notifications across two 10-agent fan-outs)
- **SessionDepth**: deep — two full 10-persona fan-outs (onboarding baseline + PR18 review), ~20 agent reports synthesized, COMPONENTS.md authored, review posted to GitHub
- **Personas Active**: all 10 (security-engineer, it-architect, project-manager, project-engineer, ux-designer, code-reviewer, database-engineer, sre, qa-engineer, technical-writer) — twice each
- **Beads Touched**: created project-b-96k, project-b-lgk, project-b-a0w, project-b-ar6 (onboarding value-gated findings). None filed for PR18 per PO direction.

## 1. User Value Delivered

Concrete and shipped:

- **A posted, actionable review on PR #18** (github.com/egyptiangio/project-b/pull/18#issuecomment-4868244353) — the session's actual deliverable per the PO's closing scope statement. The review is not generic: it contains findings verified by *execution* (idempotence bug in the display-name prepend, first-icon-only regex) and by *data* (joining overrides.db against stations.db to expose 360 silent logo clobbers behind a misleading "0 conflicts" claim), plus a broken-bead/stale-body process catch. The PR author gets a fix list they could not have gotten from a single-reviewer pass.
- **COMPONENTS.md** now exists at the repo root (home-lab, all components), unblocking every future team ceremony without re-litigating tier.
- **An onboarding baseline** identifying the project's two real gaps (untested merge/emit core; no failure notification or backup drill) against an otherwise unusually healthy codebase — captured as 4 backlog beads the PO can sequence or ignore.

The onboarding was instrumental value (a prerequisite the tooling forced), not something the PO asked for on its own. It happened to surface genuinely useful findings, but the session's cost-to-value ratio was widened by it — see Sections 3–5.

## 2. What We Did Well Together

The authorization exchange at the onboard gate. When team-review refused to run without COMPONENTS.md, the PO answered in one line: "Do the onboard. You do not need my permission. home-lab tier." That single message resolved three things that usually take a round-trip each — authorization, autonomy level, and the tier declaration that Step 3 of the onboard would otherwise have paused on. It let the entire 10-agent onboarding run and synthesize without a single blocking question, and the pre-declared tier was injected into every agent brief so no persona wasted output on enterprise-grade findings.

## 3. What the PO Could Improve

**The scope of the PR18 review — "post it for the dev, don't bring me decisions" — arrived at the end instead of the beginning.** The original ask was "Lets get the team to review PR18." I synthesized the 10 reports into a PO-facing package: a conflicts section, three framed decisions (the 360-logo call, the bundled-feature call, the leave-draft gate), and a follow-up-beads offer. Only after the review was posted did the PO clarify: "All we are doing is posting the review of 18 — it's up to the original dev to fix." Had that been in the first message ("get the team to review PR18 and post the findings for the author"), the synthesis would have been written for the author's audience from the start, the decision-framing turns (and the re-surfaced decision list after the credits question) would not have happened, and the session would have been two turns shorter. The information existed only in the PO's head; nothing in the repo or the PR indicated the author was a third party whose fixes weren't ours to plan.

## 4. What the Agent Got Wrong

**Assigning haiku to the SRE reviewer produced a miscalibrated report I then had to spend synthesis effort correcting.** The SRE agent rated the missing temp-file cleanup "CRITICAL — FAIL, fix required before merge" and asserted a disk-fill scenario ("hourly builds → gigabytes") not supported by the code's actual failure behavior. The engineer and code reviewer (both sonnet) verified the same code path and established the correct severity: loud failure, original file intact, orphaned temp = cheap Warn. The synthesis caught and tempered it — but a wrong severity that survives into a posted review damages the review's credibility with the author, and it was luck that two other personas covered the same lines. The home-lab downshift rule says haiku; judgment should have overridden it for any persona whose review touches code behavior. (Secondary miss, same theme: the beads pre-commit hook didn't auto-export the new beads as assumed — verified and fixed with an explicit `bd export` + amend, but the first commit went in without checking.)

## 5. What Would Make the Project Better

**A cheap metadata preflight before any multi-agent PR review.** Four of ten reviewers independently spent effort discovering the same three process facts: the PR body was stale relative to head, the cited bead (project-b-nw8) didn't exist, and a second feature was bundled. A 30-second orchestrator preflight — `gh pr view --json commits` vs body, `bd show <cited-bead>`, diff-vs-description scan — would have either (a) bounced the PR back to the author for hygiene before spending a 10-persona review on it, or (b) put those facts in every brief so reviewers spent their tokens on code instead of forensics. Worth adding to the team-review skill's Step 0.

## 6. Persona Perspectives

### Security Engineer
- **User value assessment**: Real value both passes — verified the committed overrides.db byte-for-byte against its claims (protects the operator from a poisoned tracked binary) rather than checkbox scanning. Confirmed escaping at every injection site before anyone relied on the doc's word.
- **Session assessment**: Heard and correctly weighted. My findings were P3s and were presented as P3s — nobody inflated them for effect.
- **What I'd flag**: The onboarding validateOutputPath denylist finding is still unfiled; if the scheduler tier ever moves to small-team (the architect's open note), it becomes a real gap.
- **Disagreement**: None this session; the tier calibration held.

### IT Architect
- **User value assessment**: The PR18 layering verification (id-freeze constraint, verified against rewriteIds' positional contract) is the kind of finding that prevents a future "why did all my channel ids change" incident for the operator.
- **Session assessment**: Good. My component inventory became COMPONENTS.md nearly verbatim, and my scheduler-tier caveat was preserved in the Notes section rather than flattened.
- **What I'd flag**: The jesmann CDN hotlink dependency is now in a posted review but recorded nowhere durable. If the PR merges without addressing it, the accepted risk exists only in a GitHub comment.
- **Disagreement**: With the DBA — I maintain the two-store base+overlay model is sound design, not duplication. The synthesis presented both positions to the author rather than picking one, which was correct.

### Project Manager
- **User value assessment**: Mixed. The PR18 review is delivered value. But the session also produced 4 beads and 6 open decision items the PO explicitly deferred or declined — work-about-work that the closing scope statement suggests wasn't wanted.
- **Session assessment**: The board catch (project-b-nw8 missing, duplicate beads) was the review's most immediately actionable finding. Session-close discipline was followed: committed, rebased, pushed, verified.
- **What I'd flag**: The onboarding decision items (project-b-55y, project-b-4yt, project-b-2em, the P3 batch) were surfaced twice and answered zero times. Either they matter — put them at the top of next session — or they don't, and we should stop carrying them.
- **Disagreement**: With the orchestrator's bead filing: the onboard skill authorized it, but filing 4 P2s minutes before a PO who then said "all we are doing is posting the review" suggests the value gate should have been a proposal, not an action.

### Project Engineer
- **User value assessment**: High — my verification that failure paths are loud and non-corrupting is what kept the posted review honest (it prevented the SRE's "corruption-class" framing from reaching the author).
- **Session assessment**: Sound. Running the PR's tests in isolation (scratchpad, symlinked node_modules) without touching the working tree was the right read-only discipline.
- **What I'd flag**: Nothing new; the review stands.
- **Disagreement**: With the SRE's severity, resolved in synthesis (correctly, in my favor).

### UX Designer
- **User value assessment**: The reseed-footgun finding protects a real future person — the operator who hand-seeds 50 display_names and loses them to a reseed. That's user harm prevented, not design preference.
- **Session assessment**: UX had a genuine seat in a backend PR review, which is rare and was warranted here (display names are the most user-visible artifact this project produces).
- **What I'd flag**: The name/id divergence trade-off is documented only in the PR body and inline comments; when display_name seeding starts, player-side confusion reports will follow unless the README paragraph lands.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: The Block on documentation was user-serving, not aesthetic — the repo's own convention exists because the operator is the only support channel, and an undocumented precedence rule means an unfixable "my logo won't change" evening.
- **Session assessment**: The dual fan-out format worked; my findings overlapped four other personas and the synthesis deduplicated rather than double-counting, which kept the posted review readable.
- **What I'd flag**: The posted review is one long comment rather than inline anchors. Fine for a draft PR; for a larger diff, inline comments would serve the author better.
- **Disagreement**: Mild, with the engineer on sax-vs-regex — I flagged reuse, they called the deviation defensible. Both positions went to the author as a comment-your-choice item, which is the right resolution.

### Database Engineer
- **User value assessment**: The 360-row clobber discovery is the single highest-value finding of the session — it changes what the operator's guides will actually show, and it was invisible to every other review method (only a cross-db join surfaced it).
- **Session assessment**: Extract-and-query against scratch copies was the right pattern; no repo state touched.
- **What I'd flag**: My "trim the seed vs. document precedence" question is now the author's call with no PO steer. Fine per the closing scope, but if the author picks override-wins, the stationdb curation for those 360 stations is dead weight going forward.
- **Disagreement**: With the architect on two-store duplication — unresolved by design (PO delegated to the author). Recorded, not averaged.

### SRE
- **User value assessment**: The temp-file and observability findings are real operator-facing issues even at corrected severity.
- **Session assessment**: My report's severity was overruled by two siblings with better evidence, and the synthesis said so explicitly. That's the system working — but I'd rather have been calibrated correctly in the first place. (Orchestrator's note in Section 4 stands: this was a model-assignment issue.)
- **What I'd flag**: Onboarding gaps project-b-a0w (backup drill) and project-b-ar6 (failure notification) are the two findings most likely to hurt the operator this year; they're filed but unsequenced.
- **Disagreement**: I rated temp-file cleanup a merge blocker; engineer + code reviewer rated it Warn. The posted review carries their calibration with my finding — acceptable, though I note the fix is 5 lines either way.

### QA Engineer
- **User value assessment**: Executing the code (not just reading it) found the two confirmed bugs in the posted review. Execution-based verification is what made this review worth posting.
- **Session assessment**: Good. My onboarding finding (merge.js untested) directly sharpened my PR18 brief — the fan-out sequencing paid off.
- **What I'd flag**: The display-name path has never touched real data (every seeded row is NULL). When seeding starts, that's the moment the untested-escaping and idempotence findings stop being theoretical.
- **Disagreement**: Anticipated engineer pushback on "edge cases that can't occur yet"; the synthesis kept my findings as Warns with the "yet" framing visible. Fair.

### Technical Writer
- **User value assessment**: The doc-gap Block protects the future operator, and it was grounded in the repo's own convention rather than an imported standard — which is why it should survive author pushback.
- **Session assessment**: Both fan-outs treated documentation as evidence to verify (spot-checking README claims against source), not a checkbox. The onboarding caught a real doc error (--conc default) that would have misled an operator.
- **What I'd flag**: This retro itself and the posted review are now the only records of several session findings (CDN risk, P3 batch). The project's own rule is bd-or-it-didn't-happen; half this session's knowledge lives outside bd.
- **Disagreement**: None.

## 7. Lessons

- **Keep**: Injecting sibling-persona context into second-wave briefs (onboarding findings → PR-review briefs). The QA and DBA reviews were sharper because their briefs carried the baseline's coverage-gap and SQLite-convention facts.
- **Keep**: Execution-based and data-based verification in review agents (run the code twice; join the two databases). Every finding in the posted review that will actually change the author's behavior came from execution or data, not reading.
- **Stop**: Downshifting review personas to haiku on code-adjacent surfaces because the tier table says so. The one haiku reviewer with code in scope produced the one miscalibrated report. Tier modulates *expectations*, and should modulate model choice only for genuinely formulaic work (the PM/TW haiku reports were fine).
- **Start**: Asking "who is the audience for this deliverable, and who owns the fixes?" at review kickoff when the PR author isn't obviously the PO. One question up front would have removed the decision-framing overhead the PO ultimately didn't want.
- **Value learning**: The PO's revealed preference this session: lean footprint, artifact-shaped deliverables (a posted review, a committed file), decisions delegated to whoever owns the work. The elaborate decision-block machinery was overhead here — useful when the PO owns the fix, noise when a third party does.
