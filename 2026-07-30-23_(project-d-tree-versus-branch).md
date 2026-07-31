# 2026-07-30-23 — project-d-tree-versus-branch

- **ModelID**: claude-opus-5
- **TurnCount**: ~110 cumulative (≈45 user messages, ≈65 assistant turns), one context compaction
- **SessionDepth**: deep — this retro covers the **ship phase** of a long session. An earlier retro (`2026-07-30-22_(project-d-orchestrator-bypassed-guard)`) covers the build phase and the hook finding; this one deliberately does not re-litigate that ground
- **Personas Active**: technical-writer (dispatched for version bump + changelog); project-engineer, QA, SRE, PM, code-reviewer, UX, database-engineer (perspectives)
- **Beads Touched**: none created this phase; the phase closed out work from ~38 beads filed earlier in the session

---

## 1. User Value Delivered

**None yet, and that is the honest answer for this phase.**

The PO asked to merge so people could preview the new UI. At the time of writing: the branch is pushed, a PR is open against the integration branch, and **two required checks are failing** — the static-analysis suite and a release-matrix job. Nothing has merged. No one outside the PO can preview anything.

What this phase *did* produce is a defect caught before it reached anyone:

**A committed stylesheet consumed brand tokens that only existed in an uncommitted file.** The consumer carried a fallback of white; the light-theme sidebar is near-white. Merging the branch as it stood would have shipped an invisible brand mark in one of three themes, to precisely the audience being invited to look at the new UI. Found by asking "does anything committed depend on what is uncommitted?" rather than by any gate — the contrast guard measures text and glyphs, not SVG fills, so it would have stayed green.

That is real value, but it is *avoided harm*, not delivered function. The session's actual user value was banked earlier and is documented in the previous retro.

---

## 2. What We Did Well Together

**The PO's "let's merge" prompted a state check instead of a merge, and the state check found the defect.**

The instruction was unambiguous — merge, let people preview. The tempting execution is `gh pr create` and move on. Instead the first action was to enumerate what was uncommitted, and the second was to ask whether anything committed *depended* on it.

That second question is the one that mattered. Four uncommitted files looked like leftover polish; the token/consumer split meant they were load-bearing. When it went back to the PO it was as a decision with the consequence measured — contrast ratio, the exact sidebar colour, which element survived on a theme-aware fallback and which did not — rather than as "there's some uncommitted stuff, what do you want to do?"

The PO answered in one word. That is what a well-framed decision looks like from both sides.

---

## 3. What the PO Could Improve

**"Lets merge. Time for folks to see the new UI" arrived with no request for the branch's state, and the branch's state was not good.**

At that moment: 102 commits had never been pushed, never seen CI, and included four uncommitted files of prior-session work that nobody had reviewed. The PO knew the work had been happening — they had been previewing it all night — but the instruction assumed readiness that had not been established by anything except my own local gate runs.

It worked out because the state check happened anyway. But the PO's framing put the burden entirely on me to volunteer the bad news, and I have demonstrably been an unreliable narrator this session — five briefs wrong on substance, four misread gate outputs. A one-line "what's the state before we merge?" would have made the check explicit rather than dependent on my judgement.

**Second, smaller:** the retro was requested while a required check was failing and I had not yet reported it. The retro is more useful written after the merge resolves — right now Section 1 has to say "nothing shipped," which will be wrong within the hour and cannot be corrected in a synced public file.

---

## 4. What the Agent Got Wrong

**I put an internal measurement into an operator-facing document, and the PO had to remove it.**

The Discord preview post included "the channels/streams split is now even. It was 58/42, leaving streams ~270px narrower." The PO's correction: *"that was mainly for the tests we were doing."*

The failure is one of audience, and it is a direct overshoot of a habit that was correct everywhere else this session. Every commit message, bead and brief tonight was improved by being specific and measured — naming the pixel counts, the contrast ratios, the element counts. That discipline is right for engineering artifacts, where the number is the evidence. I carried it unchanged into a post whose readers do not care what the ratio *was* and have no way to verify it. To them "58/42" is noise that implies the old value mattered.

