# 2026-07-31-15 — project-a download cascade debugging

- **ModelID**: claude-fable-5
- **TurnCount**: ~36 (13 genuine PO messages, ~23 assistant/system turns; heavy background-task notification traffic)
- **SessionDepth**: deep — media browse UI, nav architecture, WorkManager/foreground-service internals, Room FK semantics, byte-range HTTP resume, on-device diagnosis over adb across five CI build cycles
- **Personas Active**: project-engineer (implicit, main loop), code-reviewer (5 dispatched review passes), SRE (on-device diagnosis), database-engineer (Room CASCADE root cause), QA (flaky test fix), UX (wrap fixes), project-manager (beads hygiene)
- **Beads Touched**: bkd, bzx, 2gn, acq, 0wj, k77, 6q3 (closed); 250, 44g, glt (filed open)

## Section 1: User Value Delivered

Real, verified-on-device value:

- **Music browse works again.** Tapping an artist had shown an infinite spinner (nav-scoped ViewModel + never-passed nav args). Fixed at the root (ID-based loading), confirmed by the PO on their phone.
- **Downloads are visible and trustworthy.** The original ask — "prove the foreground download we have to attest to" — required a persistent notification. Delivering it uncovered a stack of four real defects: per-track workers silently losing foreground rights when backgrounded; a queue that could strand forever after an app update; an HTTP 416 resume loop that failed one track eternally; and the worst one, a Room `REPLACE`+CASCADE interaction that silently wiped cached track metadata on every browse refresh, orphaning download rows. All fixed; the PO's stranded 27-row queue self-healed on next launch and the PO confirmed "it is fixed now."
- **Three visual defects fixed** (wrapped segmented buttons, wrapped bottom-nav label, wrapped account names) plus two identical latent ones the review pass found before the PO could screenshot them.
- **First-run setup now offers all four account types** instead of two.
- The store-compliance attestation (dataSync foreground service) is now demonstrable — direct user value in keeping the app distributable.

No work this session was speculative; every change traced to a PO-reported symptom or a reviewer-verified sibling of one.

## Section 2: What We Did Well Together

The diagnostic loop at the end. When the PO reported "1 of 12 complete and isn't moving," we stopped guessing after one failed workaround and switched to evidence: WorkManager's own diagnostics broadcast over adb proved zero scheduled work existed; then purpose-built drainer logging (one PO tap to enable, one agent-driven UI tap over adb) produced the exact three root causes in a single log read — "26 rows skipped, no cached track" and "HTTP 416." The PO's willingness to leave the phone plugged in and unlocked, plus flip one setting on request, converted a blind cycle into a conclusive one. The final fix landed on the first attempt after instrumentation, having failed twice before it.

## Section 3: What the PO Could Improve

"Unit tests are failing" (mid-session) arrived with no run link, no failure name, and no local-vs-CI context. Diagnosis required pulling CI history to discover the latest run was actually green and the failure was a one-run-old flake in an unrelated test. A pasted run URL or the failing test name would have collapsed a full investigation round-trip into a sentence. Same pattern earlier: "still not seeing the download notification" — the log later showed the notification had appeared and vanished in 0.8s because the batch was already complete; "I tapped Download All on the album I downloaded earlier" as context would have redirected the investigation immediately. The reports were all honest and prompt — they just consistently omitted the one artifact (link, name, timing) that anchors them.

## Section 4: What the Agent Got Wrong

The reviewer's F3 note on the drainer redesign said legacy per-track jobs would be no-op'd and "their rows drain on the next real kick." I accepted that without asking the load-bearing question: *when is the next kick?* The answer was "never" — nothing kicked the drainer except a new enqueue — and the PO hit exactly that strand within the hour, mid-download, because I also installed the update over an in-flight batch. Compounding it, I then gave a workaround ("tap one more track — it will drain the whole queue") built on an unverified assumption about row states; the rows were orphaned, the workaround failed, and the PO burned a test cycle on it. Instrumentation-first (the logging that ultimately solved it) should have been the response to the *first* "still not working," not the third.

