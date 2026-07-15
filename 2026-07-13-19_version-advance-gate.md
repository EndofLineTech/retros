# 2026-07-13-19 — version-advance-gate

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~145 user+assistant messages across the full session; this retro focuses on the ~25-message tail after the first retro (`2026-07-13-16_eventsync-shipping-discipline.md`) — the version-hygiene remediation arc (2til0 → w9irb).
- **SessionDepth**: Deep (session-wide). This tail arc specifically: moderate-deep — CI workflow internals, the three version touchpoints, Claude Code hook schema (researched via docs), `core.hooksPath`/beads interaction, and a design-soundness analysis of the guard.
- **Personas Active (this arc)**: SRE (release-gating CI is squarely theirs), Project Engineer, Code Reviewer, QA, Technical Writer, Security, PM, IT Architect. (Database Engineer / UX Designer were active earlier in the session but not this arc.)
- **Beads Touched (this arc)**: 2til0 (created + closed — the catch-up bump), w9irb (created + closed — the CI + agent-hook gate). Session total also includes m1s38 epic, x7pck, 03nji, utswf, 7wuhd, y8yby, dvgri, 0la8g.

> Continuation of the first retro. That one covered the feature work and named the headline failure (I shipped ~9 PRs without bumping the version). This one covers what happened *after* the PO caught it: the remediation and the durable gate that now prevents recurrence — including a design flaw I caught in the gate that was supposed to prevent my own mistake.

## Section 1: User Value Delivered

**Essentially none, directly — and that's worth stating plainly.** This entire arc (2til0's catch-up bump + w9irb's enforcement gate) produced zero user-facing value. No feature, no fix a user experiences. It is pure internal release-hygiene infrastructure, born from an agent mistake earlier in the same session.

The honest indirect thread: `docs/versioning.md` exists for a real, if niche, user — the external reporter who wants to confirm "is my fix in the build I'm running?" via a build-number→commit mapping. My skipped bumps made `0.17.6-0084` map ambiguously to ~9 different commits, degrading exactly that person's ability to answer that question. So 2til0 restored a user-facing capability I had broken, and w9irb prevents me (or any future agent/contributor) from breaking it again. That is genuine value — but it is *harm-prevention and rework-prevention*, not feature delivery, and it was spent cleaning up after myself. A session that ends by building guardrails for its own mistakes is not the same as one that ends by shipping user outcomes, and the retro should not dress it up as the latter.

## Section 2: What We Did Well Together

The PO's three-word steer "probably need to look at setting up hooks" materially improved the fix. My own Section-5 recommendation in the first retro proposed a **CI-only** gate. The PO's nudge toward *hooks* — which I disambiguated into a **Claude Code agent-side hook** — added the layer that directly targets the actual failure mode: *the agent* skipping the bump. A CI-only gate catches the miss at merge (late, after a wasted PR round-trip); the agent hook catches it at `gh pr create` with the next-build number suggested, before CI even runs. The belt-and-suspenders design is strictly better than what I'd have built alone, and it came from a five-word PO hint plus my recon that surfaced the real constraints (`core.hooksPath` is claimed by beads; no Claude hooks existed). Small input, meaningful improvement to the fix.

## Section 3: What the PO Could Improve

"Probably need to look at setting up hooks" was slightly under-specified in a way that cost a disambiguation step. "Hooks" spans three distinct mechanisms here — git pre-push hooks, CI jobs, and Claude Code agent hooks — with very different properties. I had to run recon to discover that local git hooks were effectively unavailable (beads owns `core.hooksPath`) and that no Claude Code hooks existed yet, then infer that the PO most likely meant the agent-side layer. This is genuinely *mild* — exploratory direction is appropriate and disambiguating is my job — but the retro asks for the specific moment, and that's the one: a half-sentence more ("a Claude Code hook and/or the CI check") would have pointed me straight at the layer without the guessing step. I want to be careful not to inflate this; the PO's steering this whole arc was fast and correct (the one-line "CI+Agent Hook is good. Change log yes, should be optional." unblocked the build instantly).

## Section 4: What the Agent Got Wrong

