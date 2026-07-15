# 2026-04-23-17 — bulk-edit-audit

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: ~75 user + assistant exchanges (heavy on CI/task-notification interleave)
- **SessionDepth**: deep — standup triage across 10 personas, full-history audit of PR #99 (5 review rounds traced), 6 PRs opened and merged, sustained CI-incident-response through an ongoing GitHub infra event
- **Personas Active**: Project Engineer (heavy — 6 agent spawns), Project Manager (standup RED), Security Engineer (M1 ReDoS finding + standup YELLOW), IT Architect (standup YELLOW, ADR-007 advocacy), Database Engineer (standup YELLOW, Stats v2 retention shape), SRE (CI flake triage, 42aai bead), QA Engineer (TDD discipline across 6 PRs), Code Reviewer (implicit, PR review discipline), Technical Writer (standup green), UX Designer (standup green)
- **Beads Touched**: `q7maj` (created+closed, review tracking), `k41e0` P1 (M1 ReDoS bypass — closed via PR #127), `z7xqy` P2 (M3 validate_rule trap — closed via PR #128), `gjoe5` P3 (M2 silent-ignore — closed via PR #131), `bh1hh` P3 (S1 N+1 — closed via PR #129), `91mcq` P3 (S2 audit-trail — closed via PR #132), `42aai` P2 (CI retry — open for 2-week observation window)

---

## Section 1 — User Value Delivered

Substantial, mostly shipped:

- **PR #99 (bulk-edit auto-creation rules) merged.** Users who manage many auto-creation rules can now multi-select and apply per-section changes in one operation instead of editing rules one by one. This was the feature the contributor (@stevencoutts, external) had iterated on through 5 review rounds; shipping it closed the "goalpost-moving" pattern the PO had flagged.
- **PR #127 (M1 ReDoS bypass fix) merged.** The bulk endpoint now rejects catastrophic-backtracking `sort_regex` patterns the same way the single-rule PUT has since bd-eio04.7. Closes a real ReDoS vector amplified to 500-rule blast radius. Users are protected from a potential DoS surface an attacker (or careless operator) could have planted at scale.
- **PR #128 (M3 scalars-only bulk skips validate_rule) merged.** Removes the latent footgun where bulk-toggling `enabled` on 50 rules could 400 + rollback if any one row had pre-existing schema-drifted conditions. Direct operator UX benefit.
- **PR #131 (M2 reject conditions/actions in bulk payload) merged.** Silent-drop of request fields eliminated — integrators now get 422 with a clear error instead of 200 OK with silently-discarded data. API contract hardening.
- **PR #129 (S1 N+1 → single fetch) merged.** 500-rule bulk-update went from ~1000 SELECT roundtrips to 1. Users see faster bulk operations; operators get all missing IDs reported in 404 instead of aborting on the first.
- **PR #130 (S2 CI docker-push retry) merged.** Infra hardening against the GHCR 50x flakes we hit 4 times today. Bead `42aai` stays in_progress for a 2-week observation window per the engineer's acceptance metric. Users benefit indirectly (faster time-to-production for future fixes).
- **PR #132 (S2 per-entity audit-trail) merged.** Bulk updates now emit per-rule journal entries with shared `batch_id` + before/after diffs, matching the pattern at `backend/routers/channels.py:800`. Admins and incident responders can answer "what did bulk-update do to rule 47 at 14:32?" — a forensic capability that was blind before.
- **Standup surfaced flow-control failure (PM RED).** Both user-facing epics (v0.18.0 Backup/Restore, v0.17.0 Stats v2) had ready foundation work with no pickup and 24h of zero activity. That surfacing didn't ship user value directly but raised the right decision to the PO.

One caveat: none of the listed work moved either v0.18.0 or v0.17.0 forward. We cleaned up the bulk-edit PR and its debris. Whether that cleanup qualifies as user-visible value or technical-debt maintenance is a judgment call — the PR #99 feature itself is user-facing, the 5 follow-up fixes are quality-of-implementation.

---

## Section 2 — What We Did Well Together

The single best exchange: **the PO's "we seem to keep moving the goalposts"** prompt at turn ~20 (after I delivered the initial engineer review of PR #99 and proposed 4 new must-fixes). That instruction pushed me to fetch the full PR history and trace every round — which caught 4 distinct goalpost moves from R1 through R5, AND my own Round 7 repetition of the pattern (re-raising N+1 and audit-trail as "should-fix" when R1 had explicitly classified them as non-blocking follow-ups). The honest self-critique in that response — naming which of my findings were legitimately new vs. which were me repeating the pattern — set the frame for the collaborative recovery path: **ship #99, fix-forward the one real security finding (M1), defer everything else to beads**. Instead of a 6th review round, we shipped PR #99 and opened 5 scoped beads. One prompt from the PO turned process slippage into a clean backlog.

---

## Section 3 — What the PO Could Improve

Two things, ordered by impact:

**1. The "1-4 are fine" response was ambiguous on merge authority.** When I laid out the 5-item post-review plan (merge #99, file beads, check M3 data, fix-forward M1, apologize), the PO said "1-4 are fine, we don't need to apologize with #5." But item 2 ("File bd immediately for M1 and do a fix-forward PR") is two actions — open a PR vs. open + merge. I interpreted as "open the PR" only. When PR #127 went green I awaited explicit merge direction, which came much later as a separate "Merge it" instruction. Later when we were parallelizing 4 engineers, the PO batched "Merge it" again after all 4 PRs landed. That's fine, but at the #127 stage it cost a 15-minute gap where I was waiting on a merge decision that the PO might have already implicitly authorized in "1-4 are fine." Clearer would be: "approved — merge each when green" or "approved — hold each for my explicit call." Explicit per-PR authority.

**2. The "engineers do work, not you" correction exposes a framing gap in the skills, not a PO failure.** At turn ~50, when PO approved option 1 for the M3 fix, I started writing the TDD test file and editing `backend/routers/auto_creation.py` directly. PO intervened: "No. engineers do work. Not you." The correction itself was clean and direct — zero time wasted once it happened. It was the second time this session I'd blurred the line (the first was PR #127 where I did the code change directly after the initial engineer review, and the PO didn't object).

**Investigation follow-up:** the PO pushed back on this retro item and asked "is it not a part of our rules, defined in the skills?" I checked:

- `~/.claude/skills/project-engineer/identity.md` line 3: *"While others draw boxes and write findings, you make it actually work for users."*
- `~/.claude/skills/project-engineer/SKILL.md` frontmatter: `when_to_use: implementation, coding, infrastructure deployment, ...`
- `~/.claude/skills/_shared/engineering-discipline.md` line 27: *"Do not interleave actions. Do not act on a partial frame."*
- `~/.claude/skills/_shared/conflict-resolution.md`: domain-authority model — each persona has authority within their domain.
- `~/.claude/CLAUDE.md`: empty (0 bytes).
- Project `CLAUDE.md`: focuses on beads/shipping/dev-container, no orchestrator-vs-engineer framing.

The rule is **architecturally inferable** (every persona has a skill; the engineer's skill is "implementation, coding"; therefore Claude-as-orchestrator should spawn the engineer for code work) but **not explicitly directive**. No text in any skill file, shared doc, or CLAUDE.md says *"Claude does not implement code — spawn the project-engineer persona for any code change."* A fresh session without that frame will default to "just do the small change" for anything that feels below the spawn-overhead threshold. That's what I did at turn ~50.

**Corrected claim:** my initial retro framing ("framed up front would have prevented the slip") was correct about the **effect** but wrong about whether the rule **exists**. The rule is structurally present. The **explicit framing** is what's missing. Candidate fix: a short section in `~/.claude/skills/_shared/engineering-discipline.md` (or a new sibling `orchestration.md`) stating *"Claude orchestrates — personas implement. For any code change, spawn the project-engineer agent. No line-count exceptions."* That turns an architectural inference into a hard directive, and the PO's correction at turn ~50 stops being a recurring ritual.

---

## Section 4 — What the Agent Got Wrong

**1. Goalpost repetition in the initial PR #99 engineer review (turn ~17).** The review added M1/M2/M3 as "must-fix" (M1 ReDoS was legitimately new; M2 silent-ignore was a real find; M3 validate_rule trap was a real find) — but also re-raised N+1 query and audit-trail pattern as "should-fix" items. Round 1 of the PR review (by the same owner, earlier) had **explicitly** classified both as non-blocking follow-ups. I repeated the exact pattern the PO was frustrated about. The prior reviews were readable via `gh api` — I should have pulled them before writing my review, not after the PO asked me to. **Cross-check findings against prior review record before submitting** is the operational lesson.

**2. `gh run rerun 24840698133 --failed` orphaned the required CodeQL Analysis check-runs (turn ~28).** At PR #99's first registry 504, I ran `gh run rerun --failed` to retry only the failed Docker push jobs. That re-queued the workflow in a state where the previously-passing `CodeQL Analysis (python)` and `CodeQL Analysis (javascript-typescript)` check-runs were dropped from branch protection's view (they passed on the old attempt but weren't re-registered on the current SHA because they hadn't failed). Combined with the workflow getting stuck in `queued` state, the PR became unmergeable despite the visible `gh pr checks` output showing everything green. I escalated to the PO with "3 ways forward" when in fact a simple `close+reopen PR` (my step 3 later) would have unstuck it. Cost: ~30 minutes of incident response + confusion. **Understand tool semantics before using them on the critical path.**

**3. Started M3 patch myself at turn ~50.** After the engineer agent diagnosed M3 (3 rules, 0 invalid) and proposed the code change, I started writing the regression tests and editing `auto_creation.py` directly — wrote `TDD test-first` content and was mid-edit when the PO stopped me with "engineers do work. Not you." Correct call on their part; I backed out, deleted the unused branch, and spawned a fresh engineer. But the slip shows I defaulted to "competent implementer" mode instead of "orchestrator" mode when the path felt small enough to just do. For small changes the cost of spawning is higher than just doing — but the discipline isn't about cost, it's about the persona firewall. I should have held the line.

---

## Section 5 — What Would Make the Project Better

**CI checkout-step retry.** We hit 3 separate `git checkout` HTTP 500s from github.com during the session (PR #130, PR #131, PR #132). PR #130 added retry on docker push (the GHCR side) but not on the `actions/checkout` step itself. Today's GitHub infra incident exposed that the checkout step has no retry wrapper. Candidate follow-up bead: wrap `actions/checkout@v4` with `nick-fields/retry` in all workflows, OR evaluate `actions/checkout@v5` (if released) for built-in retry semantics. Cost: one workflow touch + observation period. Benefit: future sessions during GitHub incidents don't spend 20+ minutes on rerun ritual.

**Refactor bulk-update tests away from `validate_rule` mocking.** The existing `test_sets_merge_streams_remove_non_matching` mocks `validate_rule` to always return valid. That mock pattern is exactly what would have hidden M3 — the reason `z7xqy` existed at all. Two engineers (PR #128 and PR #132 authors) flagged this as scope-controlled; neither refactored the existing mocks. It deserves a P3 coverage-hardening bead: remove `validate_rule` mocks from bulk-update tests where the behavior under test doesn't require them. Without that refactor, the next bulk-handler bug will also evade these tests.

---

## Section 6 — Persona Perspectives

### Security Engineer
- **User value assessment**: Real user protection shipped. PR #127 closed a ReDoS vector amplified to 500-rule blast radius — an attacker with auth could have planted a catastrophic-backtracking pattern on up to 500 rules per request that the single-rule PUT would have rejected. The fix is genuine defense, not compliance theater.
- **Session assessment**: My initial review of PR #99 caught M1 (ReDoS bypass) that three prior reviewers missed across 5 rounds — the pattern that bd-eio04.7 existed specifically to enforce was being bypassed in the new bulk endpoint. That's a real win for writeup-hygiene.
- **What I'd flag**: The goalpost-moving pattern the PO surfaced has a security implication. If reviewers keep adding must-fixes round after round, external contributors get fatigued and stop contributing. That's a supply-chain signal in miniature: an unfriendly review culture pushes work out to forks, which we then don't see. Keep reviews bounded.
- **Disagreement**: None with other personas this session. One quiet disagreement with the PO: I think M3's defense-in-depth patch was worth doing even though the prod DB had 0 invalid rows. The engineer's framing ("schema evolution arms the footgun overnight") is correct — we shouldn't trust today's data-quality state to be tomorrow's.

### IT Architect
- **User value assessment**: Mixed. The standup correctly surfaced that ADR-007 (retention policy for session_telemetry) is the gatekeeper for Stats v2 — 6 user-facing beads blocked on a scoped decision that hasn't been made. That diagnostic surfaced the right PO decision. But no architectural work actually shipped this session; we were firefighting PR #99 debris, not advancing architecture.
- **Session assessment**: The proposed ADR-007 shape (30-day raw / 90-day hourly / 365-day daily, pg_cron + materialized views) from my standup Phase 2 was concrete and actionable. It sat in the retro-writeup instead of being picked up. Architecture ready work not being pulled is the pattern.
- **What I'd flag**: PR #99's `bulk_update_auto_creation_rules` handler kept accumulating orthogonal concerns (N+1, audit trail, validation skip, regex lint, reject conditions/actions). That's five cross-cutting changes to a ~50-line function. At this rate we should extract the handler into a service class with a clearer per-concern boundary. Not urgent, but the next iteration shouldn't land as a sixth bolt-on.
- **Disagreement**: The DBA proposed `pg_cron` for Stats v2 rollups; I'd push back on adding a new Postgres extension dependency without an ADR first. `pg_cron` is fine, but it's a deployment-environment question (managed-PG compatibility, extension allowlisting, operational familiarity) that deserves explicit trade-off. App-side APScheduler may be a simpler default.

### Project Manager
- **User value assessment**: Net-positive. PR #99 shipped direct user value (bulk edit UX). PRs #127–#132 are quality-of-implementation on that same feature. Neither Backup/Restore (v0.18.0) nor Stats v2 (v0.17.0) moved — the two major user-facing epics flagged RED in standup stayed at 0h of progress this session. A full day of engineering capacity went into one feature plus its debris. That's a delivery-pace concern.
- **Session assessment**: WIP management improved mid-session — parallel 4-engineer work with worktree isolation was efficient, ~400K tokens across agents delivered 4 PRs in under 30 minutes of wall-clock. That's the pattern to repeat. But the two in-progress process beads (ADR-004, ADR-005) are still in-progress after the session — neither moved and the standup flagged them as non-user-facing WIP.
- **What I'd flag**: The contributor @stevencoutts did 5 review rounds on PR #99. If we want external contribution (we do — it's how PR #113 and #99 both came in), the review cost-to-merge ratio matters. This PR was substantively complete at R2 (after 4 must-fixes addressed) and we kept finding new must-fixes. That's process debt billed to the contributor.
- **Disagreement**: With the Security Engineer — I think the M1 ReDoS finding, while legitimate, should have been a post-merge fix-forward bead from the start. Security fought for pre-merge gating on `kgjci` (ADR-005 CodeQL) + `0i2vt.1` (secret redaction) pre-merge in standup; I think that same "pre-merge or post-merge?" test should apply to M1. We chose post-merge for M1 and it worked fine. Consistency matters.

### Project Engineer
- **User value assessment**: Real value delivered. Six PRs, all green, all tested, all merged within the session. TDD discipline held across every fix-forward (PRs #127, #128, #131, #129, #132) — each had explicit Red→Green evidence in the engineer report. The prod data check (M3) caught zero invalid rows but surfaced that the latent bug was architectural, not just data-dependent. The engineer for 91mcq discovered `batch_id` already existed in the journal schema — avoided a bogus migration. The engineer for 42aai rejected `nick-fields/retry@v3` + `Wandalen/wretry.action` in favor of native YAML retry to avoid new supply-chain surface on a CI path that builds production images. Solid judgment calls from the persona.
- **Session assessment**: The parallel-worktree pattern worked clean. Three of four PRs that shared `backend/routers/auto_creation.py` merged in conflict-free sequence (#131 before the handler-body touchers, then #129, then #132 after a clean rebase). The one rebase (#132) was handled by a fresh engineer agent and came back with `209 insertions, 31 deletions` matching the pre-rebase diff shape — no scope widening.
- **What I'd flag**: The existing `test_sets_merge_streams_remove_non_matching` mocks `validate_rule` unconditionally. That mock pattern hid M3 for months and two engineers this session flagged it but didn't refactor it. Next time we touch bulk-update logic, the mock-removal should be part of the work, not a separate bead. Coverage gaps accumulate faster than we file beads for them.
- **Disagreement**: With the PO's "engineers do work, not you" rule — yes, for sustained work. But for a 15-line patch (M1 PR #127), the agent-spawn overhead costs more than it saves. I'd carve out an exception: mechanical edits under ~20 lines with an explicit test-first plan can be agent-direct. Anything larger spawns a persona. The pure rule was easier to enforce but sub-optimal for small changes.

### UX Designer
- **User value assessment**: Silent but positive. PR #99's UX (multi-select, "Apply …" section toggles, bulk-edit modal) shipped as designed. No UX regressions filed. PR #129's behavior change (404 now lists all missing IDs instead of first) is better UX for operators.
- **Session assessment**: Didn't participate actively — this was all backend + infra work. But two standup-flagged UX beads (0i2vt.19 logo-miss red banner, 0i2vt.20 restore-complete summary, skqln.6 Users panel) stayed blocked. No movement on any user-visible UX this session.
- **What I'd flag**: Bulk-edit modal doesn't surface "values differ across selection" when the user has selected heterogeneous rules. Classic multi-edit UX challenge. The #128 engineer noted it as a follow-up bead candidate; nobody filed it. Silent overwrite of divergent values on Apply is a real usability risk as user rule-counts grow.
- **Disagreement**: None with other personas; one quiet concern about PR #99's 412-line `BulkRuleSettingsModal.tsx` not getting a full UX review before merge.

### Code Reviewer
- **User value assessment**: Quality standards held. TDD evidence on every fix-forward, clear commit messages, proper PR descriptions, beads referenced. Code quality served user reliability (the bugs fixed were real user-impact latent issues) rather than aesthetic churn.
- **Session assessment**: The initial PR #99 review from Round 1 (by the PO as owner) was thorough — multi-persona triage with specific file:line references and a non-blocking follow-ups section. That's the pattern. My Round 7 review (the engineer-delegated one) was less disciplined — it re-raised items R1 had already classified as non-blocking without acknowledging that prior classification. The goalpost move was mine to own.
- **What I'd flag**: Commit messages across the 6 PRs merged today are consistent with repo convention (`fix/feat/perf/ci(scope): subject (bd-id)` + Co-Authored-By). Good. The PRs reference beads in the body, and the bead close-on-merge discipline held. But only PR #99 had a CHANGELOG entry — the 5 fix-forwards did not. If we ship in isolation that's fine; if v0.18.x includes all 6, CHANGELOG needs retroactive reconstruction.
- **Disagreement**: With the Project Engineer's exception for sub-20-line patches. No — the discipline value is in the persona firewall, not the line count. Even a 1-line fix goes through an engineer agent if we've committed to that rule.

### Database Engineer
- **User value assessment**: No DB work shipped this session. The ADR-007 retention shape I proposed in standup Phase 2 (3-tier lifecycle: monthly-partitioned raw + hourly rollups + daily rollups) sat unpulled. Stats v2 still blocked.
- **Session assessment**: PR #129's N+1 → `.in_()` refactor is good DB hygiene — at 500 rows, 1000 roundtrips is a real p95 regression on SQLite. The engineer verified `AutoCreationRule` has no server_default / DB triggers / computed columns before dropping `session.refresh()`, which is exactly the check you want. Clean.
- **What I'd flag**: PR #132 added per-entity journal entries for bulk-update, which means a 500-rule bulk-update now writes 500 rows to `journal_entries` in one commit. At current auto_creation_rules.count = 3, this is a non-issue. At a larger install it's a proportional write amplification. Worth watching — probably file a bead to cap bulk-journal batch size if we ever see `auto_creation_rules.count > 100` in a support instance.
- **Disagreement**: With the Architect on `pg_cron` — we're on SQLite, not Postgres. That whole Stats v2 rollup discussion assumed Postgres; we need to validate whether the deployment target is migrating to Postgres or if Stats v2 rollups will run app-side under SQLite. That's a base-assumption check neither of us made in standup.

### SRE
- **User value assessment**: Positive, indirectly. PR #130 (CI docker-push retry) reduces operational toil when GHCR burps — I was the persona who proposed the retry bead (`42aai`) after seeing 4 GHCR 50x flakes in 30 minutes. The 2-week observation window I wrote into the bead description kept the bead open post-merge, which is correct — "merged" ≠ "validated" for observational fixes.
- **Session assessment**: CI incident-response was rough but survivable. ~30 minutes lost to the `gh run rerun --failed` orphaned-check-runs issue. The close/reopen workaround eventually unstuck it. No production incident, no user-visible degradation — the cost was entirely in engineer wall-clock and PO coordination.
- **What I'd flag**: We have no runbook for "GH infra flake on a pending merge." This session we discovered 5 different recovery paths (`gh run rerun --failed`, `gh run rerun`, `gh api update-branch`, close+reopen PR, `--admin` override with `enforce_admins: true` caveat). Next time, whoever hits this spends the same 30 minutes rediscovering. Candidate bead: `docs/runbooks/ci-gh-infra-flake.md` capturing the actual recovery tree.
- **Disagreement**: None this session — my findings aligned with the Project Engineer's 42aai implementation. One quiet concern: the retry mechanism in `build.yml` retries on any failure, not just the GHCR 50x error signature. That widens the retry blast radius — a real code failure in docker build also gets 3 attempts before surfacing. Probably acceptable for the observation window, worth tightening later.

### QA Engineer
- **User value assessment**: Test quality served users well. Every fix-forward had Red→Green TDD evidence. The M3 tests specifically exercise the real `validate_rule` path (no mocking), which is the right move — the coverage gap that hid M3 was `validate_rule` mocking. PR #132's 3 new tests exercise real journal call sequences. No aspirational tests, no coverage padding.
- **Session assessment**: TDD discipline held across all 6 agents spawned. The "write tests, watch them fail, apply code, watch them pass" pattern is the baseline. Engineers who weren't doing TDD would have shipped broken PRs in the parallel batch.
- **What I'd flag**: 2 pre-existing test failures exist in `test_alembic_baseline.py` + `test_health.py::TestSchemaVersionEndpoint` on unchanged dev. Engineers correctly identified them as not-their-regression but nobody filed a bead. If pre-existing failures accumulate we lose signal on real regressions. Candidate P3 bead: investigate + fix or mark-expected-failure those alembic tests.
- **Disagreement**: With the Project Engineer's exception for sub-20-line patches — even a 1-line fix needs its test. Line count isn't the quality gate; behavior coverage is. PR #127 was 4 lines of code + 37 lines of test; that ratio is correct.

### Technical Writer
- **User value assessment**: Minimal documentation delivered this session. PR #99 included CHANGELOG entry + docs/api.md row (good). The 5 fix-forwards did not. If we release v0.18.x containing all 6 merges, CHANGELOG will need retroactive reconstruction — that's documentation debt created by not writing it at merge time.
- **Session assessment**: I wasn't asked to participate. The runbook I'd have wanted to write (CI GH-infra-flake recovery) wasn't written. The ADR-007 retention shape proposed in standup wasn't drafted into an actual ADR document.
- **What I'd flag**: `docs/shipping.md` mentions a CHANGELOG discipline but the 5 fix-forwards this session bypassed it. Either fix-forward PRs get a CHANGELOG exemption (and the shipping doc should say so) or we're systematically underdocumenting post-merge fixes.
- **Disagreement**: With the PM's "we shipped 6 PRs today" framing — we merged 6 PRs. CHANGELOG coverage for 1 of 6 means from a user-documentation perspective we shipped 1 feature plus 5 undocumented changes. Those are different delivery claims.

---

## Section 7 — Lessons for Future Sessions

- **Keep**: Cross-checking new review findings against prior review history before submitting. When the PO asked "go through the history" it exposed goalpost repetition in my own review. Making that check pre-submission should prevent the pattern entirely.
- **Keep**: Parallel engineer work with `isolation: worktree` for orthogonal-scope beads. Four engineers delivered four PRs in ~30 minutes of wall-clock. Worktree isolation plus explicit "you may need to rebase against other engineers' merged PRs" briefing produced zero cross-contamination.
- **Stop**: Running `gh run rerun --failed` on workflow runs with mixed pass/fail jobs unless I've verified the semantics. It orphans previously-passing required check-runs from branch protection's view. Either rerun the whole workflow or understand check-run re-registration rules first.
- **Stop**: Doing code changes directly when the PO has engineers available. Even 15-line patches should go through `project-engineer` spawn — the discipline value is in the persona firewall, not time-per-change. PO correction at turn ~50 was valid; drop the line-count exception.
- **Start**: Briefing the first engineer agent of a session with the full engineer-does-work framing explicitly. "For any code change, I spawn the project-engineer agent." Establishes the rule so later slips self-correct without PO intervention.
- **Start**: Adding a CHANGELOG check to every fix-forward PR description. If we don't make CHANGELOG part of "fix-forward definition of done," documentation debt accrues at merge velocity.
- **Value learning**: Bulk operations multiply the severity class of latent bugs. The single-rule PUT had the same validate_rule trap as bulk-update, and it never generated a support ticket at 1-row blast radius. At 500-row blast radius, the same code path becomes a visible operator footgun. Future features that introduce bulk-variants of existing behavior deserve an extra look at which previously-acceptable latent behaviors become user-visible at scale. The standup-flagged "blast radius from 1 to 500" language captures it.
- **Value learning**: When a goalpost pattern gets called out by the PO, stop the current review cycle and audit the review record. The cost of pausing to self-check is small; the cost of shipping a 6th round of must-fixes on a PR that was substantively complete at round 2 is a contributor we lose.
