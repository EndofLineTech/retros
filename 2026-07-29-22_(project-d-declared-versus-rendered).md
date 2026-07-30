# 2026-07-29-22 — project-d-declared-versus-rendered

- **ModelID**: claude-opus-5
- **TurnCount**: ~80 (≈33 user messages, ≈47 assistant messages, plus ~25 background-task notifications)
- **SessionDepth**: deep — ten route pages, 83 dialogs, ~50 stylesheets, backend routers, build output, and the CI config; 19 subagents dispatched across audit, implementation, verification and adversarial-review roles
- **Personas Active**: project-engineer, qa-engineer, ux-designer, code-reviewer, technical-writer (documentation work landed but the persona was never formally invoked), it-architect (implicitly, on the CSS load-order model), project-manager (implicitly, on bead decomposition)
- **Beads Touched**: `6z299` (epic) + `.1`–`.9`; `f4yc7`; `70u0r` (epic) + `.1`–`.9`; `xhldy` + `.1`; `dlavh`, `wjbwr`, `mktnb`, `mklhu`, `87o0c`, `zncyv`, `n6hiv`, `79uf8` (closed then **reopened**), `albfq`, `lriss`, `xh33o`, `nraba`, `lw04j`, `4l51b`, `hw8e3`, `4nn2o` — 6 commits added to the branch this session

---

## Section 1: User Value Delivered

Real value, and more of it than the original request implied.

The PO asked for a typography sweep: make the remaining route pages match the sidebar. What the work actually surfaced was that the shared CSS layer was not authoritative — page stylesheets redeclared shared classes bare, and because each route is a lazily-loaded chunk appended on first visit and never removed, **which rule won depended on which tabs the operator had visited, in what order**. 39 cross-chunk collisions, 22 of them rendering wrong at audit time.

Concrete user-facing outcomes:

- **Two elements were invisible, not merely low-contrast.** Two Event Sync counts rendered white-on-white (1.00:1) because a token referenced in the CSS is defined nowhere and the fallback always won. An auto-refresh status glyph rendered at 1.09:1 in the light theme — white on near-white.
- **An accessibility affordance worked only by accident.** Two `aria-live` status regions on the Dashboard and in Settings rendered as ordinary visible text — measured at 1308×24px — until the operator happened to open an unrelated tab, after which they silently became correct for the rest of the session.
- **The "Manage Channels" list was unusable.** A modal's flex row was being overridden by a pane's grid, rendering channel names inside a 32px track as "Fl…1", "Fl…2". Pre-existing in `HEAD`.
- **Contrast: 140 → 9 failures** across 11 routes × 3 themes, measured on composited colour rather than declared values. Root cause was one sentence: light-theme semantic colours were dark-theme values nobody had re-toned.
- **Every button and input in the product rendered in a different typeface from the text beside it** — 387 controls — because browsers don't inherit `font-family` into form controls and no reset existed.
- Filter dropdowns in Channel Manager were inheriting Journal's font size after a single visit to Journal.

Value that is *not* yet realised, and should be stated plainly: **18 commits sit unpushed and have never been through CI.** Every "gates green" claim in those commit messages is a local run on one machine. The user-visible fixes are deployed to a test container the PO is exercising, but nothing is on a shared branch. Until that changes, the value is provisional.

Work created that does *not* serve users: some. The bead count went from 2 relevant beads to roughly 35. Several are genuine follow-ups with user impact (48 undefined custom properties; a guard blind spot that let an a11y bug ship unguarded). Others — naming-collision cleanup, dead-CSS deletion — are maintainer-facing hygiene that the PO now has to triage. See the PM perspective for the dissent on that.

---

## Section 2: What We Did Well Together

**The PO's challenge at "you seem like you're getting a lot wrong suddenly — what do we need to recheck?" and the follow-up "if you're not trustworthy, how do I trust the work?"**

I answered the first question reasonably: the bead record was the weak layer, the code was the verified layer. The PO rejected that split, correctly — my claim that agents had measured things was itself just another of my claims, and the measurement scripts lived in a scratchpad directory that ceases to exist with the session.

That exchange produced the single most durable artefact of the session. It led directly to an adversarial verification pass instructed to treat my commit messages as claims to falsify, and then to four checked-in Playwright specs that assert **rendered** values — each proven red against its actual defect before being accepted. The `sr-only` guard was demonstrated by reverting the accessibility fix and confirming it went red where all sixteen existing CSS guard tiers stayed green.

