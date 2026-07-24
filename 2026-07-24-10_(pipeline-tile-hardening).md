# 2026-07-24-10 — pipeline-tile-hardening

- **ModelID**: claude-fable-5
- **TurnCount**: ~51 (14 genuine PO messages, ~12 automated task notifications, ~25 assistant turns)
- **SessionDepth**: deep — two projects (project-g data pipeline, project-h web app), seven agent dispatches across four personas, three review rounds, two merges
- **Personas Active**: project-engineer (×4 dispatches), code-reviewer (4 rounds), security-engineer (1), technical-writer (2 rounds), orchestrator covering PM/SRE duties
- **Beads Touched**: None (neither project is bead-tracked)

## Section 1: User Value Delivered

Substantial and shipped, on both projects:

- **Prevented a total pipeline outage.** The project-g crawler's whole-array `JSON.stringify` save had crossed the JS engine's ~512 MiB string ceiling that morning — 8 FATAL crashes in one day, and the one "successful" run squeaked under the cap by <1 MB. Without today's streaming/atomic rewrite, every future daily sync would have hard-failed within days. Also fixed a checkpoint-ordering bug that had silently dropped ~500 changelog events (cursor saved before data), and rewound to recover them.
- **Quota-aware backoff.** Confirmed empirically from logs that the vendor API quota is a fixed daily window (~5,000 calls, resets at midnight UTC) and replaced blind 10-minute probing with sleep-to-reset plus a mandatory fallback loop. Saves ~80–140 wasted probes per lockout and makes ETAs predictable.
- **Logo coverage for end users.** +5,661 station logos recovered by name-match propagation (45% → 50% coverage); the single biggest lineup-visible gap (a channel present in ~6,000 lineups) now has its logo.
- **Fallback tiles in project-h.** Logo-less channels (5.6% of channel slots) now render a callsign/initials tile with deterministic color and a category badge instead of a blank box. Label collision rate went from 19% (pre-existing) to 2.6% — users can now visually distinguish channels that previously all looked the same.
- **Hygiene that compounds:** the PO's entire previously-uncommitted app (~500 lines modified + whole subsystems untracked) is now in git history; test infra went 0 → 74 tests; lint 28 → 0; npm audit 3 high → 0; docs refreshed to match reality.

## Section 2: What We Did Well Together

The quota-window correction. The agent estimated backfill resumption at "~1–2 hours" based on a same-day 140-minute lockout. The PO pushed back — "I don't think we're going to get the quota back today; it usually is a 24-hour period. You should be able to validate that from the logs" — and the log history proved them right: 7 of 8 resets landed within ten minutes of midnight UTC; the agent's estimate was pattern-matched on the lone anomaly. The PO didn't just correct the fact — they pointed at the evidence source, which turned a wrong ETA into (1) a corrected plan, (2) a documented, confirmed quota model in the readme, and (3) a follow-on feature (the sleep-to-reset backoff) that the PO proposed once the window was confirmed fixed. That's the collaboration working exactly as designed: PO domain knowledge + agent verification capacity.

## Section 3: What the PO Could Improve

When the agent surfaced the merge decision for the feature branch and the PO replied "Lets do the follow-up candidates" — a catch-all that authorized new work but didn't address the explicitly-pending merge question — the decision had to be re-surfaced in the next message and answered a turn later. One extra round-trip is cheap, but the pattern is the known scope-creep vector the decision-discipline rules exist for: a decision block answered with an adjacent instruction rather than an answer. A one-character addition ("1. yes. Also do the follow-up candidates") would have closed both in one turn. (Counterpoint in the PO's favor: every other decision block this session — five of them — was answered instantly and unambiguously, often with a single letter. This was the only miss.)

## Section 4: What the Agent Got Wrong

Two related misses, both about over-confident inference:

1. **The 140-minute ETA (primary).** The agent told the PO the quota would reset "within roughly an hour or two" based on a single same-day observation, without checking the full reset history that was sitting in the same log file. The PO had to prompt the verification the agent should have done before stating a number. The data to get it right was already on disk and took one grep.
2. **The prescriptive kickback that caused the second regression.** In the first review kickback, the orchestrator's fix brief prescribed a specific mechanism ("word-initials via deriveInitials") without requiring re-measurement against real data. The engineer implemented exactly that, and it doubled the collision rate — caught only because the reviewer re-ran its sampling method. The round-2 brief then again prescribed a specific default ("first 6 alnum chars"), which fixed the measured problem but reintroduced a subchannel-suffix collision class at a smaller scale. The lesson isn't that the briefs were wrong to be specific — it's that both rounds prescribed mechanism while only round 2 required the empirical gate (collision-rate sampling ≤ baseline). The gate, not the mechanism, is what finally converged it. Mechanism prescriptions without measurement gates are half a brief.

