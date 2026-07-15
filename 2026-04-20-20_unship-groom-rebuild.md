# 2026-04-20-20 — unship-groom-rebuild

- **ModelID**: claude-opus-4-7[1m]
- **TurnCount**: ~90 (user: ~45, assistant: ~45; plus ~25 background-agent notification turns that were system-delivered, not user input)
- **ContextUsed**: ~75% — heavy tool-call volume (30+ agent spawns, ~80 bash commands), long agent reports inlined into conversation, multiple full bead descriptions read
- **Personas Active**: Security Engineer, IT Architect, Project Manager, Project Engineer (executed most of the real code work), UX Designer, Code Reviewer (drove all 3 ESLint sessions + external-PR review), Database Engineer, SRE, QA Engineer, Technical Writer. All 10 were invoked for standup and grooming; 8 of 10 authored prereq beads; 3 shipped code (PE, DBA, SRE) plus Security shipped a doc.
- **Beads Touched**:
  - Closed: `mghjm`, `vgm4l`, `jrocu`, `r4zt8`, `87hfq`, `sw883`, `zjge5`, `thfal`, `qmuij`, `ak1db`, `c5wf5` (11 closures)
  - Created: `vgm4l` (rollback), `sw883` (PM intake for #73), `bqpq0` (2nd smart-sort pattern), `thfal` (#77 triage), plus 14 prereq beads from grooming: `xnqgo` `t71c5` `d42ko` `dm9lc` (ADR-001..004 from Architect), `c5wf5` (Alembic/DBA), `2d30g` (style guide/CR), `yopgt` `bwly4` `f1wnt` (CHANGELOG/runbooks/user_guide from TW), `5jy0m` (shared DBAS components/UX), `ak1db` (observability/SRE), `qmuij` (STRIDE/Security), `tp681` `4crtt` (flake/perf from QA). 18 new beads.
  - Title-added to 8 previously-titleless beads: `nmlxi`, `j9xrz`, `zqmv1`, `lx1gf`, `v28b8`, `5x6n7`, `hlcgj`, `in620`.
  - PRs opened and merged this session: #74 (sw883), #75 (lint session 2 — actually merged earlier this day), #76 (lint session 3), #77 (external, Steven Coutts), #78 (post-#77 test fix), #79 (STRIDE doc), #80 (observability), #81 (Alembic).

---

## Section 1 — What We Did Well Together

The **v0.16.0 rollback coordination**. Starting from the moment the PO said "I'm not sure I'm ready for 0.16.0 to be released, that's my fault," through the fully-verified rollback three message-rounds later. Specifically: I proposed three escalating options (soft/medium/hard) with their reversibility spelled out, the PO picked hard, and the PE agent then executed the 9-step rollback across git (main force-reset, dev rewind, tag delete, release delete, protection toggle-and-restore) and GHCR (retag `:latest`, delete `:0.16.0`/`:0.16`). We hit **two legitimate blockers mid-execution** — branch protection on main refused admin bypass, and the `gh` token lacked `delete:packages` scope — and each time the agent stopped cleanly, reported what was done vs. not, and waited for the specific authorization needed. No partial-state left behind. The PO pasted a PAT in chat (which I flagged immediately as a transcript-leak concern), I staged it to `/tmp` with mode 600 and used it only for GHCR ops, and the follow-up PE session cleaned it up. **That's the rollback going right.** Compare to what this could have looked like: silent half-rollbacks, weakened protection left on, a PAT lingering in `/tmp`. It didn't.

## Section 2 — What the PO Could Improve

**The v0.16.0 rollback was the PO's fault and the PO said so — but the earlier framing could have prevented it.** At session start, the PO said both `mghjm` (release) and `zjge5` (lint) were "queued for tomorrow" and explicitly asked me to kick them. I verified the bead's documented trigger conditions (PRs #65 / #66 merged; branch protection on main configured) mechanically and launched the release PE. **Seven hours of rollback work later**, the PO said: *"I've got bugs still that I want cleared."*

The bug list existed in the PO's head, not in beads, not in a sprint scope field, not in the release bead's trigger conditions. The trigger conditions validated "are the code dependencies in place" but not "is the PO actually ready to release." **A single sentence at session start — "verify no open P0/P1 bug beads in the sprint before firing release" — would have blocked the launch and surfaced the gap before we shipped and unshipped.**

I don't want to shift the blame entirely; I should have asked (see Section 3). But from a PO-improvement lens: release-trigger conditions should enumerate the "bugs clear" check explicitly. The bead even has a "Trigger Condition — DO NOT START" block; it just didn't list "no open sprint-bugs" as one of them.

**Secondary:** pasting a GitHub PAT directly into chat (turn ~27) is a credential leak into the transcript. We handled it safely within the session (used it, then the PE deleted the tempfile), but the transcript stores it forever. Better: `gh auth refresh` in an external terminal, or a "paste it to a file and `cat` it from there" handoff. I should have offered that path more forcefully when the refresh failed non-interactively instead of accepting the paste.

## Section 3 — What the Agent Got Wrong

**Two specific misses.**

**Miss 1 (turn ~5, early session): invoked persona agents with `subagent_type: "project-engineer"` / `"code-reviewer"` — both rejected.** The available subagent types are listed in the system prompt (claude-code-guide, Explore, general-purpose, Plan, statusline-setup). The PO's response — *"both of these should be general purpose; we fixed this yesterday in their skills"* — shows they'd explicitly addressed this pattern in the prior session. I took a shortcut based on seeing "code-reviewer" in the Agent tool docstring's example, didn't verify against the actual runtime list. Cost: one round-trip of wasted agent launches. Cheap lesson, but I should have read the subagent_type list before guessing.

**Miss 2 (turn ~55, grooming brief): I briefed the PM persona with "bd-gb5r5.3 has 1 child, bd-gb5r5.4 has 1 child."** The PM's first act was to inspect and correct me: gb5r5.3 has **5 children (.1-.5)**, gb5r5.4 has **3 children (.1-.3)**. I'd read the top-level `bd show` output, which listed only one child each — but that was only the first-level children. I didn't drill into `.3.1` to see it was itself an epic with children. The delegation brief went out with stale sizing, and only the PM's rigor caught it. Had every persona taken my brief at face value, the sprint-sizing verdict would have been systematically under-counted.

**Third, lesser miss (turn ~43): running the grooming agents in foreground** until the PO said *"why aren't you launching these as sub-agents so we can do other work?"* I'd been given this same guidance earlier in the session (QA on PR #76, CR on PR #77) and already written `feedback_background_agents.md`. The second correction on the same pattern within one session suggests I'd saved the memory but not fully internalized the trigger condition ("long-running delegated agent = background by default"). That's on me.

## Section 4 — What Would Make the Project Better

**A pre-release gate bead-linter check.** The rollback story was entirely because release-trigger conditions validated merge state but not "sprint scope clean." A simple `bd` query — "no P0/P1 bugs open with sprint-scope label for current release" — wired into `docs/shipping.md`'s pre-cut checklist would have made today's double-spend impossible. Today we have the convention that bead IDs appear in commit messages but no convention that the release bead has to verify sprint-scope-zero before launching. That's the lightest-weight fix for the session's biggest cost.

Secondarily: **empty beads should not be possible.** `bd-r4zt8` and `bd-87hfq` were created today (by me or a subagent), left empty, and sat `in_progress` for hours before the PM standup caught them as noise. `bd create` should require at minimum `--description` non-empty, or beads auto-close after N hours if they remain empty. Small guardrail, real benefit.

## Section 5 — Persona Perspectives

### Security Engineer
- **Session assessment**: Heard when I raised concerns. STRIDE threat model bead was filed, content-authored, merged same-day. The PAT-in-chat incident was flagged and handled safely.
- **What I'd flag**: **The PAT is still in the transcript.** Rotation was recommended but I don't have visibility into whether the PO did so. That's a real residual risk. Also: we now have ADR-004 (credential & trust model) and the STRIDE doc, but the actual DBAS import implementation hasn't started — the security ACs are written but not enforced in code yet.
- **Disagreement**: I'd push back harder than the PM on shipping v0.17.0 without the threat-model hardening checklist enforced. PM framed scope as a throughput question; I view it as an attack-surface question.

### IT Architect
- **Session assessment**: Good. Four ADRs filed today — first ADRs in repo history, which I've been flagging since onboarding. Getting the skeleton down before the v0.17.0 work starts is exactly right.
- **What I'd flag**: ADRs are stubs. They'll only be load-bearing once fleshed out. Starlette 1.0's FastAPI compat matrix is still an open decision; if the dep-bump epic starts before that resolves, we'll hit it hot. Also: no CI registry push still means every upgrade in bd-6rrl5 validates against a mutated container, not a fresh one. That silent correctness gap is the architect's biggest remaining yellow.
- **Disagreement**: Disagree with the PE's framing of bd-6rrl5 as "split per-child and ship." Ship per child, yes — but the coupling between React 19, vite 8, plugin-react 6, and jsdom 29 means "independent" is relative. ADR-001 should capture which pairs are actually coupled.

### Project Manager
- **Session assessment**: Mixed. Today moved real metal (11 closures, 18 creates, 5 PRs landed). But the session started with a scope miss (release trigger conditions didn't cover "bugs clear") and that cost was borne by the whole team. Grooming session produced genuinely useful prereq-bead decomposition — that was the strongest PM output of the day.
- **What I'd flag**: 46/49 of the pre-grooming backlog was "stale" per `bd stale` — mostly because titleless beads defeated the rule that "stale = not updated in 30 days." Now that we've titled 8 of those, the stale count will drop mechanically next query. Genuine stale-work rate is lower than the number suggested.
- **Disagreement**: Disagree with the SRE's implicit assumption that observability substrate (`bd-ak1db`) automatically unblocks DBAS work. It's necessary, not sufficient — runbooks still need to exist (bd-bwly4), and DBAS epic decomposition is its own multi-session grooming task. Don't short-circuit.

### Project Engineer
- **Session assessment**: Heavy delivery day — shipped sw883, test-fix #78, merged sessions 1-3 of zjge5, triaged #77's CI break, reviewed external PR #77's scope. TDD discipline held throughout (sw883 had 4 tests RED→GREEN; #78 was test-only). Container-first workflow honored.
- **What I'd flag**: The external contribution (#77) taught us something: our review rigor catches overlap and regression shape, but we don't yet have a pre-merge test-delta gate. #77 introduced a deterministic tiebreaker, existing test asserted the old contract — CR + CI would have caught it on the PR itself if CI ran the full backend suite on PR, not just post-merge. Check whether Backend Tests on PR branches has the same scope as post-merge. (Suspicion: yes, but worth verifying.)
- **Disagreement**: Push back on the DBA's P1-elevation of bd-c5wf5 (Alembic). Once it's merged (it is now), the P1 status was justified; but keeping every observability/infra prereq at P1 dilutes the signal. P1 should mean "drops everything else." We had three P1s in flight simultaneously today; that's board noise.

### UX Designer
- **Session assessment**: Heard via the shared-components bead (`bd-5jy0m`). Grooming correctly flagged that gb5r5.3 and gb5r5.4 share 60%+ component needs and shouldn't be built twice. PR #76 (zero lint) included real UX-affecting refactors — 5 modals split, VideoPlayer, AnnotationPopover — and **QA on PR #76 was killed before it ran** because PR #76 merged before QA could finish.
- **What I'd flag**: We merged PR #76 without a real interaction smoke test of the refactored modals. The CR's unit tests passed, but modal effect-timing changes are specifically the bugs unit tests miss. If there are focus-return, form-reset-on-reopen, or drag-drop regressions in the refactored surfaces, we'll discover them from a user report, not a test. The onboarding yellow on "no a11y audit" remains yellow.
- **Disagreement**: Disagree with the PM closing bd-zjge5 without a real UI smoke. The acceptance criteria were about "zero lint + CI flipped + docs" — none of which prove the UX didn't regress. A separate follow-up bead for "manual smoke of PR #76's refactored surfaces" should have been filed before closing.

### Code Reviewer
- **Session assessment**: Productive. Three lint sessions cleared the baseline to zero + flipped CI to blocking — biggest onboarding RED now GREEN. External PR #77 review was thorough and correctly flagged complementary-not-conflicting with PR #74; also flagged that #77 doesn't fix bqpq0 and offered Steven a follow-up path.
- **What I'd flag**: Zero-lint floor is intact today. Eslint 10 (bd-5x6n7) is the single biggest threat — flat-config migration will rename/remove rules and "fix later" is not acceptable. The style guide bead (`bd-2d30g`) must land before eslint 10 or we lose the floor silently.
- **Disagreement**: Disagree with the PO's autonomous-filing decision for prereq beads. It was efficient — but bead content quality varied. UX's bead is tight; two of the ADRs are thin stubs. I'd rather review-then-file than file-then-review. That's a CR-purist view; I accept the throughput tradeoff the PO made.

### Database Engineer
- **Session assessment**: Alembic landed with a baseline revision, FK enforcement, WAL verified, `docs/database_migrations.md` added. That was a session-long deliverable from initiation to merge. Biggest onboarding YELLOW ("no migration framework") is now GREEN-adjacent.
- **What I'd flag**: The 8 orphan legacy tables from the pre-v0.13 health-monitor subsystem are still in live DBs. Baseline deliberately excluded them — correct call — but they're now a "clean up in a future revision" TODO that could rot for months. File a bead explicitly for their removal before it falls off the board.
- **Disagreement**: Disagree with the SRE that `ecm_health_ready_ok` is a complete health signal. It's a binary. Without per-check tags and per-check recent-history, an intermittent Dispatcharr flap looks like a steady-state ok-with-one-failure. Needs refinement before it becomes an alert source.

### SRE
- **Session assessment**: Good. Structured logging + `/metrics` + correlation-ID middleware + cardinality discipline shipped today via `bd-ak1db`. Three of four tests for cardinality guardrails (route pattern not raw path) — exactly the hill to die on. Substrate is in place; SLOs, alerting, Prometheus wiring still in the future-bead pile.
- **What I'd flag**: Today we cut v0.16.0, rolled it back, cut it again, rewound GHCR, merged 5 PRs, and merged through an external contrib — zero of those actions produced a metric trace we can correlate post-hoc. The structured-logging endpoint shipped today, but only post-merge; the session's own operational history is unrecoverable except from `bd` notes. **Runbook for today's rollback** is the single most-owed doc on the board; I'll file it myself next session if TW hasn't by then.
- **Disagreement**: Disagree with the CR that "add tests + review catches regressions." That's the happy path. My lens says: unlabeled ambient behavior (effect timing, WS reconnect, cancellation partial states) has been the bug shape this week (PR #76 refactor risk, #77's tiebreaker contract change). We need integration + smoke tests that exercise *interactions*, not just units. bd-2lw25 E2E-in-CI has been open since 2026-04-19 and is the real answer.

### QA Engineer
- **Session assessment**: Mixed. Flake-audit bead (`bd-tp681`) and perf-baselines bead (`bd-4crtt`) filed. Good. But **QA on PR #76 was spawned, ran partially, then killed when PR #76 merged** — before QA could deliver a verdict. We merged substantial refactoring work without the interaction test I kicked off. UX flagged this too. That's a process gap: QA was in flight, PR merged anyway, QA was killed as stale.
- **What I'd flag**: Merging past an in-flight QA agent is a pattern. We did the same with v0.16.0 — release agent executed while bugs were known-but-unrecorded. Both times the in-flight work was the check that should have blocked the action. "Don't merge while QA is running on this PR" is a rule we haven't written.
- **Disagreement**: Strongly disagree with the PM closing bd-zjge5. No QA pass on the refactored surfaces. CR's unit tests aren't a substitute. I would have left it in_progress pending a smoke test and flagged it explicitly.

### Technical Writer
- **Session assessment**: Three infra beads filed (CHANGELOG, runbooks dir, user_guide dir), each small but blocking. DBA's Alembic work included `docs/database_migrations.md` inline — exactly the pattern I've been asking for (doc ships with the code, not after).
- **What I'd flag**: We closed `bd-zjge5` today. Its acceptance criteria explicitly required documentation ("document in docs/frontend/ or CLAUDE.md why --max-warnings 0 is enforced"). The `docs/frontend_lint.md` landed with PR #76, so that's met — but I want to verify it actually explains the *why* (zero-warnings is a commitment, not a default). Skim of the description suggests patterns-focused, not rationale-focused. If the doc reads as "here's how to fix each rule," it's not the *why* doc the bead asked for.
- **Disagreement**: Disagree with the PO's autonomous-filing of prereq beads without a TW review pass on each description. Several beads cross into "this needs user docs / runbook docs" territory without saying so. I would have wanted a TW editorial pass before the beads landed in the system of record.

---

## Lessons

- **Keep**: **Persona-driven delegation with clean handoffs.** The sw883 fix (PM intake → PE TDD → PR → merge → close) and the rollback (PM files bead → PE executes → stops at scope boundaries → PO authorizes → PE completes) both worked because each persona owned their lane and stopped cleanly at their scope edge. The discipline held under pressure (rollback had two real blockers mid-execution and neither left partial state).

- **Keep**: **Background agents for long delegated work.** Once corrected, it was a clear win — PO made 4 PO-level decisions while 12 grooming + filing + execution agents ran in parallel. Total wall clock for the grooming + prereq-filing + P1 execution: ~60 min real time; in foreground sequential, would have been 4+ hours.

- **Keep**: **Triage-before-groom.** Catching that 9 "titleless" beads were actually well-described (missing title field only) before burning 90 agent spawns on them saved real time. The 10-bead batch fetch to surface the stale-or-not question was worth it.

- **Stop**: **Briefing agents with state I haven't verified.** The PM's correction on gb5r5.3/.4 child counts was a quality save — but the same miss at scale could lock in a wrong estimate across 10 personas. If I'm going to delegate grooming, I need to inspect children recursively, not just the top bead.

- **Stop**: **Treating release triggers as mechanical checklists.** `mghjm`'s trigger conditions said "PR #65 merged, PR #66 merged, branch protection configured" — all true, all green, fired the release. What was missing: "are the bugs that would block release actually cleared?" That's a semantic check, not a mechanical one. When the PO says "ship it" I should still ask "bugs are clear?" — even when the bead trigger says green.

- **Stop**: **Merging past in-flight work.** PR #76 was merged while QA was running on it. PR #76's refactors may or may not have regressed the 5 modals — we don't know, because QA was killed mid-flight. A merge while a verification agent is in flight is a merge without that verification. That's the v0.16.0 pattern at a smaller scale.

- **Start**: **Pre-release bug-clear verification.** Before any release-execution agent fires, query `bd list --status open --label sprint:X --priority 0` (and P1). If non-empty, stop and confirm with PO. File this as a bead-authoring rule for future release beads and mention in `docs/shipping.md`.

- **Start**: **Minimum viable bead description.** `bd-r4zt8` and `bd-87hfq` were created and abandoned empty — two of 49 open beads were pure noise. If `bd create` enforces non-empty description, this class of artifact vanishes. File as a small task for the beads tooling owner.
