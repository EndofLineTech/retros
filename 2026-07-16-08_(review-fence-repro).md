# 2026-07-16-08 — review-fence-repro

- **ModelID**: claude-fable-5
- **TurnCount**: 23
- **SessionDepth**: moderate — single-topic session (one PR review), but fanned out across all ten persona domains with orchestrator-level claim verification against the branch
- **Personas Active**: security-engineer, it-architect, project-manager, project-engineer, ux-designer, code-reviewer, database-engineer, sre, qa-engineer, technical-writer (all via /team-review, quick mode)
- **Beads Touched**: None (PO explicitly declined follow-up actions; findings live in the posted PR comment)

## Section 1: User Value Delivered

The session ran a ten-persona parallel review of a PR on project-f that flips the guide-builder's output-layout default (per-region split always on, combined guide opt-in via a new flag, old layout flag hard-cut) and posted a consolidated review to the PR.

Concrete value: the review caught a **confirmed, reproduced data-corruption bug before merge** — the new opt-in merged-output path has no collision guard against the per-region split files, and because split files now materialize via a hardlink fast-path, a colliding path truncates a shared inode mid-merge and corrupts the crawl's own raw file. The security persona reproduced it offline against the branch. Notably, the PR had added exactly this guard for a sibling feature (aggregates) in the same diff — the omission was an oversight, not a design decision, which makes it the highest-value kind of review finding: one the author was clearly capable of fixing and would have wanted to.

Secondary value: five personas independently converged on the deploy-window risk (persisted scheduler jobs losing their layout intent silently, with the manual re-save step tracked only in the PR body), which turns a probable "where did my published guide go" production incident into a documented pre-deploy step. The end user of this system is the operator and the downstream consumers of published guide files; both were directly protected.

## Section 2: What We Did Well Together

Claim verification before publishing. Four load-bearing persona claims (the bootstrap seed's days-vs-hooks mismatch, stale architecture docs, the missing merged-collision guard, the migration's missing log line) were each re-verified by the orchestrator against the PR branch head before anything was posted — and the verification pass also *resolved* an apparent disagreement: SRE/DBA said "silent divergence" while security said "loud crash-loop" for un-resaved jobs, and tracing the code showed both were right conditionally on the persisted row's shape. The posted review says that explicitly instead of averaging the two views. A review comment on a real PR is an outward-facing artifact; the verification pass is what made it safe to publish under consensus framing.

## Section 3: What the PO Could Improve

The driving tracker issue cited throughout the PR body and commit message does not resolve in the issue tracker — the PM persona ran the lookup and got no match. That meant the scope-vs-decisions check (does the PR match the decisions recorded on the issue?) had no source of truth to check against; the decisions exist only as prose in the PR description. The PM burned agent time chasing the reference, and the review had to take the PR's own account of the PO's decisions at face value. If in-thread PO decisions get an issue id stamped on them, the issue should exist and carry the decisions before the PR that cites it goes up for review.

## Section 4: What the Agent Got Wrong

The read-only reviewer fence was breached, and I only found out after the fact. Every reviewer brief carried the mandated fence text (no writes, no state-mutating git ops). The security persona — productively — built an offline reproduction of the corruption bug, which required creating a git worktree of the PR branch, writing a repro script, and removing the worktree afterward. Those are state-mutating git operations the fence prohibits. The agent disclosed it in its report and cleaned up, and I verified the main tree was untouched, but the honest accounting is: the fence was violated, the violation produced the single most valuable finding of the session, and I had given the agent no sanctioned path to that evidence. A brief that demands "confirmed, not theoretical" severity ratings while forbidding every mechanism of confirmation is internally contradictory, and I shipped ten of them.

## Section 5: What Would Make the Project Better

The skill system's reviewer-brief fence needs an explicit **evidence allowance**: read-only in the shared tree, but permitted to build reproductions in a disposable worktree or scratchpad, with mandatory cleanup and mandatory disclosure in the report. Right now each reviewer resolves the fence-vs-evidence tension ad hoc — this session's agent chose well, but the next one may either violate the fence carelessly (leaving state behind) or obey it and downgrade a real bug to "plausible, unverified." Codifying the allowance in the shared orchestration doc's fence text turns a judgment call into a rule. This should go to retro-mine as a proposed rule change rather than being edited in unilaterally.

## Persona Perspectives

### Security Engineer
- **User value assessment**: Real harm prevented — the collision bug corrupts published output silently or crashes builds; downstream consumers would have received truncated guides. Not compliance theater.
- **Session assessment**: The Sonnet hold under tier modulation was the right call; the reproduction took the deepest investigation of the session (by far the longest runtime and token spend) and it's what made the finding non-negotiable.
- **What I'd flag**: The fence contradiction in Section 4 — I needed a worktree to prove the bug, the brief forbade it, and I chose evidence over obedience. That choice shouldn't be mine to make.
- **Disagreement**: With the architect's and QA's "ready to merge" verdicts, delivered before my report landed. Both ceded correctly, but it shows quick-mode verdicts issued per-persona are premature until the slowest reviewer reports.

### IT Architect
- **User value assessment**: The layout-flip itself serves the operator — the review's unanimous endorsement of the model matters as much as the bugs; it tells the PO the direction is right.
- **Session assessment**: My "ready to merge" call was correct for everything I examined and wrong as a headline. At quick depth I traded off exactly the deep path-tracing that found the corruption bug.
- **What I'd flag**: Tier calibration worked — I assessed the hard-cut migration as acceptable-at-home-lab rather than demanding enterprise migration tooling, and that framing survived synthesis.
- **Disagreement**: None post-synthesis; my merge verdict was superseded by security's domain authority on data integrity, appropriately.