I also wrote "**has landed on dev**" in that post while the PR was still open. I caught it myself and flagged it — but only after writing the whole thing, which means I drafted an announcement of an event that had not happened and would have handed it over had the PO not been reading carefully.

**And the substantive one:** I described the PR as "the first time any of these commits sees CI" while treating my local gates as near-equivalent coverage. They are not. Within minutes CI failed two checks that no local gate runs. I had said the words about CI being unvalidated territory without actually behaving as though I believed them — I opened the PR expecting a formality.

---

## 5. What Would Make the Project Better

**Nothing forced CI to run until 104 commits had accumulated, and the local gate suite creates a false sense that it substitutes.**

The local suite is genuinely good — type-check, lint, ~2,500 unit tests, a backend suite of ~8,000, and eight rendered-CSS browser guards that measure the compositor. It is thorough enough that passing it repeatedly *feels* like validation. It ran green all night.

But it does not run static analysis, and it does not run the release-matrix job. Those are exactly the two things that failed. The gap is not "we forgot to run something" — it is structural: those checks only exist in CI, and CI only runs on a PR.

Two things would help, in order of value:

1. **Push a long-lived branch early, even without a PR**, so branch-triggered workflows run continuously instead of everything landing at once. Discovering a static-analysis finding at commit 5 is a small fix; discovering it at commit 104 means bisecting a night's work.
2. **Name explicitly, in the shipping guide, which checks exist only in CI.** Right now an engineer running the documented local gates has no way to know what they are *not* covering. "Gates pass" and "CI will pass" are being treated as the same claim by the very document that lists the gates.

There is a related pattern worth naming: this is the third time in one session that a *guard's* coverage boundary was the problem rather than the guard's correctness — the edit hook could not see worktrees, the control guards open no dialogs, and now the local gates do not include static analysis. Each was individually reasonable; collectively they mean "green" has been over-trusted all session.

---

## 6. Persona Perspectives

### Project Engineer
- **User value assessment**: The engineering work is done and deployed to the PO's instance. The ship phase added no function, and the one defect it caught was real.
- **Session assessment**: The version bump was dispatched to a persona rather than hand-edited, which is a direct behaviour change from the correction recorded an hour earlier in the previous retro. Small, but it held under time pressure — that is when process corrections usually fail.
- **What I'd flag**: The uncommitted work was carried in the working tree for an entire session while every deploy was built from that tree. The container and the branch silently diverged, and nobody would have noticed until a fresh clone.
- **Disagreement**: With the PM on "104 commits is a delivery risk" — the risk was never the commit count, it was that they were unpushed. Commit count is a symptom.

### QA Engineer
- **User value assessment**: The guards built this session will protect users. But this phase demonstrated their boundary, which is the more useful finding.
- **Session assessment**: Local gates were run diligently and repeatedly, including re-running an agent's claimed numbers independently. That discipline was real. It was also insufficient, and the insufficiency was structural rather than sloppy.
- **What I'd flag**: **The invisible-logo defect was undetectable by every guard in the suite.** The contrast guard measures text and glyph colour; the failure was an SVG `fill` resolving to a fallback. A guard estate that has been extended eight times this session still has no assertion that a token a stylesheet consumes is actually defined.
- **Disagreement**: With the Project Engineer's framing that the local suite is "genuinely good" — a suite that passes on a branch which then fails CI twice is not good enough, whatever its test count.

### SRE
- **User value assessment**: The migration heal and the fail-soft version check protect real deployments. Neither has been validated by anything but local runs.
- **What I'd flag**: The PR body correctly states the container now needs outbound access to a third-party service. That belongs in release notes and an upgrade note, not only in a PR description that will be read once and archived. It is currently in the changelog, which is the right place — but nobody has confirmed the deployment documentation mentions it.
- **Disagreement**: None on the design. My concern is purely that an operational prerequisite is documented in artifacts operators do not read.

