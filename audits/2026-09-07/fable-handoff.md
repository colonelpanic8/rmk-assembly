## Task
Take ownership of the RMK audit follow-through. The user's exact request is: "Can we hand the findings over to fable 5.1 to actually do the fixing and give a second opinion?"
Independently assess the findings and architecture recommendations, correct any overclaims, and IMPLEMENT the validated fixes. This is an execution handoff, not a request for another report alone. Continue through tested, integrated local fixes; don't stop after a plan or the easiest findings.

## Context and evidence
Maintenance repository/workspace: /home/imalison/Projects/rmk-assembly
Read /home/imalison/Projects/rmk-assembly/AGENTS.md and its fork-fold skill and pinned guide before operating on the assembly.
Primary report: /home/imalison/Projects/rmk-assembly/audits/2026-09-07/README.md
That report has 20 primary findings, secondary findings, exact source references, six architecture proposals, validation commands, and a coverage ledger for all 36 carried entries.
The same directory contains scope.json, stack-status.txt, regression-repros.patch, all test/compile/lint logs, and four companion reports:
- input-review.md
- input-review-additional.md (a delayed independent review with additional important pointing/Morse findings)
- transport-review.md
- lighting-review.md
Read the main report AND companions. The report is evidence and hypotheses, not authority; challenge severity, trigger assumptions, and architecture where appropriate.

Audited upstream base: f8da2742971d048e0080e57d69d98f00a859e4e9
Audited assembled commit: 9a88399366a01b344cf90fb3723cae68dc4dc949
Audited tree: ad36ded1d782d52aed0c5924569993f37ca91362
Scope: 36 entries, eight coherence fixups, net 205 files / +41172 -1763.
Inspect current manifest/lock/live worktree status before changing anything: other sessions may have advanced it since this snapshot. Fix current relevant topic content without discarding newer work.

## Current state
The audit made NO production fixes, commits, pushes, pins, firmware flashes, or activations.
Audits/ was untracked in the maintenance repo at handoff. Preserve its evidence.
Detached reproduction checkout: /home/imalison/Projects/rmk-assembly/.worktrees/audit-astra-20260907
It contains test-only edits in rmk/src/host/rynk/mod.rs, keyboard.rs, storage/mod.rs, lighting/standard/tests.rs, maintenance TOML scenarios, and new Unicode/wire/Morse tests. Do not turn this assembled-based checkout into a topic branch.
Detached upstream compatibility control: /home/imalison/Projects/rmk-assembly/.worktrees/audit-astra-upstream-20260907
RMK git objects are available via .worktrees/source; .worktrees/build is generated output and may be in use by other sessions. Inspect rather than resetting it.
Regression patch applied cleanly in git apply --check against the assembled tree. It contains expected-correctness tests that intentionally FAIL until the bugs are fixed.

## Findings to assess and address
F1 missing maintenance gates for SetMorseProfileEntry, DeleteMorseProfile, SetSplitTransportForce.
F2 shared layer-metadata response Signal routes concurrent callers incorrectly.
F3 Unicode mode persisted but omitted by authoritative host boot loader.
F4 hold-trigger table can exceed macro-sized storage scratch buffer under valid larger capacity.
F5 inserting brightness enum variants changes existing Serde/Postcard wire tags (old Left Control bytes become MouseAccel0; old Play becomes BrightnessMinimum).
F6 Unicode clears physically held modifier bits and Ctrl/GUI remapping changes the Linux protocol chord.
F7 pointing keypad taps send full independent keyboard reports, clearing held keyboard state (unsafe Caret helper is inherited; new use expands it).
F8 deleted empty position combo restores as occupied tagged-empty, breaking legacy getters.
F9 pointing configuration lacks structural/numeric validation.
F10 half-duplex deadline is armed before TX and too short for valid default-baud bursts.
F11 corrupt reply prematurely releases receive ownership, allowing bus collision.
F12 transport force switches central without peer application acknowledgement and suppresses write failures.
F13 scene shards overwritten in place restore mixed old/new generations after interrupted replacement.
F14 compiled output-mode conditional scenes always receive None and never match; runtime counterpart works.
F15 powered_only_scope Local is emitted/read back but has no runtime behavioral consumer.
F16 two compile regressions: Rynk without lighting has six lighting endpoints missing cfg gates; Rynk with lighting but no storage has unconditional flash imports in pointing_config.
F17 standard generated firmware never calls pointing config loader/init or instantiates PointingLayerModes.
F18 permanent sensor-init failure plus asserted motion pin can loop without yielding; upstream failure parking guard was removed.
F19 independent full mouse reports lose pad/device/Drag/Press button ownership during keyboard mouse-key or other sensor reports.
F20 with_opposite_hand_hold(None) erases explicit unilateral_tap=false in generated profiles.
Secondary: stale queued transport forces; advertising timeout ignores selector/liveness wakeups; reset retains old queues; duplicate default nRF TIMER/PPI ownership; stale GATT battery/scan-vs-GAP identity contracts; live-combo replacement strands release state; shared all-device pointing state; incomplete combo bypasses retro-tap interruption; invalid Caret threshold creates tap storms and Scroll/Sniper ratios can overflow; linger provenance mismatch; >64-layer lighting validation; quadratic lighting scans and board memory budgets.

