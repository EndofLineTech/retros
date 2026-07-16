# 2026-07-15-22 — retro-sync-split

- **ModelID**: claude-fable-5
- **TurnCount**: 6
- **SessionDepth**: light — single-topic session in the skill system repo (claude-agent-dev-team, public, not pseudonymized per corpus convention); two SKILL.md files, README, CHANGELOG
- **Personas Active**: Technical Writer, Code Reviewer, Project Engineer (all implicit — meta-work session, no agents spawned)
- **Beads Touched**: None

## Section 1: User Value Delivered

The retro pipeline's responsibilities were split cleanly: `/retro-sync` is now push-only (scrub + push local retros), and `/retro-mine` owns pulling the corpus down for analysis. Before this, retro-mine ran the full retro-sync — which also scrubbed and pushed — just to get a pull, and retro-sync's dual identity muddied both skills' descriptions and triggers. Users of the skill system get two skills that each do one thing, with the mechanical `pull --rebase`-before-push explicitly labeled as a push prerequisite rather than a feature. Docs (README ×3, CHANGELOG, both SKILL.mds, version bumps) all updated and validated in the same session. Value shipped, not queued.

## Section 2: What We Did Well Together

The PO's framing in the opening message was a complete spec in two sentences: what retro-sync should do (push only), what it shouldn't (pull), and which skill inherits the pull (`/retro-mine` "is intended for that"). No clarifying round-trip was needed. The one judgment call — keeping `pull --rebase` as a mechanical prerequisite for pushing to a shared branch while reframing it as non-gathering — could be made confidently because the intent was unambiguous.

## Section 3: What the PO Could Improve

The second message — "first validate that all documentation is up to date" — arrived without saying whether the PO suspected a specific gap or wanted a general re-verify. The sweep found everything already consistent, so the turn produced confirmation, not correction. If the PO had a specific doc in mind that felt stale, naming it would have targeted the check; if it was general distrust of the turn-1 report, that's feedback the report itself should have earned — see Section 4. Either way, stating which would have made the validation turn cheaper or more useful.

## Section 4: What the Agent Got Wrong

At turn 2 (first work turn), I fired three README edits in parallel before reading the file — all three failed with "File has not been read yet" and cost a wasted round trip. The read-before-edit requirement is mechanical and known; batching edits against an unread file is a self-inflicted retry. Secondary: my turn-1 completion report listed the files touched but didn't state that a full-repo reference sweep had been done, which likely prompted the PO's "validate all documentation" follow-up. A report that says "grepped every retro-sync/retro-mine reference repo-wide; these N locations are all of them" preempts the verification request.

## Section 5: What Would Make the Project Better

The skill system now has a convention worth writing down: when one skill's process step invokes another skill (`retro` step 6 → `retro-sync`, formerly `retro-mine` step 1 → `retro-sync`), a behavior change in the invoked skill silently changes the invoker. This session caught the retro-mine dependency because the PO named it; the `retro` skill's step 6 happened to still be correct by luck of phrasing ("push the retro"). A one-line habit — grep for the skill's name across all SKILL.mds whenever changing its contract — would make cross-skill contract changes systematically safe instead of memory-dependent.

## Persona Perspectives

### Technical Writer
- **User value assessment**: Real value — the old retro-sync description promised pull behavior the PO no longer wants, and stale skill descriptions actively misroute skill invocation (descriptions are the routing surface). Fixing description + when_to_use triggers is functional, not cosmetic.
- **Session assessment**: Good. All four doc surfaces (SKILL.md frontmatter, SKILL.md body, README, CHANGELOG) updated in one pass; historical CHANGELOG entries correctly left as historical record rather than rewritten.
- **What I'd flag**: The when_to_use triggers "gather retros, pull retros" were removed from retro-sync but not added to retro-mine's triggers. Someone saying "pull retros" now matches neither skill cleanly. Minor, but it's a routing gap created by this change.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: The diff was minimal and correct — reframing rather than restructuring, versions bumped, no scope creep into scrub.sh or installer.
- **Session assessment**: The parallel-edit-before-read failure (Section 4) is a process slip, cheap this time but the same reflex on a longer chain wastes more. Otherwise the edit discipline was sound: grep-driven completeness, diff-verified installed copies.
- **What I'd flag**: The claim "installed copies are linked to the repo" was inferred from cp's "identical (not copied)" message, then later verified by diff at the PO's prompting. The verification should have happened before the claim, not after — claims are hypotheses until verified.
- **Disagreement**: With the Technical Writer's severity on the trigger gap — "pull retros" falling through to retro-mine's "mine retros" semantics is close enough that routing will usually recover. Watch item, not a fix-now.

### Project Engineer
- **User value assessment**: The judgment call to keep `pull --rebase` before push was right — a push-only skill that can't push to a shared branch is broken, and the alternative (force-push or per-machine branches) is worse. Labeling it "prerequisite, not gathering" preserves the PO's intent without breaking the mechanics.
- **Session assessment**: Clean. The bootstrap edge case (retro-mine pulling when ~/retros isn't a repo yet) was handled by deferring to retro-sync's existing bootstrap rather than duplicating it.
- **What I'd flag**: retro-mine's pull step has no conflict-handling note; a rebase conflict in ~/retros mid-mine would strand the user. Retros are append-mostly so it's unlikely — acceptable ceiling, but it's undocumented.
- **Disagreement**: None.

## Lessons

- **Keep**: Grep-driven completeness sweeps when changing a skill's contract — every reference to the changed skill across all SKILL.mds, README, CHANGELOG, installers, checked in one pass.
- **Stop**: Issuing Edit calls against files not yet read in the session, especially batched — the failure is guaranteed and the retry round-trip is pure waste.
- **Start**: Completion reports that state the verification performed ("swept N references repo-wide, all updated"), not just the files touched — it preempts the PO's trust-but-verify follow-up.
- **Value learning**: The PO wants skills with single responsibilities even when the combined behavior was mechanically convenient — "one run does both directions" read as a feature when written, but muddied intent in practice. Convenience bundling in skill design is a smell.
