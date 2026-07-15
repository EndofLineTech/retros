# 2026-07-15-14 — retro-pipeline-buildout

- **ModelID**: claude-fable-5
- **TurnCount**: ~13 (6 user, ~7 assistant)
- **SessionDepth**: moderate — single-topic (retro pipeline) but spanning skill design, shell scripting, a public data-publishing decision, and live testing against a 30-file corpus
- **Personas Active**: Security Engineer, Project Engineer, Code Reviewer, Technical Writer, Project Manager, QA Engineer (all consulted implicitly — this was meta-work on the skill system, where the orchestrator edits directly)
- **Beads Touched**: None (skill-system repo has no board)

## 1. User Value Delivered

Real and shipped. The PO's concrete pain — retros stranded per-machine in `~/retros/`, moved by hand with SCP — is gone. This session delivered:

- A public single source of truth for retros, with the full 30-retro backlog scrubbed, pseudonymized, and published
- `/retro-sync`: one command that pushes local retros and pulls everyone else's, with a fork+PR path so *other users of the skill system* can contribute — turning a private habit into a shared learning corpus
- Two-layer anonymization: generation-time rules in `/retro` (write clean in the first place) backed by a deterministic scrub script (stable project pseudonyms from a private local map, applied to content and filenames; redaction of emails, IPs, credential formats, and denylist terms)
- `/retro-mine`: closes the loop by turning the corpus into proposed rule changes for the skill system, with an evidence threshold (≥2 retros), rule-not-working detection, and PO approval before any edit

The value chain is complete: retros → corpus → mined patterns → rule changes → future retros validate. Before this session only the first and last links existed, connected by manual labor.

## 2. What We Did Well Together

The anonymity-level exchange. The first assistant reply flagged "how anonymous is anonymous?" as the one decision that changes the design, with an explicit recommendation to defer pseudonymization until a sharing need existed. When the PO later said "we will want to be more invasive — I want others, who are using this skill, to contribute," that single sentence carried everything needed: the requirement (pseudonymization), the reason (public contribution), and the scope (corpus as a learning commons). The design pivoted in one turn with no clarification round-trip. Flagging the fork in the road early is what made the late answer cheap.

## 3. What the PO Could Improve

The design-changing decision arrived after implementation started, in fragments. The sequence was: (turn 3) "create a skill that sends them to [the org]" — which didn't answer the flagged anonymity question; (turn 4, as an interrupt) "I'm okay with it being public"; (turn 5, mid-implementation) "we will want to be more invasive." Each fragment was clear, but the repo had already been created private, the scrub already built to the secrets-only spec, and the backlog already scrubbed once — all of which was redone. Answering "how anonymous, and who's the audience?" at turn 3, when it was explicitly posed, would have collapsed three passes into one. The flag-early pattern only pays off if the flagged decision gets answered before the build starts.

## 4. What the Agent Got Wrong

Published 30 historical retros to a public repo on grep-level spot checks rather than a full read. The scrub caught what it was built to catch (5 internal IPs, filesystem usernames, a third-party GitHub handle), but the residue check was driven by which strings the agent thought to grep — the third-party handle was found by luck of reading one grep excerpt, not by systematic sweep. Identifying feature names (milestone codes, UI pane names) survived into the public corpus and were flagged to the PO only afterward as "worth a skim." For a one-way action — public publishing is cached and indexed even if later deleted — the diligence bar should have been a structured review of every file (or an explicit PO sign-off on a sampled review), not greps plus confidence. Secondary miss, same root: the repo was created before the visibility decision was in hand, forcing the PO to interrupt.

## 5. What Would Make the Project Better

`scrub.sh` has no self-test — and the skill system's own engineering discipline says non-trivial logic ships with one runnable check. The scrub is a security boundary (it's what stands between a leaked credential and a public repo), it has regex logic with known sharp edges (word-boundary behavior, map ordering where longer names must precede shorter names they contain, case-insensitivity), and it was validated only by running it against the live corpus and eyeballing. A fixture file with planted secrets + an assert that they all come out redacted would take minutes and would catch regressions every time the patterns are touched. Second, structural: the project map and denylist are per-machine files with a documented "copy it once" ceiling — fine for one person, but the multi-contributor vision means every contributor independently maintaining scrub hygiene with no shared floor.

## Persona Perspectives

### Security Engineer
- **User value assessment**: The two-layer design (generate clean, scrub as backstop) is the right shape, and pseudonymization at the filename level was a catch most designs miss. This protects real people — a third-party GitHub username was in the corpus and is now redacted.
- **Session assessment**: Security was load-bearing in the design from the first reply, not bolted on. Heard throughout.
- **What I'd flag**: The publishing sequence. We scrubbed, *then* published, but the review between those steps was grep-driven sampling. For irreversible disclosure, the control should be enumerative (every file, every category) or explicitly accepted by the PO as sampled. Also: the scrub regexes have never been tested against known-bad fixtures — we know they caught what was there, not what they'd miss.
- **Disagreement**: With the Project Engineer's "ship it, the scrub ran clean" framing at push time. Clean output from an untested filter is weak evidence.

