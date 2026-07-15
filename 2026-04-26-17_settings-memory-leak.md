# 2026-04-26-17 — settings memory leak

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: ~18 (9 user / 9 assistant, plus system-reminder injections)
- **SessionDepth**: moderate — single bug, but traced through three interacting components (`SettingsTab`, `useHashRoute`, `ScheduledTasksSection` + `NotificationCenter`), browser-verified end-to-end, shipped via PR with one CI iteration.
- **Personas Active**: project-engineer (implementation, debugging), code-reviewer (style guide, lint), qa-engineer (verification protocol), technical-writer (CHANGELOG entry, comment compression). Implicit only — not invoked via Skill.
- **Beads Touched**: `project-d-5xg17` (created, in_progress, updated with root cause, closed).

## Section 1: User Value Delivered

Real and measurable. The reporter on gh #207 was hitting 5 GB Edge memory growth that escalated to browser crashes — a P2 "ECM is severely degraded" report. We shipped a 14-line fix in PR #209 (`v0.16.0-0070`), browser-verified with the worst-case trigger primed (stranded sessionStorage key, never-matching taskId): pre-fix this scenario would push thousands of history entries within seconds and balloon the heap; post-fix, 7 navigations produced exactly 7 history entries and the heap was reclaimed by GC. Posted a comment on the issue directing the reporter to pull `dev`. The user no longer crashes their browser when navigating Settings. That's the win.

A second, durable win: the fix to `useHashRoute.setHash` (bail out when the route is unchanged) is defense-in-depth that immunizes the entire app against the same fresh-object-on-every-call pattern that caused this bug AND the recharts/Stats bug from PR #208 the day before. Future no-deps `useEffect`s in any consumer can't reproduce this failure mode.

## Section 2: What We Did Well Together

The PO's framing in the opening message ("Ignore the analysis in the issue as we're not sure it is true. What we can confirm is there appears to be a memory leak in the frontend") set the right relationship between agent and prior LLM output. I took it seriously: I read the actual code — `SettingsTab.tsx:319-324`, `useHashRoute.ts:71-77`, `ScheduledTasksSection.tsx:401-415`, `NotificationCenter.tsx:326-328` — and derived the loop mechanism myself before noting that the analysis happened to be correct on mechanism but had a subtler trigger story (the "any Settings navigation" claim was contingent on the sessionStorage key being set first). The PO confirmed direction with "you can proceed" only after that verification was on the table. This is the right division of labor: PO doesn't trust an unverified analysis, agent does the verification, neither blindly follows.

## Section 3: What the PO Could Improve

At turn H4 ("Our PR failed tests; please check"), my immediately-prior status message had shown all 5 required checks **green** and the PR **merged at f9cfc96a**, with post-merge `Tests` and Docker runs `in_progress`. The PO appears to have been looking at an earlier `gh pr checks 209 --watch` output that included the first push's Frontend Tests failure (which I'd then fixed and pushed again). The "please check" was a no-op turn for me — everything was already green.

A more efficient prompt would have been: "I'm seeing 'Frontend Tests fail 1m1s' in the run — is that the lint regression you mentioned, or something new?" That would have surfaced the source of the confusion (looking at stale check output) in one turn instead of forcing me to re-verify state from scratch.

Same thing happened at turn H7, where the PO pasted the lint error verbatim and asked "did you clear it?" — that was the **same** lint failure from the first push of #209, which had been fixed in commit `1e8996e4` and merged. Asking "is the eslint-disable placement on dev still wrong?" with a pointer to what they were reading would have been a one-line confirmation rather than re-fetching the code from `origin/dev`.

The pattern: when CI history is long, pointing me at *which* check output the PO is reading saves a turn. "I see X — is that current or stale?" beats "X failed — please check."

## Section 4: What the Agent Got Wrong

