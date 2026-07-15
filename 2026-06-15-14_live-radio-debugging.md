# 2026-06-15-14 — live-radio-debugging

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~55 (deep, multi-day-feeling single session; ~22 PO messages, ~22 assistant turns, several `AskUserQuestion` decision rounds)
- **SessionDepth**: deep — spanned the whole Stellar now-playing subsystem (client/poller/WS/snapshot), the web player architecture, logo resolution, IPTV promotion, station data model, observability, and deployment/runtime behavior; ~8 milestones shipped (M33 follow-ups → M39)
- **Personas Active**: project-engineer, code-reviewer, qa-engineer, database-engineer, security-engineer (html.escape removal review), SRE (observability/WS), UX-designer (play button, grouping, now-playing display), IT-architect (runtime-settings + grouping model), technical-writer (CHANGELOG/deployment docs), project-manager (epic/bead structure)
- **Beads Touched**: epics `nws` (M33), `aeq` (M34), `ivs` (M35), `e1n` (M36), `icd` (M39); standalone `ckg` (M37), `bbd`, `l6z`, `6d4`; follow-ups nws.13–.20, aeq.1–.10, ivs.1–.8, e1n.1–.8, ckg.1/.2, l6z.1, icd.1–.7; test-infra `kgn`, `l8v`. Version `1.0.0-0010` → `1.0.0-0022`.

---

## Section 1: User Value Delivered

Substantial, real, shipped-and-CI-green value. The session began with "live radio matching isn't working — 90s on 9 shows no radio data" and ended with the entire StellarTunerLog now-playing feature actually functioning for the first time, plus several requested UX features.

Concretely:
- **The headline win:** M33 had shipped "complete" but the StellarClient pydantic models were built against a *guessed* API shape (`/v1/nowplaying` keys under `stations` not `channels`; `/v1/channels` is a dict keyed by id, not a list). The feature had never worked against the real API. Fixed (`nws.20`), verified against the live API (90s on 9 → id 8206 → now-playing). Without this, *nothing* downstream mattered.
- **Auto-match wiring** (`nws.13`): the M33.3 resolver existed but was never invoked in production — relays never auto-matched. Wired into create/edit + an hourly re-resolve.
- **Manual re-match buttons** (`nws.16`), **relay logos** from the Stellar catalog + EPG/IPTV backfill (`aeq.2`), **album art in the station detail panel** (`aeq.1`), **audio that keeps playing across navigation** (`aeq.1` — hoisted the live audio element to the app shell), **multi-station now-playing list** (`aeq.3`).
- **The WS "Reconnecting" flap** (`bbd`): root-caused to redis-py 8.0 changing `DEFAULT_SOCKET_TIMEOUT` to 5s, which killed the blocking idle `pubsub.listen()`. Fixed across all three WS consumers.
- **Display + cold-start fixes** (`l6z`/`6d4`): "Don&#x27;t Speak" double-escape and the 30s-blank-on-startup.
- **Stations perf** (`ivs.1`): list endpoint was leaking an httpx client per station + N serialized Redis reads.
- **M39 features the PO explicitly asked for:** inline play button in the list, IPTV bulk-add (multi-select promote), and custom many-to-many station groups.

Net: a broken-on-arrival flagship feature is now working end-to-end, and three net-new UX capabilities shipped. Users (the operator + Adagio listeners) get functioning live-radio now-playing, logos, organized stations, and one-click play. This is genuine outcome delivery, not motion.

One caveat on "value vs. churn": a meaningful slice of mid-session work (M34's reconnect attempt, M35's WS-resilience/proxy-doc work) chased the WS flap with the *wrong* root cause and didn't fix it — it produced useful hardening but not the fix. That's work-adjacent-to-value that the eventual evidence-based fix superseded. See Sections 4 and 7.

## Section 2: What We Did Well Together

**The PO's demand for observability was the single best move of the session.** After two rounds of speculative WS-flap fixes that didn't hold, the PO asked: *"What logging do you have to prove what the issue is? If we don't have logging, it needs to be."* That reframed the whole effort. It led to M36 (structured WS open/close/pong-timeout logs, slow-request warnings, event-loop-lag detection, per-step list timing), and the very next reproduction produced a definitive root cause — `redis.TimeoutError ... in _pump_redis` every ~5s — that immediately refuted the event-loop hypothesis (zero `loop.lag`) and pointed straight at the pub/sub idle read. We went from "plausible theories" to "here is the exact line and the redis-py 8.0 default that causes it." That is exactly how hard-to-reproduce production bugs should be run, and the PO forced the discipline when the agent hadn't.

