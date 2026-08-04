# 2026-08-04-17 — project-d-verification-had-blindspots

- **ModelID**: claude-opus-5
- **TurnCount**: ~85 (12 PO messages, one of them mid-turn; the rest agent turns, heavily tool-weighted)
- **SessionDepth**: deep — full disaster-recovery exercise against a disposable stack, six defects diagnosed to root cause in both our code and an upstream dependency's source, six fixes implemented and shipped, plus a substantial rewrite of the exercise's own procedure
- **Personas Active**: QA Engineer (lead), Project Engineer (4 dispatches), Technical Writer (1 dispatch), SRE, Security Engineer, Code Reviewer, Project Manager, IT Architect, UX Designer, Database Engineer
- **Beads Touched**: created `a429n` (run tracking), `cytzj`, `lvfwd`, `y65si`, `6pilh`, `2o0cz`, `dfkbn`, `nlg9i`, `pi9k2`, `x548h`, `jy006`; priorities raised on `lvfwd`/`y65si`/`6pilh`/`2o0cz` (P0) and `dfkbn`/`cytzj` (P1); attempted and failed to annotate `1zwmr` (does not exist)

---

## Section 1: User Value Delivered

Substantial, and honestly stated: **the product's disaster recovery did not work, nobody knew, and now it does — probably.**

The premise of the exercise was that the restore path had never had a real round-trip test. That turned out to be exactly the right thing to distrust. Taking a full backup, destroying both services and their volumes, rolling out clean, and restoring produced:

- Two defects that **aborted the restore outright and rolled it back**, each triggered by something ordinary — a custom stream profile, and rebuilding the companion service with a differently-named admin.
- Past those, a restore that reported `success … failed 0` while producing an instance where **not one channel could play**, every logo and guide link was gone, a profile built to *exclude* items had silently reverted to including everything, and non-default settings were back at defaults.

For an operator who has actually lost their instance, that is close to the worst possible failure: the recovery tool reports success and hands back something that looks right and does nothing. All six defects are fixed and merged; a new image is published and I verified the fixes are physically present in it rather than trusting CI.

**The honest caveat:** none of it has been verified end-to-end against a live stack. Every fix is unit/mock-level. The value is "very likely delivered" rather than "proven delivered", and the whole point of the exercise was that unproven recovery paths are exactly what bite you. The session that proved a code path was never exercised then shipped its replacement unexercised. Run 2 exists to close that, and until it runs this is a strong inference, not a result.

Secondary value: the exercise's own procedure was hardened (its spec grew by about a third), and four additional defects were filed that nobody was looking for, including an unauthenticated-endpoint finding and a discovery that the upstream service returns provider credentials in cleartext to admin callers — which means our redactor is the *only* thing keeping a live credential out of a standard backup artifact.

---

## Section 2: What We Did Well Together

**The PO's mid-turn message: "If the agent is using APIs wrong, let it use the [upstream] repo to learn how to use the APIs from their code."**

That single sentence was the highest-leverage thing that happened all session, and it worked because it corrected a *method*, not a conclusion. The PO didn't tell me what I'd gotten wrong. They told me how to find out.

I had written six bead descriptions from black-box observation — watching HTTP responses and inferring causes. Reading the pinned image's own source instead:

- Corrected **four of six** root causes I had confidently written down.
- Found the exact upstream line where a serializer declares a field optional and then indexes it unconditionally — turning my guess into a citation.
- Surfaced **three upstream bugs**, one of them the cleartext-credential disclosure that materially raises the stakes on our redaction code.
- Overturned one of my *own* subsequent claims: I briefed an engineer that a protected profile's field "can legitimately be remapped", and the source showed the guard compares a field's `name` against a set containing its `attname`, so the one field its comment permits is the one it rejects. The engineer pushed back with the citation and was right.

The pattern worth keeping: when the PO sees an agent reasoning from symptoms, pointing it at a source of ground truth beats correcting the symptom-level conclusion. It fixed one bug in my process and six in my output.

---

## Section 3: What the PO Could Improve

**The tempo the PO set — "ASAP", then "Now", then "commit and push" — put six P0/P1 fixes onto the shared branch on unit tests alone, ahead of the verification run that was the entire point of the session.**

The sequence: the exercise proved a recovery path had never been exercised and was comprehensively broken. The PO said the blockers *"need to be fixed ASAP"*. Then, on the remaining four, *"1. Yes. 2. Now."* Then, before any live re-verification, *"Can we commit and push the code so we can get a new container built."* Merged the same session.

