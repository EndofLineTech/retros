# 2026-07-12-03 — event-sync-orchestration

- **ModelID**: claude-fable-5 (main loop for most of the session), switched to claude-opus-4-8 after the review-queue engineer hit a Fable-5 usage limit; subagents ran Opus-class (default) except two deliberate Sonnet dispatches (docs ti939.1.6, enabled-groups filter x82s3)
- **TurnCount**: ~130 messages (1 opening directive + a long tail of background-task notifications, ~6 genuine mid-session PO messages: "stipp going?", "keep going", "what are phases 2 and 3?", "Do phase 2 as well", the enabled-groups feedback, and the "enabled on account is correct" confirmation)
- **SessionDepth**: deep — full Event Sync epic (Phases 1A + 1B + Phase 2), 13 PRs, matcher/schema/executor/engine/frontend/docs/MCP surfaces, live container verification across two deploy cycles
- **Personas Active**: Project Engineer (many dispatches), Code Reviewer (persona-reviewer, ~13 review rounds), Technical Writer (Sonnet), plus latent SRE / Security / DBA / QA / UX perspectives baked into the beads and surfaced in review
- **Beads Touched**: Created + closed — ti939.1.1–1.6, ti939.1 (epic), ti939.2.1–2.4, ti939.2 (epic), ti939.3.1–3.4, sfysz, hirm6, z4y4a, ixujz, x82s3. Created (open follow-ups) — 8fq6x, 8rzkq, odnnf, xeli0, 72z02. Left open by design — ti939.3.5, ti939.4.*, epic ti939 (Phase 3 out of scope)

## Section 1: User Value Delivered

Real, shipped, verified-on-the-operator's-own-instance value. The user's problem — N duplicate channels per live sports/PPV event across multiple IPTV providers — now has an end-to-end answer:

- **Phase 1A (preview-only):** the operator can configure an event_sync rule, test parse patterns against live stream names, and preview exactly which secondary-provider streams would attach to which master channels, with scores, bands, team-token verdicts, parse failures, and an unmatched list — **zero writes**. This is the trust-building instrument that inverts the prior 1,341-false-positive-merge incident.
- **Phase 1B (attach):** a manual run actually attaches matched streams to master channels — capped, journaled, reversible via rollback, and structurally manual-run-only.
- **Phase 2 (automation + conveniences):** opt-in auto-run (default off), dummy-project-b auto-assignment, a confirmed/journaled guided auto-sync toggle, and an ambiguous-band review queue keyed on content fingerprints.
- **The feedback fix (x82s3):** group pickers went from listing **1,120** groups to the **26** actually enabled on their provider account — a measured 43× noise reduction on the operator's real instance, verified live.

Nothing shipped that nobody asked for. Every merged PR traces to a bead whose description records the decided design and PO constraints. The one place we produced work-that-creates-work is deliberate: five follow-up beads filed from review findings and live verification, each a genuine defect or gap, not scope-padding.

## Section 2: What We Did Well Together

The **preview-first phasing held under pressure and paid off precisely where designed.** The clearest moment: during PR #614 review, the code-reviewer probed the two shipped generic no-`@` patterns against the real matcher and found they mangled their own example strings — collapsing every same-minute event to an identical garbage title and manufacturing **1.0-confidence attach-band false positives** between unrelated events. That is the exact shape of the 1,341-incident. It was caught *before any operator saw it*, in a preview-only phase, and the fix landed with a permanent cross-language JSON-fixture parity gate so it can never silently regress. The PO's original instinct to gate the write path on preview validation is what made that catch cheap instead of catastrophic — the collaboration worked because the PO front-loaded that constraint into the epic and I enforced it batch by batch rather than optimizing it away.

## Section 3: What the PO Could Improve

**The scope grew twice mid-flight, each time as a terse directive with the decision authority delegated back to me — and the second expansion ("Do phase 2 as well, i can answer questions when i get home so you may need to make decisions for me") handed me a security-relevant feature epic to run essentially unsupervised.** That worked out, but it put me in the position of making calls that genuinely deserved a human — most notably choosing the `int()` taint-cut over dismissing two CodeQL SSRF alerts (I deliberately kept dismissal authority with the PO), and shipping the guided auto_channel_sync toggle — the *first sanctioned Dispatcharr group-settings write in the project's history* — reviewed only by an agent. Neither was wrong, but "you may need to make decisions for me" is a blank check on a feature whose whole design premise is *precision-over-recall because wrong writes are expensive*. A single sentence in that message — "don't ship anything that writes to Dispatcharr settings without me" or "auto-run is fine to enable by default" vs "keep it off" — would have removed the highest-stakes guesses. The delegation was efficient; a one-line guardrail on the irreversible surfaces would have made it safe rather than merely lucky.

