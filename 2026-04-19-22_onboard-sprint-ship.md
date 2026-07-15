# 2026-04-19-22 — onboard — sprint — ship

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: ~100 (long session; onboarding → multi-PR sprint → merge wave)
- **ContextUsed**: ~70% (many long agent reports from onboarding + multiple scope-refinement cycles)
- **Personas Active**: Project Manager (scrum master), Security Engineer, IT Architect, UX Designer, Code Reviewer, Database Engineer, SRE, QA Engineer, Technical Writer, Project Engineer
- **Beads Touched**:
  - Created this session: `lbrhp` (onboarding), `v3xfl` (P0 security), `844yr` (P1 WAL+FK), `w0iyw` (P1 health), `0lvkk` (P1 ErrorBoundary), `t8xw3` (P0 CI gates), `8w33i` (P1 branch protection), `nmlxi` (P2 coverage), `zjge5` (P1 lint baseline), `2lw25` (P2 E2E-in-CI, reframed), `trfd0` (P2 modals fixme), `nv36s` (P1 CHANGELOG), `vighz` (P1 SECURITY.md), `ehf2q` (P2 CONTRIBUTING.md), `mghjm` (P1 release execution)
  - Closed this session: `lbrhp`, `t8xw3`, `trfd0`, `844yr`, `w0iyw`, `nv36s`, `vighz`, `ehf2q`, `v3xfl`, `8w33i`, `0lvkk` (11 closed)
  - Still open: `mghjm` (release execution, parked for tomorrow), `zjge5` (lint cleanup), `nmlxi` (coverage), `2lw25` (E2E infra), `8w33i`-follow-ups, `gb5r5` tree (v0.17.0), plus older backlog items

## Section 1: What We Did Well Together

**The Path A/B/C framing on E2E scope.** When the engineer's preflight surfaced that modals.spec.ts's 21 conditional skips weren't fixture-shaped and would need a Dispatcharr mock sidecar (10-14h lift), we could have spiraled. Instead: PM framed three options (full mock, hybrid with `test.fixme()`, pull E2E entirely), each with honest trade-offs. PO made a clean "take C" call in one message. The engineer then landed the E2E removal cleanly as a forward-only commit. That three-turn exchange converted a scope surprise into a clean decision without any architectural panic. The follow-on `2lw25` reframe preserved the institutional knowledge for future investment.

Also notable: the v3xfl defensive pass. Engineer was told to fix two specific sites; they chose to grep the whole codebase for ffmpeg/ffprobe invocations and categorize every one as safe or unsafe before writing the patch. That's exactly the "don't trust bead history at face value" discipline that matters — it prevented another `ppe28.1` partial-fix pattern.

## Section 2: What the PO Could Improve

**Sprint goal was re-stated three times with mutually inconsistent implications.** Early on, the PM proposed a "stabilization sprint" (close onboarding gaps). The PO then said **"Sprint goal: Get 0.16.0 fully released"** — a *release* framing, not a *stabilization* framing. The PM's follow-up brief correctly deferred `0lvkk` / `w0iyw` / `zjge5` out of 0.16.0 to honor the release framing. Then the PO said **"those three can be in scope"** — expanding back into stabilization territory.

Each shift forced a re-plan. The PM re-framed once for stabilization, once for release, once for release-plus. If the PO had committed upfront to "release sprint that also includes quality improvements that don't destabilize" (which is what ended up happening), the plan would have been tighter from the start.

This isn't about being indecisive — the PO is decisive and fast. It's about the *framing* being looser than the decisions. A sprint goal statement that captures both the ship-blocker and the quality-improvement scope up front would save a re-plan.

## Section 3: What the Agent Got Wrong

**I skipped worktree isolation when spawning parallel implementation agents, and it predictably caused a branch collision.** At roughly turn 60, I spawned the Database Engineer (844yr) and QA Engineer (trfd0) in parallel without `isolation: "worktree"`. Both agents operated in the same git working tree with a single `.git/HEAD`. Predictable consequence: QA's commit landed on DBA's branch mid-work because the DBA's `git checkout` had switched HEAD in between QA's `git add` and `git commit`. QA recovered via `git branch -f` pointer manipulation; DBA recovered via cherry-pick + reset. No work was lost, but the 844yr branch had an extra stray commit as parent when pushed to origin.