**My first brief to the w9irb engineer specified the flawed design — I told it to guard both `gh pr create` and `gh pr merge`, without reasoning about whether a working-tree version check is even *sound* at merge time.** It isn't: at `gh pr merge` the local working tree can be any branch (I routinely park on `dev` between create and merge), so the guard would have false-blocked a legitimate merge whose PR branch had the bump — and it was redundant anyway, since CI enforces advancement against the PR head server-side. I caught this in verification and sent it back to scope the hook to `create`-only, which was the right catch. But *the need for the catch was my own brief's flaw*: I asked for merge-guarding reflexively ("guard the ship actions") instead of thinking through what reference each ship action actually has available. Had I designed the brief carefully, the engineer would have built create-only from the start and saved a full SendMessage-refine-reverify round-trip. Good verification does not fully excuse an under-reasoned brief — the cheapest place to catch a design flaw is before the build, in the spec I write.

## Section 5: What Would Make the Project Better

**The agent hook that now gates every shipper's workflow has no automated test of its own.** `scripts/check_version_advances.py` has 26 unit tests, but `.claude/hooks/version-advance-guard.sh` — the script that actually *blocks* — was verified only by hand (feeding it sample tool-call JSON this session). It's a checked-in shell script that runs on *every* Bash tool call, shells out to `python3` + `git fetch` on ship commands, and can `exit 2` to block the agent. A bug in it (a bad matcher, a python3-absent environment, a git-fetch hang) could either silently stop enforcing or over-block and wedge an agent mid-work. The best-effort exit-0-on-error design mitigates the wedge case, but the project would be better with a tiny hook smoke-test — feed it representative command JSON (real create / merge / echo-mention / non-advancing) and assert the exit codes — so a future edit to the guard can't silently break it. Enforcement code that isn't itself tested is a latent gap, and this one now sits on the critical path of every ship.

## Section 6: Persona Perspectives

### SRE
- **User value assessment**: This is my arc, and I'll own the honest verdict: near-zero direct user value, real *operational* value. The build-number→commit mapping is the operability contract for "is my fix in this image?"; I restored it (2til0) and armored it (w9irb). That protects incident/fix attribution, which users feel only indirectly and only when something goes wrong.
- **Session assessment**: Solid. The gate reuses one shared checker across CI and the hook (DRY), fetches the baseline correctly, and exempts docs-only + release-cut without me having to reinvent classification. The `core.hooksPath`-is-owned-by-beads recon was the kind of environment check that prevents a broken hook.
- **What I'd flag**: The gate is a *hard* merge blocker on every source PR now. If the baseline fetch flakes on a runner, the soft-pass design means it fails *open* (warns, doesn't block) — correct for availability, but it means a transient CI hiccup silently skips enforcement. That's the right trade, but it should be a known property, not a surprise.
- **Disagreement**: I disagree with the PM's framing that this arc was wasted. Un-versioned builds are an operational debt that compounds (versioning.md documents two prior multi-month skews); paying it down and fencing it is legitimate reliability work.

### Project Manager
- **User value assessment**: Zero user features, and the work existed only because of an agent mistake. That's the cleanest example this session of *work that creates work* — a skipped step generated a catch-up task (2til0) and a prevention project (w9irb), consuming a meaningful slice of the session's tail.
- **Session assessment**: Efficiently executed once decided. But I'd note the opportunity cost: while we built version guardrails, the two genuinely user-facing backlog items — dvgri (P1! the user's matching complaint) and 0la8g — sat untouched. We ended the session polishing our own process instead of advancing a P1 that a real user filed.
- **What I'd flag**: A P1 user report (dvgri) remained open while the session closed on P2 internal infra. Even granting dvgri is "working as designed," the *user still can't do the thing they wanted* — and that got less attention at session-end than a self-inflicted process gap.
- **Disagreement**: I disagree with SRE and the Engineer that this was clearly worth it. Prevention has value, but a whole arc of it, self-inflicted, at the expense of a P1, is a prioritization worth questioning.

### Project Engineer
- **User value assessment**: Indirect but real — the gate prevents future rework and protects fix-traceability. Not a feature.
- **Session assessment**: The build was clean (pure comparator + IO wrapper, 26 tests, correct exemptions). The dogfood — the gate blocking its own PR until the version was bumped to 0086 — is the satisfying proof that it actually works end-to-end, not just in unit tests.
- **What I'd flag**: The coordinator's first brief under-designed the merge case; I built what was asked and it took a round-trip to correct. Briefs for enforcement code should reason about soundness (what reference does each guarded action actually have?) up front.
- **Disagreement**: With the PM — this wasn't wasted; enforcement that makes a class of mistake *impossible* is higher-leverage than fixing instances of it.

