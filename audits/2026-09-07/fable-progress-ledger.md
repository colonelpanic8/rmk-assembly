# Fable 5.1 follow-through ledger

Started 2026-09-07 (UTC evening). Owner: Paseo agent 7d712173 (claude-fable-5-1).
Scope: independent second opinion on the audit, implementation of validated
fixes on topic branches, refresh of the stack to live upstream main, and a
locally reproduced reassembly. No hardware flashing, no downstream activation.

## State discovered at handoff

- Maintenance repo master `80e2547`; lock base `f8da2742971d`, assembled
  `9a88399366a0`, tree `ad36ded1d782`. Manifest matches lock.
- Live upstream main: `8b4d1b312d8ef2eeed55705a5f33977ab13f0a55` (51 commits
  past the pinned base). Notable: storage flush barrier (`7797b0141`), oneshot
  sibling modifiers (`52444a8d6`), combo pending keys (`0f9860af4`), Unicode
  keymap aliases (`acd1254be`), Trouble role refactor (`2815178a4`), dongle
  state/events, release version bumps (rmk 0.8.3, rmk-types 0.3.0).
- A separate Astra agent (Paseo 381ac2a4) is still running and had already
  pushed fixes to six topic branches before this handoff:
  - `feat/morse-retro-tap` 804106665 (retro-tap interruption before timeout)
  - `feat/consumer-keycodes` 26a1399d2 (F5 wire tags, appends variants)
  - `fold/unicode-input` 954715911 (F6 literal reports, no held-state mutation)
  - `fold/runtime-behavior-parity` d642d0748 (auto-mouse runtime config)
  - `fold/opposite-hand-hold` 450c57951 (F20 hand policy exclusivity)
  - `feat/lighting-layer-indicator-conditions` 2f0104f1e (F16 lighting gates)
  It is mid-flight on pointing_config storage gating / lifecycle (F16b, F17)
  and PR 1031 lighting output policy + USB HID transport. I sent it a
  coordination message claiming everything else and asking it to write
  `astra-fix-manifest.md` when done.
- Topic heads that conflict with upstream main `8b4d1b3` (need rebase before
  the stack can build): split-reliability, lighting-v2 (PR 1031),
  connection-selection, split-app (PR 984), single-pass-storage-read,
  pointing-runtime-config-pinned, runtime-behavior-parity,
  battery-power-transitions, ble-name-template, ephemeral-split-layer-state,
  position-combos, nrf-half-duplex-split, persistent-layer-metadata,
  lighting-layer-indicator-conditions, ctrl-gui-swap.

## Finding dispositions (living)

| # | Disposition | Notes |
|---|---|---|
| F1 | confirmed; fixed (fork/feat/maintenance-mode 7e8ceb172): unclassified commands fail closed, `Cmd::ENDPOINTS` list + test flags any command without a policy; later topics' commands get classified in their coherence fixups | Cmd is a u16 newtype, so a compile-time exhaustive match is impossible |
| F2 | confirmed, pending fix | persistent-layer-metadata; upstream now has a Flush barrier pattern to align with |
| F3 | confirmed, pending fix | cross-topic: single-pass boot loader vs unicode-input |
| F4 | confirmed; fixed (fork/feat/morse-hold-trigger-positions 9a850ef9d): scratch buffer sized from the hold-trigger capacity; full-table store test passes at default and 160-position capacity | |
| F5 | confirmed, fixed by Astra 26a1399d2 | to be verified in rebuilt tree |
| F6 | confirmed, fixed by Astra 954715911 | to be verified |
| F7 | confirmed; fixed (fork/fold/pointing-runtime-config-pinned 47b02afc7): caret/keypad taps are queued on a virtual-key channel the keyboard loop selects on and processes through the normal key-action path (held keys, modifiers, swap all compose); pointing no longer builds keyboard reports | consumer/system usages also go through the keyboard's action path |
| F8 | confirmed; fixed (fork/feat/position-combos cd4c6bf7f, rebased on 8b4d1b3): empty definitions persist as the legacy empty combo and any empty stored combo restores as vacant; 55 combo tests pass | |
| F9 | confirmed; fixed (47b02afc7): full-candidate validation before any mutation (capacity, unique devices/overrides, declared devices, layer bound, positive thresholds); rejected writes keep the revision; scroll/sniper/caret arithmetic saturates | |
| F10-F12 | confirmed by source reading, pending fix | nrf-half-duplex-split |
| F13 | confirmed, pending fix | PR 1031 scene persistence generations |
| F14 | confirmed, pending fix | PR 1031 compiled conditional scenes |
| F15 | confirmed, pending fix | PR 1031 powered_only_scope |
| F16 | half fixed by Astra 2f0104f1e; pointing half in flight (Astra) | |
| F17 | in flight (Astra) | |
| F18 | confirmed as an ASSEMBLY defect: the guard exists on every pointing topic; rerere pair 87fc5636 (entry fold/pointing-runtime-config-pinned) dropped upstream's init-wait loop. Topic now carries a parking test (47b02afc7); the resolution is corrected during the rebuild | |
| F19 | confirmed; fixed (47b02afc7): per-device register of device-owned held buttons; keyboard mouse reports and every pointing processor OR the union in | bounded fix; per-device transient state for the all-devices processor left as-is |
| F20 | Astra's 450c57951 did NOT pass the audit's exact chain (`new(Some(false)).with_opposite_hand_hold(None)` still lost the explicit false). Fixed on top (fork/fold/opposite-hand-hold 37d88e42f): an absent option now leaves the packed hand policy untouched; rmk-types Morse tests + 11 hand-policy scenarios pass | |