The Agent tool explicitly supports `isolation: "worktree"` for exactly this case. I had read the tool description but defaulted to no-isolation because my earlier parallel spawns (the read-only onboarding personas) worked fine. I didn't update my mental model when the spawns became *write-mode* (commit/push). Only after the QA report surfaced the collision did I save the lesson to memory and use isolation on the next PE spawn for PR #70.

Also second-order: I filed the three doc beads (`nv36s`, `vighz`, `ehf2q`) myself before spawning the Technical Writer, after the PO had already corrected me once earlier that session ("engineers do the work"). Needed the correction twice before internalizing the pattern that personas own their domain work including bead creation.

## Section 4: What Would Make the Project Better

**The 71 existing ESLint errors + 98 warnings are grandfathered but un-scheduled.** `t8xw3` wired CI gates that enforce changed-files-only lint on PRs — new violations block, existing ones pass. The cleanup is tracked in bead `zjge5` (P1) but has no owner, no target sprint, no timeline. While it sits, every new PR that touches a file with pre-existing violations has to either fix them (scope creep) or leave them untouched (tacit acceptance). PR #70 hit this exact tension — the engineer had to add a file-scope `eslint-disable` on `App.tsx` to get green, with a TODO pointing at `zjge5`. If `zjge5` doesn't land within 1-2 sprints, that disable pattern will spread. The project would benefit from `zjge5` getting a hard deadline (e.g., "must land before v0.17.0 cut") rather than an open-ended P1 tag.

Related: the 3-overlapping-architecture-docs issue (Tech Writer flagged twice during onboarding and CONTRIBUTING.md work) is still not beaded. Documentation debt accumulates.

## Section 5: Persona Perspectives

### Security Engineer
- **Session assessment**: Got heard. `v3xfl` was triaged as P0, implemented cleanly with a defensive grep pass + 3 regression tests verifying `-protocol_whitelist` is present in both invocations. The `ppe28.1`-overstated-completion pattern was exactly what I flagged during onboarding, and the engineer verified they didn't just trust bead history.
- **What I'd flag**: `python-jose` transitive `ecdsa` CVE-2024-23342 is documented in the CI `pip-audit` ignore list as "unused" (HS256 path uses cryptography backend). That's verified at the current snapshot but isn't actively monitored. If a future dependency change pulls `ecdsa` into the HS256 signing path, nobody is watching. Small risk, but no owner.
- **Disagreement**: With PM, who scoped `2lw25` (Dispatcharr mock / E2E-in-CI) as pure QA hygiene. E2E that exercises authn middleware is a security-relevant gate too. A middleware regression between now and whenever `2lw25` lands goes uncaught until a user hits it.

### IT Architect
- **Session assessment**: Onboarding surfaced the Alembic gap (60+ hand-rolled `ALTER TABLE` statements in `database.py`). I recommended adopting Alembic before the next schema change. That recommendation is NOT yet in a bead. Migration debt keeps compounding.
- **What I'd flag**: `844yr` added the PRAGMA event listener — good defensive move — but the fundamental migration mechanism is unchanged. Every new column still goes through `init_db()`'s imperative branch. This is a "cost doubles every 6 months" problem. Put a bead on it before v0.17.0.
- **Disagreement**: With PM, who deferred `2lw25` (Dispatcharr mock) because "not release-blocking." Architecturally, the mock isn't just a testing convenience — it's the validation harness for our integration boundary. Without it, every Dispatcharr API change (like the 0.23.0 auth compat we handled in v0.16.0-0037) is caught by users, not CI.