Each step was individually defensible, and the last one had a genuinely good reason — a published image removes the local-build workaround, and getting one requires landing on the shared branch. I said so at the time and proceeded. **So this is not solely the PO's to carry:** I flagged the inversion in one sentence and complied, when the session's own central finding was that unexercised recovery paths don't work. Naming it here and not in Section 4 would be the self-serving move the prior retro warned about.

What is specifically the PO's: **"ASAP" and "Now" are urgency signals that don't distinguish "fix this quickly" from "fix this and ship it before verifying it."** They read identically to an agent, and I resolved the ambiguity in the direction of speed three times running. "Fix it now, ship after run 2" would have cost nothing to say and would have produced a different, better sequence — the fixes ready, the drill run, then a merge backed by evidence instead of inference. Instead we now hold six merged fixes whose live behaviour is unknown, on the exact feature where "we believed it worked" was the finding.

**A smaller, sharper one:** the *"don't launch, clear context first"* instruction arrived after the PO had already approved the re-run, after I had set the control lines, verified the environment, and started a container build. That build was interrupted mid-flight. The instinct was right — a fresh session for a long run is correct — it just arrived one approval late. Attaching it to the "yes, re-run it" would have saved the attempt.

---

## Section 4: What the Agent Got Wrong

**Twice I verified a claim with an instrument too weak to support it, and both times the weak instrument returned a confident, wrong-shaped answer.**

**First, and worse:** the inventory captured every credential as a boolean — "is something set here" — because the spec said never to emit the value. Sound intent. But a redacted restore writes a *literal placeholder string* into the password field, and a placeholder is non-empty, so the boolean read `True` on both sides. My diff reported the account **byte-identical** while the instance could not authenticate and served zero streams. The measurement I built to detect exactly this class of drift declared it clean.

I caught it by manually dumping a raw field while poking at something else — luck as much as rigor. Had I trusted my own instrument, I'd have reported a false PASS on the single most consequential defect of the session. I implemented the boolean as specified without once asking "what value would satisfy this check while still being broken?" — a question I had all the context to ask and didn't.

**Second, at the very end:** verifying the shipped image, I grepped for the first textual occurrence of two symbols to confirm import ordering and reported `WRONG`. It had matched a rollback deleter map, nothing to do with import order. The actual ordering registries were correct. The engineer had *explicitly warned in its report* that a nearby construct was a decoy — and I walked into a variant of the same trap while checking its work.

**Also worth recording:** mid-run I saw log lines showing objects being restored after the UI said "Restore failed" and asserted the restore was "actually applying successfully and the UI is wrong." It had failed and rolled back. I retracted it in the report rather than quietly deleting it, but the error was reading partial evidence and announcing a conclusion when the authoritative source — the task history — was one call away.

The through-line, and it connects to the prior retro on this project: **that retro's Section 4 said "the retro is a good instrument and a lagging one."** Today the failure was the same shape one level down — the *verification* instrument was lagging, and the thing that caught both blind spots was a second check of a different shape, applied semi-accidentally. Two sessions running, the story has been an instrument that looked authoritative and wasn't.

**Then I did it twice more, in this retro, in the section meant to hold the PO accountable.**

Draft one of Section 3 criticised the PO for a standing instruction that conflicted with the project's orchestration hook. The PO **never gave it** — it came from harness-level session guidance in my system prompt, and is absent from their global config, the project config, and every hook and setting they own. They caught it with the obvious question: *where did I say that?*

I rewrote it. Draft two criticised the PO for an exercise spec that asserted product behaviour it had never verified — a logo-seeding instruction that produced an empty artifact category, and two tracker references that resolve to nothing. **The PO didn't write that spec either.** A prior Claude session did; the tracking bead names the author outright. They caught that one too.

Two fabricated attributions, in consecutive drafts, in the one section whose whole purpose is accurate accountability, destined for a public repo. And the second came *immediately after* I wrote the corrective rule — "verify it came from that person, grep for it before writing it" — and then did not apply it to the very next paragraph. Writing down the lesson and violating it in the same edit is a more interesting failure than the original error: it shows the rule was recorded, not internalised.

The mechanism is the same weak-instrument pattern one level up. I reached for a *plausible* source instead of a *verified* one, exactly as I reached for a plausible boolean and a plausible grep. Four instances now, escalating in consequence: a broken instance reported clean, a correct fix reported wrong, and twice a criticism attributed to someone who didn't earn it. Had the PO not pushed back both times, the corpus would carry two invented complaints about them — and worse for anyone reading, each would have pointed the fix at the wrong layer entirely.