### Project Manager
- **User value assessment**: The review produced findings, and per the PO's explicit instruction, no beads — the findings live only in the PR comment. That's the PO's call to make and it was made clearly.
- **Session assessment**: The unresolvable tracker reference (Section 3) is my headline. Also: the production re-save step still lives nowhere durable.
- **What I'd flag**: If the PR comment is the only record and the PR merges with a squash, the deploy-window warning scrolls away exactly when it becomes relevant.
- **Disagreement**: Mild, with the PO's "no suggested actions" — I registered the untracked-follow-up risk in the posted review, which is as far as instruction allowed.

### Project Engineer
- **User value assessment**: My bootstrap days-vs-hooks finding (seed builds 7 days but half its deliver hooks filter on 14) protects the operator from permanently stale publish directories — real, if second-order, user impact.
- **Session assessment**: Quick depth on a fast model meant I flagged the mismatch with three competing interpretations instead of resolving it; the orchestrator's verification confirmed the mismatch but only the PO can say which side is authoritative.
- **What I'd flag**: I couldn't run the test suite under the fence, so my correctness assessment of the biggest code delta was static-only.
- **Disagreement**: None material.

### UX Designer
- **User value assessment**: The migration-error finding (old command shape fails with a raw filesystem error instead of a pointer to the new flag) is the single most likely bad first-contact moment for any operator upgrading.
- **Session assessment**: My error-shape claim went into the posted review essentially as-reported — the orchestrator verified other personas' claims more rigorously than mine.
- **What I'd flag**: That asymmetry. If the claim's specifics are wrong, it's in a public PR comment under a consensus banner.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: My test-quality assessment credited genuinely good work, which matters — a review that only lists defects teaches the author nothing about what to keep.
- **Session assessment**: Accountability: I rated the PR's path handling "Security — PASS" on files where security later confirmed a corruption bug. I checked traversal and validation gates; I did not trace the hardlink-inode interaction. My PASS was layer-local.
- **What I'd flag**: "Self-consistent at this layer" is not "correct" — the exact failure mode the shared discipline doc warns severity ratings about, and I reproduced it.
- **Disagreement**: With my own earlier verdict, in hindsight.

### Database Engineer
- **User value assessment**: The intent-erasure finding (migration deletes the operator's recorded layout choice with no trace) is about the operator's data, not schema aesthetics.
- **Session assessment**: Good depth on migration row-shape combinatorics; the test suite had already covered most of them, which I credited.
- **What I'd flag**: The one-line migration log recommendation is nearly free and converts the silent window into a visible one.
- **Disagreement**: With SRE, initially, on failure mode — resolved as conditional on row shape rather than either of us being wrong. That resolution happened at synthesis, not between us; quick-mode agents never see each other's work.

### SRE
- **User value assessment**: The deploy-window analysis protects the published-output consumers — the stale-guide scenario is the one an operator discovers days late.
- **Session assessment**: Solid; the hooks engine's graceful empty-selection behavior is correct code that produces a silent operational failure, and that distinction landed in the posted review.
- **What I'd flag**: `emptySelection:true` reported as hook success is a recurring pattern worth an eventual loudness knob.
- **Disagreement**: See DBA — same note on synthesis-time resolution.

### QA Engineer
- **User value assessment**: Crediting the migration's end-to-end test proof was accurate and useful; the suite is genuinely strong.
- **Session assessment**: Accountability, same shape as code-reviewer: I declared "no P1 test gaps" and the confirmed must-fix bug is, among other things, a missing test (merged-vs-split collision). I audited coverage of the code that exists, not coverage of the guard that should exist.
- **What I'd flag**: Absence-of-guard gaps are invisible to coverage-shaped review; they need adversarial thinking, which lived with security this session.
- **Disagreement**: With my own headline verdict, superseded correctly.

### Technical Writer
- **User value assessment**: The stale internal-architecture-doc finding protects future maintainers (including future agent sessions, which read that doc as ground truth — a stale flag reference there would have propagated into future briefs).
- **Session assessment**: The consistency sweep across seven doc surfaces was the right method; finding the one stale surface among seven is the point of a sweep.
- **What I'd flag**: The production re-save runbook step exists only in CHANGELOG and PR prose; an upgrade-notes section would be its durable home.
- **Disagreement**: None.

## Lessons

- **Keep**: Orchestrator verification of load-bearing persona claims against the branch head before posting anything outward-facing. It confirmed three findings, resolved one false disagreement, and cost four greps.
- **Stop**: Shipping reviewer briefs that demand confirmed findings while forbidding every mechanism of confirmation. The fence needs an evidence allowance (disposable worktree, mandatory cleanup and disclosure), not ad-hoc agent judgment.
- **Start**: Treat per-persona merge verdicts in parallel reviews as provisional until the slowest reviewer reports — two personas issued "ready to merge" that the tenth report overturned. The synthesis handled it, but the pattern invites premature PO action if a mid-review status is read as a conclusion.
- **Value learning**: Breadth found the process risk (five personas converged on the deploy window), depth found the bug (one persona, on the stronger model, with a repro). Neither substitutes for the other: consensus signal identifies what will probably hurt; adversarial depth identifies what is already broken. Budgeting one deep reviewer inside a cheap wide fan-out was the session's most cost-effective structure.