## Section 3: What the PO Could Improve

**Deployment-environment facts arrived late, after they had already shaped wasted work.** The critical fact "I do not have a reverse proxy in front of this, I'm direct to the app" was not stated until *after* M35 had shipped WS-resilience work whose framing (the corrected `docs/deployment.md` proxy examples, the "proxy idle timeout" narrative) was premised on a reverse proxy existing. The earlier diagnosis (`/api/v1/ws/` vs `/ws/` proxy mismatch + 30s idle timeout) was a plausible-but-wrong theory that a single up-front sentence — "no proxy, direct to uvicorn" — would have pruned immediately. Similarly, the runtime facts that actually mattered (redis-py version behavior, whether the *worker* image had been redeployed, that 90s on 9 was a fresh/cold start) only surfaced through back-and-forth.

This isn't a request for "more context" in the abstract — it's specific: **when reporting a production symptom, lead with the deployment shape (proxy/no proxy, what was redeployed, how it's run).** The agent should have asked for these sooner too (Section 4), but the proxy detour is the clearest case where one early fact from the PO would have saved a milestone's worth of misaimed effort.

(Honorable mention, much milder: "the build failed" with no detail required a dig to find it was the codegen-drift CI step — a pasted job link or the failing step name would have shortcut it.)

## Section 4: What the Agent Got Wrong

**I shipped fixes for unproven hypotheses instead of insisting on evidence first.** The WS flap took *three* attempts. M34.1 treated it as a frontend reconnect/dedup issue; M35 treated it as a reverse-proxy idle-timeout issue (and "hardened" accordingly); only after the PO demanded logging (M36) did the real cause — redis-py 8.0's 5s socket timeout on idle pub/sub — surface. I had the tools to instrument or to hit the live API on day one and chose code-reading-and-reasoning instead. The lesson the PO had to teach me ("prove it with logging") is one I should have applied unprompted the moment the first fix didn't hold. Relatedly, I built `nws.13`/`nws.16` *on top of* a StellarClient I never validated against the real endpoint — I didn't curl the live API until `nws.20`, several milestones in. The very first diagnostic for an external integration should have been "does the client parse the real response?"

**Two concrete process slips, both self-inflicted CI/merge pain:**
- **The codegen-drift CI failure (`nws.19`)** and a later near-miss: I "verified" no `schema.ts` drift with a `grep` shortcut instead of actually running `pnpm codegen`. CI caught it; I owned it and made "always run the real codegen drift check" a standing rule for the rest of the session — but it cost a round-trip first.
- **The worktree merge-from-wrong-directory mistake (M39.2 merge):** I ran `git merge m39-station-groups` from *inside the worktree on that branch*, producing a silent no-op ("Already up to date"). This is the exact footgun CLAUDE.md's worktree-aware-git rule warns about — and I violated my own documented discipline. I caught it immediately (the "Already up to date" was wrong), verified nothing was lost, and redid it correctly from the main checkout, but it was avoidable with the `pwd && git branch` check the rule mandates.

## Section 5: What Would Make the Project Better

**Contract-test external integrations against recorded real responses.** The entire `nws.20` failure class — a shipped, reviewed, fully-unit-tested feature that never worked because every test mocked a *guessed* API shape — is preventable. Any client for a third-party API (StellarTunerLog today; others later) should ship with at least one fixture captured from the *real* endpoint and a test that parses it, plus ideally a nightly/opt-in live contract check. The unit tests were green the whole time the feature was 100% broken; that's the most dangerous possible state and it should be structurally impossible to reach again. (This session already converted the Stellar fixtures to real-shaped — institutionalize the pattern.)

Secondary, both already recurring: (1) the documented local web gate doesn't include CI's `pnpm codegen` drift check, which produced repeated red-CI surprises — add it so local == CI; (2) the long-worktree-path test-DB provisioning flake (`kgn`/`l8v`) intermittently poisons full-suite runs and erodes trust in the gate.

## Section 6: Persona Perspectives

### Security Engineer
- **User value assessment**: Real. The `l6z` html.escape removal review protected users from a display bug *without* reintroducing XSS — I enumerated every sink (React text, XMLTV `saxutils.escape`, JSON/M3U/Xtream) before signing off. The `aeq.2` strict-SSRF posture on the third-party Stellar logo fetch closed a genuine (if low-likelihood) SSRF gap on attacker-influenceable URLs.
- **Session assessment**: Security got proper attention exactly where it mattered (reversing the M33 XSS defense was treated as a security decision, not a cosmetic one).
- **What I'd flag**: The new admin bulk-promote and group-mutation endpoints all correctly use `require_jwt_admin_user`, but authz coverage on the group mutations was a *test* gap caught only in review — auth posture should come with its tests by default, not as a kickback.
- **Disagreement**: I pushed harder than the code-reviewer initially did to treat the escape change as security-blocking-review-required; that was the right call.

### IT Architect
- **User value assessment**: The architectural reuse decisions served users directly — hoisting the live `<audio>` to the app shell (so audio survives navigation) and making `asyncio_debug` a runtime setting mirroring `stellar_enabled` both solved real UX/ops problems without bespoke machinery.
- **Session assessment**: Trade-offs were mostly explicit and the PO was given real choices (grouping model, bulk-add scope) via structured decisions.
- **What I'd flag**: The grouping data model was decided well (many-to-many, admin-curated), but the bigger architectural smell of the session is that M33 shipped a subsystem whose external contract was never validated — an integration boundary with no contract is an architecture gap, not just a test gap.
- **Disagreement**: None material.

### Project Manager
- **User value assessment**: High output, but with visible rework. ~8 milestones merged; however M34/M35's WS work was partially superseded by `bbd`, and several merges needed review-kickback cycles. The work was organized cleanly in beads/epics throughout.
- **Session assessment**: Good decomposition and traceability (every fix had a bead; follow-ups filed not dropped). Cadence frustrated the PO twice ("Is anything going on?", "What is happening for 40 minutes?") because long subagent runs look like hangs.
- **What I'd flag**: The repeated full quality-gate suite (~16–19 min) per milestone, plus two-to-three reviewers each, is the dominant wall-clock cost and the source of the "is it hung?" perception. Worth a faster inner loop (targeted gates during iteration, full suite at merge).
- **Disagreement**: I'd push back on any "we shipped a lot, great session" framing — the WS flap took three tries; velocity masked a diagnosis-quality problem until the PO forced evidence.

### Project Engineer
- **User value assessment**: Implementations delivered the asked-for value and reused existing infrastructure well (the shared `useTuneIn` hook, the shared `_promote_one_channel` helper, the runtime-settings pattern).
- **Session assessment**: TDD held; engineers caught their own subtle bugs (the `MissingGreenlet` from a session-bound user object after per-channel commit in bulk-promote was a genuinely sharp catch and was then regression-guarded).
- **What I'd flag**: Several engineer subagent runs ended truncated mid-test-run without committing, forcing continuation passes — a real efficiency leak; explicit "run gates in foreground and finish this turn" instructions helped.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: Strong. Inline play from the list, grouped stations, now-playing on every card, and album art in the detail panel are all direct answers to how the operator actually uses the app.
- **Session assessment**: The PO's real-world testing was the UX feedback loop, and it worked — each round surfaced concrete friction (flap, blank now-playing, escaped apostrophes).
- **What I'd flag**: Cold-start now-playing still isn't truly instant (first data waits on one poll); acceptable, but the PO's mental model ("the page should already have queried") deserves a small UX affordance (a loading shimmer vs. blank) — filed-adjacent, worth doing.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: Quality gates caught user-facing risks, not aesthetics — the dead live-error alert after the audio hoist (QA), the infinite-scroll >200-id cap that would silently kill now-playing, the single-promote-no-regression check.
- **Session assessment**: The two-reviewer pass repeatedly earned its keep; on M34.1 QA caught a regression code-review missed, and on M37 code-review verified the per-tick DB read was pool-safe.
- **What I'd flag**: A recurring pattern of *correct code, missing tests* (authz on mutations, mid-loop error recovery) — reviews kept having to kick back for test coverage that should have shipped with the feature.
- **Disagreement**: On `bbd` and `icd.3`, code-review said "no blocking" while QA said "blocking (test gaps)"; QA was right both times — for an observability feature and a partial-success endpoint, untested critical paths are blocking.

### Database Engineer
- **User value assessment**: The `icd.2` station-groups schema serves a real organizing need; the migration was reversible with correct FK cascades and a deliberate reverse-lookup index (so station-delete doesn't seq-scan memberships).
- **Session assessment**: The mandatory schema-change DBA review happened and was thorough; no migration shipped without it. `aeq.2` correctly reused the existing `'stellar'` cover-art source — no needless migration.
- **What I'd flag**: The long-worktree-path scratch-DB collision (`l8v`) and the create-all parity flake (`kgn`) are test-infra debt that undermines confidence in DB tests specifically.
- **Disagreement**: None.

### SRE
- **User value assessment**: This is *my* session. The M36 instrumentation directly converted an unreproducible user-facing flap into a one-line root cause. Observability paid for itself within one reproduction.
- **Session assessment**: Reliability work protected the user experience (WS stability, cold-start, perf), not vanity metrics.
- **What I'd flag**: We're still missing a WS-close-reason metric on `/metrics` (only logs); and the redis-py 8.0 default socket timeout now applies to *all* normal ops — appropriate, but worth a dashboard alert if any hot-path op trends toward 5s.
- **Disagreement**: I'd argue (gently) against the early agent instinct to fix-by-reasoning — the whole flap saga is the case study for "instrument first."

### QA Engineer
- **User value assessment**: Testing stayed focused on user-facing behavior (does the flap stop, does the logo load, does the title render correctly, does partial bulk-promote persist earlier successes).
- **Session assessment**: QA was the sharpest reviewer this session — caught the dead live-error alert, the untested mid-loop session-poisoning path, the authz gaps, and insisted on DB-level (not response-body) assertions for partial success.
- **What I'd flag**: The recurring need to *add* the regression guard after the fact (e.g., proving the bulk-promote test fails when the UUID fix is reverted) — good that it happened, better if the failing test came first.
- **Disagreement**: Repeatedly diverged from code-review on what counts as blocking; QA's "test gap on a critical path = blocking" stance was vindicated each time.

### Technical Writer
- **User value assessment**: The CHANGELOG entries and the corrected `docs/deployment.md` WS-proxy section serve operators directly (even though *this* operator has no proxy, the docs were genuinely wrong: `/ws/*` vs `/api/v1/ws/*`).
- **Session assessment**: Docs were updated alongside code, not deferred.
- **What I'd flag**: The runtime/env interactions we discovered (redis-py 8.0 timeout, asyncio_debug DB-overrides-env precedence, cold-start poll behavior) belong in an operator runbook, not just CHANGELOG bullets — filed as follow-ups but worth consolidating.
- **Disagreement**: None.

## Section 7: Lessons for Future Sessions

- **Keep**: Observability-before-fix when a bug isn't reproducible by reasoning. The M36 instrument-then-reproduce cycle is the template — it turned three failed guesses into one proven fix. Also keep the two-reviewer (3 with DBA) gate; QA caught real, shippable-bug-preventing issues every milestone.
- **Stop**: Shipping fixes for unproven root-cause hypotheses. Two WS "fixes" merged before the cause was known. Also stop "verifying" generated-artifact drift (or any gate) with a proxy check like `grep` — run the actual command. And stop running git merges from inside a worktree — `pwd && git branch --show-current` before every merge/push, per the existing rule.
- **Start**: For any external-API client, on day one: hit the real endpoint and assert the real response parses through the models, and commit a real-captured fixture + contract test. `nws.20` (a green-tested feature that never worked against reality) is the cautionary tale. Also start asking the PO for the deployment shape (proxy? what was redeployed? how is it run?) at the first sign of an environment-specific symptom.
- **Value learning**: The user didn't need "a now-playing feature" — M33 had delivered that on paper. They needed it to *actually parse the real API*, *stay connected*, *render text correctly*, and *show up promptly*. The gap between "feature merged + tests green" and "feature works for the user in production" was the entire session. Verify against reality, not against our model of reality.

## Memory candidates (durable, recurring)
1. External-API integrations: validate the client against the live endpoint + a real-captured fixture before building on it — unit tests against guessed shapes hid a 100%-broken feature through full review.
2. Local web gate must include CI's `pnpm codegen && git diff --exit-code` drift check; a version bump or any endpoint/docstring change can drift `schema.ts`. Always *run* codegen, never grep-guess.
3. Instrument-then-reproduce for unreproducible production bugs; do not ship speculative root-cause fixes.