## Section 4: What the Agent Got Wrong

**During the final live verification I misread a stale browser bundle as a possible regression and nearly reported it as one.** At the Phase 2 wrap I loaded the editor, saw the Advanced section rendering only three controls (no auto-run checkbox, no dummy-project-b picker) and the quick-path copy still saying "never runs from the unattended refresh task," and my first framing was "this contradicts what we shipped — possible merge-interaction regression." I did the right thing next (checked the deployed bundle on disk, found both controls present, correctly diagnosed browser cache and cache-busted), so nothing false reached the PO. But the *initial* interpretation jumped toward "regression in the merged code" before ruling out the cheap, obvious cause — a cached tab from a verification session hours earlier. The orchestration discipline I'd been applying to agents ("verify premises before briefing") is exactly what I under-applied to my own first read of the UI. A disciplined verifier checks the artifact under test is the artifact deployed *before* forming a hypothesis about the code.

## Section 5: What Would Make the Project Better

**A deploy-provenance signal the UI actually exposes.** Twice this session the container's `/api/version` reported `0.17.6-0043` (a baked `ECM_VERSION` env var) while the running code was `0073`/`0074`, and the frontend served a stale cached bundle independent of the backend. Live verification had to reconstruct "what is actually running" from three indirect sources (grepping `main.py` in the container, grepping bundle strings, cache-busting the browser). For a container-first workflow where "edit locally, deploy, verify" is the core loop, the absence of a trustworthy "what commit/bundle is live right now" readout is a recurring tax — it cost real time here and is exactly the kind of thing that lets a stale-deploy bug masquerade as a code bug. A footer/health field that reports the *actual* git SHA of the running backend AND the built bundle hash would pay for itself.

## Section 6: Persona Perspectives

### Security Engineer
- **User value assessment**: High and real. The team-token hard-reject, the schema-mandatory scoping, the 0.80 floor clamp, and — critically — the config-stripping catch in PR #612 (backup restore silently reverting an event_sync rule to a standard rule with dormant `create_channel` actions live) all protect the operator from destructive false writes. That last one was a genuine latent footgun, caught in review.
- **Session assessment**: Security rails were treated as non-negotiable throughout and independently re-verified, not trusted. The fingerprint-keying constraint on the review queue (no channel/stream IDs in any identity column) survived an explicit adversarial review pass.
- **What I'd flag**: The guided auto-sync toggle is the project's first sanctioned Dispatcharr group-settings write, and it shipped under agent-only review during an unsupervised window. It's confirm-gated at the API *and* the TS type level and journaled — genuinely well-built — but a human eyeball on the first write-to-provider-settings surface would have been proportionate to its irreversibility (snapshot restore does NOT revert it).
- **Disagreement**: I'm less comfortable than the PM with how fast the auto-run + toggle epic moved unsupervised. Velocity was real; the irreversibility profile of two of those surfaces warranted a checkpoint the PM's "shipped clean" framing glosses over.

### IT Architect
- **User value assessment**: The master-group-as-reference model (ECM never creates/deletes channels; Dispatcharr owns lifecycle; ECM only attaches) is the decision that makes this feature safe to build at all, and it served the user by keeping the blast radius to "an extra stream on a channel" rather than "a channel graph ECM half-owns."
- **Session assessment**: Trade-offs were explicit and recorded in beads. The single-resolver / dry-run-parity-by-construction choice (preview and attach call the identical `resolve_event_sync`) is the architectural spine that made "preview validates the real behavior" true by construction rather than by test alone.
- **What I'd flag**: The stateless-recompute-plus-fingerprints design held, but Phase 2 introduced the first new table (`event_sync_reviews`) — the epic scoped "no new tables" to Phase 1, correctly, but the boundary is now a thing future work must re-read rather than assume.
- **Disagreement**: None material. The DBA and I agree the review-queue schema was the right call.

