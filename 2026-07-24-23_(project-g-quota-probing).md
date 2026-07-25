# 2026-07-24-23 — project-g-quota-probing

- **ModelID**: claude-fable-5
- **TurnCount**: ~60 (including 6 cron-fired status turns and 4 background-task notifications)
- **SessionDepth**: deep — vendor API surface mapped end-to-end, three probe crawlers written and run, one data-integrity audit built, cross-endpoint feasibility study for an project-f product
- **Personas Active**: Project Engineer, SRE, Database Engineer, IT Architect, Project Manager, QA Engineer, Security Engineer, Code Reviewer, Technical Writer (UX Designer marginal)
- **Beads Touched**: None

## Section 1: User Value Delivered

Real, layered value:

1. **Freshness restored**: verified the overnight changelog crawl (project-g) completed after its quota lockout, then loaded the refreshed dataset (~46k lineups, ~16.7M postal rows) into project-h's database via the existing containerized loader with an atomic table swap. The user-facing app is now current.
2. **A new entitlement dataset + integrity audit**: built a crawler for the vendor's country-entitlement endpoint (turned out to cost ~1 API call total) and an audit script that diffs the vendor's official per-country roster against our materialized data. Baseline finding: ~11.6k roster entries missing locally and ~3.6k local extras — investigated to root cause (systematic scope disagreement between two vendor endpoints, NOT replay loss on our side). That converts a scary number into a documented, evidence-backed vendor question.
3. **A decision-grade feasibility study**: measured, with ~4,300 quota-bounded API calls, exactly what an project-f product could and couldn't be built from this vendor's changelog API. Schedules: tractable (~one quota-day for the full forward window; proven by capturing 2.3M objects). Program titles: not tractable (no by-ID lookup anywhere; 0.08% resolution rate from recent-window walks against a 4.9B-event stream; maintenance alone would eat ~90% of daily quota). The PO ended the session with a clear, justified "stop" instead of a months-long crawl fantasy — that avoided cost is the biggest value item.
4. **The updateId→time mapping trick**: discovered empirically that changelog cursor positions map monotonically to event time, recovering a "start from date X" capability the API officially lacks. This reduced a projected 163-day backfill to a single evening for the schedule layer.

No production feature shipped beyond the data refresh, but every experiment ended in a documented number that changed a build/don't-build decision. That is the point of a spike.

## Section 2: What We Did Well Together

The mid-flight redirect at the availabilityday probe. The agent surfaced that the epoch walk was burning quota on dense, useless 2018-era data and framed a single crisp decision (redirect to a head-region walk vs. continue), with a recommendation. The PO answered in one word via the option picker, the epoch probe was checkpointed and stopped within a minute, and the head walk started from 820M — capturing the *useful* window instead. The pattern — agent detects mid-run that the plan is spending budget on the wrong region, quantifies the alternative, PO decides in seconds — is exactly how a spike should steer.

## Section 3: What the PO Could Improve

The lockout experiment's actual goal surfaced only at the end. The framing at kickoff was "let's see how much we can get before we get locked out" — a mechanics-framed goal — and the agent built a probe optimized for finding the wall. Over the next hour the real question emerged through successive prompts ("how long to gather all the data?", "could we build an project-f?", "let's test the titles layer", "do the Japan test"): the PO wanted a *feasibility verdict for an project-f product*, not the wall itself. The session got there, but by a path that included a deliberately-diagnostic probe whose captured data (2.3M schedule objects) was structurally unusable for the actual goal and was deleted hours later. Had the opening prompt been "I want to know if we can build an project-f from this API — start with schedules," the probe parser would have kept source IDs and timespans from page one, and the evening would have produced a reusable dataset *and* the same feasibility numbers. Related, smaller: "Lets give it a test and try and get locked out" arrived while the probe was already running toward lockout — a signal the preceding status message hadn't landed before the next instruction went out.

## Section 4: What the Agent Got Wrong

Two concrete misses:

1. **The probe parser was too minimal at the redesign moment.** When the head walk was launched (a deliberate second design pass), the agent kept the skeleton parser ({id, day, schedule-count}) from the diagnostic epoch probe. Adding source_id and timespans would have cost nothing measurable and would have made the 2.3M captured objects answer the PO's very next question ("could we build an project-f out of this data?") with data instead of caveats. The agent had already seen source_id in the sample XML when the walk started. Diagnostic framing was carried past the point where the goal had shifted.
2. **An unverified claim went to the PO as fact.** The agent stated Austria's entitlement roster listed "~157 providers" — a grep across a two-object response counted both countries' providers. It was corrected one turn later by the agent's own check, but the wrong number had already been used to motivate a hypothesis ("compacted record lost providers") in user-facing text. Cross-object grep counts on single-line XML are exactly the kind of measurement that needs verification before being presented.

## Section 5: What Would Make the Project Better

