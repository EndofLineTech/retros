# 2026-07-16-19 — event-sync-triple-ship

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~72 (12 genuine PO messages, ~25 background-task notifications, ~35 assistant turns)
- **SessionDepth**: deep — user log-bundle forensics, matcher/resolver/engine internals, two design spikes, three implementation dispatches, two ship flows, one CI incident diagnosis
- **Personas Active**: project-engineer (×3 build/ship dispatches), qa-engineer (verification spike), project-engineer-as-reviewer spike (read-only design spike), orchestrator; relevant but not dispatched: code-reviewer, SRE, technical-writer, UX
- **Beads Touched**: bead-yjchp (created→closed, PR #671), bead-jqwfq (created→closed, PR #673), bead-io0tv (created→closed, PR #674), created: bead-fw3ub (docs), bead-wawwh, bead-t6bin, bead-2ey2y; read: bead-dvgri

## Section 1: User Value Delivered

High and concrete. An end user supplied a debug bundle reporting "event matching doesn't run after M3U refresh" and "matching is still wrong sometimes." The session shipped three merged PRs into dev:

1. **PR #671 (build 0105)** — a venue-conflict rail that stops a *confirmed* wrong auto-attach (a 0.90-scoring pair for two different-venue events, reproduced byte-for-byte from the user's bundle); journal UI visibility for event-sync entries the backend was already writing (the user's exact complaint); a pre-flight message that stops advising an action that would break the user's other rule; bundle exports so the next "why didn't it fire" is diagnosable remotely.
2. **PR #673 (build 0107)** — a stale-name guard: dateless provider names left over from yesterday no longer auto-attach to today's events; they land in the review queue instead. Live-validated against real snapshot data before the behavior flip (41/41 detections verified, zero false demotes).
3. **PR #674 (build 0108)** — preferred-provider stream ordering on event-sync master channels, unlocking the user's stated plan ("get the desired stream at the top"), self-healing on every run.

Equally valuable: the diagnosis showed **the matcher was mostly right** — the "not running on refresh" complaints were a config knot (one group serving as master in one rule and secondary in another) and a not-opted-in rule, both invisible to the user. We relayed operator-actionable guidance that requires no code at all, plus one premise correction (stream order changes what plays, not which placeholder guide-data profile applies) that saved the user from building on a false assumption.

## Section 2: What We Did Well Together

The PO's decision turnaround was the best I've seen in this project. Every `DECISIONS NEEDED` block came back as terse numbered answers ("1. Yes. 2. Yes, do analysis/spike on this. 3. Yes, verification spike as well.") — zero ambiguity, zero re-litigating, which is exactly what the decision-vs-status discipline is for. The standout moment: when I presented ship-readiness for the third PR, the PO said "Before we ship, we need to check what the failures are on our CI/CD first." That gate was correct — it forced a real diagnosis (GitHub's 503 "Unicorn" page inside all four Docker build jobs) instead of shipping onto an assumed-healthy pipeline, and it turned a vague "some job failed" into a repaired image gap.

## Section 3: What the PO Could Improve

A second orchestrator session was running against the same main checkout throughout this one, and it cost real cycles twice: mid-way through the first build, the other session flipped the checkout from the engineer's branch to dev and pulled (the engineer burned a recovery cycle re-pointing its branch and re-running a contaminated test suite), and later the checkout sat on that session's feature branch at the exact moment I needed to dispatch the next build — forcing a switch to worktree isolation that this project's own docs describe as buggy (cwd-trap). Both were absorbed, but the standing default here is "non-worktree, sequential engineers" precisely because the worktree mechanism is unreliable — running two uncoordinated writer sessions against one checkout silently invalidates that default. If parallel sessions are going to be normal, say so at session start so isolation is chosen deliberately instead of discovered mid-dispatch.

## Section 4: What the Agent Got Wrong

My first independent backend gate verification ran with the ambient system Python instead of the project venv — 9 tests failed on a stale `cryptography` library, and I initially treated it as a discrepancy against the engineer's report. Worse, my follow-up "list the failures" re-run launched from the wrong cwd (the harness had silently reset it), producing an empty, invalid result I had to kill. Two full-suite runs (~10 minutes) and one misleading status line to the PO ("9 failures — need to see which tests") before I found the interpreter mismatch. The engineer had it right all along. I encoded the venv-python requirement into every subsequent dispatch brief, but the orchestrator's own verification path should have used the project interpreter from the first run — the information (`.venv/` in the repo root) was one `ls` away. Same session, smaller version of the same sloppiness: I armed a backoff timer whose deadline was recomputed every loop iteration (would never fire) and had to kill and rewrite it.

## Section 5: What Would Make the Project Better

Pin the gates in a script. Three separate actors this session (orchestrator twice, engineers by instruction) had to *know* that the backend suite only passes under `.venv/bin/python`, that frontend tools need a specific fnm path, and that worktrees need a bootstrap script first. A single `scripts/gate.sh` (or `make verify`) that selects the right interpreter and tool paths itself would have prevented the Section-4 failure outright and shrunk every dispatch brief. Related, smaller: the hook suite showed three frictions in one session — the version-advance guard false-blocks worktree ships unless `git -C` is embedded in the same command (bit both ship flows), the bead-commit regex never matches this repo's non-numeric bead suffixes (guard is inert), and the persona-bead firewall allowed an engineer's `bd update --notes` early in the session but blocked the identical call later (inconsistent, confused the agent). The hooks need a self-test pass; a bead for guard self-testing already exists.

## Persona Perspectives

### Project Engineer
- **User value assessment**: All three builds trace directly to field-reported harm or a stated user plan. Nothing speculative shipped.
- **Session assessment**: Briefs were unusually good — the spike-first pattern meant the jqwfq build brief contained verified facts (which metadata is useless, which store answers the question) instead of hypotheses. Post-rebase gate re-runs before push were enforced both ships.
- **What I'd flag**: The worktree cwd-trap briefing worked, but only because it was pasted verbatim into every brief. That knowledge should live in tooling, not prompts.
- **Disagreement**: None with the approach; mild pushback on Section 4 — the orchestrator's venv miss was also *my* report's fault for not stating which interpreter I used the first time.

### QA Engineer
- **User value assessment**: The live-validation stage boundary in the stale-guard design is exactly right: the heuristic was proven against real provider data (41/41, spot-checked) *before* it could change attach behavior for users.
- **Session assessment**: Strong. My spike's "unpinned load-bearing side effect" finding (skipped merges registering channels for reorder) got a regression test in the very next build rather than being filed and forgotten.
- **What I'd flag**: The venue rail and stale guard both demote to the review queue — nobody measured the combined queue volume a heavy provider will now generate per day. If it's dozens of rows daily, users will stop reading the queue, and the safety mechanism degrades to a write-only log.
- **Disagreement**: With the engineer's default-on choice for the stale guard knob — defensible, but it shifts burden to exactly the recurring-event users bead-t6bin says will suffer daily re-accept churn. I'd have wanted the churn bead fixed *first* or the default flipped after the first field complaint.

### Code Reviewer
- **User value assessment**: The rails protect users from silent wrong attaches — real quality, not aesthetics.
- **Session assessment**: Here's my problem: three PRs merged and **no dedicated code-review pass happened on any of them**. The orchestrator read the first PR's core diff and CI ran (CodeQL, Semgrep, full suites), but PRs #673 and #674 merged on engineer report + independent gate re-runs alone. Gates prove the tests pass; they don't review the tests.
- **What I'd flag**: The shipped-in-one-day cadence was possible partly because review was skipped. The venue rail's stop-token list and the staleness module's timezone boundary are exactly the kind of code where a reviewer finds the edge the author didn't test.
- **Disagreement**: With the PM's velocity framing below. One of these days a "gates green, live-verified" build will carry a design flaw the gates can't see, and this session set the precedent that review is optional when the PO says ship.

### SRE
- **User value assessment**: The CI diagnosis protected users concretely — without it, image-pulling users would have gotten stale builds while `dev` claimed three shipped fixes.
- **Session assessment**: Good incident handling: exact failing step identified (GitHub's own 503 page inside the metadata action), correct call that it was platform-not-us, re-run only after diagnosis, watcher to completion.
- **What I'd flag**: A *non-required* Docker build job failed at merge time and nothing alerted anyone — it was caught because an engineer happened to read the checks list. Publish-path jobs that gate user-visible artifacts should either be required or should page/notify on failure.
- **Disagreement**: None.

### Project Manager
- **User value assessment**: Three user-tracing ships in one session, each behind an explicit PO decision, all beads closed with evidence. The board is honest.
- **Session assessment**: The decision cadence was the enabler. But note the ratio: three beads closed, five created (docs, rename-blindness, churn, pre-flight warning, plus the out-of-repo hook fix left homeless). Backlog grew net +2 while feeling like a cleanup session.
- **What I'd flag**: The docs bead from ship #1 (bead-fw3ub) is still open while two more shipped features added user-facing behavior. Docs debt compounds quietly.
- **Disagreement**: With the code reviewer — the two later PRs were spike-designed, corpus-pinned, live-verified, and independently gate-checked; calling that "review skipped" undersells the controls that *did* run. I'd accept the risk trade again for field-bug fixes.

### Technical Writer
- **User value assessment**: Mixed. Ships #2 and #3 included docs updates in-PR (good), but ship #1's rail — the one that changes which matches silently stop attaching — is still undocumented (bead-fw3ub). An operator seeing new "venue_token_conflict" rows in their review queue today has no doc explaining why.
- **Session assessment**: The bundle-export additions are documentation in the best sense: the next debug bundle answers "why didn't my rule fire" by itself.
- **What I'd flag**: Prioritize bead-fw3ub before the next release note goes out; the release notes will otherwise describe behavior the user guide contradicts.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: The premise correction relayed to the user (stream order ≠ placeholder guide-data selection) is the highest-leverage UX act of the session — it stopped them from building a workflow on a false mental model.
- **Session assessment**: Good instinct throughout to prefer "demote to review" over "hard reject" — keeps the human in the loop.
- **What I'd flag**: Echoing QA — two new demotion sources feed a review queue whose accept decisions expire daily for recurring events (bead-t6bin). That's a treadmill, and treadmills train users to rubber-stamp. The fingerprint design decision is genuinely hard, but it's now the UX bottleneck of the whole event-sync feature.
- **Disagreement**: With the default-on knob, same grounds as QA.

### Security Engineer
- **User value assessment**: Low-stakes session for security; nothing shipped touches auth or input trust boundaries beyond name parsing that already existed.
- **Session assessment**: Appropriate care in the right places: the debug bundle analyzed was already provider-redacted; the test fixture captured from a live DB was specified as redacted/truncated in the brief.
- **What I'd flag**: Verify the committed snapshot fixture actually got the redaction pass the brief allowed for — provider group names and stream names are low-sensitivity but the check should be explicit, not assumed. One-minute review against the fixture file.
- **Disagreement**: None.

### Database Engineer
- **User value assessment**: The staleness lookup rides an existing snapshot store with one query per account and a bounded JSON parse — user-facing latency respected, no new tables, no migration.
- **Session assessment**: Sound. The spike correctly rejected a design (per-stream age knob) that the data model cannot support rather than bending the model to the feature.
- **What I'd flag**: The 500-names-per-group snapshot cap is now load-bearing for a matching behavior. If someone raises the cap or changes capture cadence for unrelated reasons, they won't know a matcher heuristic depends on its semantics — the fail-open test pins behavior, but a comment at the cap's definition should point back.
- **Disagreement**: None.

### IT Architect
- **User value assessment**: The demote-to-review band is becoming the system's uniform answer to uncertainty (venue conflict, staleness, score band) — a coherent architecture users can build a mental model around.
- **Session assessment**: Good precedent discipline: both new rails mirrored the existing one instead of inventing parallel mechanisms; the ordering feature reused dormant columns instead of adding config surface.
- **What I'd flag**: Three uncertainty sources now converge on one queue with one fingerprint scheme designed for none of them specifically. Before a fourth rail lands, the review-queue identity model (bead-t6bin) deserves a real design pass — it's the convergence point.
- **Disagreement**: None.

## Lessons

- **Keep**: Spike-before-build for anything touching matching behavior. Both spikes converted "plausible designs" into "verified facts + one recommendation," and both builds shipped without design rework. Also keep: independent gate re-runs and merge-state verification before telling the PO anything is done — the trust-nothing rule cost ~10 minutes per ship and made every "shipped" claim checkable.
- **Stop**: Running verification gates with whatever interpreter/tooling is ambient. The environment is part of the gate. (Also stop letting two writer sessions share a checkout without an explicit plan.)
- **Start**: A repo-pinned gate script that encodes interpreter, tool paths, and worktree bootstrap — so orchestrator and every dispatched agent run the *same* gate by construction. And schedule a dedicated code-review pass when a session ships more than one PR, even when each individually looked low-risk.
- **Value learning**: The user said "matching doesn't work." The matcher was almost entirely correct — what was broken was *visibility*: silently skipped rules, journal entries the UI couldn't show, a review queue doing its job invisibly, and advice that would have broken a second rule. For mature features, "it doesn't work" reports are more often observability gaps than logic bugs — diagnose from the user's artifacts before touching the algorithm.
