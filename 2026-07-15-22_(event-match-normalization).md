# 2026-07-15-22 — event-match-normalization

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~24 (3 genuine user turns — initial task+constraint, "Yes please" to ship, /retro — the rest assistant tool-cycles and background-task notifications)
- **SessionDepth**: moderate — single-domain (event-sync matcher) but with real production-evidence tracing, a fix, full verification, and a concurrent-session coordination overlay
- **Personas Active**: project-engineer (implemented), code-reviewer, qa-engineer, sre, security-engineer, technical-writer, project-manager
- **Beads Touched**: project-d-79k6b (created, in_progress, closed — shipped in PR #664)

## Section 1: User Value Delivered

Real and verified. Two live Flo Racing event streams that operators expected to auto-attach to their master channels were silently landing in the "ambiguous" review queue instead of attaching. The operator would have had to hand-resolve each one every time these events recur. Root cause was two un-normalized characters in the provider-supplied master names — a backtick used as an apostrophe (`Joker` + backtick + `s`) and an underscore (`Utica_Rome`) — both of which dropped the fuzzy title score below the 0.90 teamless-attach floor.

The fix normalizes both in the shared LOCALS name cleaner and was verified against the *actual* production debug-bundle data (the exact scores 0.837 and 0.703 were reproduced before any code changed, then confirmed to lift to 1.0 attach after). A hyphen-spelling variant of the same event recovered as a bonus. Shipped to `dev` in PR #664 with a frozen precision/recall corpus gate green plus three new regression entries, so it can't silently regress.

This is the good kind of session: a narrow, well-evidenced correctness fix that removes recurring manual toil, with a regression net left behind.

## Section 2: What We Did Well Together

Evidence before hypothesis. The PO didn't just say "these don't match" — they pasted the sync'd names, the expected matches, AND the already-matching controls (GAF, IWK marked "all matched"). That control data was quietly load-bearing: it let me confirm the fix lifted the failures to 1.0 *without* moving the controls off 1.0, which is how I knew I hadn't over-normalized. On my side, pulling the debug bundle's `event_sync_matching.json` and reproducing the exact ambiguous-band scores against the real functions before touching code meant the whole fix rested on measured behavior, not a plausible-sounding theory. The PO's inclusion of controls + my insistence on reproducing first combined into a tight diagnose→fix→verify loop.

## Section 3: What the PO Could Improve

The "keeping things separate will be necessary" constraint was stated as important but under-specified, and the gap turned out to matter more than it looked. The PO knew a second session was active but didn't say *what it was touching*. I discovered only at cleanup that the other session had a worktree on `fix/gh645-dedup-matchkey` — dedup-matcher work — while **my** change was in `dedup_matcher.py`. That's a genuine near-collision on a shared file, not an abstract one. It merged clean only because the other branch hadn't landed yet; when gh645 merges it may still conflict with my now-merged `_normalize` change. Thirty seconds of "the other session is on gh644 docs and gh645 dedup-matchkey" up front would have told me my target file overlapped theirs, and I'd have flagged the overlap risk to you *before* shipping rather than noticing it in the worktree-list output afterward.

## Section 4: What the Agent Got Wrong

Two misfires, both recoverable but real:

1. **I misattributed the hook block on `bd create`.** When my first two `bd create` attempts were denied, I read it as a policy boundary ("orchestrator can't write beads, that's persona work") and routed bead creation into the engineer's dispatch brief — where it *also* got blocked, because subagents genuinely can't do board writes. The actual cause was mundane: my command text contained a backtick and a `>=`, which the pretooluse hook's redirect/substitution detector flagged as shell metacharacters. Board creation *is* orchestrator territory. I only figured this out on the third attempt. Net cost: one wasted instruction to the engineer and a round-trip.

2. **My first independent repro was built wrong and briefly looked alarming.** I called `score_pair` without parse `patterns`, so *every* case — including the GAF control that should be 1.0 — came back 0.0/reject. For a moment that reads like "the fix broke everything." I'd already seen the `score_pair` signature earlier in the session, so I should have known it needs patterns and gone straight to `_fuzzy_title_score` on parsed titles (which is the layer the fix actually touches). I recovered by recognizing it as a harness error, but a sharper first move would have skipped the scare.

## Section 5: What Would Make the Project Better

A cross-session file-scope registry. Two concurrent Claude sessions on one checkout have zero shared visibility into which files each is editing. This session's fix touched `dedup_matcher.py` while the other session had an open dedup worktree; I found out by reading `git worktree list` during cleanup, not by any mechanism designed to surface it. A lightweight convention — each session writes its intended file scope somewhere shared (a scratch file, a bead label, even a branch-description) at dispatch time — would turn silent overlap risk into an explicit, early signal. Related, smaller: the pretooluse hook's false-positive on benign shell metacharacters (`>=`, backticks) inside `bd` description text is a recurring friction; description text is data, not a command line, and shouldn't trip the redirect detector.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: High. The change removes recurring operator toil on a real, reproduced failure, not a hypothetical. LOCALS-only scoping meant the load-bearing CONSERVATIVE dedup path stayed byte-for-byte identical — no blast radius on the 1,341-merge-incident invariant.
- **Session assessment**: Sound. Reproduce-first, fix, verify-through-real-functions, regression-net. Extending `_team_tokens` to the same apostrophe set was a justified generalization of a pre-existing latent inconsistency (backtick names split into two tokens while straight-apostrophe names fused).
- **What I'd flag**: The fix lives in a file the other session was concurrently modifying. The clean merge into `dev` is not proof of no future conflict — the sibling dedup branch is still out there.
- **Disagreement**: I'd push back on the code reviewer if they wanted the apostrophe char-set factored into one shared module constant right now — the engineer deliberately kept two local constants with a "keep in sync" comment to avoid a cross-module import of a private name. That's a defensible call, not a defect.

### Code Reviewer
- **User value assessment**: The quality discipline here directly served users — the corpus regression entries mean these exact events stay matched across future matcher changes.
- **Session assessment**: Good. Verified independently (I re-ran the diff, the targeted suites, and the repro myself rather than trusting the agent's self-report), confirmed the 9 failures were pre-existing crypto-env, confirmed ReDoS-safe regex.
- **What I'd flag**: Two copies of the apostrophe character class (`_LOCALS_APOSTROPHE_RE` and `_TEAM_APOSTROPHE_RE`) with a comment asking humans to keep them in sync is a real drift risk. A single shared frozen constant would be safer. Minor, but it's the kind of thing that rots.
- **Disagreement**: With the engineer, mildly — see above. "Keep in sync by comment" is how char-sets diverge six months later.

### QA Engineer
- **User value assessment**: Testing focused on user-facing behavior (does the event attach?) not coverage metrics — correct emphasis.
- **Session assessment**: Strong. Three append-only corpus entries built from real debug-bundle names, plus CONSERVATIVE-preservation fixtures proving the change is LOCALS-only, plus the hyphen variant as an extra case.
- **What I'd flag**: The new corpus entries needed a synthetic parseable date suffix because the live rule used time-only patterns — so the corpus exercises the *scoring* layer faithfully but not the exact *parse* path the production rule uses. That's a small fidelity gap between the regression net and the live config.
- **Disagreement**: None material.

### SRE
- **User value assessment**: This is reliability-of-outcome work — fewer events falling into a manual queue is less operator load and fewer missed live-event attachments.
- **Session assessment**: The decision to verify via pure-Python matcher functions instead of deploying to and restarting the shared container was the right operational call with a second session possibly using that container.
- **What I'd flag**: The near-collision on `dedup_matcher.py` between two sessions is an operational hazard, not just a code one — concurrent uncoordinated writes to a shared checkout is exactly the class of thing that produces "works in isolation, breaks on merge" incidents.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: Directly protective. This cleaner runs on attacker-controllable M3U/stream names inline on the event loop, so a non-linear regex here is a real DoS surface. Keeping the new patterns as bounded character classes with single quantifiers (Semgrep-validated, 0 findings) protected users from that.
- **Session assessment**: The ReDoS constraint was treated as first-class, matching the file's existing hardening notes.
- **What I'd flag**: Nothing new introduced. The apostrophe/underscore steps are constant-time.
- **Disagreement**: None.

### Technical Writer
- **User value assessment**: The CHANGELOG `[Unreleased] → Fixed` entry and the dense in-code comments (citing the bead, the failing scores, the ReDoS rationale) serve the next dev who touches this cleaner — that's the audience that gets burned by silent normalization behavior.
- **Session assessment**: Adequate for the change size. No user-facing doc needed.
- **What I'd flag**: The placeholder bead slug lived in code/tests/corpus until the ship step swapped it for the real id — a reminder that "I'll pin the id later" is a real follow-up, not a freebie.
- **Disagreement**: None.

### Project Manager
- **User value assessment**: Clean value-to-effort ratio. One bead, one PR, one recurring toil source removed, regression-protected.
- **Session assessment**: Well-sequenced: diagnose → dispatch → verify → ship → cleanup → close, with the decision point (ship?) surfaced clearly and separately.
- **What I'd flag**: The concurrency risk was managed reactively (discovered at cleanup) rather than planned for at intake. For a repo with two live sessions, "what is the other session's file scope" belongs in the opening triage, not the closing worktree-list read.
- **Disagreement**: With any "it merged clean, so concurrency was handled well" reading — clean merge was partly luck of ordering, not solely process.

## Lessons

- **Keep**: Reproduce the failure against the real functions on real data (debug-bundle `event_sync_matching.json`) *before* writing the fix, and keep the PO-supplied already-passing cases as controls to prove no regression. This grounded the entire session.
- **Stop**: Misreading a hook denial as a policy boundary. When `bd`/tool calls get blocked, first check whether my *command text* contains shell metacharacters (backtick, `>=`, redirects) before concluding it's an orchestration-firewall rule.
- **Start**: At intake of any session flagged "another session is active," immediately establish the other session's file/branch scope (`git worktree list`, `git branch`, ask the PO) and compare against my intended target files — surface any overlap to the PO *before* dispatching, not at cleanup.
- **Value learning**: Operators experience matcher precision as toil, not as a score. A teamless event stuck one hundredth of a point below an invisible floor is, to the user, an event that "just doesn't work" and has to be fixed by hand every week. Sub-threshold near-misses in a review queue are a user-value problem, not a tuning curiosity.
