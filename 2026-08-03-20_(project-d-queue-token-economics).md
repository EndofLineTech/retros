# 2026-08-03-20 — project-d-queue-token-economics

- **ModelID**: claude-fable-5
- **TurnCount**: ~215 total for the session; this retro covers roughly the back half (~turn 100 onward), after the first fix merged
- **SessionDepth**: deep — a seven-PR shipping queue with two review rounds each, a ten-persona ceremony, and a design-then-implement arc for a new test-infrastructure guard
- **Personas Active**: project-engineer (7 implementation dispatches), code-reviewer (9 review dispatches incl. delta rounds), technical-writer (3), it-architect (1 design), plus the full ten-persona team review earlier in the session
- **Beads Touched**: closed — q6xjl, y6zg6, tyei5, kahzn, lsa0s, zt3kf, d20jf, r9oqx, nvhg7; in flight — ax0kf; filed — 63lik, fexq1, 75j5q, czmph, ej4co, ciabe, wd2av, 0cpld, yducf, qfejq

## Section 1: User Value Delivered

Substantial and mostly verifiable. Seven PRs merged (builds 0015→0021, with an eighth in review), turning a documentation test's findings into shipped fixes:

- **The disaster-recovery path works and is proven.** The restore round-trip that failed 7/7 now succeeds against a live instance. Downstream of that: the dry-run preview no longer certifies applies that would fail; failed counts now reach task summaries and notifications instead of dying in a rich report nobody opens; degraded backups report warning-level with the affected category named instead of clean success; and the last dead upstream endpoint was remapped to the real resource.
- **A user-facing safety gap closed.** Both restore-apply paths now require type-to-confirm. The engineer found a *second* unguarded path while fixing the first — one that let an operator skip the dry run entirely.
- **The operator guide shipped**, including 21 articles that existed in the repo but were unreachable from the site nav.
- **The developer doc whose staleness seeded the whole bug class was corrected**, and the commitment nobody was honoring ("restore must stay periodically exercised") got an actual runbook procedure.

Ten beads were filed for findings that surfaced along the way — that's backlog growth, but each is a real defect or gap discovered by work the PO authorized, not invented scope.

The honest caveat: all of this is on the `dev` branch. The PO's production instance still runs an older build, so none of it reaches the actual operator until a normal update. "Shipped" here means "merged and proven on a test instance," not "in the user's hands."

## Section 2: What We Did Well Together

**The PO asked the single highest-leverage question of the session, and asked it about a number rather than a feeling**: "How much waste is there in that sweep in tokens right now? I noticed it's up around almost 300K." That question surfaced a systemic orchestration defect that ~15 fix rounds had hidden in plain sight (Section 4). It worked because it was anchored to an observable — a counter the PO was watching — not to a vibe about the agent being slow. A generic "are we being efficient?" would have gotten a defensive non-answer.

Runner-up, same shape: when a review came back Approved-with-notes, the PO consistently chose "fix now, then merge" over "merge and backlog." That kept five separate classes of half-finished work out of the backlog, and in one case (a review Block on the backup severity fix) it caught a bug that would have shipped the original defect in its worst form.

## Section 3: What the PO Could Improve

**"Merge it" arrived while the code review was still running, on the one PR where the review was the entire point.** The PR added a test-infrastructure guard whose only value is failing when it should; CI passing proves nothing there, because a vacuous guard passes CI beautifully. I held the merge and surfaced the trade-off, and the PO chose to wait — and that review then returned a Block plus two Warns, including proof that four ordinary refactors would leave the guard silently green.

The instruction wasn't wrong so much as early: it was issued a couple of minutes after the review was dispatched, in a session where every prior PR had merged promptly. The cost of complying silently would have been shipping an unverified guard and killing the agent that was about to prove it had holes. A half-sentence — "merge when the review clears" — would have carried the same intent with none of the risk, and it's what the PO actually meant.

## Section 4: What the Agent Got Wrong

**Every fix round in this phase was an agent resumption, and most of them shouldn't have been.** When an agent is resumed to address review findings, it carries its entire prior transcript. The implementation agent for the final bead finished at ~217K tokens; resuming it for a five-item fix meant that round *started* at 217K and climbed toward 300K — to deliver changes across about four files.