### Project Manager
- **User value assessment**: Every closed bead maps to shipped operator value; no orphaned work. 13 PRs merged, all gates green, all traceable.
- **Session assessment**: Strict dependency-ordered execution, one coherent batch at a time, review-before-merge without exception. The mid-run rate-limit failure of the review-queue engineer was handled by re-dispatch-with-continuation rather than orchestrator-finish — the process held under stress.
- **What I'd flag**: Two scope expansions (follow-ups, then all of Phase 2) roughly doubled the session mid-flight. Delivered cleanly, but "we finished the project" was declared twice before it was actually done — a reminder that a stated Definition of Done should survive contact with "also do X."
- **Disagreement**: I'm more sanguine than Security about the unsupervised Phase 2 window — the review gate never lifted, so "unsupervised by the PO" ≠ "unreviewed." But I concede the toggle was a fair place for a human checkpoint.

### Project Engineer
- **User value assessment**: The implementations delivered the asked-for value and nothing speculative. The x82s3 filter is the cleanest example — 1,120→26 groups is a change the operator *felt* immediately.
- **Session assessment**: TDD held, gates were real, and the continuation-engineer hand-off (auditing an interrupted engineer's uncommitted tree against spec rather than trusting it) is exactly the discipline that keeps a rate-limit failure from becoming a silent quality hole. Version-bump lockstep and CHANGELOG discipline were consistent.
- **What I'd flag**: The recurring stale-frontend-bundle friction is an engineering-workflow gap, not just an ops one — the container-first deploy for the frontend has no cache-invalidation story beyond "clear assets and hope the browser cooperates."
- **Disagreement**: None with Code Reviewer — the review rounds materially improved the merged code every single time.

### UX Designer
- **User value assessment**: The enabled-groups fix is the session's purest UX win — it came from real user feedback and removed real friction, verified against ground-truth counts. The per-candidate-evidence review queue (raw names side by side, not an opaque score) directly encodes the human-factors lesson from the 1,341 incident.
- **Session assessment**: Accessibility baselines (text+icon never color-alone, ARIA on bands, keyboard-nav, no modal-only flows) were carried through every frontend batch, not bolted on.
- **What I'd flag**: The badge-tooltip inaccuracy I caught in live verification (`xeli0` — "never on unattended refresh" is false for auto-run rules) is a small instance of a real pattern: copy that was true at Phase 1 and quietly went stale as features landed. This is the *second* stale-copy sweep this session (the first was the ti939.2.4 rider). Copy needs to be treated as a versioned artifact that features update, not set-and-forget.
- **Disagreement**: I'd have pushed to fix `xeli0` before calling the wrap done rather than filing it — a user hovering that badge is actively misinformed. The orchestrator batched it; defensible at 3 AM, but it's user-facing wrong *now*.

### Code Reviewer
- **User value assessment**: Quality standards caught bugs the operator would actually have hit — the `@`-separator false positives, the guessed-date leak, the config-stripping resurrection, the contested-candidate tie-break that would attach prelims to the main card. These were user-harm defects, not aesthetics.
- **Session assessment**: Every PR got an adversarial pass; findings were empirically reproduced against the real matcher, not asserted; the orchestrator independently re-ran gates before every merge rather than trusting agent self-reports. This is the discipline working as designed.
- **What I'd flag**: The `odnnf` follow-up (no direct test that a tripped-breaker/pre-flight-fail unattended run enqueues nothing — it's only structurally guaranteed) is the kind of "proven by construction, not by assertion" gap that's fine until a refactor moves the construction. Worth closing before Phase 3 touches that path.
- **Disagreement**: None — but I want it on record that "APPROVE WITH NITS" appeared often enough that the nits are a real backlog (8fq6x, 8rzkq, odnnf, xeli0), not noise. Shipping-with-nits is correct only if the nits actually get worked.

### Database Engineer
- **User value assessment**: The `event_sync_reviews` table serves a real user need (decisions that survive refreshes) and the fingerprint keying is what makes those decisions durable across provider stream-ID churn — a direct user-facing correctness property.
- **Session assessment**: Migration 0031 and 0032 were both guarded, up/down tested, backward-compat verified, and the 0025 idempotency-guard fix (exposed by a full-chain replay) was a genuine latent-drift catch.
- **What I'd flag**: `8fq6x` is mine to worry about — DBAS config-restore does `delete()` on rules, and the FK CASCADE silently drops answered review decisions. Self-rebuilding pending items is fine; losing *answered* decisions on a restore is data the operator won't know they lost. This needs the PO decision I flagged.
- **Disagreement**: None with the Architect on the new-table call; it was the right reuse of the ADR-008 pattern.

