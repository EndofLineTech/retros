# 2026-08-03-09 — project-d-doctest-to-fix

- **ModelID**: claude-fable-5
- **TurnCount**: ~145 (8 substantive PO messages, ~12 background-task notifications, ~125 assistant turns)
- **SessionDepth**: deep — full naive documentation walk of a rendered docs site (102 pages crawled, ~30 articles exercised interactively against a live test instance), then a product-bug investigation, fix, two review rounds, live verification, and merge
- **Personas Active**: orchestrator-as-tester (doc-test protocol), project-engineer (investigation + implementation, separate dispatches), code-reviewer (ship-sequence round), then all ten personas in a /team-review ceremony
- **Beads Touched**: created+closed: qfonx (run tracking), q6xjl (the product bug); created: lsa0s, zt3kf (later updated with parked notes), y6zg6 (dependency recorded), d20jf, kahzn, r9oqx, nvhg7, ax0kf

## Section 1: User Value Delivered

Two distinct chunks of real value:

1. **The documentation was proven trustworthy for new operators.** Run 4 of a recurring "naive operator" doc test walked the twice-fixed guide end-to-end on a fresh install. For the first time in the series, the entire getting-started path produced zero defects — every prior run's fixes were independently confirmed by someone who hadn't read the answer key. A full-site crawl (102 pages) found zero broken links/images, retiring a historical class of ~122 dead links. Users of this guide now genuinely can get from nothing to working channels.
2. **A broken disaster-recovery path was found, fixed, and proven within the same session.** The doc test discovered that restoring a just-taken backup failed 7/7 and rolled back — the product's core "your data is safe" promise was false. The root causes (three API-contract bugs where the client had been written against a guessed upstream REST surface) were investigated, fixed with recorded-real-response fixtures, reviewed twice, and proven with a live same-instance round-trip that reproduced the original failure scenario and now succeeds. An operator who loses their container tomorrow can actually restore.

Secondary value: four documentation defects filed with precise evidence, nine well-scoped backlog beads with recorded dependencies, and a systemic finding (contract-test coverage exists for only 3 of ~20 critical client paths) turned into a design bead rather than left as review commentary.

## Section 2: What We Did Well Together

The **decision cadence on the fix arc**. The PO answered every decision block within minutes and in exactly the granularity offered ("1. Yes, authorized. 2. Yes, remap. 3. Yes" and later "2. Recommendation is good. Approved. 3. a. 1. Ship it."). Because decisions were framed as numbered options with recommendations, zero clarification round-trips were needed across an investigation → fix → review → live-proof → merge pipeline that involved five agent dispatches and one production-adjacent write authorization. The specific moment that made this work: the live-apply-to-production question was surfaced *early* (while the engineer was still building), so the authorization was already in hand when the PR went green — no stall at the gate.

## Section 3: What the PO Could Improve

**The /team-review of PR 765 arrived after the ship sequence had already run its review.** By the time the ceremony was invoked, the PR had a completed code-review round (approved, five dispositioned findings), independently re-verified gates, a posted review record, and a live end-to-end proof — and my message on the table was literally the merge ask. The ten-persona ceremony then re-reviewed the same 14-file diff. It did surface real value (the stale developer-API doc, the periodic-exercise gap flagged by three personas, the systemic contract-test question), but roughly half its output re-verified what the first round had already established, and one decision (finding dispositions) had to be re-collected because the ceremony superseded the pending version of the same question. If the intent was always "this fix gets the full team treatment," saying so at the "Yes, authorized" moment — before the single-reviewer round — would have gotten the same findings for roughly half the total review spend and avoided asking the PO the disposition question twice.

## Section 4: What the Agent Got Wrong

**A defensive-scripting fallback nearly put the test artifact into the legacy full-restore card.** During the doc-test's backup verification, my upload script located file inputs by walking ancestors for the modern restore card, and when that lookup returned nothing it *fell back to index 0* — which was the legacy whole-config-restore card's input, a control whose adjacent button replaces the entire configuration and reloads the app. Nothing fired only because that card requires a separate button click. On a destructive-adjacent surface, "fall back to the first match" is exactly the wrong failure mode; the script should have thrown on lookup failure. I caught it in the output, cleared the input, and rebuilt the flow — but the near-miss was self-inflicted and avoidable by writing fail-closed automation from the start. (Same-session smaller versions of the same pattern: a wrong-button probe that downloaded a YAML export instead of the backup zip, and a leak-gate that initially checked non-rendering attributes.)