Minor: an addendum item queued to a running batch agent arrived after the agent had finalized, and was silently not done — caught only because the orchestrator checked for the overrides stanza rather than trusting "all done." Queued mid-run messages need a completion-check, not an assumption of delivery.

## Section 5: What Would Make the Project Better

**Put project-g under git.** The pipeline directory uses a `.bak-YYYYMMDD` copy convention instead of version control — today produced six backup files across four edits, and "which version is deployed" was answered by comparing mtimes against log timestamps (which itself caused a documentation error via UTC-vs-local confusion). Every discipline this team runs — review diffs, kickback cycles, merge decisions, deploy-time provenance — got harder or impossible in that directory. The scripts are now load-bearing production infrastructure with a systemd deployment; they've outgrown the scratch-directory workflow. A `git init` plus a one-line "restart the service after editing" hook note would have made three of today's verification steps trivial.

## Persona Perspectives

*(Synthesized by the orchestrator from full session context rather than spawned agents — every dispatch and review round this session is in-context, and per-persona subagents would only see a summary of it.)*

### Project Engineer
- **User value assessment**: High — the crash fix and backoff protect the data pipeline users depend on; the tile work is directly user-visible. Nothing built this session was speculative.
- **Session assessment**: TDD held throughout (52→74 tests); the required live-DB sampling caught my own first-attempt regression in the follow-up batch before commit, which unit tests alone would have missed. The staged verification demands (prove the streaming writer standalone before touching the live service; prove the ESM loaders in the real container) were the right calls.
- **What I'd flag**: The round-1 collision fix regressed because I validated against the reviewer's cited examples instead of the data distribution. Example-driven tests pin examples; only sampling pins distributions.
- **Disagreement**: With the orchestrator — the round-1 kickback brief prescribed the mechanism that regressed. Shared blame is fine, but briefs should prescribe acceptance criteria harder than they prescribe implementations.

### Code Reviewer
- **User value assessment**: The four review rounds directly protected users: the callsign truncation bug would have rendered meaningless colliding labels on the majority of fallback tiles — precisely the thing the feature exists to prevent.
- **Session assessment**: Method-based re-verification (re-running the sampling, not re-checking the cited examples) caught a regression that example-based re-review would have approved. That's the strongest process validation this session produced.
- **What I'd flag**: The engineer's test suites twice validated the fix against reviewer-cited examples rather than the data shape the reviewer's method targets. Tests that mirror the review conversation are a weaker contract than tests that mirror the data.
- **Disagreement**: None with the final state; approved on evidence.

### Security Engineer
- **User value assessment**: Honest answer — modest. All three audit findings were confirmed non-reachable, so the override fix removed scanner noise and future-proofed rather than closing an active hole. The genuinely valuable output was the reachability analysis and the recorded guardrail (image-optimizer config change ⇒ re-assess), which prevents a silent future flip to reachable.
- **Session assessment**: Good — investigation before remediation, remediation tested in isolation before application, guardrail written into the commit record rather than a chat message.
- **What I'd flag**: A `.env` with live API keys sits in the working tree of a repo that only today acquired real git history. It was verified excluded from every commit this session, but "verified each time" should become "structurally impossible" — confirm `.gitignore` coverage and consider a pre-commit secrets check.
- **Disagreement**: None.

### Technical Writer
- **User value assessment**: The readme refresh serves a real operator (the PO at 7 AM wondering why the crawler is sleeping) — status, confirmed quota model, script inventory, and the restart-after-edit gotcha are all questions this session actually had to answer the hard way.
- **Session assessment**: One real failure: I shipped a fabricated incident narrative into documentation by comparing UTC log timestamps against local-time file mtimes. The orchestrator caught it on evidence review. Docs claims need the same verification bar as code claims.
- **What I'd flag**: project-h got zero documentation this session despite gaining a test toolchain, an env-var config knob, a cross-project file dependency, and new UI behavior. The next developer learns all of that from git archaeology.
- **Disagreement**: None.

