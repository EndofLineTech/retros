# 2026-07-21-14 — crawl-pipeline-buildout

- **ModelID**: claude-opus-4-8
- **TurnCount**: ~70 (user + assistant messages)
- **SessionDepth**: deep — spanned crawler engineering, systemd service management, quota/rate-limit handling, third-party API integration (TMS), data modeling, and a published reference artifact
- **Personas Active**: Project Engineer, SRE, Database Engineer, Technical Writer, Code Reviewer, Security Engineer, QA Engineer, IT Architect, Project Manager, UX Designer
- **Beads Touched**: None (no bd usage this session)

## Section 1: User Value Delivered

Substantial and real. The session started from a false premise the PO held ("the crawler should be running every day and it looked like it stalled / had way more data to go") and delivered:

- **Corrected a fundamental miscalculation.** The project's own README estimated a "month-long, ~287k-page" crawl. That estimate divided a sparse cursor's max *value* by page size, conflating an ID with a count. The crawl had actually already completed. Surfacing this prevented weeks of imagined waiting and a pointless "why is it stuck" investigation.
- **Built incremental daily sync** on the existing lineup crawler (the `done:true` flag had been terminal; reinterpreted as "caught up, resume from cursor").
- **Built a full second crawler from scratch** (GVD `/v1/sources`, ~110k stations) with the same quota-aware/resumable/systemd design, enabling `source_id → station name` resolution.
- **Resolved logos** the cheap way: a free local join against an existing sister-project station DB (~32k logos, zero API calls), then a targeted by-ID fetch for the ~53k gap — after correctly *rejecting* a 185M-object media crawl as disproportionate.
- **Field enrichment** directly serving the stated product goal (customers pulling local-provider data): `subtype` (HD/SD), `categories`, `xids` (incl. the TMS cross-ref key), `content_languages`, and — flagged proactively — `provider_type` (cable/satellite/OTA), which was being silently discarded.
- **A published, theme-aware data-model reference** mapping entity relationships and, critically, which dataset/field is *authoritative* for each question (with real traps documented, e.g. `source.country` lies).

This is genuine forward motion toward a real product outcome, not work-that-creates-work. The one caveat: two long jobs (lineup re-seed, TMS logo fetch) are still running at session close, and the final logo-merge + coverage report remain undone.

## Section 2: What We Did Well Together

**The scoping-before-committing discipline on the media crawl.** When the PO asked to resolve logos "for every Gracenote ID," the reflexive move was to crawl GVD's media dataset. Instead, the session measured it first: page density at five stream positions, the referenced-vs-total ratio, the subtype mix (logos were ~1.1% of ~185M objects), and confirmed no by-ID lookup existed. That evidence turned "sure, I'll crawl it" into "this is disproportionate — here's a targeted alternative," and the PO's terse "a" (scope it) plus later "we can pivot to TMS Logos" showed the framing landed. A multi-week crawl was avoided by spending ten minutes measuring instead of assuming.

## Section 3: What the PO Could Improve

**The dedicated TMS key existed the whole time and was disclosed only after a quota-contention detour.** Late in the session, after both long jobs were competing on a single API key and hitting quota walls, I presented a three-option sequencing decision (pause one job to let the other finish). The PO's response was to hand over a *separate* TMS key ("Use this key for TMS... The lineup re-seed can use the other key") — which instantly dissolved the contention. That key resolved the whole problem, and it was available all along. Had it surfaced when the TMS fetch was first designed, the entire quota-contention analysis, the sequencing decision, and one fetcher restart would never have been needed. The lesson isn't a criticism of withholding — it's that **credential/quota inventory is a first-class input to crawl design**, and volunteering "there's a second key for TMS" early is cheap insurance against an expensive detour.

Secondary: on the authorization caveat (whether sustained daily crawling of the primary key was sanctioned by the key owner), the PO answered "I don't care, install the timer." That's the PO's call to make — but it was waved off rather than resolved, and it remains an open risk (a standing daily job on a key of unconfirmed sanction).

## Section 4: What the Agent Got Wrong

**The `pkill -f` self-kill.** When restarting the TMS fetcher to fix a host-normalization bug, I ran `pkill -f gvd_tms_logos_fetch.js` — but my own shell's command line *contained that string*, so pkill killed the shell before the subsequent `rm` executed. The partial (wrong-host) output was never cleared, the relaunch resumed on top of it, and I got a mixed-host output file that needed a second, cleaner teardown (via the proper task-stop mechanism). A careless process-management move; `pkill -f` against a pattern that matches the running command is a known footgun and I walked into it.

