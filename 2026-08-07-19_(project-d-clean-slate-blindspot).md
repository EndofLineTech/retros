# 2026-08-07-19 — project-d-clean-slate-blindspot

- **ModelID**: claude-opus-5
- **TurnCount**: ~78 (12 genuine PO inputs including 3 structured decision responses; ~66 assistant turns)
- **SessionDepth**: deep — a full ~2h20m disposable-stack drill across two products, four restore rounds, then three shipped PRs touching backend, frontend and docs, then a style audit
- **Personas Active**: QA Engineer (the drill itself), Project Engineer (2 dispatches), Technical Writer (3 dispatches), SRE, Security Engineer, Code Reviewer, IT Architect, UX Designer, Project Manager, Database Engineer
- **Beads Touched**: 8 — created and closed a run-tracking bead; filed 3 product defect beads (all fixed, merged, closed); filed and closed 2 docs beads; filed 1 follow-up obligation bead (left open, deliberately); filed and closed 1 style-violation bead

---

## Section 1: User Value Delivered

Real, and unusually traceable for a session that started as a test exercise.

The session ran a scripted disaster-recovery drill in "doc-following" mode: complete a full backup → destroy → restore round trip using **only the published documentation**, no repository access. The point is to find out whether an operator holding nothing but the docs can actually recover their instance.

**Value shipped:**

1. **Two silent data-shaping defects found and fixed.** Restoring onto an instance that already has content (the migration and disaster-recovery case) silently kept the destination's grouping instead of the archive's, and adopted a same-named-but-different container object while the report claimed its contents had been compared. Both reported success with zero failures. An operator would have read a clean report over a lineup that had quietly kept the wrong organisation. Now reported under the default mode, and reconciled under the overwrite mode.
2. **A reporting defect fixed**: the dry-run preview diverged from the apply on one category and omitted another category entirely — including the one that synthesizes placeholder records on a match miss, i.e. exactly what an operator most wants forewarning about.
3. **Five documentation defects fixed and live**, one of them consequential: the guide asserted that a particular configuration "isn't discoverable from the product's own screens; you have to go to [the other product] to set it up." That is false — the product has a first-class editor for it. The docs were sending operators to the wrong application. Verified gone from the published site.
4. **The drill's own procedure was extended** so future runs actually exercise the populated-target path, which no run had ever done.

**Work created that doesn't directly serve users**: one open follow-up bead (run the new procedure to prove the three fixes against a live populated target). That is a genuine obligation, not busywork — the fixes are proven by tests, not by a drill, and the docs say so.

---

## Section 2: What We Did Well Together

**The PO's mid-turn interrupt, roughly turn 30: "Wait. How much are you doing that's API-driven versus playwright driven?"**

I had been drifting. The drill's mode requires following the *published documentation*, which is written for the UI. I had started using the API wherever it was "equivalent" because it was faster and more reliable to script. That drift was quietly hollowing out the entire run: measuring via API doesn't test what an operator following the docs would actually hit, and the headline finding ("can someone with only the docs recover?") would have been unearned.

The PO caught it with one question, mid-turn, before I'd gone too far. I laid out the actual split honestly, flagged that I'd been drifting and why it mattered, and offered three scoping options. The PO chose the strictest: UI for every documented step, API only for reading state back and for the one creation path the docs themselves say has no UI.

That interrupt is the reason the run's conclusions are worth anything. It also directly produced the strongest documentation finding of the session — the "you have to go to the other product" defect is only discoverable by actually driving the product's own screens and noticing the control the docs claim doesn't exist.

---

## Section 3: What the PO Could Improve

**Partial answers to multi-part questions, twice, with real downstream cost.**

Two concrete instances:

1. **The stray "rem".** Around turn 52 the PO sent a single message: `rem`. I couldn't tell whether it meant *remove*, *remind*, *remaining*, or a stray keystroke, so I flagged it and continued the work already in flight. I re-raised it in three subsequent turns. It was never answered. That's not catastrophic, but it left an unresolved thread I had to keep carrying and re-surfacing, and if it *did* mean something I've silently dropped it.