## Delegation

Sol (codex default account) hit its usage limit before any agent produced
output; the user's own `codex-colonel` account only offers Terra/Luna/5.5, so
the six branch-owning agents run on `codex-colonel/gpt-5.6-terra` (xhigh for
transport, pointing/HID ownership and lighting persistence; high for the
rebase-heavy groups). `codex-ben` has Sol but is another person's account and
was not used. Astra (Paseo 381ac2a4) stopped after its manifest.

| Agent | Branches | Findings |
|---|---|---|
| 621ffeda | feat/nrf-half-duplex-split | F10 F11 F12, transport secondary 4/5/6 |
| c81e7638 | single-pass-storage-read, persistent-layer-metadata, ble-name-template, runtime-behavior-parity | rebases, F2 |
| 932cbf0c | position-combos, ctrl-gui-swap, maintenance-mode, morse-hold-trigger-positions | rebases, F1 (exhaustive gate), F4, F8, combo-replace |
| 9989b83c | split-reliability, connection-selection, split-app (PR984), battery-power-transitions, ephemeral-split-layer-state | rebases, GATT battery unavailable |
| d84eb41f | glove80-rmk/lighting-v2 (PR1031), lighting-layer-indicator-conditions | rebases, F13 F14 F15, lighting F3/F4 |
| 7d15b7b4 | fold/pointing-runtime-config-pinned | rebase, F7 F9 F18 F19, numeric validation |

Fable keeps: retro-tap combo-interception gap (morse-retro-tap), all
coherence fixups (F1 cross-topic gates, F3 unicode boot arm, F17 pointing boot
arm, exhaustive-gate classification for later entries), the F18 rerere
correction, base bump, prune, build, locked reproduction, commits.

## Secondary findings

