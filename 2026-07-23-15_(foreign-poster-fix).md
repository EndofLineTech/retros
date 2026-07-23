# 2026-07-23-15 — foreign-poster-fix

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~34 messages (7 genuine user turns, the rest assistant turns and background-task notifications)
- **SessionDepth**: deep — one user-reported bug traced from symptom to a falsified premise in the code, across CDN forensics, a live API probe, three review rounds, and a merge
- **Personas Active**: project-engineer, code-reviewer, database-engineer (all dispatched); QA, technical-writer, PM, SRE, security, IT-architect, UX (relevant, not separately invoked)
- **Beads Touched**: one bug bead created and left IN_PROGRESS (deliberately — user-reported bug, not done at merge); three follow-up beads created (station-logo premise re-validation, WAL + immediate-txn, orphaned-worker credit burn)

## Section 1: User Value Delivered

Real, direct user value — with an honest asterisk. A user reported that every movie in the US guide showed a poster in a foreign language (German for one title, French/Spanish/Telugu for others). The root cause was a genuinely wrong premise baked into the code: it treated the trailing letters of an image asset id as a revision counter when they are actually a language slot, and it picked whichever localization sorted first off the CDN. The fix stops guessing and asks the upstream API for its own language-correct pick. A second, previously uncatalogued failure mode (titles whose art lived outside the assumed family were permanently sealed "no poster exists") was cleared by the same change.

The asterisk: nothing has shipped to the user yet. No live build has emitted a real poster into a real guide — that requires a scheduled run we didn't authorize this session. The bead is correctly left open pending deploy + reporter confirmation. So the session produced a merged, reviewed, verified fix that is *staged* value, not *delivered* value. That distinction was kept honest throughout rather than being papered over at merge.

## Section 2: What We Did Well Together

The probe-first decision. When I surfaced the diagnosis, I flagged that I could not tell whether the bug was in one module (the poster resolver) or two (that plus a global image-aspect switch), and asked for authorization for a bounded ≤10-call API probe to settle it before committing to a fix scope. The PO answered crisply: "1. a. Yes, probe. 2. A. yes." That probe exonerated the aspect switch — my initial CDN evidence had looked alarming because I'd sampled the wrong image-style family — and prevented an engineer from being sent to change a grid parameter that was never broken. A single terse authorization for a ten-second question saved a wrong fix. The collaboration worked because the decision was framed as a scoped fork with a recommendation, and the PO resolved it in one line.

## Section 3: What the PO Could Improve

One decision was left hanging across three separate messages: whether to cap the first post-upgrade run's paid-API spend. I re-surfaced it each time (correctly — it's a deploy-time decision, not a merge blocker), but the PO consistently answered only the immediately-actionable question in front of them and let the deploy-time one roll forward unanswered. It is still unanswered at session end. That's fine *today*, but the migration this PR ships spends real credits per row on its first run, and the count is genuinely unknowable in advance. The risk is that the decision surfaces for the first time as an invoice line rather than a choice, because it kept getting deferred instead of parked with an explicit "decide at deploy." A one-line "park the cap decision until we schedule the run" would have closed the loop cleanly instead of leaving it as an open thread I have to keep re-raising.

## Section 4: What the Agent Got Wrong