At turn A3 (the first push of PR #209), I shipped a lint regression that broke the Frontend Tests check. The cause: I placed `// eslint-disable-next-line react-hooks/exhaustive-deps` on the line **before** `useEffect(() => {`, but `react-hooks/exhaustive-deps` emits its diagnostic on the dep-array line (`}, []);`) — so my disable directive was unused (caught by `--report-unused-disable-directives`) AND the actual warning was unsuppressed.

This was avoidable in two ways:

1. **I ran `npm test`, `npm run build`, and `tsc --noEmit` locally before pushing — but I did not run `npm run lint`.** `docs/shipping.md` §1 lists `npm test` and `npm run build` but not lint explicitly, and I followed that list literally. I should have noticed that the script defines `eslint . --report-unused-disable-directives --max-warnings 0` and run it.

2. **The existing pattern was 24 lines above me in the same file.** `SettingsTab.tsx:295-301` already had this exact pattern with the disable directive correctly placed right above the `}, [initialSettingsPage]);` line. I didn't read or copy it — I wrote my own placement from intuition, and got it wrong.

Cost: one CI cycle (~3 min), one extra commit on the branch, one round-trip with the PO who flagged it.

Adjacent miss: my first pass at the code comments was multi-line and referenced `gh #207` and `PR #208`. I caught and compressed both before pushing — but only after writing them. CLAUDE.md is unambiguous on both ("no multi-line comment blocks" and "Don't reference the current task, fix, or callers — those belong in the PR description"). Should have written terse and rule-compliant from the start instead of revising.

## Section 5: What Would Make the Project Better

`docs/shipping.md` §1 "Run Quality Gates (MANDATORY)" lists `npm test && npm run build` for the frontend — which is what I ran — but the **CI** also runs `npm run lint` as a blocking step inside the `Frontend Tests` job. The shipping doc and the CI are out of sync: the doc tells the agent (and any human shipper) to run a strict subset of what CI will block on. Add `npm run lint` to the §1 list so the local quality gate matches the CI gate. This would have caught my regression in 30 seconds locally instead of in CI three minutes after the push.

Stretch goal: a pre-push hook that runs lint+test+typecheck. Would prevent the same failure mode for any agent or human who reads the docs less carefully than they should.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: Strong. We shipped a real bug fix for a real user complaint, with code-level verification (read every component on the path) and runtime verification (DevTools-instrumented browser test with the worst-case trigger). No premature abstraction, no scope creep — minimal correct fix, two files, 14 net lines.
- **Session assessment**: TDD wasn't applicable (this was a bug fix, and the existing `useHashRoute.test.ts` suite already covered the route-change behavior in a way that the bail-out doesn't break). Container-first development was followed — local edit → build → `docker cp` → browser verify → PR. Bead created before reading code, closed after merge. Beads workflow was clean.
- **What I'd flag**: I missed `npm run lint` in my pre-push gate. This is on me, but it's also on the docs (see Section 5). The fact that I "knew" about `--report-unused-disable-directives` from the failure log but hadn't thought about it before pushing is a discipline failure I should internalize.
- **Disagreement**: None with QA — we agree the verification was sound. Mild tension with the code reviewer over the eslint-disable directive itself: arguably the cleanest fix would have been to move `setActivePage` definition above the effect AND wrap it in `useCallback`, eliminating the dep-warning entirely. I chose the lighter `[]`+disable path because the wrap was already in the file (`SettingsTab.tsx:295-301`) and adding a useCallback for a function only used in click handlers would have been local-only churn. The reviewer is right that the eslint-disable is a code smell; I'm right that the alternative was scope expansion for a memory-leak fix.

