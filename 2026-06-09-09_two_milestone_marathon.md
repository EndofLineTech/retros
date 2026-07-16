# 2026-06-09-09 — two_milestone_marathon

- **ModelID**: claude-opus-4-8 (1M context)
- **TurnCount**: ~85 messages (deep session; estimate — many multi-tool assistant turns). Notably, every blocking defect this session was caught *after* turn 40 — in the M27 build, not the planning.
- **SessionDepth**: deep — two milestones planned + groomed, one spike, two milestones built-and-merged (M26, M27), plus diagnosis of a live bug and planning of two more milestones (M28/M29). Spanned the full stack: Alembic migration, scanner internals, destructive DB ops, FastAPI endpoints, React UI, in-Docker web gate.
- **Personas Active**: all ten. Grooming spawned all 10; the spike used database-engineer (lead) + project-engineer + it-architect; M26/M27 reviews used code-reviewer, qa-engineer, database-engineer, security-engineer; project-engineer drove every build pass.
- **Beads Touched**: M26 epic `project-c-4jp` + children `4jp.1–4jp.6` (closed); M27 epic `project-c-p03` + children `p03.1–p03.13` incl. spike `p03.5` (impl closed, nits `.11/.12/.13` open); M28 epic `project-c-fow` + `fow.1–fow.3` (planned); M29 epic `project-c-68t` + `68t.1–68t.3` (planned); `project-c-d15` (optional tag-writeback, open).

---

## 1. User Value Delivered

Real, shipped value — two milestones merged to `dev`, both tracing to features the PO explicitly asked for.

- **M26 (v1.0.0-0003)** gave listeners the table-stakes player controls that were missing: repeat one/all, shuffle (album / artist / whole library), per-track technical info (bitrate/codec/size), and richer album info (genre, disc count, MBID). These are things a user touches every listening session.
- **M27 (v1.0.0-0004)** solved the PO's concrete library-management pain: albums that don't auto-group could not be fixed at all before this. Now an operator can merge, reassign, and split albums, edit album descriptions, and *discover* mis-grouped albums — and critically, those manual fixes **survive a rescan** (the whole reason the spike existed).
- **A real, silent bug was correctly diagnosed**: Top Tracks/Artists never populate because the web player never POSTs `/play-history`. We didn't fix it this session, but the diagnosis is precise and the fix is scoped in `fow.1` — that's value, not make-work, because the PO had been staring at empty stats with no idea why.
- **Avoided make-work**: investigation revealed the PO's "I can't manually refresh IPTV" was not a bug (the feature exists) — so we did *not* build a duplicate. The session redirected that energy to the real gap (IPTV groups).

No work was created that doesn't serve a user. The deferred nit beads (`p03.11/.12/.13`) are genuine backlog, honestly classified as non-blocking.

## 2. What We Did Well Together

**The grooming → spike → build sequence on M27's durability problem.** During grooming, four independent personas (code-reviewer, architect, DBA, engineer) each surfaced the same load-bearing risk: a manual album merge would be silently undone by the next library scan. Rather than letting me guess a mechanism, the PO answered the durability question with **"Run a spike first."** That was the right call and it paid off concretely: the 3-persona spike produced one specific design (`tracks.album_pinned`, DB-authoritative grouping, ADR 0008), and that design then drove all three M27 build passes without a single rework of the *mechanism*. The destructive merge/split ops, the scanner guard, and the durability test all fell out of one correct decision made at the right moment. Choosing investigation over speed at a genuine fork is exactly when a spike earns its cost.

## 3. What the PO Could Improve

**The IPTV item was framed as a bug that turned out to be a different, unstated need — and the real need only surfaced after I'd spent an investigation confirming the "bug" didn't exist.** The PO listed "I can't manually refresh an M3U/XC account" as bug #2. I dispatched an Explore agent, which found the manual-refresh endpoint *and* the UI button already exist and work. Only when I reported that back and asked "what's the actual gap?" did the PO reveal the real problem: *"I don't see any way to enable any groups and create any IPTV groups/station groups."* That's an entirely different feature (channel categorization), and it's the one that became M29's centerpiece.

The cost was a wasted investigation branch and a round-trip. If the original framing had been "my IPTV channels aren't organized into groups in the client" instead of "I can't manually refresh," we'd have pointed the investigation at the right subsystem (the provider category surface) from the start. The lesson isn't "give more context" — it's that **a symptom reported as the wrong mechanism sends the investigation in the wrong direction.** When something feels missing, describing the *outcome you can't achieve* ("channels aren't grouped") beats guessing the *mechanism* ("refresh is broken").

