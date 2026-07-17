# 2026-07-16-20 — hooks-to-retros

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~21 (7 genuine PO messages, 1 background-task notification, ~13 assistant turns)
- **SessionDepth**: light — backlog survey, bug-batch closeout, one hook fix built and reverted, one process rule established
- **Personas Active**: orchestrator only (no dispatches); relevant: project-engineer, code-reviewer, qa-engineer, project-manager
- **Beads Touched**: closed bead-xzdx9, bead-bn0wa, bead-0vao3 (shipped bug fixes), closed bead-uq7u0 + bead-ovqng (migrated to retro corpus per PO decision); read: all open beads

## Section 1: User Value Delivered

Concrete but mostly closeout-shaped. The session cleared project-d's bug column: three field-reported bugs (a P1 silent backend crash under concurrent cookie-auth requests, a sort-vs-numbering discoverability gap, and whitespace-variant duplicate channels) were already fixed, merged, and answered on their upstream issue threads by prior sessions — this session verified that state independently, confirmed all three issue replies were posted and awaiting reporter confirmation, and closed the beads.

One piece of real user-facing value was caught and fixed in passing: the development container was running a frontend build ~2 hours stale, missing a merged PR's Event Sync fixes (venue-conflict rail, journal visibility). Diffing the container assets against a fresh build from the dev branch caught it; a rebuild-and-redeploy brought the container current, verified by asset diff.

The session also produced work that was then deliberately discarded (see Section 4): a complete, tested fix for the org commit-guard hook's bead-ID regex — built before the PO clarified scope, reverted cleanly after. The fix design is preserved in Section 5 of this retro so the work isn't lost, just re-routed.

## Section 2: What We Did Well Together

The PO's mid-flight interrupt. While I was mid-implementation on the commit-guard hook fix, the PO stopped the session with "Wait... what're you doing? Why are you changing the hooks?" — at exactly the right moment: the fix was built and self-checked but nothing was committed or pushed. The recovery cost was one `git checkout` and one bead status reset. Every subsequent PO answer was terse and decisive ("I just want to fix the [product] bugs", "Close them"), and the session ended with the PO converting the incident into a durable process rule rather than just a one-off correction.

## Section 3: What the PO Could Improve

"Lets finish the bugs" was issued over a 4-bead set where one bead was not a product bug at all — it was a defect in shared org-level agent tooling, filed in the project tracker as type `bug`. The instruction carried an implicit scope (product bugs only) that only became visible after implementation had started. One scoping clause — "the product bugs" — in the original message would have saved a built-and-reverted fix. This is a minor cost and the deeper failure was the agent's (Section 4), but the pattern is worth naming: when a batch instruction is issued over a set the PO hasn't just reviewed item-by-item, the odd-one-out item is where scope assumptions diverge.

## Section 4: What the Agent Got Wrong

I treated tracker-type as authorization scope. bead-uq7u0 was filed as a `bug` in the project's tracker, so "finish the bugs" appeared to cover it — but its subject was the org-wide persona-system commit-guard hook, whose working copy is **live infrastructure**: the harness settings register the hook by its repo path, so editing the file changes enforcement behavior for every project on the machine, immediately. I claimed the bead, designed the fix, edited the live hook, and extended its self-check suite — all before surfacing "this bead is org tooling, not product code; include it?" as a decision. The right order was decision first, implementation second; a `DECISIONS NEEDED` line would have cost the PO five seconds and me nothing. I did flag it correctly *after* the interrupt (including the "this edit is already live behavior" disclosure and a clean-revert offer), but that disclosure should have led, not trailed. The fix itself was verified working (all self-check cases green) and then discarded — wasted work that a question would have avoided.

## Section 5: What Would Make the Project Better

**The rule the PO established this session, verbatim intent: hook fixes are functions of the claude code personas — they belong in the retro corpus, not in project backlogs.** Project trackers keep accumulating beads about the agent-team's own enforcement hooks (two open in project-d alone this week; a third hook friction was fixed via PR days ago). Those beads sit in a product backlog where no product persona will ever pick them up, while the actual change-control path for the persona system is retro → retro-mine → proposed rule change → PO approval. This retro implements the rule: both hook beads are closed in the tracker and their full content is preserved below for retro-mine.

**Payload for retro-mine — hook defects awaiting upstream action in claude-agent-dev-team:**

