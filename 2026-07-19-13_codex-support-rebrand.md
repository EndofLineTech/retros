# 2026-07-19-13 — codex-support-rebrand

- **ModelID**: GPT-5 Codex
- **TurnCount**: 55
- **SessionDepth**: deep
- **Personas Active**: Project Manager, Project Engineer, IT Architect, Security Engineer, Code Reviewer, QA Engineer, Technical Writer
- **Beads Touched**: None

## User Value Delivered

This session moved ai-agent-dev-team from a Claude Code-only installer toward a genuine dual-host tool. Users can now install the same skills for Claude Code and Codex CLI, globally or per project, receive host-appropriate instruction files, and install a shared enforcement hook for both tools. The public repository and local checkout were rebranded to reflect that wider scope. Existing installations migrate their managed instruction markers instead of accumulating duplicate blocks.

The work also exposed and corrected an important usability misunderstanding: Codex skills were installed and loaded successfully, but Codex invokes them with `$skill-name`, not Claude Code's `/skill-name`. Confirming the live session catalog turned an apparent installation failure into precise user guidance.

The work is not fully shipped yet. The latest Codex-hook changes remain uncommitted on the open feature branch, and Codex cannot mechanically enforce the ceremony gate because skill activation is not exposed as a tool call.

## What We Did Well Together

The strongest collaboration moment was the PO restarting the session and immediately reporting that the skills appeared unusable. Instead of accepting the symptom as proof of a failed installer, we inspected the resumed session record and found the team skills in the actual injected catalog. That distinguished discovery failure from invocation-syntax mismatch and produced a concrete answer: `$team-plan` in Codex rather than `/team-plan`.

## What the PO Could Improve

The PO expanded the target in several small steps—first “install for Codex,” then repository rename, then full rebrand, then documentation parity, then hooks, then Beads/project instructions. Each step was reasonable, but stating the full compatibility target up front would have reduced repeated sweeps through the same installers and README. A framing such as “make this a first-class Claude Code and Codex CLI project, including skills, instructions, hooks, docs, naming, and project onboarding” would have made the acceptance boundary visible before implementation began.

## What the Agent Got Wrong

The agent initially told the PO that a restart should make the skills available without verifying the new session's actual skill catalog or explaining Codex's invocation syntax. That answer conflated discovery with invocation. The restart did work; the missing piece was `$skill-name`. The agent should have checked the supported invocation model and explicitly documented the syntax difference during the original installer work.

A second miss was treating “install the same project for Codex” primarily as a destination-path change. Codex hooks, `apply_patch` tool semantics, hook trust, and skill invocation behavior are distinct integration surfaces. Those should have been included in the initial compatibility matrix rather than discovered through follow-up questions.

## What Would Make the Project Better

Add a checked-in compatibility matrix and an automated installer acceptance test covering both hosts. The matrix should enumerate skills, invocation syntax, global instructions, project instructions, hooks, trust requirements, subagent support, ceremony enforcement, and known host limitations. The test should validate generated Claude settings and Codex `hooks.json`, legacy migration, idempotency, and uninstall preservation. This would turn “supports both” from prose into a reviewable contract.

## Persona Perspectives

### Project Manager
- **User value assessment**: The session delivered a much broader usable surface than the original installer change, but the scope arrived incrementally and caused repeated implementation passes.
- **Session assessment**: Decisions were quick and unambiguous once surfaced. The missing artifact was an upfront definition of parity.
- **What I'd flag**: The uncommitted Codex-hook changes mean the latest value is installed locally but not yet durable in the repository.
- **Disagreement**: The engineer may view each follow-up as a natural incremental enhancement; from delivery's perspective, they form one compatibility epic that should have had explicit acceptance criteria.

### Project Engineer
- **User value assessment**: Users now get working skill discovery, instruction loading, and hook registration in both supported CLIs.
- **Session assessment**: Temporary-home tests protected real configuration during development, and the real installer was run only after those checks passed.
- **What I'd flag**: PowerShell hook changes were inspected but not executed because PowerShell was unavailable. The Codex hook configuration was structurally verified, but the non-interactive test did not exercise every live subagent payload variant.
- **Disagreement**: A compatibility matrix is useful, but executable fixtures should be the primary contract; prose alone will drift.

### IT Architect
- **User value assessment**: Separating shared policy from host-specific discovery and configuration is the correct direction and reduces divergent behavior.
- **Session assessment**: Reusing one dispatcher was better than creating independent Claude and Codex policy implementations.
- **What I'd flag**: The shared orchestration document still contains Claude-centric terminology and model assumptions. Installation parity does not yet imply orchestration-semantic parity.
- **Disagreement**: The current direct-skill layout is suitable for local authoring, but broader distribution may eventually be cleaner as a Codex plugin while retaining Claude installation support.

### Security Engineer
- **User value assessment**: Extending deterministic edit, Bash-mutation, and commit guards to Codex reduces the chance that users receive weaker protections merely by changing hosts.
- **Session assessment**: The work correctly consulted current Codex hook documentation and added `apply_patch` path parsing rather than assuming Claude tool shapes.
- **What I'd flag**: Hook trust is intentionally user-mediated, so installation alone does not activate enforcement. Path parsing and host payload differences remain security-sensitive and require adversarial review before merge.
- **Disagreement**: The PM may emphasize shipping the parity feature quickly; enforcement code should not ship without the mandatory security and code-review passes described by the project's own orchestration rules.

### Code Reviewer
- **User value assessment**: Idempotency and preservation tests protect users' existing configuration, which is more valuable than cosmetic parity.
- **Session assessment**: The implementation added focused self-checks for Codex patch handling and tested preservation of unrelated `hooks.json` entries.
- **What I'd flag**: Installer logic is duplicated across Bash and PowerShell, increasing drift risk. The new changes need review before the PR is ready.
- **Disagreement**: The engineer's smoke tests are necessary but not sufficient evidence for PowerShell correctness or every Codex hook event shape.

### QA Engineer
- **User value assessment**: The session validated the paths users are most likely to exercise: fresh install, repeated install, project install, local install, migration, and uninstall.
- **Session assessment**: Testing against temporary homes avoided contaminating user state, followed by a deliberate real install verification.
- **What I'd flag**: A live Codex run should confirm that the trusted hook blocks an onboarded root-agent `apply_patch` while allowing permitted instruction/config edits. PowerShell needs a CI runner.
- **Disagreement**: Static JSON validation is not the same as end-to-end hook execution; it should not be reported as complete runtime verification.

### Technical Writer
- **User value assessment**: The README now explains both installation layouts and the hook distinction, but it initially failed to teach the most immediate user action: invoke Codex skills with `$`, not `/`.
- **Session assessment**: Naming, clone URLs, project structure, Windows gaps, and hook limitations were updated consistently.
- **What I'd flag**: Add a “Claude Code vs Codex CLI quick reference” near installation rather than scattering differences across sections.
- **Disagreement**: Engineering documentation tends to focus on filesystem correctness; for users, invocation syntax and hook trust are the first-run experience and deserve higher prominence.

## Lessons

- **Keep**: Verify reported symptoms against live runtime state before changing the installer; the session catalog proved discovery was working.
- **Stop**: Treating cross-host support as copying files into a second directory without enumerating host-specific behavior.
- **Start**: Define and test a compatibility matrix before claiming first-class support for another agent host.
- **Value learning**: Users experience “support” through invocation and trust flows, not through filesystem layout. A correctly installed skill that the user does not know how to invoke is functionally unavailable.