A smaller, second instance: answering the artist-shuffle question with **"Why not both? Give the option."** A binary with materially different implementation costs was resolved by deferring the choice back to me. It worked out (a shuffle menu), but it's the "just do what makes sense" pattern — the two options had different downstream UI surface and I had to commit to a design the PO hadn't actually seen.

## 4. What the Agent Got Wrong

**I claimed I would "verify independently" but hadn't ensured I had the means to — my first independent M27 Pass A test run failed because the engineer had torn down its ephemeral test database, and I only then scrambled to provision one.** The Pass A engineer's report ended with "run alone on dedicated ephemeral PG/Redis now removed." I launched my independent full suite against the worktree — and it died at DB provisioning (`Connect call failed ('[IP]', 5432)`). I had to diagnose the failure, find the project's test-DB convention, stand up `project-c-m27test-pg`/`-redis` myself, *then* run the verification. The verification discipline was sound; the *foresight* wasn't. "Engineer removed its DB" was right there in the report — I should have provisioned the controlled DB before declaring I'd independently verify, not after a failed run forced it.

Secondary, and more embarrassing for a session that was heavy on environment-control discipline: **I twice killed my own shell with `pkill -f` patterns that matched the very command running them** (once `"seq 1 120"`, once `"m27-album-metadata-matching/.venv/bin/pytest"`), each time exiting 144 mid-cleanup. Recovered immediately both times, but it's sloppy — `pkill -f` matches its own argv, and I knew that the second time.

## 5. What Would Make the Project Better

**A standard, isolated "owned test DB per verification" harness — because test-DB contention caused a recurring class of false signals and wasted cycles across both milestones.** The pattern repeated three distinct ways this session:
1. In M26, a **code-reviewer ran its own pytest against the same test DB concurrently with mine**, corrupting both runs (the "rows committed by fixtures invisible" symptom).
2. In M27 Pass B, the **engineer's own runs orphaned pytest processes deadlocked on the shared DB for ~30 minutes**, which it had to detect via `ps` and kill.
3. My **first Pass A verification failed entirely** because there was no DB to connect to.

Each was individually survivable, but together they cost real time and produced at least one near-miss false signal. The project already has `_dbprovision.py` (unique DB per run) and a `test-db-clean` Make target — the missing piece is a **convention that every verification (orchestrator or reviewer) runs against a dedicated, single-occupant Postgres on a known port, with explicit "this DB is mine for the duration" ownership.** Codify it (a `make verify-db` that brings up a per-worktree container on a free port, plus a rule that reviewers never touch the orchestrator's DB) and an entire category of "is this failure real or contention?" disappears. This is the highest-leverage process fix the session surfaced.

## 6. Persona Perspectives

### Database Engineer
- **User value assessment**: High. The Pass A migration is the substrate every album-metadata feature stands on, and the durability column (`album_pinned`) is *the* thing that makes manual matching actually usable rather than a fix that evaporates on the next scan.
- **Session assessment**: My review was the standout catch of the session — I traced a **blocking cascade** in Pass A that the code-reviewer rated non-blocking: the resurrection fix made an MBID-index collision reachable, and the scanner's per-file `except` doesn't roll back, so one stray file poisons the session and aborts the *entire* scan. That's a bug every operator who removes-then-re-adds an album would eventually hit.
- **What I'd flag**: The `uq_albums_musicbrainz_release_id` index is still unfiltered by `deleted_at` — correctly deferred (merge NULLs the loser's MBID), but it's a standing invariant Pass B had to honor and future work must keep honoring.
- **Disagreement**: I explicitly diverged from the code-reviewer, who saw the same MBID collision and rated it non-blocking/documented. The orchestrator adjudicated in my favor after reading the scan loop. That divergence *was the review working* — averaging us would have shipped a scan-aborting bug.

### Security Engineer
- **User value assessment**: Direct user protection. Without my pass, the merge/split endpoints could have shipped reachable by any authenticated listener — destructive library surgery as a broken-access-control hole. I verified all three (plus the new PATCH) gate on `get_current_admin_user` with direct 403 negative tests.
- **Session assessment**: Heard and acted on. The description-edit XSS posture (React-escaped, no markdown, CSP backstop, `max_length`) was verified concretely with an injection regression test.
- **What I'd flag**: The pre-existing, codebase-wide question — `get_current_admin_user` does **not** block app-password tokens, so an admin's leaked `adn-` token reaches every destructive admin endpoint. Not an M27 regression; filed as a PO-facing note in `p03.12`. It should not stay un-decided forever.
- **Disagreement**: I'd push to JWT-gate the whole admin surface; the engineer/architect view is "consistent with existing contract, defer." I conceded it's not an M27 blocker, but I don't think "it's already inconsistent everywhere" is a permanent answer.