### Code Reviewer
- **User value assessment**: Quality discipline served users here — the verification protocol caught that the auto-navigate-to-scheduled-tasks behavior still worked correctly post-fix (one nav, not zero, not loop). A "minimal" fix that broke an intended flow would have failed the user differently.
- **Session assessment**: The minimal-fix discipline held. The agent declined to also "fix" `setActivePage` (the analysis-in-issue's third proposed change), correctly arguing it was over-engineering once the loop driver and the route-churn were both gone. CHANGELOG entry was specific (cited bd-a7l1z relationship, named both files, gave verification numbers) instead of vague.
- **What I'd flag**: Two things. (1) The first-pass code comments violated style guide: multi-line comment block in `SettingsTab.tsx`, multi-line in `useHashRoute.ts`, both with task/issue references. The agent caught and compressed these *after* the PO would have seen them in a normal review flow — that's not the shape of a self-correcting reviewer. Comments should be rule-compliant on first write. (2) The eslint-disable-next-line in `SettingsTab.tsx` is a code smell in a file that already has 4884 lines and is, according to the directory comment, due for decomposition. Each disable directive that lands raises the bar for future reviewers to argue against the next one. Tracking-bead-worthy.
- **Disagreement**: With the technical writer — I think the CHANGELOG entry is too long. It's a faithful description of the bug, but a Fixed-section bullet shouldn't require five sentences. The technical writer would say (correctly) that `bd-a7l1z` two days ago set the precedent for forensic detail in CHANGELOG entries. I think the precedent is wrong. Push back on it next time.

### QA Engineer
- **User value assessment**: The verification protocol was the right one. Priming the worst-case trigger (`taskId: '__definitely_not_a_real_task__'`, which never matches and never gets cleared) tests the actual stranded-key scenario the user is hitting, not a synthetic happy path. The 7-page navigation hammer test gave a clean signal (`historyDelta == 7`) that pre-fix would have failed obviously (tens of thousands of entries within seconds).
- **Session assessment**: Good. Used `performance.memory` for heap trend, `window.history.length` for the route-churn signal — the latter is a sharper signal than memory because GC noise drowns out small leaks but `pushState` calls are deterministic. The unit test suite (1231 tests, 18 useHashRoute tests) was treated as a regression net, not as primary evidence.
- **What I'd flag**: We did not write a regression test for this bug. The two ways to do it would be: (a) a `useHashRoute.test.ts` case asserting that `pushState` is NOT called when `setHash` is invoked with the current route, or (b) a `SettingsTab.test.tsx` integration test priming the sessionStorage key, mounting the component, and asserting that no infinite loop occurs (e.g., bounded render count). (a) is cheap and would cement the fix; (b) is more valuable but would require a SettingsTab test harness that doesn't currently exist for this component. (a) should be a follow-up bead.
- **Disagreement**: With project-engineer — I think the missing regression test is more important than the missing lint step. The lint regression cost one CI cycle. A future agent or refactor could re-introduce the loop driver under a different sessionStorage key and have no test to catch it. I'd prioritize the regression test bead.

### Technical Writer
- **User value assessment**: The CHANGELOG entry tells a future operator (or future agent) exactly why this fix was needed and what the symptom looked like. The PR description carries the same detail at PR-discoverable scope. The issue comment to the reporter is in second person and gives them a concrete action ("pull `dev`, confirm"). All three audiences served.
- **Session assessment**: Documentation discipline was followed — CHANGELOG updated as part of the ship, not after. PR body had clear sections (Summary / Root cause / Fix / Test plan). Issue comment was concise.
- **What I'd flag**: The PR description used a markdown link (`[gh #207](...)`) instead of the GitHub closing keyword (`Closes #207`), so the issue didn't auto-close on merge. The agent caught this when the PO asked about leaving a note — but if the PO hadn't asked, the issue would still be open with no notification to the reporter that the fix shipped. Add to memory: when a PR closes a `gh` issue, use the literal `Closes #N` keyword, not a markdown link to it. Markdown links are for `gh-N` references that are NOT meant to auto-close.
- **Disagreement**: With code-reviewer (see their entry). The CHANGELOG length is contentious. My position: precedent matters; this codebase has chosen forensic CHANGELOG entries (see bd-a7l1z, bd-zv6pi, bd-i75ax above mine), and a four-sentence "we fixed a render loop" entry would be visibly thinner than its peers. If we want to change the convention, do it deliberately, not by writing the first short entry against type.

### SRE
- **User value assessment**: The leak's blast radius was the user's browser tab, not server-side, so reliability instrumentation didn't catch it (and probably can't — heap profiling per browser tab is not in our observability stack). User-reported via the issue tracker. That's the failure mode SRE has to accept for client-side leaks.
- **Session assessment**: Reasonable. The bug surfaced in production via user report at v0.16.0-0064; the recharts loop in PR #208 was found internally; this loop took an extra two days because it required a specific user action (Edit-Schedule click, then a deleted/missing task) to arm. That's not a process gap, it's the long tail of state-dependent React bugs.
- **What I'd flag**: We have no client-side reliability metric for this class of bug. Two render loops in 36 hours (recharts/Stats and Settings/scheduled-tasks) both share a "fresh object literal on every render → setState → re-render" shape, and both went undetected until users reported memory exhaustion or tab crashes. A simple `window.history.length` watchdog (alarm if it grows by > N entries in M seconds without a user navigation event) would catch this whole family of bugs in dev builds. Beads-worthy: `feat/dev-render-loop-watchdog`. Cost: tiny. Coverage: high for this exact failure mode.
- **Disagreement**: None. SRE position is additive, not in tension with anyone.