- Retro-tap under an incomplete combo (input-review-additional #9): REFUTED. The Morse key's release runs `dispatch_combos` first, which dispatches the waiting combo key and demotes the candidate before the release is evaluated. Scenario `retro_tap_suppressed_by_incomplete_combo_press` passes unchanged on fork/feat/morse-retro-tap (797aa2a61 adds it as a guard).
- ctrl-gui-swap rebased onto 8b4d1b3 (2ecacafb4, pushed); swap scenarios pass.

## Work log

- Read AGENTS.md, pinned fork-fold guide, all audit reports, regression patch.
- Verified live upstream OID and per-topic base conflicts (merge-tree).

## Second opinion on the architecture proposals (draft, refined at the end)

1. One report owner per HID interface: agreed in substance, but the full
   event-sourced reducer is more than the bugs need. Implemented the bounded
   form: pointing taps become virtual key events consumed by the keyboard loop
   (F7), device-owned mouse buttons live in one register every full report ORs
   in (F19), and Unicode emits literal protocol reports without touching held
   state and then restores the ordinary report (Astra, F6). Active-instance
   separation for combos/Morse (input-review proposal 2) is not done; see F8
   notes for the replace-while-active limitation.
2. Declarative protocol schema with stable wire tags and access classes: the
   wire-tag half is solved by appending variants (F5) plus an old-fixture test;
   the access-class half cannot be a compile-time exhaustive match because
   `Cmd` is a `u16` newtype, so the implemented shape is fail-closed
   classification (`maintenance_policy -> Option<bool>`, unclassified = locked)
   plus a generated `Cmd::ENDPOINTS` list and a test that fails on any
   unclassified command. That gives the same guarantee one step later (in the
   test run of the assembled tree) without rewriting the endpoint macro.
3. One restore/commit lifecycle: partially. Boot restoration of Unicode mode
   (F3) and pointing config (F17) is being re-homed into the single-pass loader
   via coherence fixups; tombstone normalisation (F8) and validation before
   mutation (F9) landed on their topics; multi-record commits with generations
   (F13) are on the lighting agent. A general typed validate/normalise/apply
   framework is not introduced: each topic is still a minimal upstream diff and
   a shared framework would make every topic depend on one more.
4. Split phase owner with generation/ACK: agreed; delegated to the transport
   agent with that exact design (F10-F12 plus stale-force/reset/advertising
   items). Static reasoning only; no cable measurement was possible here.
5. One lighting evaluation snapshot: agreed for F14/F15 (engine passes mode
   and power context to compiled sources); delegated to the lighting agent.
6. Whole-board resource validation before codegen: agreed as direction; only
   the pieces tied to findings (pointing config, hold-trigger capacity,
   lighting layer bound) are being done. Timer/PPI allocation for multi-port
   nRF and Unicode index validation remain open (recorded as limitations).

## Assembly refresh (2026-09-07, later)

- Base repinned to live upstream main `8b4d1b312d8ef2eeed55705a5f33977ab13f0a55`.
- All 15 base-conflicting topics rebased and pushed (see `agent-reports/`):
  lighting-v2 (PR 1031, now 82064a605), lighting-layer-indicator-conditions
  (632e7521e), nrf-half-duplex-split (f46e7337d, with F10-F12 and transport
  items 4/5/6), persistent-layer-metadata (48947f8d4, F2), ble-name-template,
  runtime-behavior-parity, battery-power-transitions, ephemeral-split-layer-state,
  split-reliability, connection-selection, split-app (PR 984, one commit dropped
  as obsoleted by upstream 9447a6b12), single-pass-storage-read, position-combos
  (cd4c6bf7f, F8), ctrl-gui-swap, pointing-runtime-config-pinned (47b02afc7).
- Topic fixes on top: maintenance-mode ae81411c1 (F1 fail-closed policy),
  morse-hold-trigger-positions 9a850ef9d (F4), opposite-hand-hold 37d88e42f
  (F20), morse-retro-tap 797aa2a61 (guard scenario), pointing-mode-event-priority
  6d64cff13 (F18: this topic is what removed upstream's parking loop; restored).
- Build: 35 entries (final-formatting-coherence removed: its content is now
  produced by per-entry formatting; ble-name-template fixup removed: already on
  the rebased branch). New fixups: unicode-input (F3 boot arm),
  lighting-layer-indicator-conditions (Advanced endpoint policy). Re-captured:
  runtime-behavior-parity (adds auto-mouse test `buttons`, keeps unicode arm),
  maintenance-mode (changelog line, dedupe), position-combos (PositionCombo boot
  arm + vacancy, F8 test on the single-pass loader, dedupe), nrf-half-duplex-split
  (split command policy), persistent-layer-metadata (GetLayerMetadata policy).
  Standalone patch runtime-lighting-wake-layers regenerated against the new tree.
- First assembled commit `28021b81638002ffb63265a6ed2587e2c69cabbf` (tree
  `886c6f1dd84a`), then feat/device-data bb4e94249 (type alias for the audit's
  pre-existing clippy `type_complexity` lint) was repinned. FINAL assembled
  commit `56927cd1bb6492d2b8138e8f56fc1a60522364e8`, tree
  `fc0b328b46250c10babc83c3f59b00db0b656725`; `build --locked` reproduced the
  tree exactly (commit f8df003b2). `prune --dry-run`: nothing to prune.
- Final-tree verification (detached checkout of 56927cd1): `cargo check --lib`
  for `rynk`, `rynk,lighting`, `rynk,storage`, no-default (F16 closed); clippy
  `-D warnings -A clippy::new_without_default` on `rynk,storage,lighting,split`
  clean; nextest over maintenance/unicode/boot/combo/pointing/mouse/lighting/
  storage/split/morse/keyboard/rynk filters: 710 passed, 1 failed (the
  pre-existing lighting extended-conditional test under `_ble`); the audit's
  cross-topic `audit_unicode_swap` scenario passes when dropped into the tree.
- Storage ordinals: every StorageKey/StorageData variant present in the previous
  assembly keeps its ordinal; the new lighting generation keys and the indicator
  branch's V3 keys sit at the end (V3 rules persisted by the previous build must
  be re-sent once, as the lighting agent noted).
- Known boundary incoherence (pre-existing): entries 10-12 replay old pairs that
  leave `rmk/src/keyboard/morse.rs` unparsable until entry 13 repairs it.
- Known topic-side test failure (pre-existing on lighting-v2, not an assembly
  regression): `host::rynk::handlers::lighting::tests::scene_endpoints_flow_through_adapter_and_engine`
  fails with `InvalidRequest` on the extended-conditional commit when `_ble`
  is enabled; passes on the lighting agent's `rynk,storage,lighting` set.