### IT Architect
- **User value assessment**: The source-of-truth framing I brought to the spike (DB authoritative for *operator-decided grouping*, files authoritative for *content*) is what kept the durability design coherent and rejected tag-writeback — which would have failed on read-only mounts and couldn't express a same-title split.
- **Session assessment**: Trade-offs were explicit and the provider-surface boundary held — I verified all of M26/M27 is `/api/v1`-only with zero project-a/Xtream contract impact, which de-risked the whole effort.
- **What I'd flag**: M29's IPTV-groups work *will* touch the project-a category contract — that's a different risk class than M27 and needs me in the loop early.
- **Disagreement**: None material this session; the DB-only mechanism I argued for is what shipped.

### Project Manager
- **User value assessment**: Two milestones shipped, both PO-requested, no make-work. The decomposition I pushed in grooming (splitting the monolithic `p03.3` matching bead into merge/reassign/split, adding the discovery bead) made the build tractable and reviewable.
- **Session assessment**: Work was organized into clean passes (A/B/C) with explicit gates between. Sequencing held.
- **What I'd flag**: This was a *very* long single session — two full build-and-merge cycles plus planning for two more milestones. The risk of marathon sessions is that fatigue-class errors (the pkill self-kills, the out-of-dependency-order bead closes) creep in late. Consider checkpointing at milestone boundaries.
- **Disagreement**: I'd gently push against the QA-engineer's implicit "ship it, the tests are thorough" — the *volume* of test coverage was high, but two real assertion gaps still slipped to the orchestrator's independent run and the Pass C review. Coverage count ≠ coverage quality.

### Project Engineer
- **User value assessment**: I delivered the features the PO asked for, end to end, with TDD and honest gate reporting (including disclosing my own orphaned-pytest contention in Pass B rather than hiding it).
- **Session assessment**: Disciplined overall — the savepoint fix, the direct-pin foot-gun avoidance, the durability test all landed correctly.
- **What I'd flag**: One of my passes opened its report with a **confabulation** — "all five sub-tasks were already implemented by a prior interrupted session" — when the commits were my own work. The orchestrator caught it (the branch was created from `dev` minutes earlier; there *was* no prior session) and verified everything anyway. That kind of narrative invention erodes trust even when the code is correct.
- **Disagreement**: I'd defend deferring the N+1 `by_id` loops and the app-password question as genuinely non-blocking at single-operator scale — and the reviewers agreed.

### UX Designer
- **User value assessment**: The matching UI solves a real operator problem, and the discovery surface (which I argued for in grooming) is what makes it *findable* rather than an escape hatch nobody discovers.
- **Session assessment**: My grooming flow-specs were carried into the build, and the destructive-action confirmations + 409-force flow shipped.
- **What I'd flag**: The matching UI shipped **functional-first without the dedicated design pass I flagged as warranted in grooming**. The merge/reassign/split flows interact (tag write-back? cover art ownership? playlist/history on merge?) and those were resolved implicitly by the engineer rather than designed. It works, but "works" and "the operator understands what's about to happen to their library" are different bars for a destructive feature.
- **Disagreement**: I'm less sanguine than the PM/engineer that the matching UI is *done*. It's shippable; it isn't polished. The `AlbumPicker`-bounded-at-20 and the un-tested `AdminAlbumProblems` merge loop are symptoms of UI built under build-velocity pressure.

### Code Reviewer
- **User value assessment**: My catches protected users from real bugs — the Pass A "pinned track via `TrackRepo.upsert` is a silent no-op" foot-gun would have made Pass B's reassign silently fail to move tracks, the worst kind of destructive-op bug (looks like it worked, didn't).
- **Session assessment**: Strong. Call-chain analysis caught the foot-gun; I verified the production-path tests genuinely exercise the scan loop, not leaves.
- **What I'd flag**: I rated the MBID-collision scan error as non-blocking and the DBA rated its *cascade* as blocking — the DBA was right because I stopped at "one file errors" without tracing the session-poisoning propagation. I under-traced the failure mode.
- **Disagreement**: Resolved against me on the cascade. Good — that's the second reviewer earning their seat.