That pattern repeated across the whole queue. The actual work in those rounds: `git add` a file that was written but never committed. Retarget one markdown anchor. Add one field to an identity key. Reword a runbook bullet. Clamp a string. Each of those is a tight brief plus a file path — 40–60K of fresh-agent work — and each instead re-paid a full implementation history.

Two aggravating factors I also missed: one agent reported that a Bash timeout had auto-backgrounded **three duplicate full-suite invocations** (8,000+ tests each) and I didn't catch it in its report; and I ran all three roles of the final bead on the top model tier when the design phase didn't need it.

What makes this a real failure rather than a tuning note is that **I never noticed**. Each individual resumption felt justified in the moment — "the agent has the context, continuity is valuable" — and the aggregate was invisible to me because I was reading completion reports, not counters. The PO caught it by looking at a number I had access to the whole time and never summed.

## Section 5: What Would Make the Project Better

**Make dispatch mode an explicit decision with a default, instead of an unexamined habit.** The rule that falls out of this session: a fix round gets a *fresh* agent with the findings as its spec; resumption is reserved for when the fix changes reasoning the agent just built, not merely code it just wrote. Review delta-verification is the interesting edge — a reviewer's method (what it broke, how it proved a guard fails) sounds like it needs continuity, but in practice the findings themselves are a complete brief: "break it these four ways, confirm each goes red."

Second, and cheaper: **orchestrators should tally, not just read.** A per-bead token line next to each merge would have made this visible on PR #2 instead of PR #8. The information was in every completion notification; nothing aggregated it.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: High. Seven merges, every one closing a defect an operator could actually hit. The premise correction on the endpoint remap (the bead named the wrong target; the live schema said otherwise) is the kind of thing that only happens when the implementer is allowed to check rather than obey.
- **Session assessment**: The investigate-then-implement split kept churn near zero — implementation briefs cited confirmed file:line causes, so no dispatch spent tokens rediscovering the problem.
- **What I'd flag**: I wrote 167 lines and 22 tests for the version advisory and then never committed the file. The review caught it. That is exactly the class of mistake the "state the verification performed" rule exists to prevent, and my own completion report claimed test-first discipline for code that shipped with no direct tests.
- **Disagreement**: With the orchestrator's framing that fix rounds are "mechanical." Two of them weren't — the identity-key fix required reasoning about what state a created record would have upstream, and the extractor-completeness fix is a design change. Fresh agents there would have needed most of the context rebuilt anyway.

### Code Reviewer
- **User value assessment**: The reviews earned their cost this phase. A Block on the backup-severity PR caught the original bug surviving in its worst form (all categories silently clean); a Block on the runbook caught guidance that would have taught an operator to dismiss a real failure mid-incident; a Block on the final PR caught a guard that four ordinary refactors would leave silently green.
- **Session assessment**: Delta-review discipline held — every remediation round was verified by re-running the original method, not by re-reading the cited lines.
- **What I'd flag**: The best findings all came from *construction* — breaking things in disposable worktrees and watching what happened — not from reading. Reviews that only read would have approved all three of the above.
- **Disagreement**: With the PM's instinct to keep review scope tight for speed. The eleven-way break test looked like overkill until it produced the two findings that mattered.

### IT Architect
- **User value assessment**: The ADR converts a third recurrence into a decision rather than a fourth incident. Its most valuable sentence is the limitation, stated plainly: the breadth sweep would not have caught either of the original root causes, and treating it as a substitute for deep per-path fixtures is the main way the ADR gets misread.
- **Session assessment**: Measurement before recommendation — 92 call sites extracted and diffed against the live document — meant the design argued from numbers, not taste.
- **What I'd flag**: The guessed-endpoint bug class originated in a code comment that promised later verification and was never revisited. Deferred-verification comments are unowned commitments; they should become tracked work the day they're written.
- **Disagreement**: With QA's "acceptable at this tier" framing on contract breadth. The upstream publishes a machine-readable contract for free — guessing when the answer is downloadable is a choice, not a constraint.

### Project Manager
- **User value assessment**: Good throughput, but the ledger is honest only if the backlog growth is counted: nine beads closed, ten filed. Several of the new ones are genuinely more valuable than what they replaced, but "we shipped seven PRs" alone overstates the net.
- **Session assessment**: The one-PR-at-a-time queue was the right call given the version-bump lockstep, and per-PR merge asks kept the PO in control without stalling.
- **What I'd flag**: Token spend was invisible to the delivery view all session. We tracked merges and beads; nobody tracked cost until the PO asked. A delivery report that omits the resource line isn't a delivery report.
- **Disagreement**: With the code reviewer on the deepest reviews — but I lost that argument on evidence, twice, and the Blocks justify the spend.

