# 2026-07-02-14 — global-coverage-sweep

- **ModelID**: claude-fable-5
- **TurnCount**: ~118 (≈58 user-side turns including ~20 background-task/monitor notifications, ≈60 assistant turns)
- **SessionDepth**: deep — spanned scheduler UI archaeology, container docs, a new CLI verb with tests, a CLI-wide bug-class fix, ~90 minutes of live multi-country discovery crawls, an external-repo parity audit, and a 198-country API sweep
- **Personas Active**: Project Engineer, Code Reviewer, QA Engineer, Technical Writer, SRE, Project Manager, IT Architect, Security Engineer, UX Designer (consulted implicitly), Database Engineer (marginal)
- **Beads Touched**: project-b-2xb, project-b-du0 (created, open); project-b-lq2, project-b-e6b, project-b-7bq, project-b-9cy, project-b-p8o, project-b-982 (created+closed); project-b-0ju (created, later closed by fix); project-b-751 (closed early in session); project-b-lyy (**accidentally closed, immediately reopened**)

## Section 1: User Value Delivered

Concrete, shipped, and user-facing:

- **Coverage went from 8 regions of mixed provenance to 56 countries** — the entire provable footprint of the tmsapi API. End users of this project-b can now build guides for all of Western Europe, the Americas, the full Caribbean, Australia, and (undocumented by the vendor) Sint Maarten and South Korea. This is the product's core value — guide data — multiplied by 7×.
- **Correctness fixes with direct user impact**: Argentina's coverage was silently junk (1 national-default lineup; now 577 stations/24 lineups via CPA-format postals), Honduras was 7 stations (now 388), and the `discover X --conc` NaN bug could overwrite a good coverage CSV with an empty one while reporting success. All fixed, tested, merged.
- **`bootstrap` verb** — operators can regenerate a scheduler job seed from whatever coverage exists, disabled-by-default so a scratch CSV never silently burns proxy quota.
- **Institutional knowledge captured**: `data/README.md` now cites the official vendor country enum *and* the empirical sweep methodology with confirmed-absent list; `docs/CONTAINER.md` documents the container as a system. The next maintainer doesn't re-derive any of today.
- Five PRs merged (#19, #21, #31, #32, #34) plus #17 earlier; beads current.

Work that creates more work (honestly): project-b-2xb and project-b-du0 (scheduler-UI gaps) were filed, not fixed — deliberate deferral, but the UI still can't drive the features the CLI grew today. And the ISO3 rename is a breaking change that will generate at least one support moment for anyone with a seeded scheduler DB.

## Section 2: What We Did Well Together

The **ARG mid-crawl catch and fix**. When Argentina came back "successful" at 1 lineup/122 stations, the anomaly was flagged against its neighbors (Colombia 30 lineups) instead of accepted, the PO said "Okay, lets fix project-b-7bq," and within ~20 minutes we had: a live 3-postal probe proving the CPA-format hypothesis, a GeoNames-derived seed regeneration (whose 1,976 codes turned out to be the seed's literal origin), and a re-discovery producing real coverage. The PO then extended it unprompted — "Can you also fix the postal code input CSV" — which was exactly the root-cause instinct. That loop (anomaly → hypothesis → cheap probe → root-cause fix → re-verify) repeated for HND, PAN, and VGB and is the reason the final dataset is trustworthy rather than merely large.

## Section 3: What the PO Could Improve

**The PR-workflow preference surfaced only after work had been pushed to master.** The container docs were committed and pushed directly to master (following CLAUDE.md's mandatory "PUSH TO REMOTE" session protocol), and only then came "Throw a PR for this as a documentation update" — which forced a revert commit on master (`77708c8`) plus a cherry-pick branch to reconstruct a reviewable PR, leaving permanent revert noise in history. The same preference was re-stated later ("We will want to commit those via a PR") for the coverage work. One sentence at session start — "everything goes through PRs" — would have avoided the revert dance entirely. Related and worth the PO's attention: **CLAUDE.md's session-completion protocol actively contradicts this preference** (it mandates direct `git push` to remote as the definition of done). That contradiction is a standing trap for any future agent session; it should be edited to say "push the feature branch and open a PR."

Minor second instance of the same pattern: "I'd prefer to consolidate, one. Two, lets do DEU and UK." left the consolidation *target* (ISO3 vs ISO2 naming) unstated for a breaking rename touching every doc and config. The ISO3 inference was correct, but a wrong guess would have been expensive to unwind post-merge.

## Section 4: What the Agent Got Wrong

**I chose direct-to-master pushes despite visible evidence this repo works by PR.** The git log at session start showed PR merges (#14, #15, #16) as the repo's entire recent history; when CLAUDE.md's protocol said "push," I followed it literally for the beads commit and the container docs instead of reading the room and branching. The PO shouldn't have had to ask for a PR after the fact — the revert commit on master is my noise, not theirs.

Secondary accountability items, briefly: the `bootstrap` verb went to PR with **10 review-confirmed defects** including a scheduler crash-loop (case-folded region names colliding on the jobs PK) and shipped `enabled: true` job seeds that the PO had to correct to disabled — I wrote the disabled-by-default rationale convincingly *after* being told, which means I could have reasoned it before. I fat-fingered `bd close project-b-lyy` on an unrelated open issue (caught and reopened same minute). And I piped the 198-country sweep through `tail -40`, buffering all live output — when the PO asked "How goes so far," I had nothing to show and had to admit the self-inflicted blindness.

## Section 5: What Would Make the Project Better

**Discovery needs built-in anomaly detection.** Every bad-coverage incident today (ARG, HND, and the risk in the `--conc` NaN bug) shared one signature: discovery *succeeded* while producing implausibly thin results — 1 lineup from 1,950 postals, 7 stations for a country of 10 million. Each was caught by a human eyeballing station counts against neighbors. `discover` should warn (or exit nonzero with `--strict`) when heuristics fire: unique-lineups == 1 with a large postal seed, stations-per-capita far below regional peers, or every postal resolving to the identical lineup (the fallback signature). That turns today's luck into a repeatable guardrail — the difference between "we caught ARG" and "we catch the next ARG."

Also worth doing: `.beads/issues.jsonl` conflicted on *every single PR merge* today; the id-keyed union-merge I did by hand three times should be a committed merge driver or a `bd` subcommand.

## Persona Perspectives

### Security Engineer
- **User value assessment**: Real: egress discipline protects the shared proxy pool (an operational asset users depend on) and the API key. No compliance theater this session.
- **Session assessment**: Good — explicit egress confirmation via structured question before the crawl; proxy credentials counted but never printed; `.env` never committed; the concurrency envelope (≤0.7× pool) respected throughout.
- **What I'd flag**: The country comb probed a commercial API with ~500 requests using pseudo-postals — legitimate key usage, but it's the kind of exploratory traffic worth noting in the bead (it was). Also: 56 regions now hard-require `GN_TMSAPI_KEY` with no gracenote fallback; key rotation/quota exhaustion is now a single point of failure for nearly all coverage.
- **Disagreement**: None material.

### IT Architect
- **User value assessment**: The "region = data, not code" principle proved itself — 48 countries added without one code branch. That architecture is why today was even possible.
- **Session assessment**: Mostly sound, one process gap: the ISO3 consolidation was a **breaking public-interface change** (region names are the CLI's UX) decided from a three-word directive and shipped inside a coverage PR. It deserved a written decision record before execution, not after.
- **What I'd flag**: `DISCOVER_PROVIDERS` being an explicit three-country list (not a region predicate) silently produced tmsapi-only CSVs for PRI/DOM/COL — dual-provider redundancy quietly narrowed. Nobody decided that; the code shape did.
- **Disagreement**: With the PM's "five PRs merged" framing — PR #31 bundled a breaking rename with 39 data files, which is exactly how breaking changes sneak past review.

### Project Manager
- **User value assessment**: High throughput that all traces to user outcomes (coverage, correctness, operability). The two open beads (project-b-2xb, project-b-du0) are honest deferred work, properly scoped.
- **Session assessment**: Beads discipline was excellent — every substantive piece of work has a created/closed issue with evidence. The accidental project-b-lyy close was caught in-turn. Merge sequencing (21 before 31) was handled with conflicts pre-resolved as the PO asked.
- **What I'd flag**: The CLAUDE.md push-protocol vs. PR-workflow contradiction (Section 3) is a process bug that will bite every future session until the file is edited. Filing that edit as a bead should have happened today; it didn't.
- **Disagreement**: I accept the Architect's point on #31's bundling, but note the PO explicitly directed consolidation onto that branch mid-review; splitting it would have traded reviewability for churn.

### Project Engineer
- **User value assessment**: The bootstrap verb and arg-guards are real operator value; the crawl tooling (queue script, resumable spools honored, worker isolation via git worktree during live runs) delivered the coverage without stepping on the running system.
- **Session assessment**: TDD held for pure logic (bootstrap 16→20 assertions, argv 29 spawn-based). Using a worktree to fix code while crawls ran the old code was the right call. Shell craft was shakier: three failed launches (case-in-`$()`, xargs `-I` length limit, stale cwd) — all recovered fast, all avoidable.
- **What I'd flag**: The cwd bug that killed the first DEU/GBR launch is a footgun class: background commands inherit mutable shell state. Absolute paths everywhere in run scripts, always.
- **Disagreement**: With Code Reviewer below — partially. Yes, review caught 10 defects; but the offline test suite genuinely couldn't catch the casing bug without a case-sensitive-volume simulation. The lesson is a testing-environment gap, not just insufficient care.

### Code Reviewer
- **User value assessment**: The review pass on PR #19 protected real users: the casing/PK-collision finding was a first-boot **scheduler crash-loop** — the worst possible container UX — and the `--programs-limit ten` → uncapped-paid-calls finding protected users' money.
- **Session assessment**: The workflow-backed review earned its cost (10 confirmed defects from ~586k tokens). But note the direction of causality: the agent's self-authored code needed external adversarial review to be safe, and the PO had to *ask* for the review. It wasn't offered.
- **What I'd flag**: `enabled: true` shipping as the bootstrap default was a design defect caught by the PO, not by review or tests. Defaults are the highest-leverage code there is; they deserve explicit adversarial thought ("what's the worst thing this default does on the messiest real data/ dir?").
- **Disagreement**: With the Engineer's framing — the casing bug *was* findable pre-review: `seedJobsIfEmpty`'s PK constraint is visible in `db.js`, and mapping generated ids onto it is exactly the integration thinking the unit tests skipped.

### Database Engineer
- **User value assessment**: Marginal role today; the jsonl union-merge preserved all 194 issues across three conflicted merges — that's team-facing data integrity, not aesthetics.
- **Session assessment**: The id-keyed, newest-timestamp-wins merge was the correct conflict semantics for an append-mostly export. SQLite spool behavior (resume, cleanup-on-success) performed exactly as designed under 39 concurrent-ish crawls.
- **What I'd flag**: `.beads/issues.jsonl` is a guaranteed merge hotspot — line-ordered JSON export in a shared file. A custom merge driver (union by id) in `.gitattributes` would eliminate the recurring hand-resolution.
- **Disagreement**: None.

### SRE
- **User value assessment**: The monitoring pattern (per-country DONE/FAIL events via Monitor, long-poll fallbacks, resumable-by-design crawls) kept a 90-minute multi-stage operation observable and interruptible — that's operator experience, which is user experience for this product.
- **Session assessment**: Strong on the crawl (LPT queue, dynamic workers, concurrency envelope from empirical data, per-country logs). Weak on the comb: `| tail -40` buffered all output, making "how goes?" unanswerable mid-run — an observability self-own on a 198-probe operation.
- **What I'd flag**: The 30s per-probe timeout in the comb was the only thing bounding worst-case wall time; sweeps against external APIs should always log a heartbeat (N/total) to their output file, not just terminal results.
- **Disagreement**: None; the Engineer's cwd footgun flag is seconded — that failure was silent for two runs' worth of launch latency.

### QA Engineer
- **User value assessment**: Test additions targeted user-visible failure modes (bad CLI values, disabled-by-default, symlink resilience) — behavior, not coverage metrics. Good.
- **Session assessment**: The gap is **live-output validation**. Thirty-nine coverage CSVs were authored and merged with zero automated assertion on their *content* — plausibility was checked by a human reading station counts in chat. ARG proved that "discover exits 0" is not a quality gate.
- **What I'd flag**: Section 5's anomaly-detection proposal is the top QA priority for this codebase — I'd extend it to a `discover --verify` mode comparing against the prior CSV on refresh (station-count regression = warn).
- **Disagreement**: With the PM's throughput framing: five merged PRs in one session with one reviewer (the PO) means review depth was necessarily thin on the data-heavy PRs. Nobody diffed 34k lines of Korean postal codes. That's an accepted risk, but it should be *named* as one.

### Technical Writer
- **User value assessment**: High. `docs/CONTAINER.md` serves the next container maintainer; the rewritten "Supported countries" section — official enum URL + empirical method + confirmed-absent list — answers the exact question a future maintainer will ask ("are we missing countries?") with evidence, not folklore. Today's session literally *was* that future maintainer's question, asked by the PO; now it's answered in-repo.
- **Session assessment**: Docs shipped with every change (verb help text, README, CLAUDE.md counts, PR bodies as change narratives). The stale-docs finds (gracenote-era "Europe hard-gated", the COL/PAN anomaly note, the test-file counts) were corrected on contact rather than left.
- **What I'd flag**: CLAUDE.md's session-completion section is now the most dangerous stale doc in the repo (Section 3). Whoever edits it should also reconcile "23 files" → the count keeps drifting; consider removing the number entirely.
- **Disagreement**: None.

### UX Designer
- **User value assessment**: The disabled-by-default bootstrap decision (PO's call) is the session's best UX judgment: a fresh deployment shows a full inventory of buildable regions with zero surprise cost. "An inventory of what you can build; turn on what you want" is a genuinely good mental model.
- **Session assessment**: The scheduler UI's users were identified as underserved early (project-b-2xb, project-b-du0) and then... left underserved. Acceptable triage, but after today the CLI-vs-UI capability gap is much wider: the UI can't express `--provider`, can't run `channels`, and now faces 56 regions in whatever region-picker exists.
- **What I'd flag**: The ISO3 rename breaks muscle memory (`us` → `usa`) with no alias or helpful error mapping. `project-b.js us` now says "input not found" — it should say "did you mean usa? (regions were renamed to ISO3)". One string, big kindness.
- **Disagreement**: With shipping the rename without that error message; the Architect's "needed an ADR" point is process, mine is the actual user moment of breakage.

## Lessons

- **Keep**: The anomaly → cheap-probe → root-cause → re-verify loop (ARG/HND/PAN/VGB). Also: isolating code changes in git worktrees while long-running processes execute the old tree — zero interference across three concurrent workstreams.
- **Stop**: Direct-to-master pushes in this repo, regardless of what CLAUDE.md's protocol says — the PO's actual workflow is PR-everything. Also stop piping long-running probe output through `tail`/`head` (buffering kills observability).
- **Start**: Offering the code review proactively on self-authored features before the PO asks; adding plausibility checks to discovery output (Section 5); proposing a `.gitattributes` merge driver for `.beads/issues.jsonl`.
- **Value learning**: The PO's questions were consistently better product instincts than the backlog: "does the UI support this?", "do these match the vendor's docs?", "did we miss any countries?" each opened the session's most valuable workstream. The assumption that coverage gaps were API limitations ("none locally", "anomalous 400s") was wrong in every single case investigated — they were all seed-format problems. Default to distrusting "the API doesn't serve X" claims until probed with correctly-formatted input.