Runner-up: I told the PO the failure "points at endpoint drift" before dispatching the investigation; the investigation refuted drift — the endpoints were fine, the client had guessed wrong identifiers/formats. The dispatch brief hedged correctly (it explicitly asked the investigator to test the alternative), but my prose to the PO front-loaded the wrong hypothesis with more confidence than the evidence held.

## Section 5: What Would Make the Project Better

**Package the doc-test browser rig as a reusable harness.** Every run of this recurring documentation test rebuilds the same infrastructure from prose instructions: a long-lived browser holder with crash guards, a CDP-attach step library, a PII-obfuscating screenshot gate with a leak assertion, page/tab helpers, and a site crawler. Run 4 spent its first ~20 turns reconstructing (and re-debugging) what runs 1–3 had already built and discarded. A small versioned harness (or skill) with the holder, the capture gate, and the fail-closed selector conventions from Section 4 baked in would make run 5 start at the first article instead of at `npm install` — and would make the leak gate a tested artifact rather than something rewritten under time pressure next to live credentials.

## Persona Perspectives

### Security Engineer
- **User value assessment**: Real protection: the capture gate kept subscription credentials (rendered in plain URLs by the product) out of 50+ screenshots destined for a public docs repo, and the fix preserved credential-hygiene invariants through new code paths — verified by tracing, not by trusting comments.
- **Session assessment**: Security got genuine attention without being the bottleneck. The live-apply-to-production authorization was handled correctly: explicit PO gate, identical-value write, fresh-backup approach to close the stale-values window.
- **What I'd flag**: The orchestrator's screenshot leak gate was written ad hoc during setup and had to be corrected mid-run (initially checked non-rendering attributes). It held, but a safety control built under way-finding pressure is a control you got lucky with. Also: the ship-round reviewer ran its verification inside the production container's /tmp — disclosed and cleaned, but the brief should have named an allowed execution environment explicitly.
- **Disagreement**: None material this session.

### IT Architect
- **User value assessment**: The fix was tactical and correctly so; the durable value is the drift-strategy ADR + contract-sweep bead (ax0kf), which converts the third recurrence of a bug class into a design decision instead of a fourth incident.
- **Session assessment**: Good layering discipline — the resolver landed at the right seam, and the team resisted folding the systemic answer into the hotfix PR.
- **What I'd flag**: The client was originally written against a *guessed* REST surface with a code comment promising later verification that never happened. The lesson isn't the bug; it's that deferred-verification comments are unowned commitments. They should become beads at creation time.
- **Disagreement**: With the QA framing that contract breadth is optional at this tier — the upstream's schema is free and machine-readable; at any tier, guessing when the contract is published is a choice, not a constraint.

### Project Manager
- **User value assessment**: High throughput to real outcomes, but note the ratio: one merged fix and ~9 new backlog items. Backlog growth was PO-authorized at each step, yet the board now carries a "restore round-trip" story spread across six beads with no epic grouping — my grouping recommendation from the review was not adopted this session.
- **Session assessment**: Decision hygiene was excellent (numbered blocks, fast answers). My pre-merge ask to close the missing-test finding was overruled per review-history discipline; process-correct, and the dissent was recorded rather than averaged away — that's how it should work.
- **What I'd flag**: The disposition question was asked twice (pre-ceremony and post-ceremony) because the team review was invoked after the ship sequence — sequencing cost a PO round-trip and duplicated review spend.
- **Disagreement**: I still hold that a five-minute test closing the exact original failure signature belonged in the fix PR. The counter-rule won; the cost is that kahzn now carries merge risk for a test that had a free ride available.

