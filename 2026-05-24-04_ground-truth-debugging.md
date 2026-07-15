# 2026-05-24-04 — ground-truth-debugging

- **ModelID**: claude-opus-4-7
- **TurnCount**: ~250+ across the full (compacted + visible) session; this continuation arc was ~45 assistant turns. The 0emgo.4 root cause was not found until ~turn 30 of the continuation — deep into the session, which is exactly where this retro's central failure lives.
- **SessionDepth**: deep — full data-path trace of the auto-creation engine/executor/model/router/MCP-formatter, a DB record autopsy, an Alembic migration with drift-test reconciliation, two ship cycles, epic closeout.
- **Personas Active**: Project Engineer, Database Engineer, QA Engineer, Code Reviewer, SRE, Technical Writer, IT Architect, Project Manager (Security & UX consulted lightly)
- **Beads Touched**: project-d-0emgo.4 (closed), project-d-0emgo.6 (closed), project-d-0emgo (epic, closed). Earlier-in-session children .1/.2/.3/.5 already shipped.

## Section 1: User Value Delivered

Real, shipped value to two user classes:

1. **Operators reading auto-creation run summaries** now get honest numbers. Before, a dry-run reported `Channels touched: 0` while merging 26 streams — a number that was actively misleading (it told an operator nothing happened when something did). After 0emgo.4: `Stream merges: 26, Channels touched: 26`, and `Channels updated` is reserved for genuine property updates. A run summary an operator can trust is the whole point of a run summary.

2. **Agents driving ECM via MCP** now get an unambiguous `list_channels` line — `#10440 (id=12319): name` — so the API id used by mutate/lookup tools isn't confused with the channel `#number`. This is a correctness fix for the most common foot-gun in agent-driven channel ops.