Two more, briefly: (1) I launched the TMS fetcher as a plain session-bound background process and only made it a durable systemd service *after* the PO asked "will these survive session close?" — given the established systemd pattern for the other crawlers, long-running jobs should have been durable from the outset, not retrofitted. (2) I twice mis-estimated crawl scale from the wrong prior — assuming the sources and media endpoints would inherit the lineup endpoint's 10-objects-per-page cap; sources actually returned 1,000/page and finished in ~3 minutes, not the "multi-day" I'd initially framed.

## Section 5: What Would Make the Project Better

**A standing "endpoint capabilities probe" as step zero of any crawl.** I mis-estimated scale twice because I reasoned from one endpoint's behavior instead of measuring each. A tiny pre-flight — page size honored, by-ID lookup supported y/n, cursor density, quota error shape — takes one request per endpoint and would have produced correct ETAs from the start. Pair it with a **written key/quota/endpoint inventory** (which keys exist, which hosts they authorize, what the quota window is, whether sustained use is sanctioned). The single-key contention and the surprise second key both trace to that inventory living only in the PO's head.

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: High. Every crawler change mapped to a concrete capability (incremental freshness, per-country splits, name/logo resolution, provider type). No speculative features.
- **Session assessment**: Sound. Offline parser validation against saved XML before every multi-hour crawl was the right discipline; syntax-checks and backups preceded every edit.
- **What I'd flag**: The `pkill` incident was avoidable engineering carelessness. Also, three near-identical crawlers now share copy-pasted quota/retry/checkpoint logic — that's fine for three, but a shared module is overdue before a fourth.
- **Disagreement**: I think the SRE overweights the "retrofitted durability" criticism — shipping a working process fast and hardening it when persistence became a stated requirement is a reasonable order of operations.

### SRE
- **User value assessment**: The durability work directly protects the outcome — jobs survive session close and reboot with checkpointed resume, so the PO's data actually arrives.
- **Session assessment**: Strong on resilience: quota-aware sleep-poll, resumable checkpoints, systemd + linger, self-regulating on shared quota.
- **What I'd flag**: Durability was reactive, not designed-in — the TMS fetcher ran session-bound until the PO happened to ask about session close. A long job should be systemd-managed from birth. Also: no alerting on the running jobs; "tail the log" is the only health check.
- **Disagreement**: Against the Project Engineer — "harden when needed" is fine for a prototype, but these were multi-hour production data pulls from the first launch; born-durable was the correct default here.

### Database Engineer
- **User value assessment**: The authoritative-source matrix is the highest-leverage artifact of the session — it prevents downstream consumers from trusting `source.country` (which misreports) or looking for channel numbers in the wrong dataset.
- **Session assessment**: The materialized-map-from-changelog model is sound; keying everything on the Gracenote ID makes the joins clean.
- **What I'd flag**: "Current-state only" (history collapsed) was surfaced honestly, but it forecloses change-diffing — a plausible near-term product feature ("what changed in your lineup"). Worth a daily snapshot or event-log retention decision before it's needed.
- **Disagreement**: None material.

### Technical Writer
- **User value assessment**: The data-model reference serves a real audience (the PO building the product, and future consumers of these files). It made the have/have-not boundary legible.
- **Session assessment**: Good — the doc was updated as understanding evolved (coverage section, logo status, the gold-keys fix) rather than written once and abandoned.
- **What I'd flag**: The reference lives as a published artifact, not in the repo next to the crawlers. If the schema drifts, the doc won't travel with the code.
- **Disagreement**: None.

### Code Reviewer
- **User value assessment**: Quality discipline caught real user-facing issues (host normalization to the durable CDN; parser correctness verified against live data before committing).
- **Session assessment**: Mostly solid. Backups and offline validation before each change were exemplary.
- **What I'd flag**: The PO caught a CSS specificity bug the agent shipped (primary keys claimed to render gold but didn't — `.fieldrow .fn` outspecified `.pk`). The reviewer-in-the-loop should have been the agent, not the PO. And the `pkill` self-kill would have been caught by a moment's thought about what the pattern matches.
- **Disagreement**: I side with the SRE over the Project Engineer on durability — a claimed-but-untested behavior (gold keys) and a retrofitted-durability process are the same class of miss: shipping without verifying the property actually holds.

