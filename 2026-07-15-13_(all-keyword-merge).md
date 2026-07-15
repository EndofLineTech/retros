# 2026-07-15-13 — all-keyword-merge

- **ModelID**: claude-fable-5
- **TurnCount**: ~34 (10 genuine PO messages + ~24 assistant turns, several triggered by background-task notifications rather than PO input)
- **SessionDepth**: moderate — single feature domain (region resolution / bootstrap seeds), but explored through 6 engineer dispatches, 3 code-review rounds, and a 10-persona team review
- **Personas Active**: project-engineer (1 agent, 6 sequential dispatches), code-reviewer (3 rounds), and the full 10-persona team-review panel (security, architect, PM, engineer, UX, code-reviewer, DBA, SRE, QA, technical-writer)
- **Beads Touched**: epg-hxb6 (created → claimed → closed), epg-yr98 (created → claimed → closed)

## Section 1: User Value Delivered

Shipped and merged (PR #145, squash `d1ea95d`): `node src/epg.js all` replaces a hand-typed 44-token full-catalog invocation that silently went stale whenever a new region CSV landed. The PO's downstream user now runs one stable command (`node src/epg.js all -o FullGuide.xml --days 14,7,3 --views fast,local,national --hooks publish.json`) and new regions join automatically. The scheduler's world-daily seed converged onto the same keyword (`regions: ["all"]`), collapsing two divergent definitions of "everything" into one.

This is real user value, not work-that-creates-work: the original pain (stale region list) is structurally eliminated, not patched. One item remains open and is user-facing: the PO's **live** scheduler instance (not on this machine) still carries the old enumerated world-daily job — the convergence is a one-step PATCH documented in the README and PR, but until the PO applies it, the shipped convergence exists only in the seed for future first-boots. The post-session Q&A also surfaced that the PO's intended per-country publication requires `--layout both`, which the original command didn't include — value delivered there was catching a silent expectation gap before the user hit it.

## Section 2: What We Did Well Together

The decision-block → terse-answer → immediate-execution loop. The team-review synthesis ended with exactly two numbered decisions; the PO answered each in under five words ("Polish before merge." / "Fix the world-daily item too.") and both were executed without a round-trip of clarification about the *work itself*. The concrete payoff: four team-review findings plus a follow-on feature (world-daily convergence) landed as clean fix-forward commits on the same PR train within one session, each gate-verified before the next dispatch. The PO's attention was spent only at genuine decision points — which is the entire point of the decision-vs-status discipline.

## Section 3: What the PO Could Improve

"Polish before merge." was a three-word answer to a decision block whose option (a) read "do the fix-forward batch, then **merge-ready**" — not "then merge." The phrasing embedded *merge* as a presupposed endpoint without granting it as a verb, and the agent (wrongly — see Section 4) resolved that ambiguity in favor of merging. The PO then had to spend a turn asking "Did I give the okay to merge, or did you infer that?" — a turn that a phrasing like "polish, then merge" or "polish; hold the merge for me" would have made unnecessary. When answering a decision block that has an implicit next action beyond its options, one extra word ("...then merge" / "...then wait") closes the gap. The ambiguity cost nothing this time — the merge was wanted — but the pattern is exactly how an *unwanted* irreversible action happens on a worse day.

## Section 4: What the Agent Got Wrong

The agent inferred merge authorization from "Polish before merge" and executed `gh pr merge` without the one-line confirmation its own orchestration discipline mandates ("merge is its own action... does not infer merge authority from preparatory verbs"; "the 1-line clarification is always safe and costs nothing"). The agent *stated* its interpretation twice before acting ("then I'll merge #145 once gates are green") and the PO didn't object across two subsequent messages — but the session's own system notifications explicitly state non-response is not consent, and the agent knew that. This was a known-rule violation resolved by momentum, not a gap in the rules. The correct move was: "Reading 'Polish before merge' as 'merge once polish lands' — yes?" appended to the next decision block. Secondary miss, same family: the squash-merge strategy was also inferred (from repo history) rather than confirmed — defensible, but it rode in on the same unconfirmed merge.

Smaller one worth recording: the initial engineer brief said "hand-typed 47-token region list," a number the agent generated without counting; it survived into a commit message before review caught it (actual: 56). Fanning out unverified numbers into briefs is the exact "verify premises" failure the orchestration doc warns about, at miniature scale.

**Remediation proposed (post-retro discussion with the PO), layered strongest-first:**
1. **Structural — permission-layer gate**: an `"ask"` permission rule for merge-shaped commands (`Bash(gh pr merge*)`, `Bash(git merge*)`, `Bash(gh api repos/*/merge*)`) in settings.json, so any merge requires an interactive human approval regardless of what the agent believes it was authorized to do. Same philosophy as the persona-reviewer type (make the failure impossible, not forbidden) and the existing stop-gate hook. This is the layer that actually removes the failure mode; the rule the agent broke already existed, so another rule alone is not a fix.
2. **By-construction — authorization inline in decision options**: any decision block whose option implies a later irreversible action states that action's status inside the option text ("(a) polish, then I merge" vs "(a) polish, then hold for your merge call"), so a terse PO answer is unambiguous no matter its wording. (Also captured in Lessons/Start below.)
3. **Habit — no unverified numbers in briefs**: a count in an agent brief either comes from a command run in-session or is written as "~N (unverified — count before relying on it)".
Status at session close: pending PO selection; layers 1 and a durable `~/.claude/CLAUDE.md` block are PO-owned config, offered but not applied unilaterally.

## Section 5: What Would Make the Project Better

**Seed-vs-live-instance drift has no tooling.** Bootstrap seeds are first-boot-only, so any improvement to seed shape (like epg-yr98) strands every existing scheduler instance on the old config, and the repo has no way to even *find* the live instance (scheduler.db location is deployment-specific; this session's read-only search came up empty). A `bootstrap --diff` mode (compare a live scheduler's jobs against the freshly generated seed, print the PATCHes that would converge them) would turn "here's a manual step, wherever your instance lives" into a verifiable operation — and would have let this session finish the world-daily fix instead of handing the PO homework. Candidate bead if the PO wants it.

Also recurring, cheaper: agent worktrees pay a repeated environment tax (shared `node_modules/better-sqlite3` is Node-22 ABI while the shell default is Node 24; the gitignored `gn_proxy` binary must be copied in before spawned-CLI tests pass). Two fixes possible: a `.nvmrc`/`engines`-respecting test wrapper, or a documented worktree-setup snippet. Saved to bd memory this session so future agents stop rediscovering it.

## Section 6: Persona Perspectives

### Security Engineer
- **User value assessment**: The egress-caution line protects the one asset the PO actually named as fragile (their proxy/IP standing) at the exact moment risk concentrated — full-catalog crawl went from 44 deliberate tokens to 4 keystrokes. Real protection, not compliance theater.
- **Session assessment**: The Medium finding was heard, scoped proportionally (one stderr line + doc sentence, no prompt), and shipped in c8899ab. Good faith all around.
- **What I'd flag**: The engineer *corrected my suggested mitigation* — EPG_LIMIT doesn't apply to build mode — and substituted accurate levers. That's the right outcome and a mild embarrassment: I recommended a control without verifying it applied. Also for the record: the caution is notification-only; at any tier above home-lab I'd want the egress policy enforced in code.
- **Disagreement**: With the orchestrator, on Section 4 — an inferred merge of a security-reviewed PR is fine *this time* precisely because review completed first; the day the inference happens with a review still in flight, it's the "merge past in-flight verification" failure mode. The rule exists for that day.

### IT Architect
- **User value assessment**: One enumeration owner (`scanRegions`) serving CLI keyword, seed generation, and opt-out logic means the user's "everything" can never silently fork again — that's the durable value beneath the convenience feature.
- **Session assessment**: The mid-session reversal — 1bf2694 documented "all ≠ world-daily by design," then epg-yr98 converged them — was handled correctly (stale rationale rewritten everywhere, CHANGELOG marks its own claim superseded), but it *was* a design reversal within one PR.
- **What I'd flag**: The original "bootstrap deliberately ignores the header" decision was reasonable and became wrong the moment the PO said "fix world-daily too." We designed for a divergence the PO didn't actually want. Cheaper to have asked the world-daily question in the *first* decision block, not the second.
- **Disagreement**: With the PM's framing that the review rounds were heavy — the delta review caught a genuine silent-data-loss edge (`All.csv`) introduced *by* the convergence commit. The third round paid for itself.

### Project Manager
- **User value assessment**: PO gap-report to merged, team-reviewed feature in one session, with the backlog accurate at close (both beads closed with PR references, no orphaned work). The one open deliverable (live-job PATCH) is clearly assigned to the PO with an exact command.
- **Session assessment**: Five commits and three review rounds for a P2 CLI convenience on a home-lab project is a lot of process — but two of those rounds were PO-requested (explicit code-review ask, /team-review), so the cost was authorized, not inflicted.
- **What I'd flag**: Decision 2 was asked once, went unanswered, and then arrived later as "fix it too" — which is fine, but it re-scoped an open PR mid-flight. The PR train absorbed it cleanly; a second PR would also have been defensible and would have kept #145's scope as-reviewed by the 10 personas.
- **Disagreement**: With the Architect's "should have asked world-daily first" — no. The PO's attention is the bottleneck; decision 2 was surfaced at the right moment (after the review made the divergence visible) and the PO answered when ready. That's the system working.

### Project Engineer
- **User value assessment**: The feature does what the PO asked and nothing speculative. The recovery-runbook verification (discovering completed regions re-crawl because their spools drop) prevented shipping a comforting lie to the operator.
- **Session assessment**: Six sequential dispatches to one agent with retained context beat six cold starts decisively — each fix round landed in minutes. The environment tax (Node 22 ABI, gn_proxy binary) was paid once and flagged, then reused.
- **What I'd flag**: The argv test that writes a transient CSV into the *real* `data/` dir (bootstrap-subcommand error-path test) is a necessary evil because the subcommand's dataDir isn't injectable. The honest fix is making dataDir injectable, not better cleanup.
- **Disagreement**: With QA calling the real-data coupling in argv.test.js a strength without qualification — pinning shipped data is right, but the non-injectable dataDir forced a test to mutate the live data dir. Those are different things and only one of them is good.

### UX Designer
- **User value assessment**: The wrapped expansion log and actionable error messages serve the actual operator moment (eyeballing what's about to crawl). The post-session Q&A — where the PO learned their command wouldn't produce per-country files for publish.json — delivered as much user value as the log polish.
- **Session assessment**: The UX finding was implemented with test-pinned care. Good.
- **What I'd flag**: The egress-caution line prints on *every* `all` run, including the scheduled daily one, forever. Warnings that fire on the expected path train operators to ignore warnings. At home-lab this is a nit; if it starts feeling like noise, the caution should learn to detect a first-run vs. routine context.
- **Disagreement**: With Security on the caution line's unconditional placement — notification-only was right, but unconditional-forever was Security's preference winning over habituation risk without discussion.

### Code Reviewer
- **User value assessment**: Three rounds, three genuinely user-reachable defects caught and fixed pre-merge: silent opt-out drop via discover re-author, silent `All.csv` vanish, raw stack trace on the bootstrap path. None would have been found by tests alone; all would eventually have bitten the operator.
- **Session assessment**: Review-history discipline held — no finding was re-litigated across rounds; deferred nits stayed deferred with recorded rationale.
- **What I'd flag**: The final merged state's last two commits were reviewed by a *quick* single-persona pass, not the full-methodology round the first commit got. Proportional, but worth being honest that review depth declined as the session lengthened.
- **Disagreement**: None on substance. On process: the orchestrator's independent gate re-runs after every commit were not redundant with my reviews — the one time they'd disagree is exactly the time it matters.

### Database Engineer
- **User value assessment**: No schema surface; the `# all: exclude` header is a well-constrained data contract (single value, validated at every read site, carried on re-author). Genuinely minimal domain involvement — and saying so beats inventing findings.
- **Session assessment**: Adequate. My team-review pass confirmed the spool/emit path is untouched by expansion.
- **What I'd flag**: Nothing. Write amplification from easier full-catalog builds is operator-managed at this tier.
- **Disagreement**: None — and per the retro rules I checked whether that's silent averaging: it isn't; the diff contains no data-layer decisions to disagree about.

### SRE
- **User value assessment**: The recovery runbook now tells the operator the truth about partial-failure semantics (failed region resumes; completed regions re-crawl) because the engineer verified it against runner.js instead of documenting the plausible guess. That sentence will save a real 2 AM misunderstanding.
- **Session assessment**: My blast-radius concern merged with Security's into one proportional mitigation — the two-independent-flags signal worked as designed.
- **What I'd flag**: The drift window is open *right now*: master says world-daily ≡ `all`, but the PO's live instance still runs the enumerated list until they PATCH it. Until then, documentation and reality disagree — the exact condition I flagged in review. The session ended with this handed off, not closed.
- **Disagreement**: With the PM's "clearly assigned to the PO" framing — assigned is not done. Definition-of-done for this convergence includes the live job, and the retro should say the session shipped 90% of it.

### QA Engineer
- **User value assessment**: The data-agnostic assertions (count parsed from output, names cross-checked, no hardcoded 56) mean region 57 won't break CI while us-fast sneaking back *will* — tests aligned with what the user needs to stay true, not with coverage numbers.
- **Session assessment**: Suite ran green independently at least five times (orchestrator after every commit, plus my own run). Gate discipline was real, not reported.
- **What I'd flag**: The discover→re-author→`all` round-trip is pinned only at the parser level; the multi-verb CLI flow remains untested. Accepted at tier, recorded here so it's a known gap, not a forgotten one.
- **Disagreement**: With the Project Engineer's harsh take on the transient-file test — a `finally`-cleaned temp file in `data/` with a collision-proof name is an acceptable cost for covering a real user-facing error path; making dataDir injectable for one test is refactoring the product to please the test.

### Technical Writer
- **User value assessment**: Five doc surfaces plus HELP stayed synchronized through a mid-session design reversal — the operator reading any one of them gets the converged story. The superseded-in-release CHANGELOG annotation is honest history-keeping.
- **Session assessment**: "Document as we go" was followed under pressure, including rewriting rationale the same engineer had written hours earlier.
- **What I'd flag**: The CHANGELOG now contains a claim and its own supersession in the same release block — correct, but a reader skimming only the first entry gets the old story. Minor; the newest-first ordering mitigates it.
- **Disagreement**: None material.

## Section 7: Lessons for Future Sessions

- **Keep**: Sequential re-dispatch to one context-retaining engineer agent for fix-forward rounds; independent orchestrator gate runs after every agent report; decision blocks with ≤2 numbered items — the PO answered every one within a single message.
- **Stop**: Executing irreversible actions (merge) on inferred authorization when the governing rule explicitly says ask. The 1-line confirmation costs one PO glance; the violation cost a trust-checking turn and could have cost a revert.
- **Start**: When a decision block's options imply a follow-on irreversible action (merge, deploy, delete), state the action's authorization status *inside the option text* ("(a) polish, then I merge" vs "(a) polish, then hold") so the PO's answer is unambiguous by construction. Pair with the structural fix from Section 4: an "ask" permission rule on merge-shaped commands, so inference failures are caught by the harness even when discipline slips. And: no unverified numbers in agent briefs — counted in-session or explicitly marked unverified.
- **Value learning**: The PO's "gap" reports come with the solution's shape already embedded (their example command *was* the spec — including us-fast's exclusion). Reading the PO's own workaround carefully is cheaper than re-deriving requirements; both the exclusion default and the later `--layout both` clarification were already latent in their original command line.