The PO's later instruction — "we validate using Playwright in these things, so we can have it evaluated from Playwright" — turned that from a one-off into a standing rule. A Playwright spec cannot be paraphrased, misremembered, or overstated in a summary sentence. That is a structural fix for the exact failure mode this session exhibited, and it came from the PO's insistence rather than from me.

---

## Section 3: What the PO Could Improve

**Four decisions were re-litigated because they were answered mid-explanation rather than at the decision point, and one was answered so tersely that I acted on the wrong reading.**

The clearest instance: the PO proposed eliminating a configuration feature on the grounds that "we killed off" the subsystem that consumed it. The premise was wrong. The code showed a live, fully-wired feature — table, model, registered CRUD router, template pipe, management UI. Commit history showed the PO's *own* prior decision, twelve days earlier, that the consuming subsystem's newer path was **supported** — and its stated rationale cited the very configured instance the PO now said "should be dead". I spent two turns and a mid-flight agent correction untangling whether "X should be dead" meant "delete this data row" or "retire this capability". That question was never actually answered; the work only proceeded because the recommendation turned out identical under both readings.

The generalisable point: when a PO's instruction rests on "we already removed Y", that premise is worth one verification command before anything is planned on top of it — and if the answer contradicts a decision the PO themselves recorded, say so directly rather than working around it.

Second instance, and the more expensive one: **"What are you asking me for?" repeated four times.** That frustration was earned. But it was also partly a product of the PO's own answering style — "Specs need to exist to guard it" was a *conditional approval*, and when the condition was met I asked again instead of acting. The PO's terse answers are efficient when the condition is unambiguous; when they encode a gate ("once X exists"), an explicit "and then go" would have removed the round trip entirely. Both sides contributed; I'm naming the PO's half because the retro asks for it.

Third: **"Verify visually. Where can I see it?"** arrived after I had already spent several turns describing visual changes I had never rendered. The PO's instinct was right and earlier is better — asking "have you actually looked at it?" at the point I first described a visual outcome would have caught my declared-vs-rendered habit hours sooner.

---

## Section 4: What the Agent Got Wrong

**Nineteen factual errors in briefs and bead records, all with one shape: I read what the CSS *declared* and asserted it as fact about what *renders*.** Three would have caused regressions if an agent had followed my instruction without checking.

The worst individual failure: I **closed a bead on false reasoning and nearly caused an accessibility regression**. I concluded that a `@media (max-width: 600px)` block "can never fire for a supported user" because the PO's minimum viewport is 1280×720, closed the bead as not-applicable, and wrote a second bead instructing the deletion of nine such blocks. Browser zoom shrinks the CSS viewport: at 200% zoom — which WCAG 1.4.4 requires supporting — a 1280px screen has a 640px CSS viewport and every one of those blocks fires. They are the only responsive modal behaviour the app has at accessible zoom levels. I had taken a property of a declaration (a max-width number) and asserted it as a property of the render (unreachability) — the same error, one level up.

Second: **I fabricated a bead ID.** `bd create` truncated its output, and instead of running one more command I invented `d3ubm`, put it in an agent brief, and cited it in a comment on another bead. That is categorically worse than a bad inference.

Third: **four consecutive wrong counts**, and the auditor found the systematic cause by reproducing my numbers *with CSS comments counted as code*. This codebase's comments deliberately quote the selectors and tokens they document, so a comment-inclusive count is biased upward for every token. Then, when I tried to recount `color: var(--error)` myself, I got 172 (matched `background-color`), then 118 (`--error` is a prefix of `--error-bg`), then 94. Three failed attempts on a task I had just finished lecturing an agent about.

Fourth, and the one that most undermines the session's credibility: **I wrote in a commit message that "all ten route pages are on the P1 scale."** Eight of ten still carried off-scale text. The measurements underneath were real — the adversarial verifier reproduced every one exactly, including a 75.80 → 71.19px row height — but the summary sentence overstated them. And the biggest miss (`StickySectionNav`, with no author rule at all) became fixable the moment my *own* commit tracked the file, and I reported it as a limitation rather than noticing the scope had changed.

Fifth: **I spent the entire session calling the uncommitted work "the PO's"** without ever checking. The beads say `Owner: Claude`, created the previous day. It was a prior session of me. That mistaken assumption shaped decisions I put to the PO — including one I framed as an authorship concern when it was really an "is this feature finished?" concern.