### Project Manager
- **Session assessment**: Strong shipping velocity — 6 PRs merged, 11 beads closed in one session, release-execution bead fully documented for tomorrow's continuation. The Path A/B/C framing on E2E + the bench-activation brief got adopted and executed.
- **What I'd flag**: Velocity-at-any-cost was the mode today. `zjge5` is P1 but unowned. `2lw25` is deferred indefinitely. `gb5r5` (v0.17.0) is staged but not committed. Backlog is growing faster than it's closing. That's fine for one session but flags a grooming-pass need before starting v0.17.0.
- **Disagreement**: With Code Reviewer on the file-scope `eslint-disable` in `App.tsx`. CR thinks it's a code smell worth blocking the merge over. From my lens, it's a tracked, time-bounded compromise (PR #70 grandfathered warnings that belong to `zjge5`, with a TODO comment) — merging and shipping the ErrorBoundary beats blocking on style cleanup that has its own bead. Ship the value, park the style.

### Project Engineer
- **Session assessment**: Multi-cycle context switching — `t8xw3` (3 iterations including preflights) → `v3xfl` → `8w33i` → `0lvkk` → CI debug round for #70 → PR scrub work. Worktree isolation learned the hard way on the second wave. Pre-flight discipline continues to pay off — two different preflights surfaced scope-altering findings (ajv override, modals architecture) that kept beads honest.
- **What I'd flag**: Scratch branches aren't being cleaned up. `ci/test-t8xw3`, `security/v3xfl-protocol-whitelist`, `db/844yr-wal-foreign-keys`, `ux/0lvkk-error-boundary`, `sre/w0iyw-deep-health`, `test/trfd0-modals-fixme`, `docs/release-foundation`, `docs/shipping-branch-protection-note` all still exist on origin post-merge. Branch hygiene pass needed, especially before any future PR work that might accidentally push to a stale branch.
- **Disagreement**: With QA on the `test.fixme()` choice for modals.spec.ts. `test.fixme()` semantic is "broken, should work but doesn't" — implying the test itself has a bug. What's actually true is "this precondition isn't seeded in our CI environment." A tagged `test.skip()` with a TODO comment referencing `2lw25` would have been more semantically accurate. Fixme will fire when 2lw25's sidecar finally seeds the state; if the test body itself has a latent issue, we won't know until then.

### UX Designer
- **Session assessment**: Not explicitly consulted for `0lvkk`. The PE implemented the ErrorBoundary with sensible engineering defaults (centered card, reload button, theme-token colors). No UX-polish-follow-up bead was filed. The bead description had said "UX Designer (spec) + Project Engineer (exec)" — a shortcut was taken.
- **What I'd flag**: The ErrorBoundary fallback is *functional*, not *designed*. Copy ("An error occurred"), icon (none), instrumentation (console.error only), offer-to-report behavior (none) are all engineering defaults. A real designed error state would cover: what error class this is (network vs. render vs. business logic), what the user can try (reload, report, contact), what state is preserved. Worth a polish pass post-release.
- **Disagreement**: With PE, who handled `0lvkk` solo. The bead explicitly carved out a UX spec role. Shortcut was functionally fine; UX-wise we shipped less than the bead promised.

### Code Reviewer
- **Session assessment**: CI gates now enforce TypeScript + changed-files-only ESLint + `--max-warnings 0`. That's a huge baseline-lifting win. But the file-scope `/* eslint-disable react-hooks/exhaustive-deps */` at the top of `App.tsx` to unblock PR #70 is a mixed signal. It's correct that those 9 warnings belong to `zjge5`. But a file-scope disable is *less reviewable* than 9 per-line disables with per-reason comments.
- **What I'd flag**: The file-scope-disable pattern is being set as precedent. If `zjge5` doesn't land soon, future engineers will read `App.tsx`'s header and conclude this pattern is acceptable. Either fix `zjge5` within the next sprint OR add a comment policy that discourages file-scope disables in other files.
- **Disagreement**: With PM on the above — PM says ship-it-park-style, I say the pattern cost is higher than one compromise. Compromise: ship `0lvkk` but require that `zjge5` gets a hard deadline attached.

### Database Engineer
- **Session assessment**: `844yr` landed cleanly. WAL + synchronous=NORMAL + foreign_keys=ON on every SQLite connection. One latent test bug surfaced (`test_persist_changes` was inserting orphan FK rows) — fixed inline. 7 new tests. The worktree collision with QA was recovered via cherry-pick + reset.
- **What I'd flag**: We enabled WAL but have zero monitoring on WAL checkpoint behavior. Checkpoints happen implicitly or via the `wal_checkpoint(TRUNCATE)` call in `backup.py`. Under write-heavy load, WAL files can grow. SRE's new `/api/health/ready` should eventually include a check on `PRAGMA wal_checkpoint` return values. Not blocking; future improvement.
- **Disagreement**: Mostly aligned. I'd note: agreement with SRE that the health endpoint deepening was the right call. Shared interest with Architect in Alembic — if the architect's Alembic bead lands, I want a review seat.

### SRE
- **Session assessment**: `/api/health/ready` landed with the right layered approach — cheap liveness (`/api/health`) stays untouched for the Dockerfile HEALTHCHECK; rich readiness (`/api/health/ready`) does the subsystem verification. Dispatcharr ping is cached for 30s to avoid hammering. ffprobe check reuses existing helper. Good.
- **What I'd flag**: No Prometheus `/metrics` endpoint still. For operators who want to monitor ECM with Grafana (home-lab power users, exactly our user base), the only health signal is parsing logs. The onboarding flagged this as a potential improvement; no bead was filed. Worth filing for v0.17.0.
- **Disagreement**: With Architect on priorities. Architect wants Alembic (dev-time migration tooling). I want `/metrics` (ops-time observability). Both are P2-ish but we should pick one to advance first; I'd pick `/metrics` because the audience (operators running ECM today) is larger than the audience for schema migrations (ECM maintainers, solo team).

### QA Engineer
- **Session assessment**: `trfd0` landed cleanly — 21 conditional `test.skip()` converted to `test.fixme(condition, message)` with references to `2lw25`. Silent green is dead on that file. CI test workflow is now live.
- **What I'd flag**: Full-repo lint surfaces 169 problems (71 errors + 98 warnings). CI only gates on changed-files in PR mode. New code that introduces similar patterns but touches a clean file won't be caught — it'll become grandfathered the moment it merges. Suggests the full-lint should migrate from `continue-on-error: true` to blocking within a reasonable window (say, after `zjge5` knocks the baseline to zero).
- **Disagreement**: With PE on the `test.fixme()` vs. `test.skip()` debate. I picked fixme deliberately because `test.skip()` was the status quo and was SILENT. `test.fixme()` is LOUD — it fails the run. That loud-failure is what breaks the silent-green antipattern. If the test body is latent-buggy, we find out at the right time (when `2lw25` seeds state). Worth the semantic imperfection.

### Technical Writer
- **Session assessment**: Three deliverables landed — CHANGELOG pointer, SECURITY.md with GHSA disclosure, CONTRIBUTING.md for humans. v0.16.0 release notes drafted and attached to bead `nv36s` for release-day paste. The "Why + How to apply" pattern I flagged during onboarding got reinforced in the new docs.
- **What I'd flag**: The 3-overlapping-architecture-docs problem (`architecture.md`, `backend_architecture.md`, `project_architecture.md`) is *still not beaded* despite being flagged during onboarding AND during the CONTRIBUTING.md work. CONTRIBUTING.md links `project_architecture.md` as canonical but the other two still exist and will drift. Somebody needs to own the consolidation.
- **Disagreement**: With PE on `8w33i`'s `docs/shipping.md` note — functionally correct but spatially isolated. A reader hitting the shipping doc sees the branch protection section in context, but a reader coming from README.md or CONTRIBUTING.md has no path to discover it. Cross-reference would have cost two lines and helped discoverability.

## Section 6: Lessons for Future Sessions

- **Keep**: Engineer preflight discipline — `t8xw3` and `v3xfl` both caught scope-altering findings before committing work. The "stop and report" contract is high-value.
- **Keep**: Path A/B/C scope-decision framing — pose three concrete options with trade-offs when an assumption fails. The PO closed the E2E question in one message because the options were clear.
- **Keep**: Defensive sweeps — the v3xfl engineer's whole-codebase grep for ffmpeg/ffprobe invocations caught nothing new but proved we'd covered the whole surface. That's the pattern that prevents `ppe28.1`-style partial fixes.
- **Stop**: Spawning parallel implementation agents without `isolation: "worktree"`. One shared `.git/HEAD` means branch collisions. Saved to memory.
- **Stop**: Filing domain-specific beads before delegating to the owning persona. Two corrections in one session on this. Saved to memory.
- **Start**: Explicit sprint-goal commitment at the top of any planning session. If the PO says "release" but quality improvements creep in, the PM should force a single commitment before planning.
- **Start**: Scratch-branch cleanup as a post-merge ritual. Leaving 8 scratch branches on origin is low-cost but cumulative noise.
- **Start**: A "mghjm-style" release-execution bead template for future releases — this session's bead is reusable as a checklist for 0.17.0 and beyond.

## Worktree-isolation issue called out by the PO

The PO specifically flagged the worktree-collision issue for this retro (their words: "That should come up during our retro"). Captured in Section 3 and in the Lessons list. Already saved to memory as `feedback_worktree_isolation.md`. This should be the baseline for parallel agent spawning going forward.