### Database Engineer vs Code Reviewer (the session's key disagreement)
This is worth its own line: the same MBID-index collision was seen by both, rated **non-blocking** by code-review and **blocking** by the DBA. The difference was depth of failure-propagation tracing — the DBA followed the poisoned session to a full scan abort; the code-reviewer stopped at the per-file error. The orchestrator independently read the scan loop and confirmed the DBA. **Two reviewers with different depth produced the catch; one reviewer would have shipped it.**

### SRE
- **User value assessment**: The savepoint fix directly protects the user's most important operation — a full library scan no longer aborts because of one bad file. And the `/play-history` diagnosis identified why a shipped feature (stats) is silently dead.
- **Session assessment**: Reliability got real attention — the per-file savepoint, the soft-delete/tombstone correctness for project-a sync, the `random-tracks` cap.
- **What I'd flag**: The recurring test-DB contention (Section 5) is an *operational* smell in the dev workflow itself. We kept stepping on a shared resource — the same class of incident the global CLAUDE.md "control the environment" rule was written about (M21b). We followed the rule reactively each time; we don't yet have the tooling to prevent it.
- **Disagreement**: None, but I'd amplify the PM's marathon-session caution — late-session resource-handling errors are exactly the on-call-at-3am failure pattern.

### QA Engineer
- **User value assessment**: My Pass C catch — the merge/split web tests asserted the `force` flag but **not the POST URL/body** — guarded against a catastrophic source/target inversion (merging the wrong direction destroys the wrong album). That's a user-data-loss-class bug a green test would have hidden.
- **Session assessment**: The destructive-op test matrices (idempotency, concurrency, durability-survives-rescan) were genuinely thorough.
- **What I'd flag**: A flaky test (`by_artist_and_title` determinism, same-transaction `created_at` tie → random UUID tiebreak) **passed in the engineer's run and failed in the orchestrator's independent run** — it should have been caught in *my* Pass A review sweep, not by the orchestrator's full run. My flakiness sweep ran *after* the orchestrator already found it.
- **Disagreement**: I'll push back on any read that "QA was thorough so quality was high" — two real gaps (the flaky test, the missing URL/body assertions) reached late stages. Thorough volume, imperfect net.

### Technical Writer
- **User value assessment**: ADR 0008 captures *why* manual grouping is DB-authoritative — that decision is non-obvious and would have been re-litigated without the record. OpenAPI snapshots stayed current for project-a/third-party clients.
- **Session assessment**: Decisions were captured (ADR, dense bead descriptions, the spike close-out).
- **What I'd flag**: **The operator runbook for album matching never got written.** In grooming I flagged `runbooks/album-matching.md` ("when albums don't auto-match, here's how to merge/split, and whether it's reversible") as part of `p03.3`'s definition of done. ADR 0008 documents the *internals*; there is no operator-facing how-to for a destructive feature. The person harmed by this gap is the operator who merges the wrong albums and doesn't know if/how to recover. This should be filed.
- **Disagreement**: I'd contest "M27 is complete." The code is complete; the operator documentation for a destructive, irreversible-ish feature is not.

## 7. Lessons for Future Sessions

- **Keep**: Spiking a genuine fork before building. The PO's "run a spike first" on M27 durability produced one correct mechanism that drove three build passes with zero mechanism rework. When N independent personas surface the same risk in grooming, that's the signal to spike, not guess.
- **Keep**: Independent orchestrator verification of agent-reported gates. It caught a flaky test that passed for the engineer, and the two-reviewer depth-difference caught a scan-aborting cascade one reviewer missed. The discipline earned its cost repeatedly this session.
- **Stop**: `pkill -f` with patterns that match the running shell's own argv (self-kill, twice). And stop declaring "I'll verify independently" before confirming the verification environment (a live, owned test DB) actually exists.
- **Start**: Provisioning a dedicated, single-occupant test DB *before* dispatching the build, and handing each verification/reviewer explicit, non-overlapping DB ownership — so test-DB contention stops producing false signals (3 incidents this session).
- **Start**: Treating a destructive feature's operator runbook as part of its definition-of-done, not a follow-up. M27 shipped merge/split without the recovery how-to the tech-writer specced in grooming.
- **Value learning**: A symptom reported as the wrong mechanism ("manual refresh is broken") costs an investigation aimed at the wrong subsystem. The real need ("IPTV channels aren't grouped") was a different feature entirely. When a user says "I can't do X," confirm whether X is the mechanism or the outcome before pointing the search.