A quota ledger. The session's most repeated mental operation was hand-estimating remaining daily API quota ("~850 + 1,218 + ~150 probes ≈ …") across three concurrent consumers (two systemd crawlers, ad-hoc probes, experiment scripts). Every crawler already logs page counts; a tiny shared counter file (or a `quota` subcommand that sums today's calls from the logs) would replace guesswork, prevent an accidental starvation of the production timers, and make "can experiment X fit today?" a lookup instead of an estimate. Second, now evidence-backed: the vendor conversation (bulk seed files + quota tier) is the gating item for any project-f ambition — three separate experiments this session all terminated at that same wall.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: The crawlers and audit delivered real answers; the probe scripts were honest spikes and correctly deleted after yielding their numbers.
- **Session assessment**: Cloning the proven checkpoint/quota machinery for each new crawler was fast and safe. But we now have three near-identical copies of quota-wall handling, atomic writes, and sax parsing across scripts — the fourth copy is where a bug fix misses one.
- **What I'd flag**: The parser-minimalism miss (Section 4.1) is the engineering lesson: when a spike's goal shifts from "measure cost" to "assess the data," widen capture at the next rewrite — disk was never the constraint.
- **Disagreement**: With the Code Reviewer on urgency of deduplicating the crawler scaffold — the scripts that duplicated it are deleted; the two survivors share lineage but diverge legitimately. Extract a lib when the *next* crawler is requested, not before.

### SRE
- **User value assessment**: Quota discipline protected the production timers — the session deliberately stopped short of lockout, leaving ~700 calls so the 3:42 AM and 5:09 AM incremental runs would not stall. That protected tomorrow's data freshness for users.
- **Session assessment**: Good: checkpointed stops, background runs with notifications, no orphaned processes verified at shutdown. Bad: for ~40 minutes two walkers hammered the same endpoint concurrently while the vendor documents single-threaded changelog walking — it worked, but we bet an unknown ToS interpretation on it without flagging the risk explicitly at launch time.
- **What I'd flag**: The maintenance math is an operational red flag for any future project-f commitment: schedule+catalog deltas alone consume ~90% of the daily quota. That is not an architecture that survives contact with a second consumer of the same key.
- **Disagreement**: With the Project Manager's framing of the evening as "cheap experimentation" — it consumed ~86% of a shared production quota window. It was *well-budgeted*, but it was not cheap, and one more "let's also test X" would have starved the morning timers.

### Database Engineer
- **User value assessment**: The atomic staging-swap load worked exactly as designed — users never saw partial state during a 5.5-minute, 16.7M-row refresh. Counts were verified against crawl output post-load, not assumed.
- **Session assessment**: Sound. One note: the audit script parses per-country JSON files precisely because the monolithic map file crossed the JS max-string ceiling — the second time this ceiling shaped a design this week.
- **What I'd flag**: The 3,595 "extra" lineups the PO chose to leave unflagged in the DB are a silent-staleness risk; if guide UX ever surfaces them, revisit.
- **Disagreement**: None material.

### IT Architect
- **User value assessment**: The session's architecture output is a validated market-split design: vendor changelog for lineups/sources/schedules, bulk-seed-or-nothing for titles, third-party direct-lookup only for markets it covers (the PO correctly caught that it excludes Japan). That is a real architecture decision grounded in measurements, not vibes.
- **Session assessment**: The updateId→time discovery was the architectural pivot of the night and was validated across three endpoints before being relied upon — proper.
- **What I'd flag**: Everything now hinges on a vendor conversation (seed files + quota). Until that lands, no project-f commitments should be made; the feasibility study should be written up as a one-page decision doc while the numbers are fresh.
- **Disagreement**: With the Technical Writer below — I'd argue the retro-plus-readme is *not* sufficient capture for the feasibility numbers; they justify a standalone artifact because the vendor conversation will be conducted by a human who won't read a session transcript.

### Project Manager
- **User value assessment**: High-value evening: data refreshed, an audit capability added, and an expensive build path *closed* with evidence. Closing paths is delivery.
- **Session assessment**: The decisions-vs-status discipline worked — three explicit decision points were presented cleanly and the PO answered each in seconds ("None of the above", "Head-walk first"). The 5-minute cron loop, however, produced updates the PO was often ahead of — they were actively driving every 2–3 minutes anyway.
- **What I'd flag**: Section 3's finding is a process lesson for both sides: spike charters should state the *decision* the spike serves, not the activity ("find the wall").
- **Disagreement**: With SRE's cost framing — the quota was going to reset in 20 hours regardless; unspent experimental quota has zero salvage value. Spending 86% of a renewable budget on a decisive answer is the correct trade.

### QA Engineer
- **User value assessment**: The Japan test was a genuinely well-designed experiment: hypothesis, budget, three stages with independent measurements, quota-wall handling as a valid outcome, JSON report. Its 0.08% result is the single most decision-relevant number produced all night.
- **Session assessment**: Probe scripts got no tests — acceptable for deleted spikes. The audit script (which survives) has no test either, and its country-file regex silently skipped the NO_COUNTRY bucket until the agent noticed mid-investigation — that near-miss is exactly what a fixture test would catch.
- **What I'd flag**: The Austria grep miscount (Section 4.2) — measurement code deserves the same skepticism as production code when its output drives PO-facing claims.
- **Disagreement**: With the Project Engineer's "delete the spike scripts" hygiene — deleting the *Japan test script* discarded a reusable three-stage harness whose regeneration cost is nontrivial. Data deletion: agreed. Harness deletion: I'd have kept it.