The real content of that first item survives, correctly labelled: a harness instruction and a project hook did conflict, and I adjudicated between them six times. That's a genuine finding about **instruction-layer conflict** — it just isn't PO feedback, and filing it as such would have sent the wrong fix to every reader.

The prior retro's Section 4 noted putting something under "What the PO Could Improve" that was "accurate in the narrow sense and self-serving in the broader one." Both of today's skipped the narrow accuracy too.

---

## Section 5: What Would Make the Project Better

**Agent-authored specs need the same "is this verified or assumed?" discipline as agent-authored code — and right now they get none.**

The exercise spec that drove this whole session was written by a previous agent session, and it stated product behaviour as established fact in two places where it was untested guess. It instructed "assign logos to at least three channels so the binary/logo category is non-empty" — I did exactly that, took both artifacts, and found the category empty anyway, because the product auto-assigns logos as remote URLs and only *uploaded files* populate that subtree. Both artifacts had to be re-taken. It also directed me to attach evidence to two tracker items that don't exist, so a genuinely useful result ended up stranded where nobody following the breadcrumb will find it.

Neither is a failure of writing a spec from incomplete knowledge — discovering that is what a first run is *for*. The failure is that the spec gave no way to tell its verified claims from its assumed ones, and a downstream agent has no reason to distrust a confidently-worded instruction. A spec that says "do X so that Y is true" invites the reader to treat Y as achieved once X is done.

Cheap fix, and the hardening pass this session started it: mark behavioural assertions as assumptions until a run confirms them, and have the run promote them. That gives the next agent a reason to verify the unpromoted ones instead of inheriting them as fact. This matters more as agent-written specs proliferate — they read with the same authority as human-written ones and carry none of the accountability.

**A second, narrower one: bead references in code and docs should be verified to resolve, because several currently don't, and one of them cost real evidence this session.**

A test in the restore suite is skipped with a message pointing at a bead for context. That bead does not exist in the tracker. Neither does a second one referenced in the same file, in a core module, in the changelog, and in two docs — five files pointing at two identifiers that resolve to nothing.

This wasn't academic. The exercise produced precisely the evidence one of those beads was waiting for: it needed a seeded instance to confirm ordering behaviour, and I built one and confirmed it survives a full round trip on both artifact variants. I went to attach that evidence and there was nothing to attach it to. It's recorded in the run report, which is not where anyone following the breadcrumb in the test file will look.

A CI check that every `bead-id`-shaped reference in code and docs resolves in the tracker would be cheap and would stop the codebase accumulating breadcrumbs to nowhere. The deeper issue is that a *skipped test* pointing at an unreadable bead is worse than no comment: it looks like a documented decision and is actually a dead end.

---

## Section 6: Persona Perspectives

### QA Engineer
- **User value assessment**: Highest-value session this project has had for users, and the value is inseparable from having run the thing for real. Every one of the six defects required a genuine destroy-and-restore against live services. None would have surfaced in unit tests, because the unit tests were green the entire time the feature was comprehensively broken.
- **Session assessment**: The exercise design was sound and its execution was disciplined — assert-then-destroy ran four times, artifacts were hash-verified before every wipe, and the diff compared on natural keys. But the test design contained a hole that let a P0 through, and that is my failure, not the executor's.
- **What I'd flag**: We shipped six P0/P1 fixes to a shared branch on unit tests alone. The entire premise of this session was that unexercised recovery paths don't work. We have now replaced an unexercised recovery path with a different unexercised recovery path and merged it. That is a strictly better position than before and it is not a proven one.
- **Disagreement**: With the Project Manager. Landing on the shared branch to get a published image was pragmatic and I understand the sequencing, but it inverted verify-then-ship on exactly the code path whose lack of verification we had just spent a session proving was dangerous.

### Project Engineer
- **User value assessment**: Real. Restore now completes in the ordinary cases that previously aborted, and the categories that silently vanished are carried. The rebind pass is the one that turns "the objects came back" into "the instance came back".
- **Session assessment**: TDD held — every dispatch confirmed red before green, and one engineer proved its own test had teeth by reverting a single change and watching the assertion fail rather than assuming it would. The dispatch briefs were detailed enough that engineers could contradict them from source, which they did four times.
- **What I'd flag**: Two of my line-number references in the bead descriptions were runtime, not repo. An engineer following them literally would have "fixed" the wrong construct and changed nothing. Black-box bead-writing produces plausible, wrong navigation.
- **Disagreement**: With the Code Reviewer on the counter that now has two producers with different meanings. I'd rather ship the honest number today and split the semantics in a follow-up than block a P0 on a rename touching a dozen assertions.