### SRE
- **User value assessment**: The rollback verification (masters never deleted, streams-only restore) and the circuit-breaker-gates-auto-run work protect the operator's channel graph from a bad unattended run — real reliability value, not observability theater.
- **Session assessment**: The ship-blocker (ti939.2.3) was treated as a blocker; the surgical-unmerge preference (sfysz) closed a real "rollback reverts Dispatcharr's own churn" gap. The run-summary line as the "3 AM drift detector" is exactly the right instinct.
- **What I'd flag**: The `ixujz` note stands — the breaker gates auto-run *today* only because manual is the sole trigger before Phase 2's flag; that's now shipped, and the breaker/auto-run interaction is load-bearing. And the deploy-provenance gap (Section 5) is squarely an SRE concern: you cannot operate what you cannot confirm is running.
- **Disagreement**: None, but I'll amplify Security — the unsupervised window is an operational risk posture the PM under-weights.

### QA Engineer
- **User value assessment**: The frozen corpus gate (precision ≥97% / recall ≥85%, add-one-pair-per-bug-forever policy) is the single highest-leverage user-protecting artifact built this session — it turns "did we regress the matcher" from a judgment call into a CI fact.
- **Session assessment**: The 7-scenario lifecycle suite encodes the verified Dispatcharr behaviors as fixtures, so a future upstream change surfaces as a test conversation instead of a production surprise. Dry-run parity is pinned, not assumed.
- **What I'd flag**: E2E honesty was good (the attach segment was correctly demoted to a documented manual script because the live instance has no auto-sync-ON master group and toggling is forbidden) — but that means the *fully live* attach path has never been exercised end-to-end on real data. It's covered by mocked integration tests; it is not covered by reality. The operator validating on their own providers (the PO's stated next step) is the real acceptance test.
- **Disagreement**: None.

### Technical Writer
- **User value assessment**: `docs/event_sync.md` is genuinely dual-audience and cookbook-verified (patterns copy-paste-tested against the real parser), and it's the operator's roadmap for the PO's own next step. Documentation here directly enables the trust-building the whole feature depends on.
- **Session assessment**: Docs shipped in the same window as the features, not after, and the verbatim not-self-healing sentence (a hard PO constraint) was carried correctly and re-verified in review.
- **What I'd flag**: The stale-copy pattern (Section UX, `xeli0`) is my concern too — docs and UI copy drifted as Phase 1 assumptions were superseded by Phase 1B/2 reality. The docs were kept current *because a bead forced it each time*; the UI badge wasn't, because no bead owned it. Copy-as-code needs an owner per surface.
- **Disagreement**: None.

## Section 7: Lessons for Future Sessions

## Lessons

- **Keep**: Preview-first phasing with a frozen regression gate before any write path. It caught three separate would-be-incidents (the `@`-pattern collapse, the guessed-date leak, the config-stripping resurrection) *cheaply*, in preview-only phases, exactly where the design intended. And keep the independent-gate-re-run-before-merge discipline — it caught real divergence between agent self-reports and branch state more than once.
- **Stop**: Forming a "the merged code regressed" hypothesis from a UI observation before confirming the artifact under test is the artifact deployed. The stale-bundle near-miss at the wrap was self-inflicted by skipping my own "verify premises" rule on the frontend.
- **Start**: Treating user-facing copy (UI strings + docs) as a versioned artifact with a per-surface owner, so that feature X updates the copy that feature X invalidates — instead of discovering stale "preview-only"/"never unattended" strings two sweeps in a row. And: when a directive delegates decision authority on an *irreversible* surface (writes to an external system, default-on automation), extract a one-line guardrail from the PO before proceeding rather than choosing conservatively-but-silently.
- **Value learning**: The operator's most acute pain wasn't the matcher sophistication we spent the most effort on — it was **1,120 groups in a dropdown**. The single smallest change of the session (a client-only enabled-filter, 26 vs 1,120) may be the one the user notices first and most. Real-usage friction on a real instance outranks assumed sophistication; the sooner the operator's actual data is in front of the feature, the sooner that kind of gap surfaces.