2. **The unanswered enforcement question — this one has consequences.** When I reported the style-rule violation, I asked two things: what scope to fix, and whether to add an automated guard. The PO answered `Docs only.` — the scope half — and did not address the guard. I asked again in the closing summary. Still unaddressed at session end.

That matters because the evidence for needing the guard is now strong: two separate agent documentation passes introduced 144 violations of a rule the project already documents as canonical, and three reviewers (two writer agents and me) shipped them through two merged PRs without noticing. Without enforcement, the next agent almost certainly repeats it, and we'll spend a fourth PR on it. Fixing the symptom while declining to answer the question about the cause is the expensive half of the choice.

A one-word "no" or "later" would have closed it. Silence reads as neither, so it stays open and I keep spending turns re-raising it.

---

## Section 4: What the Agent Got Wrong

**I introduced 144 violations of a rule the project explicitly documents as canonical, and never opened the file that contains it.**

The project's instructions name a style guide as *the canonical reference for conventions*. That guide contains an unambiguous rule against em-dashes in prose, and even prescribes the rewrites. I never read it. Across two PRs I wrote and reviewed documentation prose that used em-dashes at roughly **60× the density of the article's own established voice** (0.2 per 1,000 words before, 12.9 after). Two technical-writer subagents didn't catch it either — because *my briefs never told them the rule existed*, and I hadn't looked.

It surfaced only because the PO asked directly, at turn ~66, after three PRs had merged. That's a straightforward process failure on my part: I dispatched writers repeatedly without ever grounding them in the project's own style authority.

Three smaller ones from the same session, all worth naming:

- **I nearly reported my own tooling bug as a product defect.** My HTML-to-text extractor truncated the published compose file and ate `<placeholder>` tokens. I was one step from filing a BLOCKER doc finding before checking the raw HTML.
- **I pointed a subagent at pre-fix evidence.** When briefing the new drill procedure, I cited the run's own findings file as the source for what a passing run should assert. That file documents the *broken* behaviour — its findings are the inverse of the fixed behaviour. The subagent caught it and reconciled correctly. Had it followed my brief literally, we'd have shipped a test procedure that asserts the bug.
- **My harness leaked where the drill's own rules forbid leaking.** The drill mandates a PII gate on every screenshot because the product renders credential-bearing URLs. My driver's *error* path called the screenshot function directly, bypassing the gate, and wrote 17 ungated frames from pages showing live provider credentials. I found and deleted them, but the drill exists partly to protect against exactly that and my tooling had the hole.

---

## Section 5: What Would Make the Project Better

**Test and drill fixtures that always start from a clean slate systematically hide the merge/upgrade path — and this project just proved it across eleven runs.**

Every restore drill before this one wiped the destination first. On an empty destination, the two "what happens to things you already have" modes are *indistinguishable* — there's nothing to preserve or overwrite — so a green result meant nothing about either. Two defect classes were therefore **structurally unreachable** for eleven runs, and both turned out to be silent, success-reporting, data-shaping bugs in the single most safety-critical path the product has.

The generalizable version: any harness whose setup is "destroy, then create" only ever tests the greenfield path. The install case gets eleven runs of coverage; the *upgrade/merge/migrate* case gets zero, while being the case with more state, more conflict, and more ways to silently do the wrong thing.

Concretely for this project, two things:

1. **Make "populated destination" a standing fixture dimension**, not a special round someone remembers to add. It's now a documented procedure step, which is progress, but a procedure step relies on a human choosing to run it.
2. **Automate the style rule.** A grep-based check over documentation and code comments for the banned character would make the canonical guide self-enforcing rather than dependent on someone thinking to look. This is the second retro-worthy instance of agents drifting from this project's documented prose conventions — the article's own two pre-existing violations came from an *earlier* agent doc pass.

   Worth noting against my own work: **this retro contains 41 em-dashes in 3,787 words**, and the previous session's retro contains 42 in 3,974. The retro corpus has no such rule, so neither is a violation. But it demonstrates the point precisely — the tic is not project-specific drift that better briefing fixes, it is a property of how the model writes by default, present in every register it produces. Anywhere a project cares about it, only mechanical enforcement will hold the line; asking the agent to remember will not.

