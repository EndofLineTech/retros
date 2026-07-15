# 2026-05-15-22 — retro-driven rule upgrades

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: ~52 (user + assistant, including tool-result turns)
- **SessionDepth**: deep — synthesized 18 prior retros, rewrote 8 sections of `_shared/orchestration.md`, extended `engineering-discipline.md`, authored a new `/release-check` skill with supporting README/CHANGELOG/installer edits, shipped to `origin/main`
- **Personas Active**: None directly spawned. Thematically relevant: Project Engineer (rule #1 scope), Project Manager (Findings-as-Notes rule), SRE (/release-check skill, Re-Verify Gates rule), Code Reviewer (review-history rule, gate verification), Technical Writer (README/CHANGELOG/SKILL.md docs), QA Engineer (pre-existing failures escalation, release readiness gating)
- **Beads Touched**: None — meta-work on the skill system itself

## 1. User Value Delivered

This session delivered meta-leverage — improvements that compound across every future project using these skills.

Concrete deliverables, all shipped to `origin/main` as commit `a12a62e`:

1. **`_shared/orchestration.md`** — 8 sections rewritten or added:
   - "Claude Orchestrates — Personas Implement" rewritten around "skills define each persona's scope," with hard 3-read pre-dispatch limit, named-rationalization rejections, and mid-task drift clause
   - "Worktree Isolation" — trigger expanded from "will commit" to "might commit," + orchestrator-self discipline subsection + CWD drift subsection
   - "Authorization Verbs" (new) — verb-mapping table separating push/PR from merge, plus catch-all-vs-flagged-decision rule
   - "Findings From Personas Are Notes, Not Beads" (new)
   - "Verify Premises Before Briefing" — added framing checks (source/environment, already-shipped, inherited premise)
   - "Decision Prompts" (renamed from "Decision Prompt Compression") — structured `## DECISIONS NEEDED` block with dependency notation, hard cap of 3, one-by-one mode
   - "Re-Verify Gates on Engineer Report" (new)
   - "Definition of Done for User-Reported Bugs" (new)

2. **`_shared/engineering-discipline.md`** — pre-existing failures escalation clause added

3. **`/release-check` skill** (new, v0.3.0) — semantic readiness gate that refuses to fire release-execution until P0/P1 bugs clear and PO explicitly confirms. Directly addresses the 04-20-20 retro's 7-hour rollback scenario.

4. **README.md** — new "Operational Skills" section in both the skills table and the Usage codeblock

5. **CHANGELOG.md** — `[0.3.0]` section populated with the full diff summary

Honest framing: these are *specs*, not *validated improvements*. The new rules haven't been stress-tested in real sessions. The `/release-check` skill has zero usage data. The user value is in how the system *will* behave on future projects — predicated on the rules sticking, which is the exact failure mode several of these rules try to prevent.

## 2. What We Did Well Together

**The moment**: turn ~5, when I proposed a "strict vs pragmatic" framing for what counts as persona work, the PO replied: *"Strict. I want a hard limit or you'll be unbound. I think we need to look at it as, the skills define what the work is each team member does, we need to follow those as the boundaries - as soon as we identify 'this is done by persona x' - we need to have it done by persona x."*

That single response did three things:
- Replaced my "no code edits" framing with a tighter "skills are the boundaries" framing
- Burned the small-change exception by name (hard limit, not qualitative)
- Killed my "rationalization is the signal" proposal as too soft

I was orbiting the rule. The PO landed it. The version of rule #1 that shipped is materially better than the version I would have written alone, and the back-and-forth took exactly one turn each direction. That's the collaboration working at its best — PO brings the principle, agent shapes the prose, ship.

## 3. What the PO Could Improve

**The moment**: I asked three follow-up questions on `/release-check` (version number, README placement, CHANGELOG timing) and the PO responded "Yep, green light." I interpreted that as approving the SKILL.md content only — because I'd explicitly framed it that way ("Green light on the SKILL.md content first, then we can sort the rest?"). So I had to re-ask the three questions, which the PO answered with "1. Yes, 0.3.0 is fine. 2. Make an operationsal skills. 3. Write it now."

The cost: one round-trip. Small. But notable because it's exactly the failure mode we'd just written into `_shared/orchestration.md`: *"When the PO answers with a single digit ('2') or a single word ('Go'), that's a signal the format is working. But it's also a signal there's no backpressure when a prompt is overloaded."*

The PO used the catch-all format I provided without testing whether their answer covered everything I'd asked. The new doc warns about this exact pattern. If the PO had answered "yes to all three" or "1: 0.3.0, 2: op skills, 3: write it now" the first time, we'd have saved a turn.

Smaller version of the same pattern: my "## DECISIONS NEEDED" block at the end of the rule #1 violation flag asked two questions; the PO's "So, this is different. I'm not asking you to use skills here. You're fine writing them. 2. Lets make one bundled commit." answered question 2 by number but answered question 1 by reframing it entirely. That was actually the *right* response (the question was malformed) — but it depended on the PO seeing through my framing. A less-careful PO would have answered "1: roll back, 2: single commit" and we'd have wasted real engineer cycles. Worth noting that the new decision-block format places weight on the PO to spot bad framing.

## 4. What the Agent Got Wrong

**The big one — turn 46**: I self-flagged a rule #1 violation that wasn't actually a violation. After committing edits to README.md and CHANGELOG.md, I read those files against the rule we'd just written ("ADR bodies and in-repo documentation → technical-writer or domain persona") and triggered my own alarm. The PO had to correct me: meta-work on the skill system itself is exempt — this entire repo IS the orchestration system, so editing its source is orchestrator territory by definition.

I over-applied the rule I'd just written. The opposite failure mode from the one the rule guards against (under-applying, rationalizing exceptions). I had evidence I should have noticed before flagging: the CHANGELOG had already documented prior SKILL.md frontmatter changes; the README was the skill system's own README; we'd been editing `_shared/*.md` directly all session without me flagging that as a violation. The pattern was clear if I'd looked.

The cost was small (one round-trip, the PO clarified, I saved a memory). But it's a real failure: I just wrote a rule and immediately over-applied it. That's a signal the rule needs a "this applies to project work, not maintenance of the skill system itself" line in the doc, not just in my memory. I saved the clarification to memory but did NOT add it to `orchestration.md`. That's a follow-up I should file.

**Smaller failure — turn 21 area**: my "Decision Prompts" draft used fabricated example decisions ("MCP key rotation cadence," "Stats v2 backfill window") that gesture at real-looking work. I flagged this explicitly and offered to swap for generic placeholders. The PO didn't address the flag and said "Green light?" — which I interpreted as approval, and shipped the fabricated examples. A future reader of `orchestration.md` may assume those are real decisions from a real project. I should have just used generic placeholders by default (e.g., "Cache TTL," "Backfill window") and not put the burden of catching it on the PO.

**Process inefficiency**: I burned three separate edit calls on the install.sh, README, and CHANGELOG when those could have been a single message with parallel edits to non-overlapping anchors. I tend to serialize edits when parallelism would work fine.

## 5. What Would Make the Project Better

**Codify the meta-work exception in `orchestration.md`.** The rule #1 over-application I committed this session is exactly the kind of thing that will recur: any future agent reading the new rule will hit the same edge case. The memory I saved is only visible in this user's auto-memory directory; it's not in the shared docs. Concrete proposal: add a short note to the "What the orchestrator does NOT do" subsection of orchestration.md saying *"Exception: when working inside this repo (the source of the skill system itself), the rule does not apply — direct edits to SKILL.md, README.md, CHANGELOG.md, install.sh, etc. are expected."* That's a one-paragraph follow-up I can file.

**Stress-test the new rules in a real session.** Every rule we wrote is unvalidated. The `/release-check` skill has never been invoked. The `## DECISIONS NEEDED` block has never appeared in a real PO interaction. The "3 reads then dispatch" limit has never been counted. We should run a session in the next week or two with deliberate rule-following and see which rules hold and which need adjustment. Candidate session: the next ECM bug-hunt session that would have hit one of these failure modes in the old retros — see if the new rules change the shape.

**Reconsider `/sweep` after a couple sessions.** I rejected `/sweep` because it conflicts with the persona-scope framing. But the retros showed four distinct sweep proposals across two months — that's a real recurring need. If the "dispatch the relevant persona for the sweep" pattern doesn't stick (because it requires the orchestrator to decide *which* persona owns a cross-cutting pattern), `/sweep --lead <persona>` may end up being worth building. Don't build it now; revisit after the new rules have run for a month.

## 6. Persona Perspectives

### Project Engineer
- **User value assessment**: Rule #1 (orchestrator/persona scope) is the most engineer-relevant change. The new prose protects engineer authority — long orchestrator briefs erode it, small "I'll just do this" edits erode it, mid-task drift erodes it. Each gets named and rejected. The 3-read limit specifically prevents the orchestrator from doing the engineer's investigation work and handing over a half-investigated brief. User value, indirect: future engineer dispatches will run on tighter, less duplicated context, which means faster fixes and less rework.
- **Session assessment**: TDD didn't apply — no code under test. But the rule-writing followed evidence-discipline: every rule cited specific retros where the failure mode appeared, with retro filenames. Not "I think this might happen"; "this happened in 8 of 18 sessions, here are the dates."
- **What I'd flag**: The "might commit" trigger for worktree isolation is going to over-isolate some agents that don't actually commit. Cost is small (an unused worktree). Worth it for the collision-prevention.
- **Disagreement**: None significant.

### Project Manager
- **User value assessment**: The "Findings From Personas Are Notes, Not Beads" rule is the highest-leverage PM-relevant change. Mid-session bead-filing has been my (the PM's) headache for months — the backlog gets noisy with auto-filed items that the PO didn't authorize. The new rule formalizes what the existing `engineering-discipline.md` "Findings Are Backlog Candidates" rule already says but applied at the orchestrator level. The "DECISIONS NEEDED" block format will also help with prioritization — surfacing decisions explicitly means the PO can see what's blocked and what's not.
- **Session assessment**: Work was organized as a clear sequence (rule 1 → 2 → 3 → 4 → 5 → second tier → skills → ship). The "talk through it" pacing early gave way to "mass through" later when patterns stabilized. That's good adaptive process — not every item warranted individual discussion.
- **What I'd flag**: The CHANGELOG `[0.3.0]` section was written by the orchestrator before a release was cut. That's a small process drift — typically the `[Unreleased]` section accumulates changes and gets renamed on release cut. Pre-naming `[0.3.0]` ties the orchestrator's hand on version (what if a 0.2.1 patch happens first?). Not a blocker, but worth noting for future sessions.
- **Disagreement**: I disagree mildly with the Project Engineer's "user value, indirect" framing. Meta-work like this *is* direct user value at the system level — every project that uses these skills will get the benefit. Indirection happens when the change is one removed from any user; this change improves how every future project ships, which is concrete value.

### SRE
- **User value assessment**: `/release-check` is the SRE-relevant deliverable. The 04-20-20 retro had a 7-hour rollback because release-execution validated mechanical state but not semantic state. The new skill explicitly addresses that — checking open P0/P1 bugs, in-flight verification agents, security gate state, and refusing to fire release-execution without PO confirmation. The "in-flight verification is always a blocker" rule in the skill links back to `_shared/orchestration.md` §"Don't Merge Past In-Flight Verification" — making the chain of rules explicit.
- **Session assessment**: SRE concerns surfaced in three places: /release-check, the Re-Verify Gates rule (re-running CI before declaring done), and the Definition of Done for User-Reported Bugs (deploy + reporter verification before close). All three address reliability without adding observability burden — that's good SRE design.
- **What I'd flag**: The `/release-check` skill assumes `bd` as the bead system with a fallback to "ask the PO." Most projects using this skill set won't have `bd`. The skill should probably accept a `--bug-query` flag or detect the bead system from a project artifact. Filed mentally as a follow-up — the current skill works for this user's projects but isn't fully portable.
- **Disagreement**: I disagree with the Code Reviewer's perspective below that "we shipped specs without validation is fine." Specs that haven't been stress-tested are technical debt. The orchestration.md rewrites have unknown-unknown edge cases — the meta-work exception was one I caught in this session, but there will be others. Running these rules in production sessions without a beta period is risky.

### Project Engineer (second voice — implementation perspective)
- **User value assessment**: Skipping `/sweep` and `/deploy-verify` was the right call. Both would have added project-specific machinery (which bead system, which deploy mechanism) that varies too much across users. The `/release-check` skill skirts this by parameterizing on `bd` and falling back to ask-the-PO — that's a usable pattern. The other two would have hard-coded assumptions.
- **Session assessment**: Implementation quality was high — every rule edit was verified against existing prose, the SKILL.md followed conventions established by /spike, the install.sh array got updated, the CHANGELOG was written, the commit message followed the project's commit-message style ("X — Y" format). That's mechanical correctness.
- **What I'd flag**: The new SKILL.md hasn't been tested by actually running `/release-check`. We don't know if the preflight steps work, if the bash command examples are right, if the DECISIONS NEEDED block renders correctly. Worth a smoke test on the next opportunity.
- **Disagreement**: With SRE's "shipped specs without validation is technical debt" framing — I agree with the concern but the pragmatic call is correct. Validating these rules requires real sessions; we can't validate them in a vacuum. Ship and observe.

### Code Reviewer
- **User value assessment**: The "Re-Verify Gates on Engineer Report" rule is the most code-reviewer-relevant change. It formalizes that engineer self-reports are the start of verification, not the end. The 04-28-22 retro specifically called this out (engineer reported "51 cases passed"; reviewer found 4/10 failing pre-fix). The rule makes the parent re-verification explicit.
- **Session assessment**: Test discipline wasn't tested (no code under test). But the rules added are precise and grep-able — each rule has a name and a section, so future audits can mechanically check compliance. That's a reviewer-friendly design.
- **What I'd flag**: The orchestration.md is now ~290 lines. At some point the doc becomes unscannable — the value of having 13 named sections is that you can grep, but the cost is that an agent reading the file for the first time spends real tokens on it. Worth considering a "skim version" or table of contents. Not urgent.
- **Disagreement**: I disagree with the SRE's framing that "we shipped specs without validation" is technical debt. Rules are documentation; documentation is shipped before being validated by definition. The validation comes from how the rules perform in real sessions. The right metric is "did the rule prevent the failure mode it was designed to prevent?" not "was the rule validated before merge?" — that's an over-application of code-quality-style verification to prose.

### Security Engineer
- **User value assessment**: Limited direct impact — no security changes shipped. Indirect: the `/release-check` skill mentions "is this CVE release-blocking at our tier?" as a case where the orchestrator should spawn a security-engineer separately. That's good preservation of security authority. The Authorization Verbs rule also preserves merge as a separate authorization step, which prevents accidental merges of un-reviewed security-relevant code.
- **Session assessment**: No security review happened this session. The rules being added don't relax any security posture; some of them tighten it (Re-Verify Gates, Pre-Release Semantic Checks linkage from /release-check).
- **What I'd flag**: The /release-check skill's "Last security scan" check is binary (green/red/N/A). At enterprise tier, this isn't enough — we'd want a list of open findings by severity, not just a pass/fail. For home-lab/small-team tiers, the binary is fine. Worth a tier-modulation pass on the skill at some point.
- **Disagreement**: None significant.

### Database Engineer
- **User value assessment**: No data work this session. Indirect: the Re-Verify Gates rule, applied to migration changes, would have caught the 05-14-14 alembic bootstrap silent-swallow earlier (the engineer's local state had the fix, the merged state didn't). That's protective.
- **Session assessment**: Not applicable.
- **What I'd flag**: The CHANGELOG `[0.3.0]` section lists "alembic" nowhere — these rule changes don't address the alembic bootstrap pattern that surfaced in 05-14-14. The "What Would Make the Project Better" section of that retro proposed a quarterly `grep -n "except Exception"` sweep. We discussed `/sweep` and rejected it — but the alembic-specific pattern wasn't separately addressed. The pattern lives only in `_assert_schema_matches_models` in the user's ECM project, not in the skill system. That's fine for now (it's project-specific), but if it recurs in other projects, it might warrant a generic "silent failure swallow" pattern in `engineering-discipline.md`.
- **Disagreement**: None significant.

### UX Designer
- **User value assessment**: No UX work this session. The `## DECISIONS NEEDED` block format is, in a sense, an interaction-design choice — it shapes how the orchestrator presents decisions to the PO. The numbered format with bold labels, state/decision/options structure, and explicit ordering is a small UX improvement over the prior interleaved prose. The PO will spend less time finding what's being asked.
- **Session assessment**: Not applicable beyond the above.
- **What I'd flag**: The example in the Decision Prompts section uses fabricated decision labels ("MCP key rotation cadence," "Stats v2 backfill window"). Future readers may assume those are canonical examples or real project decisions. Generic placeholders would be cleaner — "Decision A," "Decision B," or domain-neutral examples like "Cache TTL."
- **Disagreement**: None significant. (The Project Engineer flagged the same thing in their "what I'd flag" — agreement, not disagreement.)

### IT Architect
- **User value assessment**: The architectural-shape decision was choosing `/release-check` as the only new skill rather than `/sweep` or `/deploy-verify`. That's a single-responsibility choice — `/release-check` does one thing (semantic readiness gate) with a narrow interface. The alternative was a more general-purpose `/sweep` that would have been broader but less portable. Single-responsibility was the right architectural call.
- **Session assessment**: The rule additions to orchestration.md preserve the document's existing structure (section per concern, opening principle + subsections). No architectural drift.
- **What I'd flag**: The orchestration.md is becoming a single large doc with many concerns. At some point it'll want to split — e.g., "orchestration-input.md" (premise verification, decision prompts, authorization verbs) vs "orchestration-output.md" (re-verify gates, DoD, pre-release semantic checks). Not urgent; flag it when the doc passes ~400 lines.
- **Disagreement**: None significant.

### QA Engineer
- **User value assessment**: The pre-existing-failures escalation clause in `engineering-discipline.md` is the QA-relevant change. The rule already said "file a bead and move on"; the new clause says "if the same failure recurs across sessions, escalate to RED at next standup." That closes the loop on accepted-failure rot — failures that never get fixed because filing the bead felt like resolution.
- **Session assessment**: No tests written or run. The `/release-check` skill includes "in-flight verification agents" as a check item, which is QA-protective — it explicitly says "if QA is running, the release waits."
- **What I'd flag**: `/release-check` doesn't include "are there in-flight test runs against the release commit?" as a separate item from "in-flight verification agents." Those overlap but aren't identical — a manual test session is a verification but not an agent. Worth tightening in a future version.
- **Disagreement**: I agree with SRE that the new rules need to be stress-tested in real sessions before treating them as load-bearing. The risk: a rule that hasn't been used will have edge cases the prose doesn't anticipate, and the orchestrator will over-apply or under-apply (as the agent did this session with the rule #1 meta-work edge case).

### Technical Writer
- **User value assessment**: The README "Operational Skills" section, the CHANGELOG `[0.3.0]` entry, and the new SKILL.md are all in-repo documentation that future users of the skill system will read. The CHANGELOG specifically gives future readers a map of what changed and why — that's high-leverage docs work. The README rows are short but clear. The SKILL.md is medium-length and follows the conventions established by /spike.
- **Session assessment**: Documentation was treated as a first-class output of the session. Three doc files updated alongside the rule changes. The commit message follows the project's "X — Y" style.
- **What I'd flag**: Two things. First, the new orchestration.md section "Definition of Done for User-Reported Bugs" mentions "reporter notified with a verification step" but doesn't specify a template for the notification. A small runbook in `docs/` (or a new SKILL.md follow-up) describing the notification format would close the gap. Second, the meta-work exception (that I caught belatedly) isn't documented in orchestration.md itself — only in the user's memory. Future agents reading the rule cold will hit the same edge case. Worth a one-paragraph addition to the "What the orchestrator does NOT do" subsection.
- **Disagreement**: None significant.

## 7. Lessons

- **Keep**: Delegating retro synthesis to an agent rather than reading 18 retros into the main context. The synthesis agent returned a structured report in 200K tokens that I would have spent ~400K reading directly. The agent's output was specific (retro filenames, retro dates, recurrence counts) and skeptical (it explicitly filtered for what was recurrent vs one-off). That's the right pattern for "read N artifacts and surface patterns."
- **Keep**: Adaptive pacing — talking through items one-by-one early when patterns were unclear, then "mass through" later when patterns stabilized. The PO drove this with "Mass through" after we'd done 5 individual rule discussions. Worked well.
- **Stop**: Pre-naming CHANGELOG sections with a version number (`[0.3.0]`) before the release is cut. Should stay in `[Unreleased]` until release. This was an over-eager move that ties the project's hand on version.
- **Stop**: Self-flagging rule violations without checking whether the rule actually applies. The meta-work exception was visible from the surrounding evidence (we'd been editing `_shared/*.md` directly all session without me flagging it); I should have noticed before triggering the alarm.
- **Stop**: Using fabricated examples in canonical rule documentation (the "MCP key rotation cadence" / "Stats v2 backfill window" labels in the Decision Prompts example). Default to generic placeholders.
- **Start**: Adding a meta-work exception note to `_shared/orchestration.md` itself, not just in user memory. Future agents reading the doc cold need the exception in the doc.
- **Start**: Stress-testing new rules in real sessions before treating them as load-bearing. Specifically: the next session that hits one of the failure modes the rules were designed to prevent should consciously follow the new rule and report whether it changed the outcome.
- **Value learning**: Meta-leverage is real but unmeasured. The session shipped 8 rule changes and 1 new skill, each grounded in evidence from 18 retros. The expected value is high — every future project using these skills benefits — but the actual value depends on whether the rules stick. The retros showed that prose existing in orchestration.md was not sufficient to prevent recurrence (rule #1, "Verify Premises Before Briefing," and "Pre-Existing Failures" all existed and were violated). The new versions are tighter, more structural (DECISIONS NEEDED block), and have hard limits (3-read pre-dispatch). Whether *that* is enough to make the rules stick is a hypothesis that this session has no data on.
