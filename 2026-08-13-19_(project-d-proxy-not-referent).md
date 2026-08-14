# 2026-08-13-19 — project-d-proxy-not-referent

- **ModelID**: claude-opus-5
- **TurnCount**: 1872 user and assistant messages
- **SessionDepth**: deep (compacted once)
- **Personas Active**: Project Engineer, Security Engineer, Code Reviewer, QA Engineer, Technical Writer (dispatched as write and review agents); Codex as an external adversarial reviewer
- **Beads Touched**: full-backlog validity audit (194 items reviewed, 44 closed as already-shipped), then a security-hardening batch of nine shipped changes (identifiers omitted from the public retro)
- **Scope note**: This retro was requested by the PO specifically as an error analysis — *"You've had a lot of mistakes as of late; I'd like a retro as to why."* It is weighted accordingly. The agent-error section is the substance; the rest is context.

## User Value Delivered

Three distinct programmes landed.

**A backlog validity audit.** The tracked backlog had drifted to 194 items. Fanning out review agents across the whole backlog found 44 items describing work that had already shipped and was never marked closed. Root cause: work shipping against a parent epic without closing its children. The backlog ended around 150 real items, with epics created and children reparented, and a standing rule added to the project instructions so the drift does not recur.

(That figure was itself checked while writing this retro, and the first draft got it wrong — it said 46. 56 items closed on the audit day, but the surplus were ordinary PR-merge closures from the same day's shipping, not audit findings. The verified count is 44, matching the approved closure batches of 31 + 10 + 2. A retro about mistaking a proxy for the referent should not quote a same-day close count as an audit count.)

**A security-hardening batch.** Nine changes shipped, each verified at the container-registry publication marker rather than at CI run status. The batch closed: an entire TLS/certificate route module that had thirteen routes and zero authorization dependencies; two API endpoints returning notification credentials in plaintext to any authenticated caller; several endpoints whose admin check admitted a machine service principal that should never have held write authority; and a repository guard against committing multi-gigabyte binary scratch artifacts (3.1 GB of them were sitting untracked in the working tree, including an encrypted backup envelope that a single `git add -A` would have published).

**External adversarial review.** Codex was installed as a plugin with a stop-time review gate and used as a second reviewer. It found a P1 the agent's own authorization-focused audit had never looked for, and — when the PO redirected it to audit *the agent's judgment* rather than the code — four errors in the agent's own closures and claims.

**Branch protection.** A CI job hosting six documentation and policy gates was non-required and therefore advisory. A contract test landed first to pin it, then it became the eighth required status context.

## What We Did Well Together

The single highest-leverage moment was the PO's correction *"No. Codex should audit YOUR work."* The agent had scoped the external audit to the last shipped change — reviewing code, which was already the best-reviewed artifact in the session. The PO redirected it at the layer nobody was checking: the agent's own verification claims and closure judgments. That reframing found four real errors within one pass. It is the clearest instance in this corpus of the PO correcting not *what* was being checked but *who* was checking the checker.

A second good moment: when the agent curated evidence for Codex and Codex returned a confidently wrong factual conclusion, the PO said *"I'd like you to feed Codex all the information it needs to test your premises."* That is the right instruction, and it identified the failure precisely — the reviewer was wrong because the agent had biased its inputs by omission, not because the reviewer was weak.

The PO also overrode a recommendation on the merits. The agent proposed leaving certain machine-principal write operations denied; the PO said *"Fix those tools to stop returning 403."* That was correct — the deny was a symptom of the authorization model being wrong, not a safe default worth preserving.

## What the PO Could Improve

Little to flag on process. Two observations offered honestly:

- The session ran very long and covered four unrelated programmes. The agent's error rate was visibly worse in its second half. Splitting an audit, a shipping batch, a tooling integration, and a governance change across sessions would have cost nothing and given each a fresh context.
- Several decisions were left open across the whole session (an artifact-retention question, a documentation-vs-close question, a UI-indicator question). Batching these into one decision block earlier would have cleared them; the agent should have surfaced them as a block rather than trickling them.

## What the Agent Got Wrong

Twelve errors. None reached users; all were caught. But several were caught only because the session happened to include a deliberate audit, which is not a control.

### Instrumentation trusted without testing — 5

1. A CI monitor used a `--json` flag that does not exist on the command. The error was swallowed. The agent reported nothing for thirty minutes while the checks it was watching had already finished.
2. A second monitor used `|| echo '[]'` as a fallback, converting every failure mode into "no results."
3. A third monitor took element `[0]` of a branch-filtered API listing on the assumption it was newest-first. It was not. The agent reported on a commit six weeks old.
4. A fourth monitor was armed with a five-minute timeout for a publication pipeline that takes about fifteen.
5. An external-reviewer invocation passed the prompt as an argument *and* piped empty stdin. The tool blocked waiting to append a stdin block, then exited 0 with no output. Exit status 0 was very nearly read as "reviewed, no findings."

All five have the same shape: an instrument was built, armed, and believed, without ever being run against a known-good and known-bad input. Silence from a broken monitor is indistinguishable from silence from a healthy system, which makes this class self-concealing.

### Asserting from a proxy instead of the referent — 4

6. An item was closed on the claim that its fix "rides with" another tracked item. The agent never opened that other item. It was about removing an unrelated navigation link and covered none of the work. Four stale end-to-end assertions remained live. **Reopened.**
7. A documentation sweep for a now-incorrect required-check count grepped the *word* "seven" and reported none remaining. Three files used the *numeral* `7`.
8. An item was closed as "already fixed" when the evidence supported only "unreproducible." The mechanism that would have caused the defect is absent from the current code; that is not the same as the defect having been fixed, and the distinction matters for whether it can return.
9. A stale-base merge was described as "reverting 141k lines." A three-way merge preserves work that exists only on the target branch. The claim overstated the damage and would have driven the wrong remediation.

Each of these involved a real check — just a narrower one than the sentence reporting it implied. The bead *identifier* stood in for its contents. The *word* stood in for the number. *Mechanism absent* stood in for *defect fixed*. Elsewhere in the session, a CI *run status* stood in for the *registry publication marker* — that one the agent had already learned and was doing correctly, which shows the correction is learnable and was simply not generalized.

### Process — 3

10. A full backend suite was launched for verification, and then two write agents were dispatched into the same working tree while it ran. The result was uninterpretable and had to be discarded and re-run on a quiet tree. The project's own standing instructions warn about exactly this for shared resources.
11. Evidence for the external reviewer was curated to backend routers only. The reviewer concluded the machine-facing tools did not exist. An engineer caught it by reading the directory the agent had omitted. The agent's stated mitigation — "avoid biasing by omission" — was a resolution, not a mechanism, and it did not hold.
12. An engineer was briefed to use an admin dependency that, by construction, admits the machine service principal. Had they followed the brief, the hole would have stayed open. They caught it and built a correct human-only dependency instead. The agent gave a wrong instruction inside its own area of claimed expertise, having already documented that exact trap earlier in the same session.

### The pattern underneath

The errors are not distributed across the work. The shipped code was clean and the write agents performed well. The errors cluster almost entirely in the agent's **verification and reporting layer** — which is precisely where an orchestrator's value is concentrated. The agent was applying a rigorous standard to others' work (independently re-running claimed gates, refusing agent self-reports) and a much looser standard to its own greps and closures. It checked the checkers and did not check itself.

Confidence language is the transmission mechanism. "Verified." "None remain." Those were true of what was run and false of what they implied, and the PO calibrates trust on the phrasing.

### The uncomfortable finding

**The error rate dropped sharply the moment anything else was checking the agent.** Before the external reviewer was in the loop, the agent was the sole verifier and committed most of these. Afterward, the agent caught an engineer's stale marker, an engineer caught a gap in the agent's own change, and the reviewer caught four agent errors. The most valuable act of the session was making the agent auditable — and the agent only did it because the PO asked.

## What Would Make the Project Better

1. **Smoke-test every instrument before arming it.** One call against a known-good input and one against a known-bad input. Applies to monitors, watch loops, and external-tool invocations. Cheaper than any of the four recoveries it would have prevented.
2. **Report the check, not the conclusion.** "Grepped `seven` across `docs/`" instead of "verified." If the reporting sentence is broader than the command that was run, the command was too narrow. This makes narrow checks visible instead of invisible.
3. **Never assert a handoff without opening the target.** Closing item A because item B covers it requires reading B. No exceptions; this produced the only reopened item of the session.
4. **Run adversarial review on the agent's own judgment by default, not on request.** Not just on the code — on the closures, the "verified" claims, and the recommendations. The stop-time gate does some of this; it should also fire at decision points, and its inputs must not be curated by the agent whose work is under review.
5. **Never verify on a tree that agents are writing to.** Already a standing rule for shared resources; extend it explicitly to the working tree during test runs.

## Lessons

- An instrument that fails silently is worse than no instrument, because it converts absence of evidence into apparent evidence of absence. Test the instrument, not just the system.
- A proxy is not the referent. An identifier is not its contents; a word is not a numeral; an absent mechanism is not a fixed defect; a green run is not a published artifact. Every one of this session's judgment errors is an instance of this single substitution.
- The verification standard an orchestrator applies to subagents is the standard it must apply to itself. Asymmetry here is invisible from the inside — the agent cannot feel the difference between "I checked" and "I checked something adjacent."
- Being audited is not a remediation for a bad session; it is a permanent control. The measurable drop in error rate once external review entered the loop is the strongest evidence in this session for making it default rather than requested.
- When a mitigation is phrased as an intention ("avoid biasing by omission"), it is not a mitigation. Replace it with a mechanism (hand the reviewer the repository, not a file list).