## Validation already performed
16 failing regression tests across ten finding groups, eight passing controls (six existing + two upstream wire fixtures). Two feature cargo-check combinations fail. Details/commands in README.
Scoped Clippy fails with six diagnostics (five changed areas, one unchanged upstream Crc32 lint).
Reproduction Rust files pass nightly rustfmt, whitespace checks pass.
Use nextest for RMK host tests: mock clock explicitly rejects ordinary cargo test without process isolation. rmk-types integration tests work with cargo test.
Capacity repro needs the bundled capacity.toml via KEYBOARD_TOML_PATH.
No full suite or target hardware qualification was performed. Transport claims need focused timing/cancellation/error tests; don't characterize static reasoning as physical measurement.
Do not re-run the entire initial audit unnecessarily. Reuse evidence, independently validate the important assumptions, and run targeted tests as fixes land.

## Decisions and architecture to reconsider
Suggested priorities: ABI/maintenance/HID ownership/sensor starvation/build/lifecycle safety first, then split phases/force and storage consistency, then remaining validation and lighting integration.
Suggested architecture: one owner per HID interface with per-source contributions; explicit stable protocol/access metadata; shared validate-normalize-restore-commit lifecycle; split phase owner with generation/ACK selection; one coherent lighting snapshot for compiled/runtime rules.
Retain sound existing lighting service/output separation and topology validation. Choose bounded incremental refactors where they solve the underlying bug; don't impose a speculative total rewrite.
Resolve overlapping Morse policy options semantically before packing.

## Constraints and working method
- You are not alone in the codebase. Do not revert other sessions' edits; coordinate and adapt.
- Assembled branches are compiled output: NEVER hand-commit to them, base topics on them, or merge them into topics.
- Fix topic-local bugs on minimal topic branches against upstream. Genuine cross-topic repairs belong in the responsible entry's coherence fixup. Track conflict resolutions only through fork-fold's recorded rerere pairs.
- Use .worktrees/<task-or-branch> under the relevant repository for isolated topic work.
- Follow the pinned fork-fold guide, not recollection. Prior tool discovery: fork-fold was not on PATH; nix develop --command fork-assembler status worked.
- Rebuild/integrate using the recipe and prove locked-tree reproducibility after repairs. Commit coherent topic and recipe changes with explicit paths as appropriate. The user subsequently explicitly requested: 'fable should also do a "reassemble and get on the latest rmk upstream"'. Refresh to the latest live upstream RMK main, reconcile topics that have landed upstream, resolve resulting conflicts/coherence fixes, and reassemble with all validated repairs. This supersedes the earlier instruction to avoid base/topic updates.
- Do not flash hardware or alter installed/downstream system configuration merely as part of this repair handoff. Prepare concrete tested results before any external publication/activation decision.
- Read /srv/dotfiles/dotfiles/agents/DELEGATION.md before delegation. Prefer Paseo agents and self-contained prompts; assign disjoint ownership and review their changes yourself. Never substitute a different model for yourself silently.
- Model choice: user explicitly requested Fable 5.1; high effort is justified by independent judgment on cross-topic architecture, compatibility, concurrency, and coordinating repairs.
- Earlier Claude Opus attempt hit an account spend limit; some earlier Paseo launch calls timed out despite eventually creating agents. If a launch times out, inspect whether it created an agent before retrying, to avoid duplicates. The current requested Fable model is available in provider discovery.

## Acceptance criteria
- A disposition for every primary finding and material secondary issue: confirmed/fixed, rejected with evidence, or a precisely explained remaining limitation.
- Implement validated fixes, including architectural changes where warranted, with focused regression tests and required formatting/lint/type checks.
- Preserve upstream wire semantics or establish an explicit reviewed migration boundary.
- Integrate fixes through topics/fixups and locally reproduce the assembly; never rely on handwritten assembled commits.
- Verify the final base against live upstream main, preserve intended carried feature content, reconcile merged/redundant entries through the pinned stack workflow, and prove the refreshed assembly reproduces from its lock. Record the upstream OID and any remaining divergence explicitly.
- A final second-opinion summary explaining agreements/disagreements, completed commits, verification, and actual remaining risks. Keep a progress ledger in the audit directory so work survives long sessions. Continue executing rather than just handing back a plan.