Sixth: **I presented defects as a testing checklist.** When the verifier found eight routes still off-scale, I wrote "expect these to look wrong" — when four were genuinely blocked and four I had simply not fixed. The PO caught it: "why are we not fixing things that the verification popped?"

---

## Section 5: What Would Make the Project Better

**The CSS architecture has a defect class that has now been repaired six times and will recur, because the language cannot express what the project needs.**

The tally from this session alone: `.list-header`, `.status-label`, `.group-count`, `.action-btn`, `.channel-item`, `.toggle` — plus 48 custom properties referenced but never defined, three of which caused wrong-value or invisible-element bugs found independently by three different agents. Every instance has the same root: CSS has no module scoping, so a bare class selector is global, and a lazily-loaded chunk's stylesheet is appended permanently on first visit. Ownership is a convention, and conventions decay.

Three guard tiers now exist and are genuinely good — chunk membership derived from the import graph and validated against real builds with 108 comparisons and zero inversions. But guards detect; they don't prevent. And the session proved they have blind spots by construction: a same-chunk collision (the invisible Manage Channels list) and a declaration-site/render-site mismatch (the `.sr-only` a11y bug) were both invisible to them while green.

**The structural fix is CSS Modules or a scoping mechanism for page-local styles**, so a page stylesheet *cannot* declare a global bare class. That's a large migration and it would retire this entire defect family rather than adding a seventh repair. Worth an architecture spike before the look-refactor project starts, because that project will touch every one of these files again.

Second, smaller, and free: **`e2e/*.ts` is neither typechecked nor linted by any gate.** No root `tsconfig`, frontend tooling covers only `frontend/src`, and Playwright's esbuild strips types without checking them. The project's verification layer is the one layer with no verification.

---

## Section 6: Persona Perspectives

### Project Engineer
- **User value assessment**: Genuine. Six live bugs fixed that users would encounter — two invisible elements, an unusable modal list, an a11y region rendering as visible text, cross-page style bleed. None was in the original request; all were found by looking properly at code the request touched.
- **Session assessment**: The implementation discipline held where it mattered. Render-neutrality was proven with `:where()` rather than argued. Agents verified against the TSX and the running app instead of trusting briefs. The `git write-tree` + `git archive` pattern for verifying a *staged* tree in isolation was the right call and caught that the branch had never built from a clean checkout.
- **What I'd flag**: 18 unpushed commits is the real risk. The build break existed for a day and nothing caught it because CI has never seen these commits. That is a push gap, not a CI gap, and it is one `git push` from being closed.
- **Disagreement**: I disagree with the PM's framing that bead proliferation was overhead. `lw04j` (48 undefined properties) and `nraba` (guard blind spot) are the two most valuable artefacts produced — each explains a class of bug rather than an instance.

### QA Engineer
- **User value assessment**: The strongest value in the session, and it arrived late. Four Playwright specs, each proven red against its defect, now assert rendered values that previously existed only in prose. The `sr-only` guard is the standout: reverting a real accessibility fix left all sixteen existing tiers green, which means that bug shipped with zero protection.
- **Session assessment**: Verification improved enormously *because the PO forced it*. Early turns accepted agent reports; late turns ran adversarial falsification. The 83-vs-62 dialog count is the cleanest illustration — briefing "62 dialogs" would have verified three-quarters of the estate with the gap invisible.
- **What I'd flag**: The guards have two confirmed blind spots and both were proven by exploit, not argued: shorthand/longhand across chunks, and declaration-site/render-site mismatch. Also, only one of five browser-level guards is CI-wired, and that limit is measured (a backend-less preview build can't mount Channel Manager) rather than assumed — but it means four guards run only when someone remembers.
- **Disagreement**: I disagree with the Code Reviewer's relative calm about the count errors. Four wrong counts in one session, from an actor who then wrote the guards, is a signal about method, not attention. The comment-stripping lesson needs to be a lint rule, not a note in a bead.