### SRE
- **User value assessment**: This is the session that matters for reliability. Disaster recovery was documented, believed, and non-functional. The gap between "we have backups" and "we can restore" was total, and it was closed by exercising it rather than by reading it.
- **Session assessment**: The right thing was tested the right way — real destruction, real volumes, no shortcuts, and the restore target genuinely fresh every time.
- **What I'd flag**: A failed restore does not return you to where you started. The rollback leaves a third state — objects created by the companion service's own ingest survive because the ledger doesn't own them, and applied settings are explicitly non-compensatable. There is no documented recovery from that state, and an operator hitting it mid-disaster is worse off than before they tried. That deserves its own runbook entry and possibly its own bead.
- **Disagreement**: With the UX Designer on severity ordering. A confusing error message is bad; an unrecoverable intermediate state during an actual outage is worse.

### Security Engineer
- **User value assessment**: Two findings with direct user impact, neither of which was the session's goal. The upstream service returns provider credentials in cleartext to admin-level callers, which means our redactor is the sole control keeping a live credential out of a standard backup artifact — and a comment in our own code asserted the opposite. And an operator-supplied passphrase was sitting in process memory indefinitely.
- **Session assessment**: Credential handling in the fixes was careful — generated passwords never logged or echoed, the write-only behaviour confirmed at source rather than assumed, and a dedicated module so the sentinel's producer and consumer cannot drift.
- **What I'd flag**: The unauthenticated-endpoint finding is filed at P2 and I think that's low. Four endpoints answer anonymously including one that returns all settings and one that is an audit trail. I deliberately did not overstate the bead — the integration secrets were unconfigured, so cleartext disclosure is unproven — but "unproven" is a reason to run the five-minute check, not a reason to sit at P2.
- **Disagreement**: With the Project Manager on prioritisation. A placeholder credential that makes an account *look* configured is a security-adjacent defect too: it defeats every presence check a human or a script would run.

### Code Reviewer
- **User value assessment**: The quality bar held where it mattered. Tests were proven red, one engineer refused to manufacture a production change when the correct answer was "the code path is unreachable, here's a test to pin it", and gates were independently re-run rather than taken on report.
- **Session assessment**: Good. Four dispatches, all landing on one branch sequentially, no conflicts, no regressions, +23 tests net.
- **What I'd flag**: A counter now has two semantically different producers feeding a single operator-visible banner — one meaning "nothing matched this", the other "a channel lost what it had". That was a knowing compromise and it's filed, but it means the banner is now ambiguous rather than merely wrong. Also, three unit tests were confirmed to guard a code path that cannot execute; green tests that guard nothing are worse than no tests, because they read as coverage.
- **Disagreement**: With the Project Engineer above — I'd have preferred the semantics split land with the fix, since we're now shipping a knowingly ambiguous signal to operators.

### Project Manager
- **User value assessment**: Six defects found, six fixed, shipped, image published and verified, ten beads filed with full reproduction detail. Very high throughput with no scope drift.
- **Session assessment**: Decisions were surfaced cleanly and answered fast — the PO made six decisions in six exchanges with no stalling. Sequencing engineers on one branch instead of parallel worktrees avoided the conflict class this environment is known to produce.
- **What I'd flag**: Documentation is now the long pole and it's in a bad state — five files sitting uncommitted that describe workarounds for defects that are now fixed *and shipped*. Every hour they sit there, they get more wrong and more likely to be lost or landed carelessly. That's work-in-progress inventory with a decay rate.
- **Disagreement**: With QA. Verify-then-ship was not available: a published image required landing on the shared branch, and the exercise needed a published image to be worth running. The alternative was a locally-built image and a hand-edited compose file, which the PO explicitly moved away from, and which would have tested an artifact no user will ever run.

### IT Architect
- **User value assessment**: The architectural fix — routing a foreign key through the existing id-remap table — is what makes restore correct rather than accidentally-working. Without it, a restore succeeds or silently binds the wrong object depending on whether id spaces happen to align.
- **Session assessment**: Root causes were traced to design assumptions rather than patched at the symptom, which is the right depth.
- **What I'd flag**: The most dangerous artifact found all session was a *comment asserting an invariant that the code did not hold*. One said a category of entities "carry no outbound FK" — false, and it was the direct cause of a P0. Another said scheduled runs always produce the default artifact — false, and it was a second defect. Both read as authoritative documentation of intent. A wrong comment is worse than no comment because it stops the next reader from checking.
- **Disagreement**: None substantive, though I'd note the remap table already existed and was simply not applied to a misclassified category — this was a classification error, not a missing mechanism.