### Project Manager
- **User value assessment**: Bead workflow was followed: created, claimed, closed. PR-driven shipping per `docs/shipping.md` §6 was followed. CHANGELOG updated. Version bumped (0069 → 0070). Issue commented with action for reporter. End-to-end traceability from user complaint → bead → PR → release notes → user notification.
- **Session assessment**: Tight. Single bead, single PR, one CI iteration. Total elapsed under an hour. No scope creep, no batching of unrelated work.
- **What I'd flag**: Two follow-up items surfaced and were correctly NOT folded into this PR (good!), but neither got a follow-up bead created: (a) the `ScheduledTasksSection.tsx:401-415` stale-key bug (key not cleared on non-matching taskId), and (b) QA's regression test, and (c) SRE's render-loop watchdog. The agent flagged (a) in conversation and the PR description but did not file a bead. I'd file all three.
- **Disagreement**: None.

## Lessons

- **Keep**: The "ignore the analysis, verify from code" pattern. The PO's explicit framing was the right relationship to prior LLM analysis. Agent verified the mechanism, noted where the analysis was directionally right and where its trigger claim was contingent, and proposed a smaller fix (2 changes vs. 3) than the analysis recommended. This is the shape the PO wants — agent as independent verifier, not transcription engine.
- **Stop**: Skipping `npm run lint` in the local pre-push quality gate. The CI runs it as a blocking step; the agent must too. `docs/shipping.md` §1 needs updating, but agent should have known this from the script definition itself.
- **Start**: When closing a `gh` issue via PR, use the literal `Closes #N` GitHub keyword in the PR body, not a markdown link. Markdown links don't trigger the auto-close.
- **Start**: When writing a `useEffect` with intentionally narrow deps, look for the existing pattern in the same file (or nearest-neighbor file) before placing the eslint-disable directive. `react-hooks/exhaustive-deps` emits on the dep-array line, so the directive sits above the closing `}, [...]);` line, NOT above `useEffect(() => {`.
- **Value learning**: Two render loops in 36 hours (recharts in #208, settings in #209) share the same "fresh object on every render → setState → re-render" shape. This is a recurring family, not an isolated incident. The fix to `useHashRoute.setHash` (bail-out on unchanged route) immunizes that one hook against the pattern, but the pattern lives anywhere a hook returns a fresh-allocated object from `setState`. Worth a project-wide audit.

## Follow-up beads to file (PM action)

1. `bd: regression test for useHashRoute.setHash bail-out on unchanged route` (small, tests-only)
2. `bd: ScheduledTasksSection.tsx — clear ecm:open-task-editor on non-matching taskId` (small, removes a stranded-key trap that's harmless post-#209 but still semantically wrong)
3. `bd: dev-only render-loop watchdog (window.history.length growth alarm)` (small-medium, surfaces this whole family of bugs)
4. `bd: audit hooks for fresh-object-from-setState pattern` (medium, project-wide)
5. `bd: docs/shipping.md §1 — add npm run lint to local quality gates` (trivial)