### Technical Writer
- **User value assessment**: The changelog entries are genuinely operator-facing and every figure traces to a commit body. The Discord post needed a PO correction to reach the right register.
- **Session assessment**: The dispatched writer caught a compatibility cost that was never briefed — that pinning deep-link anchors breaks five previously-shared links — and put it in the changelog on the reasoning that an operator with a bookmark should learn it there rather than from a link that stops working. That is the persona doing its job rather than executing instructions.
- **What I'd flag**: **The orchestrator's habit of quantifying everything does not transfer to audience-facing writing**, and the PO had to catch it. The same instinct that makes a commit message trustworthy makes an announcement noisy.
- **Disagreement**: With the orchestrator's initial instinct to keep the removed detail in the changelog "because operators will notice" — that was right for the changelog and wrong for the post, and the distinction was nearly missed.

### Project Manager
- **User value assessment**: A night of work is sitting behind two red checks. Until it merges, the value is potential, not delivered.
- **Session assessment**: Sequencing was good within the phase — state check, decision to the PO, version bump, changelog, push, PR — and the PR body is unusually honest about its own limits.
- **What I'd flag**: **Requesting a retro while the PR is red is out of order.** The retro will be published saying nothing shipped, which will be stale almost immediately. Retros should follow resolution.
- **Disagreement**: With the QA Engineer's implication that the local suite failed — it did what it does. The failure was in *treating* it as equivalent to CI, which is a communication problem, not a tooling one.

### Code Reviewer
- **User value assessment**: The unreviewed commit was included for a defensible reason and the reason is stated in its own message. That is the right handling of a compromise.
- **What I'd flag**: One commit in this branch carries work that no persona wrote and no reviewer read, and its message says so plainly. That transparency is worth more than pretending otherwise — but it does not make the code reviewed. It ships on the PO's explicit decision, and that decision is now permanently recorded, which is the correct outcome.
- **Disagreement**: With the PM's satisfaction about sequencing — the branch should have been pushed 100 commits ago. Good sequencing at the end does not compensate for a review-free hundred-commit run.

### UX Designer
- **User value assessment**: The preview announcement is written for operators and mostly lands. The removed line proves the register was not calibrated on the first pass.
- **What I'd flag**: The post invites people to "tell us what feels wrong" without saying *how* to preview. That was a deliberate omission — the mechanism was unknown and inventing it would have been worse — but an invitation without a path is a weak invitation.
- **Disagreement**: With the Technical Writer on the audience lesson being about quantification. It is about *purpose*: the number was not too precise, it was answering a question the reader never asked.

### Database Engineer
- **User value assessment**: The destructive migration and its heal are in this PR and have never run in CI.
- **What I'd flag**: The migration was verified against a snapshot of a real database and on scratch databases in both directions, which is strong. But the branch containing it has now failed two CI checks, and until those are understood I would not assume the migration path is unaffected.
- **Disagreement**: With the general confidence in this phase. A branch carrying a destructive migration deserves more caution at merge time than a branch carrying CSS, and it was treated with the same ceremony as the rest.

---

## 7. Lessons for Future Sessions

- **Keep**: When told to merge, first ask what the working tree contains and whether anything committed *depends* on what is not. That question — not the file list — is what found a defect no guard could see.
- **Keep**: Dispatching even trivially small changes to a persona. The version bump was three literals and it still went to a persona, an hour after the retro that identified self-editing as the failure. The correction held under time pressure, which is the only test that counts.
- **Stop**: Treating a green local gate suite as a prediction about CI. They cover different things, and the difference is invisible from inside the local run. Say "local gates pass" and stop there.
- **Stop**: Carrying engineering precision into audience-facing writing. A measurement is evidence to a reviewer and noise to a reader. Ask what question the reader is asking before including a number.
- **Start**: Pushing long-lived branches early so branch-triggered CI runs continuously. A hundred commits between pushes converts a small fix into a bisection.
- **Start**: Writing announcements in the tense of what is *true now*, not what is expected shortly. "Has landed" was drafted while a PR sat open.
- **Value learning**: The PO's request was "let people see the new UI." The thing that nearly prevented that was not any of the UI work — it was **a gap between what had been previewed and what had been committed**, invisible to every automated check, and discoverable only by asking whether the committed half of the codebase depended on the uncommitted half. When a working tree and a branch diverge for an entire session, the divergence itself is the risk, regardless of how good the work in it is.