### Database Engineer
- **User value assessment**: The id-versus-natural-key discipline is the whole substance of the worst defect. A restore reassigns ids by definition; anything comparing or transmitting a source id is wrong by construction.
- **Session assessment**: The inventory's decision to compare on natural keys was correct and is what made the diff meaningful at all. Emitting id-bearing values into a separate reported-but-not-counted section, rather than silently dropping them, was the right call.
- **What I'd flag**: Membership data was being read from a key no producer has ever written — the data model assumed a shape the wire format never had. That went unnoticed because tests supplied the key by hand.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: The defects here are as much interaction failures as logic failures. An operator mid-disaster saw "Restore failed" with **no reason at all**, and the reason was only obtainable from an API call or a container log.
- **Session assessment**: UI-level observations were captured well because the exercise was driven through the real interface rather than the API — a placeholder password field that looks populated, an account status blaming the provider for a local misconfiguration, a native browser alert as a completion signal.
- **What I'd flag**: The reasonless failure banner is not filed as its own defect — it's buried inside another bead's description. It will be fixed incidentally or not at all. For a human under pressure, an error with no cause is arguably the worst thing here: it produces retries, and a retry of a partially-applied restore is how people make things worse.
- **Disagreement**: With SRE on ordering. The intermediate state is dangerous *because* the message gives the operator no basis to decide whether to retry. The message failure is upstream of the state failure in the causal chain.

### Technical Writer
- **User value assessment**: Zero delivered this session. The docs are written and correct-as-of-when-written, and they are sitting uncommitted describing workarounds for defects that no longer exist. Nothing has reached an operator.
- **Session assessment**: The dispatch worked well — the writer corrected six confirmed-wrong statements including one sending operators to a control that does not exist, and produced a self-contained procedure that inlines the compose file because the tooling deliberately lives outside the repository.
- **What I'd flag**: I reported "no broken links or anchors" and there was a broken internal anchor. It was caught by independent verification, not by me. Beyond that: the exercise's own spec is now a 950-line out-of-repo document that is the sole carrier of critical knowledge — including the fact that a stale local image would silently invalidate an entire run. That file has no version control and no backup.
- **Disagreement**: With the Project Manager on framing. The docs aren't just "the long pole" — they're the deliverable the whole exercise exists to produce. An operator can't run a recovery from a merged code fix.

---

## Section 7: Lessons for Future Sessions

## Lessons

- **Keep**: Reading the upstream dependency's actual source from the pinned image rather than inferring behaviour from responses. One PO nudge corrected four of six root causes, found three upstream bugs, and overturned one of my own briefings. It is now written into the exercise spec as a standing rule with the confirmed upstream bugs listed so a future run recognises rather than re-diagnoses them.
- **Stop**: Verifying a claim with the cheapest instrument that seems to fit. A boolean presence check was satisfied by a placeholder string; a first-occurrence grep matched an unrelated map; a retro criticism was attributed to the person most likely to have said it rather than the layer that actually did. All three returned confident answers of the wrong shape. Before trusting a check, ask what value or arrangement would satisfy it *while the thing being checked is still broken* — and if such a value exists, the check is not a check.
- **Start**: Treating provenance as a fact to verify, not infer — and actually running the check, not just recording the rule. Instructions this session arrived from at least five layers: the PO's turns, their committed config, project hooks, harness session-guidance, and a *spec authored by a previous agent session*. Only one of those is the PO. I wrote this exact rule after the first misattribution and then broke it in the next paragraph, which is the real lesson: a rule written into a document is not a rule applied. Before any line of the form "the PO did X", produce the message or the file they own that shows it. If that can't be produced in one command, the line doesn't go in.
- **Start**: Treating "what would a false PASS look like?" as a required design step for any verification artifact, at design time rather than after. The exercise spec now mandates three-state credential classification plus a fingerprint, an artifact-contents inspection before destruction, and an actual playback fetch instead of inferring from a populated field. Each of those exists because the previous version could return a clean result on a broken instance.
- **Value learning**: We assumed the recovery path mostly worked and needed proving. It did not work at all, in five separate ways, and the reporting said `success, 0 failures` throughout. The durable lesson is about *reported* success: every category count reconciled perfectly — items in, items out, exact match — for an instance that could not play a single channel. Counts reconciling is not evidence a system works; it is evidence the objects exist. For any recovery feature, the acceptance test has to be a functional one the user would recognise, not an inventory that balances.