1. **Commit-guard bead-ID regex is inert for bd-style IDs (from bead-uq7u0, recurrence: also flagged in the 2026-07-16-19 retro).** `hooks/pretooluse.py` rule 3 requires `\b[A-Za-z][A-Za-z0-9_]*-\d+\b` — an all-numeric suffix. bd-generated IDs use short alphanumeric suffixes that are frequently all letters (`-wccvo`, `-dvgri`) or digit-leading (`-0vao3`), none of which match, so the guard false-blocks every legitimately bead-referenced commit in such repos; engineers worked around it by citing PR numbers (`PR-660`), which satisfies the regex by accident. **A verified fix design exists (built, self-checked green, then reverted pending this routing):** keep the numeric pattern for `proj-42`-style repos, and additionally accept the repo's *own* bead prefix — read `issue-prefix` from `.beads/config.yaml` if set, else the git-root directory name — followed by any alphanumeric suffix with optional dotted children (`(?:\.\d+)*`). Scoping suffix-agnostic matching to the repo's own prefix preserves precision: ordinary hyphenated words ("re-run", "unit-tests") still fail the gate. Six self-check cases covered: directory-name prefix accept, dotted-child accept, hyphenated-English deny, wrong-prefix deny, config-override accept, and overridden-directory-name deny.
2. **Version-advance guard shell hook has no self-test and over-triggers (from bead-ovqng).** The project-local `.claude/hooks/version-advance-guard.sh`: (a) no fixture-based self-test exists for the shell wrapper — only its pure-Python comparator is unit-tested; the "Enforcement Code Tests Itself" discipline (which `pretooluse.py --check` follows) should extend to shell hooks, plausibly as a shared harness pattern in the skill repo; (b) its trigger match is substring/regex over raw command text, so a command merely *containing* the trigger phrase inside a quoted string spuriously fires the guard — hit in the field during a verification flow, forcing a restructure into a script file. Related, already fixed downstream but relevant as a pattern: the same hook was worktree-blind (resolved PROJECT_DIR to the main checkout when the ship ran from a worktree) — shipped as a project PR, but the pattern (hooks resolving paths against the main checkout while agents work in worktrees) is a class, not an instance.
3. **Recurrence note for clustering:** the 2026-07-16-19 retro independently flagged both items above plus a persona-bead-firewall inconsistency (same `bd update --notes` call allowed early in a session, denied later). Three hook frictions across two retros in one day is a cluster: the proposal shape is probably "hooks get a self-test contract + a field-friction review pass," not three point fixes.

## Persona Perspectives

### Project Engineer
- **User value assessment**: The stale-container catch was the only net-new user value; the bug closeouts were bookkeeping on already-shipped value. Both fine — closeout is real work.
- **Session assessment**: The hook fix itself was good engineering (precise, self-tested, reverted cleanly with zero collateral) executed under the wrong authorization. The interrupt-revert cycle worked exactly as it should.
- **What I'd flag**: Verified-working code was discarded rather than parked. A branch in the tooling repo would have preserved the artifact at zero risk; instead the design survives only as prose in this retro. If retro-mine's proposal is accepted, someone rebuilds from the description.
- **Disagreement**: With the framing that the revert was cost-free. The self-check extension (six cases) was genuinely reusable regardless of the regex decision, and it went down with the revert.

### Code Reviewer
- **User value assessment**: An inert enforcement hook is worse than no hook — it trains agents to route around guards (the PR-number workaround is exactly that). Fixing it protects commit-message discipline, which is user-adjacent via traceability.
- **Session assessment**: The fix design was right: prefix-scoped leniency instead of a globally looser regex. The rejected alternative (accept any hyphenated token) would have made the guard decorative.
- **What I'd flag**: The retro is now the only carrier of that design. Retro-mine needs to treat Section 5 payloads as engineering specs, not anecdotes — if the corpus scrub or summarization lossily compresses them, the pipeline drops exactly the content this rule routes into it.
- **Disagreement**: None with the routing rule itself; it puts hook changes under the change-control process that owns them.

### QA Engineer
- **User value assessment**: Closing three field-reported bugs only after independently re-verifying merge state, issue replies, and container currency is the right bar — closeout without verification is how "fixed" bugs come back.
- **Session assessment**: Good: the container-vs-dev asset diff (and the follow-up that the md5 mismatch was sort-locale noise, proven by a proper diff) is exactly the control-the-environment discipline. The `--check` self-test harness in the guard hook made the built-then-reverted fix verifiable in seconds.
- **What I'd flag**: bead-ovqng's core finding — shell hooks with no fixture tests — is now tracked only in the retro corpus. Until retro-mine acts, that enforcement code remains untested in the field.
- **Disagreement**: With the PM's clean-backlog framing below: removing untested-enforcement-code beads from the visible backlog is only hygiene if the receiving pipeline has a known cadence. Otherwise it's moving a risk somewhere quieter.

### Project Manager
- **User value assessment**: The bug column going to zero (with the sole survivor re-routed to its correct owner) is real backlog signal for the PO; misfiled infrastructure beads were noise in every "what's open" query.
- **Session assessment**: The PO's rule resolves a genuine ownership ambiguity: project trackers were accumulating beads no project persona would ever service. Routing them to the pipeline that can actually change the persona system is correct.
- **What I'd flag**: Two beads closed into a pipeline with no SLA. Recommend the PO treat "retro-mine pass pending with hook cluster in corpus" as a to-do with a date, not an eventually.
- **Disagreement**: None.

## Lessons

- **Keep**: Interrupt early, revert cleanly. The PO stopping a wrong-scope implementation mid-flight cost one command to unwind because nothing had been committed or pushed; container-first/commit-late made the mistake cheap.
- **Stop**: Treating tracker-type as authorization scope. A bead's `type: bug` says what it is, not whose it is — subject classification (product code vs. org tooling vs. process) must happen before claiming, and cross-boundary items go in a `DECISIONS NEEDED` block before any edit.
- **Start**: At triage/filing time, beads about the persona system's own hooks and skills should not enter project backlogs at all — capture them as retro payload directly (this session's rule). When they're found already filed, migrate them the way this retro does: full technical payload preserved, bead closed with a pointer.
- **Value learning**: The PO's backlog attention is a resource the same way review attention is — misfiled infrastructure beads don't just sit harmlessly, they dilute every backlog query the PO makes. "Where should this work live" is itself a decision worth surfacing.