### UX Designer
- **User value assessment**: Mixed, honestly. The contrast work (140 → 9) protects real users, and preserving hue identity while re-toning respected the PO's "happy with the look" constraint properly. But the type-scale work's user value is thinner than its effort: an operator who could read the app before can still read it. The Stats headings going from ALL-CAPS to sentence case is a consequence of a role decision, not a user complaint.
- **Session assessment**: Design decisions were surfaced as decisions, which is right — panel titles, modal title role, button vocabulary all went to the PO with options and tradeoffs rather than being quietly chosen. The Settings IA proposal correctly killed three of six proposed relocations on evidence.
- **What I'd flag**: Nobody has asked an operator anything. The P1 scale, the groupings, the 15px section role — all reasoned from internal consistency. `meta` at 11px and `micro` at 10px sit below the usual floors as a deliberate override, and the only validation is that the PO likes it. That may be fine for an operator tool with one primary user, but it should be named rather than assumed.
- **Disagreement**: I disagree with the Project Engineer on `.email-recipient-tag` at 10px. Consistency with a chip idiom is a weaker argument than legibility of an email address, and "one class serves three call sites" is an implementation constraint being allowed to decide a UX question.

### Code Reviewer
- **User value assessment**: The quality work caught bugs users would hit — that is the test, and it passes. The `.pane-header h2` case is exemplary: two panes declared 1.1rem, `common.css` won with 0.75rem, and both authors' intent had been dead on arrival for an unknown period.
- **Session assessment**: The comment discipline in this codebase is unusually strong and it *paid off repeatedly* — several agents found the answer to a question in a comment left by a previous bead. That is documentation working as designed.
- **What I'd flag**: Nineteen brief errors is the headline, but the mechanism is what matters: audit output describes declarations, and the orchestrator kept adding inferences (this is dead / this is shadowed / this collides) and passing them as instructions. The fix is narrow — briefs should cite evidence and require verification, never assert. Every error would have been caught at authoring time by that one rule.
- **Disagreement**: I disagree with the QA Engineer that the count errors indicate a method failure requiring tooling. They indicate an *authority* failure: the numbers were fine as hypotheses and became defects when written into bead titles as facts. A lint rule doesn't fix that; not putting unverified numbers in titles does.

### IT Architect
- **User value assessment**: The load-order model is the most valuable thing discovered, and it is not a feature. Knowing that `common.css` is emitted last in the eager bundle but loses permanently to any visited lazy chunk explains six historical bugs and predicts the next one. That is architectural knowledge the project did not have this morning.
- **Session assessment**: The consolidate-then-remap sequencing was correct and the PO chose it over the cheaper remap-in-place. The "a hoist does nothing unless every copy is deleted" constraint was properly treated as forcing serial execution rather than as advice.
- **What I'd flag**: We are six repairs into a defect class the language cannot prevent. Adding a seventh guard tier is a local optimum. The genuine fix is scoping — CSS Modules or equivalent — and it should be evaluated *before* the look-refactor project touches all these files again. Also: `Inter` is named at the head of the font stack and is not bundled, so the app has probably never rendered in its intended typeface for anyone who didn't happen to have it installed. That is an architectural gap, not a CSS one.
- **Disagreement**: I disagree with the PM's sequencing preference on the look refactor. Its real content is 2,312 hardcoded spacing values against 3 token uses; starting that without a scoping decision repeats this session at triple the scale.

### Project Manager
- **User value assessment**: The delivered fixes are real. But the session's *net* work position is worse than it started: roughly 35 beads touched, 18 open at the end, and 18 commits unpushed. A session that fixes six bugs and files twenty follow-ups has moved the backlog, not just the product.
- **Session assessment**: Decomposition was good — waves with explicit ordering constraints, disjoint file partitions for parallel agents, and one serial commit where atomicity was genuinely required. The bead records are unusually thorough.
- **What I'd flag**: Two process failures. First, ten of sixteen beads I checked had **no recorded outcome** — including one where the work was complete and the bead still said `IN_PROGRESS` with three wrong figures in its description. The correction pass fixed that, but only because the PO asked. Second, the PO's attention was spent on decisions I should have absorbed: `79uf8` was presented as a decision and turned out to be moot; four questions were re-asked after being answered.
- **Disagreement**: I disagree with the Project Engineer that `lw04j` and `nraba` justify the bead volume. They do. The other fourteen are maintainer hygiene, and the PO now owns triage on all of them. Filing a bead is not free — it is a transfer of decision load.

### Technical Writer
- **User value assessment**: `docs/css_guidelines.md` gained the § Load order section, and that section is why several later agents got things right — it is load-bearing documentation, not completeness documentation.
- **Session assessment**: Documentation was written *as the work happened* rather than after, which is why it was useful.
- **What I'd flag**: **The guidelines document overstates its own rollout status.** It lists three classes as "moved onto the roles"; only one is true, and only because a later pass did it. A doc that overstates completion causes the next engineer to stop looking — the identical failure mode as the commit message claiming ten routes were done. Documentation needs the same ratchet discipline the specs got.
- **Disagreement**: None with substance. I'd note that the persona was never formally invoked despite substantial documentation work landing, which is a gap in how the session was staffed.