---

## Section 6: Persona Perspectives

### QA Engineer
- **User value assessment**: High, and the clearest of any persona. The drill's whole premise — measure, don't trust the report — is what produced every finding. The product reported `success / failed 0` on runs where a lineup's organisation was silently wrong. Counts reconciled perfectly the entire time.
- **Session assessment**: Methodology mostly held. The blind-measurement discipline worked: all findings were flagged before opening the sealed run history, and the seal point is recorded. Playback was fetched rather than inferred, logos compared by hash, credentials by fingerprint.
- **What I'd flag**: The playback tool passed a channel on **188 bytes** — one transport packet — against the article's own stated bar of "a few hundred KB". A 2/2 PASS was reported on evidence three orders of magnitude below the documented standard. That's a false-confidence generator sitting inside the tool whose entire job is to prevent false confidence. Also: the inventory diff tool reports "did the restore reproduce the instance? **NO**" on a *correct* restore, because 375 timestamp-churn rows count as findings and bury the 4 real ones. A run trusting that headline would report a pass as a failure.
- **Disagreement**: With the Project Manager's framing that this session shipped cleanly. Two of the three product fixes are proven by unit tests that *I did not write and cannot fully audit from here*, and no live drill has exercised them. Shipping on that is defensible; calling it done is not. The follow-up bead is the only thing keeping that honest.

### Project Engineer
- **User value assessment**: Genuine. The reconcile pass closes a case where a restore silently produced a wrong-but-plausible result. That's the worst failure mode a recovery tool can have.
- **Session assessment**: The implementation was better than my brief. I specified comparison by remapped identifier; the implementing agent wrote that first, caught it with its own preview/apply parity test, and switched to comparing by name — the id version mis-reported 3 of the 7 known-drifted records. That's the agent's test catching the orchestrator's spec error.
- **What I'd flag**: The agent walked into the exact destructive-git hazard the project instructions call out — ran a checkout to revert a temporary test change and discarded two real edits with it. It caught it, re-applied, re-verified, and **reported it unprompted**. That self-report is worth more than the near-miss cost. I verified the file independently and nothing was lost.
- **Disagreement**: With the Code Reviewer on scope. The style cleanup covered documentation only; 97 violations remain in code comments and docstrings, which the same rule covers. I think that's the right call for now — a mechanical 14-file comment diff would bury the semantic history in blame for no user benefit — but it is debt, and it's recorded.

### Technical Writer
- **User value assessment**: The highest-leverage fix of the session was a documentation fix, not a code fix. The guide told operators a capability didn't exist in the product they were already using and sent them to a different application. Every operator following that article was being misdirected.
- **Session assessment**: Mixed, and I'll own the bad half. Three dispatches produced good structural work, and two of them pushed back correctly — one refused to mark fixes as drill-proven when only tests proved them, one caught the pre-fix-evidence trap. But none of us consulted the project's canonical style guide, because the orchestrator never pointed us at it.
- **What I'd flag**: The distinction the writer agent insisted on — "fixed at code/test level" versus "confirmed by a live drill" — is the single most important editorial decision this session. This article's credibility rests on only calling something *measured* when a drill measured it. Preserving that hedge through three PRs, including declining to remove it when the fix landed, is what keeps the document trustworthy.
- **Disagreement**: With the Project Manager. Shipping a third PR purely for punctuation looks like waste on a velocity view. It isn't: the article's voice is its authority, and prose that reads as machine-generated undermines a document whose entire function is to be believed by someone in the middle of a disaster recovery.