## Section 5: What Would Make the Project Better

Device iteration cost dominated this session: five ~8-minute CI cycles because release-signed APKs can only be built remotely, and the Play-signed install added a signature dead-end plus one full data wipe. Two cheap improvements: (1) an in-app diagnostics export (downloads table + WorkManager state dump) so release-build state is inspectable without shipping logging builds; (2) a debug-signed variant with an applicationId suffix, installable alongside the store app, for fast local iteration without touching the user's real data. Either one would have cut this session's wall-clock roughly in half.

## Section 6: Persona Perspectives

### Code Reviewer
- **User value assessment**: The review passes caught bugs users would have hit — the setup-wizard nav race (F1) would have made the new feature self-destruct on first tap; the drainer findings (stranded DOWNLOADING rows, corrupt-complete race, concurrent legacy drainers) were all data-integrity defects on the critical path.
- **Session assessment**: Five review cycles, every one produced at least one confirmed finding; the process earned its cost. The retry-after-error test ask on the first fix was honored.
- **What I'd flag**: My own F3 note called the legacy no-op "harmless — rows drain on the next real kick" and the orchestrator accepted it. Both of us treated a liveness question as answered when it was merely restated. Review findings that assert "X will happen later" need the same adversarial treatment as "X is correct now."
- **Disagreement**: With the engineer's sequencing — shipping the drainer redesign to the PO's device while the PO had an in-flight batch under the old design was avoidable; a review pass can't catch what deployment timing breaks.

### Project Engineer
- **User value assessment**: Every fix was root-cause, not symptom: ID-based loads, `@Upsert` over REPLACE, typed 416 restart. The ponytail discipline held — smallest correct diffs, one deliberate documented trade (sequential downloads for one honest notification).
- **Session assessment**: Solid except for deployment hygiene: `adb install -r` over an active download batch created the exact stranded state we then spent three cycles fixing.
- **What I'd flag**: The drainer shipped with zero worker-level tests; all confidence came from the TrackDownloader seam plus review. The three post-ship bugs were all in the untested worker orchestration layer. That's not coincidence.
- **Disagreement**: With the reviewer's "approve with minor follow-ups" cadence — twice the "minor" bucket contained the bug the PO hit next (F3 legacy strand). Severity calibration should weight liveness bugs higher.

### Database Engineer
- **User value assessment**: The `@Insert(REPLACE)` + `ON DELETE CASCADE` interaction was silently destroying user data (cached library + download linkage) on every browse refresh since the cache shipped. Finding it protected the offline-listening feature wholesale.
- **Session assessment**: Found by runtime diagnosis, not schema review — it should have been caught when the schema landed. REPLACE-as-DELETE+INSERT firing cascades is a known SQLite/Room footgun.
- **What I'd flag**: The `@Upsert` fix removes the accidental garbage collection REPLACE provided; server-deleted content now lingers in the offline cache until a manual clear. Accepted trade, but it needs a real eviction story eventually.
- **Disagreement**: None on the fix; mild disagreement with treating the downloads table's dual tenancy (music tracks + audiobook records sharing one table with indistinguishable bare-string IDs) as acceptable. The `getSong`-throws-therefore-ABS discriminator works but is a probe, not a design.

### SRE
- **User value assessment**: The debug-log pipeline (user exports to a shared drive; agent reads over adb) is what actually fixed the user's problem — observability was the deliverable behind the deliverable.
- **Session assessment**: Poor at the start, good at the end. Three fix attempts shipped before any instrumentation; the fix attempt *after* instrumentation was the one that worked. Also found and fixed a real gap: debug logging didn't activate until the settings UI loaded, so process-start events (exactly where background bugs live) were never logged.
- **What I'd flag**: The startup drainer kick runs on every launch and probes `getSong` for every non-music pending row every pass. Bounded and small, but it's a per-launch network cost that will hide in battery/network profiles later.
- **Disagreement**: With the engineer's ordering. Diagnostics-first is not a fallback for when fixes fail; on device-only bugs it is the fix path.

