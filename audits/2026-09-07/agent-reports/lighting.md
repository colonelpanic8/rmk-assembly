# Lighting branches: rebase and audit fixes (2026-09-07)

Source repo: `/home/imalison/Projects/rmk-assembly/.worktrees/source`. Worktrees left in place:
`.worktrees/lighting` (branch `work/lighting-refresh`) and `.worktrees/lighting-indicator`
(branch `work/indicator-refresh`).

## New heads

| Branch | Old head | New head |
| --- | --- | --- |
| `glove80-rmk/lighting-v2` (PR HaoboGu/rmk#1031) | `e909a63849c3` | `82064a6052ffc0a1012fa497b91b455a51f3c8f1` |
| `feat/lighting-layer-indicator-conditions` | `2f0104f1ea2a` | `632e7521e95629e00795b84e62559430a6495d00` |

Both rebased onto upstream `8b4d1b312d8ef2eeed55705a5f33977ab13f0a55`. The earlier Codex rebase
(stopped at commit 2/81, and it had stripped comments from the first rebased commit) was aborted
and redone from `fork/glove80-rmk/lighting-v2`. Rebase checkpoint pushed first as `e76ad42a1`.

## lighting-v2 commits added after the rebase

| Commit | Fixes |
| --- | --- |
| `3a05d456d` fix(lighting): compile without rynk | Pre-existing: `--features lighting` alone failed on two const asserts referencing `rmk_types::protocol::rynk`. Confirmed failing on the old head `e909a63` too. Asserts are now `#[cfg(feature = "rynk")]`. |
| `5c0503279` fix(lighting): evaluate compiled conditional rules against the engine policy | F14 / lighting-review F2. `RenderPolicy { output_mode, effects_enabled }` added to the compositor `RenderInput`; `Compositor::begin` takes it; `ConditionalScenes` and the runtime conditional source both read it (the runtime source no longer carries its own copy). Test `compiled_output_mode_conditions_observe_the_engine_policy`. |
| `4f1206ece` fix(lighting): let the powered-only scope select authority or local power | F15 / lighting-review F1. `LightingContext.local_powered` added next to `powered`; `KeymapLightingState` sets both from the authority's VBUS; engine helper `scoped_power` picks by `controls.powered_only_scope` for render, and `state()` reports the selected value. Test `powered_only_scope_selects_authority_or_local_power` (authority and local disagree under each scope). |
| `4a913a5cb` fix(lighting): commit persisted scene tables by generation | F13. Two shard-key generations (A = existing keys, B = new `LightingSceneShardB` / `LightingRuntimeConditionalSceneShardB`) plus commit records (`LightingSceneCommit`, `LightingRuntimeConditionalSceneCommit`: generation, len, policy, FNV-1a digest over the postcard encoding of the cells). Shard index 0 opens the generation the current commit does not name; the commit record moves last; boot loads only the committed generation and drops it on a short or digest-mismatching shard set. No commit record means the legacy header path, so tables written by earlier firmware load unchanged; the legacy headers are zeroed at first commit (the same downgrade guard the V2 conditional table already used). New keys and data variants are appended at the end of the enums. Tests `scene_rewrite_commits_a_generation_and_survives_interruption`, `runtime_conditional_rewrite_reads_only_the_committed_generation`. |
| `9b7531c43` fix(config): reject lighting configurations beyond 64 keymap layers | lighting-review F4. `lighting()` returns an error when `keymap.layers > 64`, before any wake layer is shifted. Test added to `resolves_and_validates_lighting_controls` (65 layers, wake layer 64). |
| `82064a605` docs(rynk): describe the HID transport in the protocol reference | Pre-existing: `rmk-types --features host` test `protocol_reference_is_current` failed on the old head because the generator still described the vendor bulk interface while the checked-in doc had been hand-edited to HID (`1559d128f`). Generator now emits the HID row (values checked against `rmk/src/hid.rs`); doc regenerated (padding only). |

## Indicator branch commits

Six carried commits rebased with `git rebase --onto <lighting-v2 head> b20d64db43ce`, plus:

| Commit | Fixes |
| --- | --- |
| `b668c49ce` fix(lighting): record the lingered context as frame provenance | lighting-review F3. `PresentedFrame.context` now records `snapshot.lighting_context()` (the linger-adjusted context the sources rendered from). Test `a_lingered_frame_records_the_context_it_was_rendered_from`. The linger code (`7e7ed9a3d`) exists only on this branch, so the fix could not go on lighting-v2 as the task asked. |
| `632e7521e` fix(lighting): allow the chunk-carrying command variant size | Pre-existing on the original head `2f0104f1e`: clippy `large_enum_variant` on `StandardCommand` (the layer-set/indicator fields grew the runtime conditional chunk to 382 bytes). Allowed with a comment; no allocator to box it behind. |

Formatting fixups were folded into the carried commits `71506f178` (was `7bf6292c2`; `handlers/lighting.rs`, `rynk/src/api.rs`, regenerated protocol reference) and `87d8cd846` (was `a0f2ae50e`; `rynk-wasm/src/client.rs`). That drift predates the rebase (checked with the repo `rustfmt.toml` at each original commit).

## Conflict resolutions

lighting-v2 rebase (81 commits):

- `35867342b` split app channel: upstream ungated `SleepState` and moved the sleep arm above the display arms. Kept upstream's placement and the branch's `Application` arm; dropped the now-duplicate display-gated sleep arm; dropped the `SleepStateEvent` import the branch no longer used.
- `19536a9ba` lighting system: took upstream's ungated `SleepState` over the branch's `_render_state` gate in `driver.rs`, `mod.rs`, `peripheral.rs`.
- `872846c50`, `e909a6384`: `rynk/Cargo.toml` and `rynk-usb/Cargo.toml` version pins updated to upstream's independently versioned crates (`rmk-types =0.4.0`, `rynk 0.3`); kept the branch's `lighting` feature and the HID dependencies.
- `478b7e8ff`, `81a965b23`: peripheral imports merged (`SleepStateEvent` ungated, `WpmUpdateEvent` display-gated).
- `6422bb655`: both new keymap tests kept (upstream bounds-check tests plus the branch's default-layer test).
- `18d426a2c` runtime latency: rewritten onto upstream's `update_conn_params_on_sleep_change`. The loop now `select`s the sleep event against `LATENCY_CHANGED`; a policy change while awake clears `applied` so the awake parameters are re-sent. `latency_state().effective` feeds both `default_split_conn_params` and, under `subrating`, `default_split_subrating_params`.
- `c4539169a` dead-link detection: kept the branch's derived supervision timeout (3 latency periods, 2 s floor) in a shared `awake_supervision_timeout_us` used by the plain and subrating awake parameters, replacing upstream's fixed 6 s.
- `8c3008fd6`, `94805ae62`: storage enum variant and test additions kept alongside upstream's `Flush` barrier and its test.

Indicator rebase: `33327c037` (test imports and a new test next to mine; `layers`/`indicators` fields added to the carried tests' `ConditionSet` literals), `7e7ed9a3d` (linger logic merged with the scoped-power and `output_enabled_for` helpers; the lingered snapshot is what `compositor.begin` receives together with the policy), `7bf6292c2` (V3 runtime conditional storage re-applied on the generation scheme: generation A is now the V3 shard keys, B the new key, commit path reads V3 data, legacy fallback order is commit → V3 → V2 (migrated) → V1, and commit zeroes all three legacy headers; the V3 keys are appended after the generation keys, see caveats). After the doc commit landed on lighting-v2 the indicator branch was rebased once more with no conflicts.

## Commands run (final trees)

lighting-v2 (`82064a605`):

- `cargo check --lib` for `lighting`, `rynk,lighting,storage`, `rynk,lighting,storage,split`, `rynk,_ble,split,storage,async_matrix,subrating,lighting`, `split,vial,storage,async_matrix,_ble,subrating`, `rynk,lighting,storage,split,_ble,async_matrix`: pass.
- `cargo clippy --lib -- -D warnings` for `lighting`, `rynk,lighting,storage`, `rynk,lighting,storage,split`: pass.
- nextest `rynk,storage,lighting -E 'test(lighting) | test(storage::)'`: 128 passed. Covers `rmk/src/lighting`, `host::rynk::lighting` and `host::rynk::handlers::lighting` tests, storage lighting tests, and the `rynk_lighting` integration loopback.
- nextest `rynk,_ble,split,async_matrix,storage -E 'test(storage::) | test(central) | test(latency)'`: 8 passed.
- nextest `rmk-config -E 'test(lighting)'`: 9 passed.
- nextest `rmk-types` default, `--features host`, `--features steno`: 21 / 98 / 21 passed.
- `cargo check --manifest-path rynk/Cargo.toml -p rynk-usb`: pass (this builds `rynk` too).
- `rustfmt +nightly --edition 2024 --check` on every `.rs` changed against upstream, `git diff --check`: clean.

Indicator (`632e7521e`): the same clippy set plus `--features rynk --lib` (F16), all pass; `cargo check --lib` for `rynk`, `lighting`, `rynk,lighting,storage`, `rynk,lighting,storage,split`: pass; nextest lighting|storage: 134 passed; rmk-config lighting: 9 passed; rmk-types default/host/steno: 21 / 99 / 21 passed; rustfmt and `git diff --check`: clean.

Not run / failed for environment reasons:

- `cargo check --manifest-path rynk/Cargo.toml --workspace` fails building `libdbus-sys` (no system libdbus; `nix shell nixpkgs#dbus` could not download from the `strixi-minaj:3090` substituter). `rynk-usb`, the crate `e909a63` touched, was checked directly.
- `cargo clippy --tests` on rmk (`rynk,lighting,storage`) reports a pre-existing `RefCell` held across an await in test code at `rmk/src/host/rynk/handlers/lighting.rs` (blame `5705c243fe`, the original PR). Left alone; CI's rmk clippy uses `--lib`.

## Durability of the Rynk mutation handlers (F13 follow-up question)

`scene_mutation` / `runtime_conditional_scene_mutation` return `Ok(state)` after `persist_scenes` has only queued its messages on `FLASH_CHANNEL`; the reply is queue admission, not durable success. Upstream's new `crate::storage::flush()` barrier would give a durable answer, but it is documented as single-waiter (`FLUSHED` is one `Signal`; "calls must not overlap") and is already used by `write_peer_address` on the split central, so the lighting adapter cannot call it without serializing every flush caller. Not wired; a mutex around `flush()` or a per-caller reply slot would be the follow-up. With committed generations a lost write can no longer leave a mixed table, so the reply now means "accepted and either fully applied or not at all after a reboot".

## Findings judged wrong, overclaimed, or needing a caveat

- lighting-review F3 is real but belongs to the indicator branch, not PR 1031; fixed there.
- The audit's repro `audit_compiled_conditional_scene_observes_output_mode` uses `layers`/`indicators` condition fields that exist only on the indicator branch; adapted for lighting-v2 as a two-rule test that also proves the mode switch.
- F15 is fixed in the engine and the RMK snapshot provider, but the split replica path is out of tree: the peripheral firmware that renders from `StandardReplicaState.context` must overwrite `local_powered` with its own VBUS for `PoweredOnlyScope::Local` to differ from `Authority`. Without that board change both bits carry the central's value.
- Storage key ordering on the indicator branch: the faithful rebase appends the V3 runtime conditional keys after lighting-v2's new generation keys, so they no longer sit at the indices the previously assembled build (`7bf6292c20`) wrote. A device that persisted runtime conditional rules under that build will not find them once; the host must re-send them. Scene tables are unaffected (their keys did not move). Keeping the shipped V3 indices instead would have put the indicator branch's keys ahead of lighting-v2's, which the PR could not carry.
- The task text said upstream bumped `rmk-types` to 0.3.0; upstream is at `rmk-types 0.4.0`, `rmk-config 0.8.0`, `rynk 0.3.0`, and the pins were set accordingly.
- On `subrating` builds the branch's latency policy now also drives the awake subrate request's `max_latency` and supervision timeout; previously (on the old head, before upstream added subrating) it only touched the plain connection parameters. This is a judgment call made to keep the feature effective on upstream's new structure.

## Unfinished

Nothing from the task list. Suggested follow-ups: wire a durable acknowledgement for scene persistence (see above), have the glove80 replica renderer supply `local_powered`, and fix the pre-existing `RefCell`-across-await in the handler test if `clippy --tests` is ever added to CI for rmk.