### Project Engineer
- **User value assessment**: Every artifact shipped is in the user's daily path — `/retro-sync` replaces a manual SCP chore with one command. No speculative features; the fork+PR path is the only piece built ahead of demonstrated need, and it's three lines of documented `gh` usage.
- **Session assessment**: Lazy-ladder discipline held: git as transport instead of a service, perl/sed instead of a tool, one skill instead of send+gather pairs. The scrub stayed under 60 lines.
- **What I'd flag**: The README-exclusion fix landed only because writing *this retro* tripped over it. That's the class of bug a 5-minute fixture test finds before a user does.
- **Disagreement**: Accepts the Security Engineer's fixture-test point without reservation — it's cheap and the ponytail discipline itself demands it.

### Code Reviewer
- **User value assessment**: Script quality directly protects users (contributors whose data passes through it). Mostly sound.
- **Session assessment**: The scrub script grew by accretion across three revision rounds (map support, filename renames, README exclusion) without a consolidating pass. There's minor dead weight (a reassignment after rename that nothing reads) and the map's order-sensitivity (longest-name-first) is load-bearing but only documented by example, not enforced or asserted.
- **What I'd flag**: Order-sensitive config that fails silently — a user whose map lists a short name before a longer name containing it (`Foo=x` before `FooBar=y`) gets `xBar` in their public retros and no warning. Either sort the map by key length before applying, or check and warn.
- **Disagreement**: None — but notes the QA Engineer's point below is the same finding wearing a different hat, which is itself a signal it's real.

### QA Engineer
- **User value assessment**: Testing this session was live-fire only: run the scrub on the real corpus, grep the residue. That validated the happy path against one dataset and nothing else.
- **Session assessment**: The live corpus *was* a genuinely good test bed (it caught IPs, usernames, and the codename gap), but there's no regression net. The known edge cases — map ordering, case-insensitive substring hits inside longer words, multiline key blocks — are exactly the inputs a fixture file should pin.
- **What I'd flag**: The first real multi-contributor sync (fork+PR path, rebase conflicts on concurrent pushes) has never been executed. It will first run in production, by a stranger.
- **Disagreement**: With the Project Manager's "shipped complete" framing — shipped, yes; verified, partially.

### Technical Writer
- **User value assessment**: The corpus README tells a contributor what this is, how to contribute, and what not to submit in three short sections — documentation as the interface, serving real newcomers.
- **Session assessment**: Every surface got updated in-session (README tables, quick-reference, directory tree, CHANGELOG, orchestration model table). No doc debt left behind.
- **What I'd flag**: The anonymization *exception* (the skill system itself is exempt from pseudonymization) lives in `retro/SKILL.md` but not in the corpus README's contributor guidance — an outside contributor following the README alone can't learn the map file format or the exemption rule. The contribution docs describe the what, not yet the how.
- **Disagreement**: None.

### Project Manager
- **User value assessment**: High-leverage session — three durable capabilities in one sitting, each requested, none speculative. The retro→mine loop converts a recurring manual cost (hand-run rule-mining passes) into a repeatable ceremony.
- **Session assessment**: Scope grew twice mid-session (public → pseudonymized+contributable; then retro-mine), and both grafts landed without rework spirals because each was absorbed before the next build step. Acceptable for meta-work; would be a flag on a product project.
- **What I'd flag**: Everything shipped is uncommitted in the working tree. Session ends with real deliverables one `git reset` away from gone. The commit decision sits with the PO by repo convention, but it's the open risk on the board.
- **Disagreement**: With QA's severity on the untested fork path — first-contributor friction is a next-session bead, not a blocker; there are zero external contributors today.

## Lessons

- **Keep**: Flagging the design-changing decision explicitly in the first reply ("how anonymous is anonymous?"). Even answered late, a pre-named fork made the pivot one-turn cheap.
- **Stop**: Treating grep spot-checks as sufficient review before irreversible public disclosure. Sampling is for reversible actions.
- **Start**: Fixture self-tests for any script that acts as a security boundary, written in the same session the script is born — the scrub's README bug and order-sensitivity would both have been caught by the check the discipline already prescribes.
- **Value learning**: The PO's goal wasn't storage — it was *community learning*. The first framing ("single source of truth instead of SCP") undersold it; the real requirement (others contribute, so pseudonymize hard) only surfaced when the PO reacted to a design option. Options presented concretely extract requirements that abstract questions don't.