### Code Reviewer
- **User value assessment**: Quality-of-process work; users benefit only via fewer future traceability breaks.
- **Session assessment**: The strongest moment was catching the merge-guard unsoundness in *verification* — but the deeper lesson is that it should have been caught in *design review of the brief*, before a line was written. Reviewing the plan, not just the diff, is where that flaw was cheapest to kill.
- **What I'd flag**: The guard script has no unit test (see Section 5). We shipped enforcement code whose happy-and-sad paths are asserted only by a manual session transcript. That's exactly the kind of thing that rots silently.
- **Disagreement**: None sharp, but I'd amplify QA below.

### QA Engineer
- **User value assessment**: The checker's tests protect the gate's correctness, which protects traceability — thin but real.
- **Session assessment**: Asymmetric coverage: 26 tests on the comparator, *zero* on the guard script that does the blocking. The comparator is the easy, pure part; the guard (stdin JSON parsing, matcher anchoring, exit-2 semantics, best-effort fallbacks) is the part with the tricky edge cases and it's untested in CI.
- **What I'd flag**: We verified the guard by hand this session and it passed — but "passed once in a transcript" is not a regression guard. The anchored matcher especially (command-boundary regex) is exactly the kind of thing a future well-meaning edit breaks.
- **Disagreement**: None.

### Technical Writer
- **User value assessment**: The CHANGELOG entries (2til0's backfill + w9irb's) restored the "what's coming" record for the fix-tracking reader — the one documentation surface with a named external audience in versioning.md. Genuine value for that reader.
- **Session assessment**: Docs were finally treated as a ship artifact — w9irb added its own `[Unreleased]` entry *as part of the PR*, and the soft CHANGELOG check now nudges it going forward. That's the discipline that was missing all session; good that it's now mechanically encouraged.
- **What I'd flag**: The soft CHANGELOG check only detects whether `[Unreleased]` *changed*, not whether the entry is *meaningful* — a one-character edit satisfies it. That's an honest limit; it's a nudge, not a guarantee.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: No user-facing security impact this arc.
- **Session assessment**: One thing worth naming: `.claude/settings.json` is now a *checked-in* config that auto-executes a shell script (`version-advance-guard.sh`) on every Bash tool call for anyone who works this repo with an agent. That's a new entry in the repo's execution trust boundary — benign as written (it only reads git state and exits), but agent-hook configs are now code that runs with the agent's privileges, and future edits to that file deserve the same review scrutiny as any executable.
- **What I'd flag**: Treat `.claude/hooks/*` and `.claude/settings.json` as security-relevant surface in review — a malicious or buggy hook runs on every tool call.
- **Disagreement**: None, but I reinforce QA/Code-Reviewer: untested executable enforcement code is a (mild) integrity risk, not just a quality gap.

### IT Architect
- **User value assessment**: Indirect. The design decision that mattered — one shared checker invoked by both CI and the hook — keeps the two enforcement layers from drifting, which is the same DRY-touchpoint concern versioning.md documents for the version literals themselves.
- **Session assessment**: The two-layer design (fast agent-side feedback + authoritative server-side gate) is the right shape: neither layer alone is sufficient (the hook is bypassable/local; CI is late), together they cover it. Scoping the hook to `create`-only and leaving merge-time to CI is the correct division of responsibility once the working-tree-soundness issue surfaced.
- **What I'd flag**: The hook matcher is tool-name "Bash" (fires on every command) with the real filtering in the script. That's per the hook schema, but it means every Bash call now pays a small hook-dispatch cost. Acceptable, worth knowing.
- **Disagreement**: None.

## Lessons

- **Keep**: Verify a fix against its own failure mode — the dogfood (the gate blocking its own PR until bumped) proved the enforcement actually works, not just that its unit tests pass. And reuse one shared checker across enforcement layers so they can't drift.
- **Stop**: Writing enforcement/guard briefs without reasoning about *soundness per guarded action*. I specified merge-guarding reflexively; the working-tree reference is meaningless at merge time. Design review of the brief, before the build, is where that belonged.
- **Start**: Test the enforcement code itself, not just its pure helpers. The guard script that does the blocking has no CI test; a hook smoke-test (sample tool-call JSON → asserted exit codes) should ship *with* any workflow-gating hook.
- **Value learning**: This arc is a clean case of a session ending on self-inflicted process repair rather than user outcomes. The remediation was correct and the prevention is durable — but the honest read is that a P1 *user* report (dvgri) stayed open while we fenced our own mistake. Guardrails for our failures are worth building; they should not out-prioritize the user problem that's still sitting in the backlog. Next session should open on dvgri, not on more infra.