### Security Engineer
- **User value assessment**: Neutral-to-negative on risk posture. The crawlers work, but they run sustained automated load against third-party APIs on keys whose sanctioned-use status is unconfirmed.
- **Session assessment**: The authorization caveat was raised clearly and repeatedly — good. But it was then dismissed ("I don't care") and a standing daily timer was installed anyway.
- **What I'd flag**: A daily cron hammering a rate-limited vendor key that may not be sanctioned for this use is how keys get revoked — which would break the very product depending on them. This is a user-impacting risk dressed as a compliance footnote. The hardcoded API keys also sit in plaintext in the crawler scripts and in a public-repo-bound retro's *source project* (kept out of this retro by design).
- **Disagreement**: I disagree with the PM's likely "we shipped, great" read — we shipped on a credential whose continued validity is an unmanaged risk.

### QA Engineer
- **User value assessment**: The validate-parser-offline-before-crawling pattern prevented shipping bad extractions to hours-long jobs — direct protection of the PO's time and data quality.
- **Session assessment**: Good instinct on validation, but it was ad-hoc (one-off node harnesses), not a repeatable test.
- **What I'd flag**: There are no regression tests for the parsers. Each field addition (`provider_type`, `content_languages`) was validated once by eye against a sample and never captured as a test. The next schema change could silently break extraction.
- **Disagreement**: None.

### IT Architect
- **User value assessment**: The structure-vs-schedule boundary the session drew ("we have the channel map, not the program guide") is exactly the clarity the product needs; it prevents scope confusion with the sister metadata project (project-f).
- **Session assessment**: The pipeline shape is coherent: three changelog crawlers + a cross-system join to an existing station DB.
- **What I'd flag**: The key/quota architecture is undocumented and turned out to be wrong in my head (one key vs. the actual two). Cross-project coupling (this project reads project-f's `stations.db` and `node_modules`) is convenient but fragile — a hard dependency on a sibling repo's build artifacts.
- **Disagreement**: With the Project Engineer's "shared module later" — the cross-project file coupling worries me more than the intra-project copy-paste.

### Project Manager
- **User value assessment**: Multiple concrete deliverables, each tied to the stated goal. Little wasted motion once scope was set.
- **Session assessment**: Decisions were surfaced crisply (numbered DECISIONS NEEDED blocks) and the PO answered in kind (terse "2.", "B.", "a."), which kept velocity high.
- **What I'd flag**: Two long jobs are unfinished at session close, plus a manual merge + coverage report still owed. The session ended mid-flight; there's a real risk the follow-through ("I'll do it when you're back") is never triggered. Work in flight with no owner-reminder is how loose ends rot.
- **Disagreement**: With the Security Engineer — I'd note the key risk but I don't think it blocked shipping; the PO owns that risk acceptance explicitly.

### UX Designer
- **User value assessment**: The reframe to "customers pull data about their local providers" grounded the whole back-end in a real user journey (enter ZIP → see your providers/channels/logos). The provider-type gap I'd call user-facing: people filter by "cable vs. satellite vs. streaming."
- **Session assessment**: The published artifact was genuinely designed (theme-aware, mono-for-IDs typography, semantic color reserved for authority) rather than a default template.
- **What I'd flag**: We're one layer below the user still — there's no query interface yet. The data is right; the "customer pulls it" path doesn't exist. That's the next real user milestone.
- **Disagreement**: None.

## Section 7: Lessons for Future Sessions

- **Keep**: Validate parsers/extractors offline against saved real payloads *before* launching any multi-hour crawl; scope large crawls by measurement (page size, by-ID support, payload ratio, quota shape) before committing.
- **Stop**: Reasoning about a new endpoint's scale from a *different* endpoint's behavior; using `pkill -f <pattern>` when the running command line contains that pattern (kill by PID or via the task/service manager instead).
- **Start**: Make long-running jobs durable (systemd + resume) from first launch, not after someone asks; treat credential/quota/endpoint inventory as a required design input and write it down; capture each field-extraction validation as a real regression test.
- **Value learning**: The PO's product is the *structure* graph (postal → provider → channel → station → logo), explicitly not program listings. Provider *type* and logos are user-facing filters, not backend trivia. And a "stalled" pipeline was actually finished — always verify the claimed state against the data before investigating the imagined problem.