### Technical Writer
- **User value assessment**: The operator guide shipped with 21 previously-unreachable articles, and the developer doc that misdirected three bug investigations is now correct and cites the recorded fixtures as source of truth.
- **Session assessment**: Byte-faithful staging worked — the shipping commit authored nothing but a CHANGELOG entry and version strings.
- **What I'd flag**: My first runbook draft told an operator that a denylisted key could explain a FAILED row. It can't — that lands in a different counter. Mid-incident, that line would have taught someone to wave off a genuine failure. Confidently wrong operational prose is worse than vague prose, and I wrote some.
- **Disagreement**: None this phase; the review that caught it was right.

### QA Engineer
- **User value assessment**: The new guard covers an existence/method class that invented mocks structurally cannot catch — mocks of a wrong assumption confirm the wrong assumption.
- **Session assessment**: Red-first evidence was demanded and produced on every bead, including via worktree replay when a claim looked thin.
- **What I'd flag**: The end-to-end restore proof is still a manual act performed once by a human. The runbook now prompts a monthly repeat, but nothing runs it.
- **Disagreement**: With the architect — see above. I hold that recorded fixtures per touched path is the right pace at this tier, and the sweep exists precisely so breadth doesn't require the deep treatment everywhere.

### SRE
- **User value assessment**: Real operator gains: distinguishable failure classes, exception types in resolver logs, degraded backups that announce themselves, and a maintenance habit written down where the on-call person looks.
- **Session assessment**: The severity work reused the existing warnings path rather than inventing new notification copy — the right instinct.
- **What I'd flag**: The Journal line for a degraded-but-successful backup now reads "0 ok, 1 failed" beside `success: true`. Filed, but it's the same confusing-at-a-glance failure one surface over.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: Credential hygiene held through every change — key names never reach log sinks, values never appear anywhere, and the new advisory leaks neither host nor credential.
- **Session assessment**: The one Low finding I'd carry forward is unbounded upstream text reaching a log line; it's now being clamped.
- **What I'd flag**: A guard that can't fail is a security problem, not just a quality one — it manufactures confidence in the exact area that already produced three defects. The decision to prove failure by construction was correct.
- **Disagreement**: None.

### Database Engineer
- **User value assessment**: Restore-path integrity was examined rather than assumed — rollback semantics, partial-failure consistency, and the deliberate dropping of per-instance ids at export all held up.
- **Session assessment**: The abort-on-any-failed-key policy is now documented rather than emergent, which is the more valuable half of that decision.
- **What I'd flag**: Earlier in the session I stated a module-level contract as if it were system behavior and contradicted observed evidence. Module comments describe a layer, not the system.
- **Disagreement**: With myself, appropriately.

### UX Designer
- **User value assessment**: Two genuine operator wins: restore failures are now distinguishable by class instead of a wall of identical upstream errors, and both destructive apply paths require deliberate confirmation.
- **Session assessment**: Mostly dormant this phase; the confirm-dialog work was engineer-led and correct.
- **What I'd flag**: The shared confirm dialog still lacks dialog semantics for assistive tech, on the control that now guards both config-overwriting paths. Filed rather than rushed, which I agree with.
- **Disagreement**: None.

## Lessons

- **Keep**: Fix-now over merge-and-backlog when a review returns notes; proving guards fail by construction rather than by reading test names; premise checks that let an implementer contradict the bead when live evidence disagrees.
- **Stop**: Resuming agents for mechanical fix rounds. Resumption re-pays the entire prior transcript; a five-item fix should not carry a 217K implementation history. Also stop reading completion reports without summing their cost.
- **Start**: Treating dispatch mode (fresh vs. resumed) as an explicit per-round decision with fresh as the default, and reporting a per-bead token line alongside each merge so aggregate waste is visible at PR #2 instead of PR #8.
- **Value learning**: The highest-value question of the session came from the PO watching a counter, not from any agent's self-assessment. Agents evaluate their own work on correctness and miss resource patterns entirely — every individual resumption was defensible, and the aggregate was still a systemic defect. Instrumentation the orchestrator actually sums beats introspection it doesn't.