### Code Reviewer
- **User value assessment**: The quality gates caught real things. Verifying agent-claimed gates independently — re-running the full backend suite, both frontend gates, the docs build — is what let three PRs merge without a CI surprise.
- **Session assessment**: Verification discipline held throughout. Every agent's claimed gates were re-run by the orchestrator before merge, and each PR's required contexts were checked **individually** because this repo has a known defect where duplicate same-named checks can show green over a real failure. That check mattered: both instances had to be read separately every time.
- **What I'd flag**: The em-dash failure is a review failure, not just an authoring failure. I reviewed those diffs for correctness and meaning and never once checked them against the project's documented conventions. Correctness review that ignores the style authority isn't complete review.
- **Disagreement**: With the Project Engineer on the deferred code comments. 97 violations in docstrings is not cosmetic when one of them is a shipped operator-visible string and one reaches the public API schema. I'd have taken the whole sweep in one pass.

### IT Architect
- **User value assessment**: The architecture question here was decided correctly and for the right reason. Matching containers by *name* across two instances is not a shortcut — it is the only identity available, because the archive carries no cross-instance identifier. The fix preserved that and changed the *claim* instead.
- **Session assessment**: The decision framing was the strongest part of the session. Rather than "fix the bug", the options were surfaced as report-only / reconcile-under-one-mode / always-reconcile, each with its blast radius. The PO picked the middle, which widens a documented contract but keeps the default safe.
- **What I'd flag**: That choice has a cost nobody has fully priced. The overwrite mode now moves records between containers, which is materially more destructive than what its previous description promised. The label and hint copy were updated, but an operator with muscle memory for the old meaning may not re-read them.
- **Disagreement**: With the SRE's comfort about the deferred proof. A behaviour change to the recovery path that has never run against a real populated instance is exactly the change I'd want exercised before an operator meets it during an actual outage.

### SRE
- **User value assessment**: Strong. The drill *is* the disaster-recovery exercise, and it ran end to end with real teardown, real destruction, and real verification that production was untouched.
- **Session assessment**: Safety discipline was solid. Every destructive command was preceded by an assertion showing what it would hit, and confirming that the production containers belonged to different projects and were unreachable from the drill's scope. Teardown verified containers, volumes, network, ports, and revoked the minted credential (confirmed rejected afterward).
- **What I'd flag**: The publish pipeline remains untrustworthy and this session re-proved it — the forced image re-pull reported "downloaded newer image", meaning the locally-tagged build really was stale. The build-marker gate is the only thing standing between a run and validating the wrong artifact.
- **Disagreement**: With the Architect. Gating on a live drill would have meant a build-and-publish cycle and a fresh drill before any fix reached anyone, while two silent data-shaping defects stayed live in the field. Shipping on strong tests with an explicit, tracked, documented obligation to prove it is the better risk trade. The docs say plainly it's unproven; nobody is being misled.

### Security Engineer
- **User value assessment**: One real, self-inflicted near-miss, caught and remediated. The product renders subscription credentials in stream URLs, and the repository's image directory is public.
- **Session assessment**: The screenshot gate worked exactly as designed — it *failed closed* and refused to write frames, repeatedly. The reason it failed was a genuine bug in the gate (values split across a text node and a styled child element), which was fixed properly rather than bypassed.
- **What I'd flag**: The orchestrator's own driver bypassed the gate on its error path and wrote 17 ungated frames from pages displaying live provider credentials. They were deleted, and nothing was published. But the lesson is structural: a safety gate that the happy path calls and the error path skips is not a gate. Error paths are exactly when you're least careful and most likely to capture raw state.
- **Disagreement**: None substantive, but I'll note the credential-in-cleartext finding re-confirmed this session (the redacted artifact still stores the provider username in plaintext while redacting the password) was correctly *not* re-filed as new — it's known and open. Re-confirming a known issue with fresh evidence is the right handling.