### QA Engineer
- **User value assessment**: The flaky-test fix (await API before load in 34 call sites via one helper) protects the team's trust in CI — a red main that's noise trains people to ignore red mains.
- **Session assessment**: Good instinct running the repaired test class three times before declaring stability. The 416 unit test pins real behavior (offsets, truncation, terminal state), not coverage.
- **What I'd flag**: On-device verification was ad hoc — screenshots and logcat greps composed per-cycle by hand. A scripted smoke (launch → enqueue → assert notification via dumpsys) would have made each of the five build cycles self-verifying.
- **Disagreement**: With shipping the drainer on review-plus-unit-seam confidence alone; the orchestration layer needed at least an instrumented test or a scripted device check before it hit the PO's phone.

### UX Designer
- **User value assessment**: The wrap fixes address real screenshots of broken layout, not taste. Middle-ellipsis for hostname-style names was the right truncation choice — both ends carry meaning.
- **Session assessment**: Good. The setup wizard hand-off (wizard → full Add Account screen) trades a seam (two back-to-back transitions on Back) for not duplicating an SSO flow — correct trade at this app's scale.
- **What I'd flag**: Sequential downloads make large batches noticeably slower than before; the notification communicates progress honestly, which mitigates it, but a user who remembers parallel speed may perceive regression.
- **Disagreement**: None material.

### Project Manager
- **User value assessment**: Ten beads, seven closed same-session, three filed with honest scope notes — the backlog reflects reality, including the open P2 that gates the audiobook side of the same attestation.
- **Session assessment**: Discipline held under interruption-driven work: every fix got a bead before code, reviews before merge, pushes completed. But five CI cycles for one feature area is a delivery-process smell, not an engineering one.
- **What I'd flag**: The session ended with the attestation demonstrable but the actual Play Console declaration (the P1 bead) untouched. The PO should be steered to that next — the engineering is done; the compliance step is the remaining user-facing risk.
- **Disagreement**: With the SRE's framing that instrumentation should always come first — the first two fixes (FGS restriction, startup kick) were correctly diagnosable from public logcat without a build cycle. The failure was continuing to fix blind after the *second* surprise, not starting that way.

### Technical Writer
- **User value assessment**: The code carries its history well — the CASCADE landmine comment on the DAO and the ponytail note on sequential downloads will stop the next regression at read time.
- **Session assessment**: Commit messages doubled as incident narratives; adequate. Nothing else was needed at this scale.
- **What I'd flag**: The debug-log workflow (enable toggle → export to shared drive → agent reads) is now load-bearing team process and lives nowhere but this conversation. One paragraph in the repo docs.
- **Disagreement**: None.

## Section 7: Lessons for Future Sessions

## Lessons

- **Keep**: Adversarial review on every diff, including hotfixes — five passes, five sets of confirmed findings, two of them ("setup wizard self-destructs," "queue strands forever") were bugs the PO would have hit within the day.
- **Stop**: Shipping another blind fix after a fix attempt has already surprised you once on-device. The second surprise is the signal that your model of the system is wrong; instrument before the third attempt, not after it.
- **Start**: Before installing an update over a live app, check for in-flight state the update invalidates (active downloads, running jobs) — the mid-batch install manufactured the worst bug of the session. Also: when a reviewer or engineer claims deferred work "happens later," name the concrete trigger; if no code path is the trigger, it never happens.
- **Value learning**: The PO's stated need was a notification; the actual need was *trustworthy downloads* — the notification was just the first place their absence became visible. Treating the symptom report as an entry point into the subsystem (rather than a UI ticket) is what surfaced the cache-wipe bug that had been silently corrupting the offline feature all along.