### SRE
- **User value assessment**: The backoff and atomic-write work directly protect data freshness users see. The systemd-first posture (service supervises, agents don't babysit) kept token cost proportional.
- **Session assessment**: Good instincts — log monitor instead of polling agents, one-call wall-detection instead of probe loops. The PO explicitly optimizing for token cost ("we should know for sure so we don't waste tokens") was the right operational reflex and the agent's monitor answer was the right shape.
- **What I'd flag**: A six-crash restart cascade ran at 1 AM and nobody knew until a human asked a status question eight hours later. There is no alerting on FATAL/restart-loop conditions for a pipeline that now has real consumers. A trivial addition (the existing timer plus a failure-state check that notifies) would close it.
- **Disagreement**: With the celebration of the "self-healing" restart loop — systemd restarting into the same crash six times isn't resilience, it's an undetected outage with good manners.

### QA Engineer
- **User value assessment**: The sampling gate is the session's most user-protective testing innovation — it measured what users would actually see (label distinctness across the real channel population) rather than what the code promises.
- **Session assessment**: Test strategy matured mid-session, which is the polite way of saying it started inadequate: two regressions shipped to review because the test suite pinned hand-picked examples. The fix — requiring distribution-level evidence alongside unit tests — should have been in the first brief, not the third.
- **What I'd flag**: The sampling gate currently lives in review-round prose and scratchpad scripts. It should be a committed, runnable check (even if manually triggered) so the next label-logic change doesn't depend on someone remembering this session.
- **Disagreement**: With the engineer's "52/52 passing" framing after round 2 — green suites were repeatedly true while behavior was regressing. Pass counts are not evidence of correctness for distribution-shaped problems.

### Database Engineer
- **User value assessment**: The staged, zero-downtime admin loader (verified live this session) is the right pattern and directly protects users from mid-load empty states.
- **Session assessment**: Sound — the engineer correctly distinguished the safe staged loader from the truncate-in-place root loader and declined to live-run the destructive one during lint verification.
- **What I'd flag**: The category/logo data flows file → DB with no versioning or generation stamp beyond `_meta`. When the sibling generator regenerates, there's no way to answer "which generation is the DB serving?" A generation ID carried through the load would make cross-system debugging tractable.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: Real problem, real fix — blank tiles were genuinely useless to users. Callsign-first labels, stable per-channel color, and category badges are defensible choices.
- **Session assessment**: The live headless-browser verification against real data (twice) is better UX validation than most sessions get. Accessibility was treated as a first-class review dimension, not a checkbox.
- **What I'd flag**: Badge vocabulary ("AUD", "PEG", "VOD") is designer-intuitive, not validated user-intuitive. And 2.6% of fallback labels still collide — acceptable now, but nobody has looked at *which* channels collide from a user's seat. Also the PO's genuine surprise at radio channels existing ("Wait, we get radio channels, too!?") is a signal the product's content model isn't fully visible to its own owner — worth a data-facts summary page or admin view.
- **Disagreement**: None material.

### Project Manager
- **User value assessment**: High throughput with zero scope invented by the team — every work item traced to a PO decision or a review finding. The follow-up batch was PO-authorized as a batch, which was efficient.
- **Session assessment**: Decision discipline mostly excellent (five clean single-letter decisions). One catch-all/decision collision (Section 3) and one lost mid-run addendum (Section 4) were the only process drops, both caught.
- **What I'd flag**: Neither project is bead-tracked, so today's six carried-forward notes (stale comment, alerting gap, docs gap, secrets hardening, generation stamps, badge validation) live only in this retro and the session transcript. That's exactly the kind of value that evaporates between sessions without a board.
- **Disagreement**: With doing the follow-up batch as one six-commit dispatch — it worked, but a 380k-token single-agent run with a mid-run addendum loss is past the size where per-item dispatches would have been more controllable.

### IT Architect
- **User value assessment**: No architecture invented for its own sake; the one structural addition (categories file contract between projects) was contract-first with graceful degradation, defined before either side built.
- **Session assessment**: Defining the cross-project file contract in both agents' briefs before dispatch is why two concurrent agents integrated without a single mismatch.
- **What I'd flag**: That same integration is an absolute-path file dependency from project-h's runtime into project-g's data directory (env-overridable, but the default couples a web app to a crawler's home directory layout). Fine for a single-host deployment; it becomes the first thing that breaks under any deployment change. Worth an ADR-sized note, not action.
- **Disagreement**: None.

## Lessons

- **Keep**: Method-based review re-verification — reviewers re-running their *measurement method* (fresh samples each round) rather than re-checking cited examples caught two regressions that example-checking would have approved. Also keep: empirical acceptance gates in kickback briefs ("collision rate must be ≤ X measured on live data"), which is what actually converged the label logic.
- **Stop**: Stating time estimates (or any inference) from a single observation when the full history is greppable in the same file. Both agent errors this session (ETA, doc incident narrative) were single-observation inferences contradicted by on-disk evidence.
- **Start**: (1) Treat UTC-vs-local as a named hazard when comparing log timestamps to file mtimes — it caused one wrong ETA input and one fabricated doc narrative in a single day. (2) After queuing a message to a running batch agent, verify the item landed in the final state instead of trusting the completion report. (3) Failure alerting on supervised background services — a restart cascade should page-or-notify, not wait for a status question.
- **Value learning**: The PO's operational knowledge (quota is daily) beat the agent's fresh-data inference, and the PO's surprise (radio channels exist) revealed the dataset contains content classes the product owner didn't know about. Both directions of that asymmetry matter: verify agent inferences against history, and surface dataset facts to the PO proactively — the data model is not as visible to its owner as either side assumed.