### Project Manager
- **User value assessment**: Three PRs merged, eight beads moved, three product defects and five documentation defects closed, all traceable to operator-visible harm. Good throughput for one session.
- **Session assessment**: Work was well-sequenced: measure → decide → implement → verify → ship, with decisions surfaced as explicit structured choices rather than buried in prose. The PO answered four decision points in one pass, which unblocked implementation cleanly.
- **What I'd flag**: Scope grew three times without being re-planned. The session was commissioned as "run a drill", became "fix the docs", then "fix the beads", then "audit the prose". Each step was justified, but nobody ever asked whether the third PR was the best use of the session versus, say, running the new procedure to actually prove the fixes. We now have an open obligation that requires another full drill.
- **Disagreement**: With the Technical Writer. A punctuation-only PR carrying a version bump, a full CI cycle, and a merge is real cost. I accept the credibility argument, but this should have been caught in the first PR — the fix cost is the *symptom*; the absent guard is the actual issue, and that question is still unanswered.

### UX Designer
- **User value assessment**: Modest but real. The two mode labels now say what they actually do, having previously promised only guide data and logos while the underlying behaviour widened to include organisation.
- **Session assessment**: The copy work was careful. The chosen wording was checked against how the surrounding interface already uses the term, and the preserve label was extracted to a shared constant so the two places that quote it cannot drift apart.
- **What I'd flag**: The panel that reports this now renders for a case where it previously rendered nothing — when the check *didn't run*. The implementing agent noticed that the existing title ("things you already had were left alone") is a claim about state that was inspected, and correctly changed the title and icon for the not-checked case. That's a subtle, genuinely user-protective distinction that a less careful pass would have missed.
- **Disagreement**: With the Project Engineer's deferral of the one shipped operator-visible string still carrying the banned punctuation. It's one string in a panel operators read during a recovery. That's the *least* deferrable instance of the whole set.

### Database Engineer
- **User value assessment**: Indirect but real. The defects were about identity and referential reconciliation across two instances — which record is "the same" record when the identifiers don't carry across.
- **Session assessment**: The identity reasoning was sound. Name is the only stable cross-instance key; the ordering constraint (containers restore before their members, and membership lives on the member) is genuine and correctly drove the fix into a post-pass rather than into the importer.
- **What I'd flag**: The fix reports and optionally repairs a divergence, but the underlying identity weakness stands: two distinct objects sharing a name are indistinguishable to the restore. That's tolerable today because the reporting is now honest. It stops being tolerable if anything ever automates on top of these reconciliations.
- **Disagreement**: None. The alternative (treating a name collision as a hard conflict) was correctly rejected — it would cascade to dependency failures and could degrade a large restore over one duplicated name.

---

## Section 7: Lessons for Future Sessions

## Lessons

- **Keep**: Independent verification of every agent-claimed gate before merge, and reading each required CI context *individually* on repos where duplicate same-named checks exist. This session had three PRs merge without a single CI surprise, and the duplicate-context defect was live on all three. Also keep: filing a follow-up bead that carries an unproven residual forward, rather than letting "fixed" and "proven" collapse into each other at close time.
- **Stop**: Dispatching writing agents without first grounding them in the project's canonical style authority. Three dispatches, 144 rule violations, zero of them caught internally. The orchestrator's brief is where a documented convention gets enforced or lost.
- **Start**: Reading the project's own conventions file *before* the first dispatch, not after the PO asks. And: when a harness has a safety gate, audit the **error** paths for gate bypass, not just the happy path — that's where the 17 ungated captures came from.
- **Value learning**: Users of a recovery tool are not harmed most by loud failures — they're harmed by **confident, successful-looking reports over quietly wrong state**. Every high-value finding this session was of that shape: counts reconciled, status said success, and the result was wrong. The corollary for test design: a fixture that always starts empty can only ever test the greenfield path, and the merge/upgrade path — the one with more state and more ways to be silently wrong — gets zero coverage while looking fully covered.