### Project Engineer
- **User value assessment**: The fix is the session's hard value: minimal diff, recorded fixtures, one-GET-per-run enforced by construction. The live proof means an operator-facing promise is now backed by evidence, not tests alone.
- **Session assessment**: The investigation/implementation split (read-only diagnosis first, then a fix brief citing confirmed file:line causes) meant implementation had zero exploratory churn. Git hygiene around another workstream's 82 uncommitted files held through branch, commit, merge, and checkout transitions — the hard fencing in the briefs earned its length.
- **What I'd flag**: The orchestrator's own automation quality (Section 4's index-0 fallback) was below the bar the engineering-discipline doc sets for persona work. Orchestrator-authored scripts touch the same live systems; they deserve the same fail-closed standard.
- **Disagreement**: None; the PM's pre-merge test ask was reasonable but the review-history rule exists precisely to stop goalpost drift, and I'd have made the same call.

### UX Designer
- **User value assessment**: The doc test *is* UX work — it measures the product through a newcomer's eyes. The strongest UX outcome: failure states in the restore report went from seven identical opaque errors to distinguishable, named, per-key failures.
- **Session assessment**: The naive-operator protocol was honored strictly (no source reading, ambiguity logged as defect), which is what makes its passes meaningful.
- **What I'd flag**: The run confirmed the upload restore path still applies with no confirmation dialog (d20jf) — a live safety asymmetry an operator can hit today. It's filed, but it's the kind of one-screen fix that ages badly in a backlog.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: Two rounds caught user-relevant things: the pagination silent-miss (future false "not present on destination" failures) and the fixture-provenance discipline that makes the contract pins trustworthy.
- **Session assessment**: Review-history discipline worked as designed across rounds — the ceremony round was briefed with prior dispositions and mostly respected them.
- **What I'd flag**: In quick mode on downshifted models, two persona reports contained factual errors (one claimed the missing test "was addressed in this PR"; another mis-stated which restore paths differ). Synthesis caught both, but a synthesis layer that must fact-check its inputs is spending its budget on defense.
- **Disagreement**: With the PM on pre-merge test inclusion — resolved per protocol, recorded as dissent.

### Database Engineer
- **User value assessment**: Restore-path data integrity was the session's center of gravity; confirming the rollback engine worked cleanly during the original failure was as valuable as the fix itself.
- **Session assessment**: Sound. The export format decision (drop per-instance ids, resolve at apply time) was examined and endorsed rather than assumed.
- **What I'd flag**: My own review read the module's "settings are not rolled back" contract as run-level behavior and stated it in a way that contradicted observed evidence; synthesis reconciled the layers rather than accepting either claim. The open policy question (should one benign missing key abort a cross-instance restore?) is now parked on zt3kf where it belongs.
- **Disagreement**: With my own first framing — a useful reminder that module-contract comments describe a layer, not the system.

### SRE
- **User value assessment**: The operator at 3 AM gained: distinguishable failure classes, log/report separation that survived the fix, and — if nvhg7 lands — a periodic exercise habit for the one subsystem whose failure is unrecoverable.
- **Session assessment**: Three personas independently found the "must stay periodically exercised" declaration had no mechanism. Convergence like that is the ceremony's best output.
- **What I'd flag**: The test instance's session-expiry behavior burned automation time in every run (documented in the prompt, still costly). Also, the fix's fail-closed memoization means one transient network blip fails a whole apply run — correct, but the runbook should say "retry the restore" explicitly.
- **Disagreement**: None.

### QA Engineer
- **User value assessment**: The naive full walk is the highest-value test type this project runs — it validated 14 prior fixes and found the one failure class unit tests structurally couldn't (invented mocks matched the invented client).
- **Session assessment**: Good: red-first TDD on the fix, recorded fixtures, manual live proof. Honest gap: the live round-trip remains a manual, session-scoped act; nothing re-runs it.
- **What I'd flag**: My own team-review report asserted the missing PATCH-failure test had been addressed — it hadn't. That error came from summarizing test coverage by theme instead of checking for the specific test. Accountability noted; it's also evidence for the code reviewer's point about quick-mode reliability.
- **Disagreement**: With the architect's "contract breadth at any tier" position — I hold that at this tier, recording fixtures per-touched-path is the right pace; the sweep bead exists for the deliberate version of the bigger investment.

### Technical Writer
- **User value assessment**: The session both validated the operator guide (the passes are documentation value, recorded per-article) and found the developer docs actively wrong about the upstream API — the exact staleness that seeded the original bugs.
- **Session assessment**: The defect report was written incrementally with quoted claims and screenshots — the format future fixers need.
- **What I'd flag**: The run-4 report's reclassification note (R4-DOC-002 → product defect) was updated in place; good. But the incident's tribal knowledge (framework default behaviors that cost two bugs) lives only in code comments and this retro until r9oqx lands.
- **Disagreement**: None.

## Lessons

- **Keep**: The investigate-then-implement split with confirmed file:line root causes in the fix brief, and surfacing production-write authorizations *before* they block the pipeline. Also: fresh-artifact-before-apply when proving round-trips — it eliminated an entire staleness argument.
- **Stop**: Fallback-to-first-match element selection in automation that touches destructive-adjacent UI. Lookup failure must throw. Similarly, stop treating orchestrator-authored scripts as exempt from the fail-closed standards personas are held to.
- **Start**: (1) Declare the intended review depth at fix-authorization time ("single review" vs "full team") so ceremonies compose instead of stack. (2) Convert deferred-verification code comments into beads the day they're written. (3) Package the recurring doc-test rig as a versioned harness.
- **Value learning**: The product's most load-bearing promise (restore works) was the one place documentation quality and product quality diverged hardest — every read-side claim was accurate while the write-side act failed. Naive end-to-end walks that *perform* the promised act, not just read about it, are where the highest-severity findings live.