### Security Engineer
- **User value assessment**: No user-facing security surface changed.
- **Session assessment**: The vendor API key is hardcoded in plaintext across every crawler script in the repo directory and was pasted into multiple shell commands this session. It's a metadata-read key, so blast radius is low, but it's also a *shared quota* key — leakage equals denial-of-service on our own pipeline.
- **What I'd flag**: The standing caveat that sustained crawling on this key has never been confirmed with its owner got sharper tonight: we burned 86% of a daily window on experiments. The PO declined (again) to raise it — that's their call to make, but the risk now has tonight's usage pattern attached to it.
- **Disagreement**: With the general satisfaction: everyone treats the quota as a technical constraint; it is also a *contractual signal* about intended use. I'm the only persona reading it that way, and I'd like that noted rather than averaged away.

### Code Reviewer
- **User value assessment**: Code quality served the mission — the streamed/atomic write discipline inherited from the week's earlier crash fix meant zero corrupt-file incidents across ~15 GB of tonight's writes.
- **Session assessment**: The country crawler faithfully preserved the proven patterns (map-first checkpoint ordering, quota-sleep). The visibility-net for unmodeled roster entries was a thoughtful touch. Duplication across crawlers is now real but bounded (two survivors).
- **What I'd flag**: The audit's `sort()` on `Object.entries` compares by key correctly only because country codes are uniform-prefix strings — fragile if the report shape changes. Minor.
- **Disagreement**: With QA on keeping the Japan harness: an untested, single-purpose script with a hardcoded key and hardcoded cursor constants is not a reusable asset — it's a liability that would rot. The conversation + report format is the reusable part; regeneration from that spec is cheap.

### Technical Writer
- **User value assessment**: The project-g readme got same-day updates for the country crawler and the audit baseline — including the cursor-stays-at-0 quirk that would absolutely have confused a future maintainer. That's documentation serving its actual reader.
- **Session assessment**: Strong on the surviving tools. The feasibility numbers (density curves, churn rates, 0.08%) live only in this conversation and this retro now that the probe logs are deleted.
- **What I'd flag**: Per the Architect — a one-page "project-f feasibility findings" doc should exist before the numbers go stale in memory. The vendor-conversation ask needs those numbers citable.
- **Disagreement**: I'll concede the Architect's point rather than fight it: retro + readme is 80% capture, and the missing 20% is precisely the part a human negotiator needs.

### UX Designer
- **User value assessment**: Marginal involvement, one real contribution: the "could we build an project-f?" framing kept every measurement anchored to an end-user artifact (a guide grid with titles) rather than data completeness for its own sake. The finding that placeholder schedule grids lack program refs for sports/OTT channels is a future UX problem (empty guide cells) worth remembering.
- **Session assessment**: Appropriate that no design work happened.
- **What I'd flag**: If the titles layer stays blocked, a schedule-times-only guide is a real but sharply diminished user experience — "what's on" without "what is it" tests poorly. Don't ship the halfway version without design input.
- **Disagreement**: None.

## Lessons

- **Keep**: Framing mid-run budget conflicts as single AskUserQuestion decisions with a recommendation (the head-walk redirect). Also: probing a changelog at several depths to build an updateId→time map before committing to any walk — it turned an impossible backfill into an evening.
- **Stop**: Carrying a spike's minimal-capture design past the moment its goal widens. When the question changes from "what does this cost?" to "what could we build?", the capture schema must be revisited in the same breath.
- **Start**: Writing a one-line decision charter at the top of any multi-hour spike ("this spike exists to decide X") — both parties drifted tonight because the charter was implicit. And a shared quota ledger for any key consumed by more than one process.
- **Value learning**: The PO's product interest was never "the wall" — it was "is an project-f buildable?" Mechanics-framed requests from the PO are often proxies for a product decision; asking "what decision does this inform?" at kickoff is cheap and would have changed the first hour's design.

## PO Response (post-retro)

The PO disagreed with Section 3's framing, and on review the agent largely concedes: the iterative, step-through process got further than a single large scoped prompt would have. Each probe's findings generated the next question (the titles test and the market-gap test were unknowable at kickoff), and small steps kept every pivot cheap — the mid-run redirect and the early stop were only fast because the PO was steering between short iterations rather than approving a big plan upfront. A kickoff charter stating "assess feasibility" would have pre-scoped a route through an API whose behavior was still unknown.

What survives of Section 3 after this correction: nothing on the PO's side. The durable lesson is agent-side only — when the PO's questions audibly shift from mechanics ("how much before lockout?") to product ("could we build X from this data?"), that shift is the signal to revisit what the running experiment captures. Goal drift is normal and productive in a spike; failing to re-derive design consequences from it is the miss.

- **Lesson (amended)**: Iterative beats pre-scoped for exploratory spikes against unknown systems. Keep steps small enough that the PO's next question is cheap to act on — and treat each new question as a design-review trigger for anything still running.