Both shipped to `dev` (PRs #425, #426), all required gates green, live-verified against the running container. The 0emgo epic — six production-reported auto-creation bugs — is fully closed. This session moved real user outcomes, not just code.

## Section 2: What We Did Well Together

The PO's standing authorizations made the debugging arc possible without stalls. Earlier in the session the PO had said "real world changes are fine, it's a dev instance" and "work on your suggested order." That meant when 0emgo.4 required repeated deploy → 236-second live dry-run → DB autopsy → redeploy cycles, I could run them autonomously without round-tripping for permission on each container mutation. The single most valuable moment was being able to query the live `/config/journal.db` execution record directly — the PO's blanket dev-instance authorization is what let that happen, and that query is what finally cracked the bug.

## Section 3: What the PO Could Improve

The PO accepted three prior agent attempts at 0emgo.4 (commits b6445b25, 712da198, 56a18373, each titled "fix… channels_touched") without a ground-truth verification gate — so the branch accumulated three commits that each *claimed* to fix the count and none of which did. Each prior agent reported success from passing unit tests; none was required to prove the fix against the live persisted record. A standing rule — "a counter/reporting fix is not 'done' until you show the value correct in the live surface it's reported on, not just in a unit test" — would have surfaced the persistence bug on attempt 1 instead of attempt 4. The PO's own global CLAUDE.md already says "do not trust agent-reported gate status for high-stakes work"; this is the same lesson, and it wasn't applied to the sub-agents that touched 0emgo.4.

(This is genuinely the only PO-side friction in the continuation — the PO was appropriately hands-off and the autonomy worked.)

## Section 4: What the Agent Got Wrong

I repeated the exact failure mode I was sent to fix. After reading the executor code, I declared the `add_result` chokepoint refactor **"the correct fix and I've fully diagnosed it"** and **"guaranteed correct"** — then deployed it, ran the 236s dry-run, and it *still* showed `Channels touched: 0`. I diagnosed from code-reading, not from ground truth, and over-claimed before verifying — the same overconfidence that produced the three prior dead-end commits.

The ground truth was available the entire time and cost one cheap query: the `auto_creation_executions` DB record. The moment I finally inspected it (modified_entities had 26 non-None channel ids; the table had no `channels_touched` column at all), the real bug — missing **persistence**, not missing **computation** — was obvious in seconds. I should have done the data autopsy *first*, before touching a line of executor code. Instead I burned a full refactor + deploy + 4-minute dry-run on a hypothesis. The refactor turned out to be defensible hygiene and shipped as part of the fix, but it was orthogonal to the actual defect; I got lucky that it was harmless, not right that I led with it.

Secondary miss: when the refactor's first run showed 0, my first reach was a boot-race / stale-deploy hypothesis (container started 04:10:40, run fired 04:10:50). I checked it and it was wrong. Reasonable to check, but it shows I was still preferring "the deploy didn't take" over "my fix is incomplete" — looking outward before looking at my own diagnosis.

## Section 5: What Would Make the Project Better

`auto_creation_executions` persistence is **manual and unguarded**: the engine finalize block hand-copies each `results[...]` counter to a column (`execution.streams_merged = results["streams_merged"]`, etc.). Adding a new key to the `results` dict silently does *not* persist — which is precisely how `channels_touched` was computed correctly for weeks and dropped on every save. Two structural fixes would prevent recurrence:

1. **A parity test**: assert that every numeric counter key in the engine's `results` dict has a corresponding column on `AutoCreationExecution` (and is written in finalize). This is the test that would have failed loudly the day `channels_touched` was added to the dict.
2. **A faster live-like loop**: the only way to reproduce this bug was a 236-second full dry-run against 2,682 streams. Unit tests passed *because* they exercise the in-memory `results` dict and never the persist→serialize→format path. An integration test that runs a small fixtured dry-run end-to-end through the DB record + `to_dict()` would catch this class in CI in seconds, and would also be a sane debugging harness.

The deeper pattern worth a memory entry: **for any "value is wrong in a live surface but tests pass" bug, trace compute → persist → serialize → format and inspect the persisted artifact before editing the compute layer.** Passing unit tests next to a wrong live value is itself the diagnostic signal — it localizes the bug to the layers the tests don't cross.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: High — both fixes deliver trustworthy run reporting and unambiguous channel identifiers, live-verified.
- **Session assessment**: The implementation that shipped is sound: chokepoint counting at `add_result`, column + migration + persistence, clean squashed history, all gates green on both PRs.
- **What I'd flag**: The path to that implementation wasted a full deploy/verify cycle on a hypothesis I should have falsified with a DB query first. Right destination, inefficient route.
- **Disagreement**: With the QA Engineer — I'd argue the shipped code is correct and tested; QA correctly counters that "tested" meant "in-memory tested," which is the whole reason the bug survived to production.

### Database Engineer
- **User value assessment**: The fix is fundamentally a data-layer fix, and it serves users directly — the metric now survives the write.
- **Session assessment**: Migration 0021 was done to convention: idempotent column-add, `server_default="0"` to match the ORM so the baseline drift test stays clean, fast-path smoke test updated with the manual ALTER (auto_creation_executions is a 0001 baseline table, so `create_all()` can't materialize the column). Correct and complete.
- **What I'd flag**: Hand-copying counters in the finalize block is a two-sources-of-truth smell (results dict vs. ORM columns). This *will* happen again without the parity test.
- **Disagreement**: With the IT Architect's framing that this is "just a smell" — from where I sit it's a recurring defect generator, not an aesthetic concern.

### QA Engineer
- **User value assessment**: The testing gap directly cost users — a wrong number reached the live surface and three fix attempts couldn't dislodge it because every test was green.
- **Session assessment**: This is the session's central QA lesson. 201 auto-creation tests + a dedicated `channels_touched` test class all passed while the live value was 0. The tests asserted `results["channels_touched"]` in memory and the executor chokepoint — they never round-tripped through the persisted record + `to_dict()` + MCP formatter, which is exactly where the bug lived.
- **What I'd flag**: We need one integration test that runs a dry-run and reads the *persisted* `channels_touched`, plus the results→columns parity test. Without them, the next dropped counter is invisible again.
- **Disagreement**: With Project Engineer's "sound and tested" — green unit tests next to a broken live value is a test-strategy failure, not a vindication.

### Code Reviewer
- **User value assessment**: The chokepoint refactor improves correctness durability (one place counts merges, can't drift), which protects the user-facing number long-term.
- **Session assessment**: Good: removed the vestigial `_channels_touched_ids`, kept the diff honest, squashed three misleading attempt-commits into one accurate commit with a root-cause-first message, verified the squash tree was byte-identical before force-push.
- **What I'd flag**: The refactor was presented mid-session as "the fix" when it was orthogonal to the defect. In review I'd want the commit message to not overstate the chokepoint's role — and it doesn't, which is good, but the in-session narrative did.
- **Disagreement**: None material; aligns with QA on the test gap.

### SRE
- **User value assessment**: This is squarely an observability bug — the system under-reported a real operational metric, which erodes trust in every other number in the summary.
- **Session assessment**: The "computed correctly, persisted nowhere" failure class is insidious precisely because nothing errors. Good that we traced it to the persistence boundary and live-verified the fix on the running container.
- **What I'd flag**: Any metric that travels compute → persist → surface needs a guard at the persist boundary. The parity test is an SRE concern as much as a QA one.
- **Disagreement**: None.

### Technical Writer
- **User value assessment**: The knowledge capture serves the next engineer who hits a dropped counter — the commit messages, bead notes, and migration docstring all state the real root cause (persistence, not compute) plainly.
- **Session assessment**: Strong. The 0emgo.4 bead note and PR #425 body explicitly correct the misleading prior framing; the epic closeout note summarizes all six bugs with PR references.
- **What I'd flag**: The root-cause narrative lives in commit/bead text; it should also become a memory entry so it's greppable outside git archaeology.
- **Disagreement**: None.

### IT Architect
- **User value assessment**: Neutral-to-positive — the fix serves users; the architecture concern is about preventing the next regression.
- **Session assessment**: The results-dict / ORM-column duality is an architectural seam that requires manual synchronization on every counter addition. It worked here but is load-bearing on developer memory.
- **What I'd flag**: Consider a single mapping (results key → column) or a serialization helper so the seam can't silently drift.
- **Disagreement**: With Database Engineer on severity framing (smell vs. defect generator) — I'd accept the parity test as a pragmatic guard short of refactoring the seam.

### Project Manager
- **User value assessment**: Two beads shipped + a six-issue epic closed in one continuation — strong throughput on user-facing outcomes.
- **Session assessment**: Good delivery discipline (both PRs driven to merge, gates verified explicitly, beads closed only after merge). But the 0emgo.4 line item carried three prior failed attempts before this one landed — that's accumulated rework that a verification gate would have prevented.
- **What I'd flag**: The over-claim-then-discover-it's-not-fixed loop is a delivery risk; it inflates apparent progress. "Live-verified against the reported surface" should gate any reporting-bug closure.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: No new security surface this continuation. The earlier 0emgo.5 journal entries correctly carry stream IDs only (no provider-credential-bearing URLs), which I'd reaffirm.
- **Session assessment**: Clean — DB autopsy and container deploys were on the dev instance per standing authorization; no secrets touched (public repo discipline held).
- **What I'd flag**: Nothing this session.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: 0emgo.6 is a genuine usability fix for the agent-as-user — leading with both labeled identifiers removes a real confusion, not a hypothetical one (the bead documents callers actually tripping on it).
- **Session assessment**: Scope was held tightly (compact format untouched, `create_channel`'s separate line left alone) — correct restraint.
- **What I'd flag**: Worth confirming the `—` em-dash and labeled format render cleanly in all consumer terminals; minor.
- **Disagreement**: None.

## Lessons

- **Keep**: Going to ground truth — querying the live persisted record and tracing the full data path — when a live value contradicts passing tests. It cracked a bug three prior attempts couldn't, and it cost one cheap query.
- **Stop**: Declaring a fix "guaranteed correct" / "fully diagnosed" from code-reading before live-verifying, and deploying theory-based fixes before inspecting the actual data. I did this and reproduced the very failure mode I was dispatched to fix.
- **Start**: For any counter/reporting bug, inspect the persisted artifact FIRST and trace compute → persist → serialize → format before editing the compute layer. Add a parity test asserting every `results` counter has a persisted column, plus one end-to-end dry-run integration test that reads the persisted value.
- **Value learning**: A number that is computed correctly but persisted or surfaced wrong is exactly as harmful to the user as one computed wrong — arguably worse, because the code *looks* right and the tests *are* green. "Correct" for a reported metric means correct at the surface the user reads, not correct in memory.