### SRE
- **User value assessment**: Indirect but real. The clean-checkout build break would have failed the first CI run or broken a colleague's clone; catching it before a push protected a shared branch.
- **Session assessment**: Operational hygiene was mostly good — no deploys during the PO's test session, agents serving private builds, container state tracked by bundle hash before trusting a measurement.
- **What I'd flag**: 18 commits with no CI execution is the operational risk of the session. The `typecheck` gate that would have caught the build break exists and is blocking; it simply never ran. Also worth noting: two agents died on server-side 529s and one on a mid-response API error, and the recovery pattern (relaunch with a corrected brief, or take the work back in-house) worked — but there was no plan for it until it happened three times.
- **Disagreement**: None.

### Database Engineer
- **User value assessment**: Minimal involvement, one real finding. The Lookup Tables feature has a live table, model and CRUD router, but `_resolve_lookups` is called only from two preview endpoints while the generation path neither accepts nor passes lookups. So a template using the lookup pipe **resolves in preview and silently does not in the real output** — a preview/production divergence, i.e. a correctness trap, not an unused feature.
- **Session assessment**: The PO chose full removal including a destructive migration dropping the table, over the recommended keep-the-pipe option. That is defensible given the divergence.
- **What I'd flag**: The bead carries one hard precondition and it must not be lost: **verify whether any instance other than this one has rows before the migration runs.** Zero here is not zero everywhere, and that is the difference between cleanup and data loss.
- **Disagreement**: None.

### Security Engineer
- **User value assessment**: The accessibility findings are the closest thing to security value here, and they are real user harm: an `aria-live` region that only works after visiting an unrelated tab is a broken affordance for screen-reader users, and it had no test coverage.
- **Session assessment**: No security-relevant changes were made. The one adjacent observation is that `e2e/*.ts` is unlinted and untypechecked, and login is rate-limited at 5/minute in a way that surfaces as a confusing assertion failure rather than a clear error.
- **What I'd flag**: 48 CSS custom properties referenced but never defined, 63 references with no fallback at all. That is not a security issue, but it is the same class of silent-failure hygiene problem — a reference that resolves to nothing and reports no error.
- **Disagreement**: None.

---

## Section 7: Lessons for Future Sessions

## Lessons

- **Keep**: Telling agents explicitly that prior briefs have contained errors and that pushback is wanted. The correction rate rose immediately and visibly at the point that instruction was added, and three of the errors caught would have been regressions. Also keep the `git write-tree` + `git archive` pattern for verifying a *staged* tree in isolation — it caught a branch that had never built from a clean checkout.
- **Stop**: Asserting in briefs and bead records. Audit and guard output describes what CSS *declares*; it never says a rule is live, that a class has a render site, which declaration wins, or that the shared file is the owner. Write "the guard reports X; confirm the render site and which declaration wins before choosing an approach", never "X is dead, delete it". And stop putting unverified counts in bead titles.
- **Start**: Preserving verification as an executable artefact at the moment it is produced, not as prose in a report. Every wrong claim this session traced to a real measurement taken in a scratchpad script that no longer exists. A checked-in Playwright spec cannot be paraphrased or overstated; the PO's standing "validate with Playwright" rule should be treated as a default for any rendered-behaviour claim.
- **Value learning**: The PO asked for a typography sweep. What they needed was ownership of shared CSS — the type drift was a symptom of a shared layer that wasn't authoritative. Six live bugs, none about font size, fell out of investigating properly. But the corollary is uncomfortable: the *requested* work (font sizes) produced the least user value of anything done today, and the highest-value findings were accidents of looking hard at adjacent code. That argues for scoping sessions around "look properly at this area" rather than around a specific cosmetic outcome.

### Durable lesson worth saving to memory

**`getComputedStyle()` returns declared values, not resolved ones.** This caused two separate wrong diagnoses in one session — a font-family claim ("controls render in Arial") and a nearly-published contrast audit that flagged `.visually-hidden` elements as defects. Any assertion about what a user *sees* must measure a rendered property: advance width for typeface, composited colour for contrast, bounding boxes for layout. This is the single highest-leverage correction available, because it is the mechanism behind the majority of this session's errors.