I passed a bad number up to the PO with a false confidence label. After the first engineer round, I relayed the engineer's "94.2% of rows rescued free, hundreds to low-thousands of credits" as if it were a meaningful production estimate. It wasn't — it was measured against a local cache snapshot that predates the very image-aspect switch that causes the bug, and is therefore structurally biased toward the free-rescue case. The damning part: I *already knew* the snapshot was stale. I explicitly told the database-engineer, in the brief I wrote, that the local cache was a "stale pre-switch snapshot, not representative of production." I briefed the reviewer about the staleness and then, in the same session, forwarded a number derived from that same stale snapshot to the PO without applying my own caveat to it. The DBA had to independently rediscover the staleness (and found the snapshot doesn't even have the relevant column) before I caught and withdrew the estimate. Verifying premises before briefing agents has to apply equally to numbers I brief *upward* to the PO, not just downward to subagents.

## Section 5: What Would Make the Project Better

The branch-gate hook has a known worktree defect that cost real friction this session: it reads the current branch from the *spawn* working directory rather than the command's own cwd, so an engineer spawned with cwd at the main checkout got its commit false-blocked and had to detach the main checkout's HEAD to work around it. That worked and was cleanly reverted, but "engineer detaches the main repo's HEAD to satisfy a hook" is a footgun one bad interleaving away from corrupting a parallel agent's state. Either the hook should resolve cwd correctly, or worktree-isolated agents should be reliably spawned with cwd = their worktree. This is the second class of worktree/hook interaction to bite in the shared history; it's worth fixing at the harness level rather than relying on each engineer to improvise a safe workaround.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: Direct. The change fixes content the user literally sees (a foreign-language poster) and does it by removing a guess, not adding a workaround. That's the right shape of fix.
- **Session assessment**: Sound. The three-round structure (implement → review kickback → small polish) was appropriate for a fix that turned out to have a subtle one-way-door hazard. The explicit `task.aspect` threading to close the non-MV-id asymmetry was the right call over patching the symptom.
- **What I'd flag**: The first implementation reproduced the original bug's *shape* one layer down — sealing a permanent sentinel on any unrecognised answer, where the old code needed a definitive 404. Getting that wrong on the first pass, on the exact defect class the PR exists to fix, is a signal that "don't guess" needs to extend to "don't conclude absence from ambiguity" as a reflex.
- **Disagreement**: I'd push back on the DBA's framing that the migration guard race was worth a code change this round on billing grounds — the orchestrator was right that it's a structural-invariant fix, not a live billing hazard, and the comment should say so.

### Code Reviewer
- **User value assessment**: The review caught user-affecting defects, not aesthetics — a widened one-way door that would have silently poisoned the cache for whole builds under an operator env setting, and a credit-accounting line that would have misreported egress to the PO.
- **Session assessment**: Strong. The reviewer proved the regression coverage by reverting the two modules and showing the new tests go red — that's the difference between "120 tests pass" and "these tests pin the contract."
- **What I'd flag**: The vacuous assertion the reviewer found (an assert that could never fail independently) is a small tell that test *count* was being treated as evidence in the first round. Count is not coverage.
- **Disagreement**: I think the external reviewer's approval overstated confidence by citing the withdrawn cost estimate — an approval that leans on a number the author already retracted isn't independent confirmation of that number, and the orchestrator was right to say so plainly rather than let the APPROVED banner carry it.

### Database Engineer
- **User value assessment**: The migration is what makes the fix reach existing users — without the one-time invalidation, already-poisoned rows never self-heal, and the user keeps seeing foreign posters forever. So the data work *is* the user value here, not schema hygiene.
- **Session assessment**: The migration was verified empirically (crash-mid-transaction, SIGKILL, full-disk, concurrent-lock) rather than by reading the comment — that's the bar for a bulk write against a production cache.
- **What I'd flag**: The DEFERRED-vs-IMMEDIATE transaction defect existed in a sibling function (the entity-type backfill) that this PR left untouched — the same latent bug, one function over. Fixing one instance of a shared defect and filing the other is defensible, but the filed bead has to actually get worked or the "we fixed it" story is half-true.
- **Disagreement**: I initially framed the guard race as a re-billing hazard; the orchestrator narrowed it correctly and the final comment reflects the accurate, smaller claim. I concede the narrowing.

### QA Engineer
- **User value assessment**: The tests that matter here pin user-visible behavior — an English poster is chosen, a foreign one is not, and absence is never falsely concluded. That's behavior-focused, not coverage-metric-focused.
- **Session assessment**: The red-without-the-fix demonstration (revert two files → 9 failures naming the exact arms) is the single strongest QA artifact of the session. It's what a regression test is *for*.
- **What I'd flag**: The scheduler HTTP tests flaked on port collisions under load during one run and passed on rerun. That's a pre-existing flake unrelated to this change, but a suite that intermittently fails on infrastructure noise erodes the signal of every gate run. It should be isolated so a real failure isn't dismissed as "probably the port thing."
- **Disagreement**: None material.

### Technical Writer
- **User value assessment**: The docs updated here serve the operator and the next engineer — the "ask, don't guess" rationale and the "never bare-bump the repair generation" warning are exactly the kind of knowledge that prevents a re-introduction. That's documentation with a reader.
- **What I'd flag**: A false claim ("steady state ~0 credits") survived two rounds before being corrected in the third, and it was wrong in the same optimistic direction as the withdrawn estimate. Docs that overstate the happy path are a slow-acting hazard — someone budgets against "~0" and is surprised.
- **Disagreement**: None.

### Project Manager
- **User value assessment**: The session delivered staged value and was disciplined about not claiming more — the bug bead stays open until deploy + reporter confirmation. That honesty is worth more than a premature close.
- **What I'd flag**: Three follow-up beads were created from this one fix. That's legitimate (they're real sibling defects surfaced by review), but it's also scope radiating outward from a single user report — the station-logo one in particular could be a rabbit hole. They should be triaged against actual user reports, not auto-promoted because they're adjacent.
- **Disagreement**: I'd gently disagree with the engineering instinct to bundle the sibling transaction fix into the WAL bead — that couples an easy change to a heavier one and could stall the easy fix behind the hard one.

### SRE
- **User value assessment**: The concurrency and transaction-mode work protects users indirectly — a migration that fails or double-bills under two concurrent scheduled jobs is an operational incident waiting to happen, and this session closed the live instance of it.
- **What I'd flag**: `programs.db` is still the only project database not in WAL, and the concurrency scenario (two jobs seeded on the same cron hour hitting the same cache) is real, not hypothetical. It's filed, but until it lands, the operational risk is live every scheduled hour.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: Low direct security surface this session — no auth, no untrusted input parsing in the changed path. The relevant protection is that the paid API path goes through the configured proxy per egress policy, which was verified.
- **What I'd flag**: The one-way-door sentinel is a mild integrity concern more than a security one — a wrong permanent value is hard to reverse. Adequately defended now.
- **Disagreement**: None.

### IT Architect
- **User value assessment**: The `CACHE_REPAIR_GENERATION` ladder is a small but correct architectural decision — it turns an ad-hoc one-time migration into a reusable, versioned repair mechanism the cache didn't have before. That serves future fixes, not just this one.
- **What I'd flag**: The repair generation now owns the single `user_version` slot on this database, and a sibling database uses the same slot with different (wipe-on-mismatch) semantics. Two conventions for one pragma across the codebase is a latent trap for whoever conflates them.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: This is a UX bug in the truest sense even though there's no UI to design — the user's experience of the product is a wall of movie posters in languages they can't read, in a market that is supposed to be English. Fixing content correctness *is* UX here.
- **What I'd flag**: Nobody asked what the user should see when *no* English poster exists. The fix falls back to the horizontal banner, which for at least one sampled title is *also* localized. The "absence" path may still show the user a foreign image; that edge wasn't examined from the user's eye.
- **Disagreement**: I'd push on the engineering framing that a sealed-absent row is "done" — from the user's side, a fallback to a foreign banner is the same complaint in a different aspect ratio.

## Section 7: Lessons for Future Sessions

- **Keep**: Probe-first when the fix scope forks on an unknown. One bounded, authorized diagnostic call settled a one-module-vs-two-module question and prevented a wrong fix. Frame the fork, recommend, and get the cheap fact before dispatching.
- **Stop**: Forwarding subagent-derived numbers to the PO without applying my own known caveats to them. I knew the cache snapshot was stale, briefed a reviewer about it, and still passed a number derived from it upward. Premise-verification runs in both directions.
- **Start**: When a fix touches a one-way-door state (a permanent sentinel, an irreversible write), make "what non-authoritative outcomes could reach this write?" an explicit pre-review checklist item — the first implementation sealed absence on ambiguity, exactly the class of bug the PR existed to fix.
- **Value learning**: A "poster bug" looked like an image-selection detail and was actually a falsified assumption about an upstream id format, with a blast radius across every movie. The user's report ("every movie, foreign language") was a more accurate description of the true scope than my first mental model of it. Trust the user's "every" — it often means a systemic cause, not a scattered one.

## UX-flagged open question for the PO

Worth surfacing beyond this retro: when no English poster exists for a title, the current fallback can still show a foreign-language banner. That's the same user complaint in a different aspect ratio, and it wasn't examined this session.
