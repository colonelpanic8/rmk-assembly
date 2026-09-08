# Topic-branch refresh onto upstream 8b4d1b312 (2026-09-07)

All eight branches were rebased onto `8b4d1b312d8ef2eeed55705a5f33977ab13f0a55`
(upstream/main at the time) and force-pushed with lease to
`colonelpanic8/rmk`. Every branch is a linear chain whose merge-base with the
new base is the base itself; no working tree was left dirty.

Worktrees used: `.worktrees/source/.worktrees/<short-name>` on
`work/<short-name>-refresh`. Only those eight worktrees were touched. The three
left-over rebases from the earlier session were handled as follows:
`split-reliability` and `connection-selection` were verified by range-diff and
by reading every hunk of the conflicted file, then kept (connection-selection
needed one further fix, see below); `split-app` and the already-rebased
`persistent-layer-metadata` were reset to the fork heads and redone.

Verification commands (per branch, from the worktree root):

```sh
git diff --name-only 8b4d1b312d8e..HEAD -- '*.rs' | xargs rustfmt +nightly --edition 2024 --check
git diff --check 8b4d1b312d8e..HEAD
cargo clippy --manifest-path rmk/Cargo.toml --no-default-features --features <set> --lib -- -D warnings
nix shell nixpkgs#cargo-nextest --command cargo nextest run --manifest-path rmk/Cargo.toml \
  --config-file .config/nextest.toml --no-default-features --features <set> -E '<filter>'
cargo test --manifest-path rmk-types/Cargo.toml --features host -- <names>   # where noted
```

The raw pass/fail lines for every clippy, nextest and cargo-test invocation are
in `rebases-check-summary.log` next to this file. Not run: the full suite,
example/firmware builds, on-device tests, `dfu_split` feature sets.

## Summary

| Branch | Old head | New head | Commits | Notes |
|---|---|---|---|---|
| `feat/persistent-layer-metadata` | 1c24ffd75 | `48947f8d468914f56c04465d1cf7f2fb67411c17` | 2 | F2 fix added as its own commit |
| `feat/ble-name-template` | 168f75362 | `ba90faee22e679ccade2d55a985079c6da826a40` | 1 | `BleName` moved to `rmk_types::ble` |
| `fold/runtime-behavior-parity` | d642d0748 | `ac491eff2752ea7797d7628d01afffb0e6d74bd3` | 10 | auto-mouse fix kept as head |
| `fix/battery-power-transitions` | 2ee03c2bd | `3fecb54c08271f07098f2717bd35a7cb0bff0add` | 2 | not redundant; finding 7 is a contract gap |
| `fix/ephemeral-split-layer-state` | 003b180bf | `a28b33321d502d759a34942857d579f2a324a7ae` | 1 | |
| `fold/split-reliability` | 859a18e43 | `8158777f54ebd92a2cdd3157e52465a8b6e338eb` | 3 | earlier session's rebase, verified |
| `fold/connection-selection` | c52fba62d | `e2d265d37c6c8e4f8f43c79b2787294c44242b82` | 3 | re-adds `OutputUsb`/`OutputBluetooth` |
| `glove80-rmk/split-app` (PR 984) | db53d6ec0 | `768fb809142062e44678caa553d639b860db7140` | 4 | last commit dropped as redundant |

Nothing on any branch was judged fully redundant with upstream. The only
dropped commit is PR 984's `db53d6ec0 fix(split): preserve peripheral BLE
stack ownership`, obsoleted by upstream `9447a6b12` (see below).

## feat/persistent-layer-metadata

Commits: `fc5a21d2b feat(rynk): persist logical layer metadata`,
`48947f8d4 fix(storage): keep concurrent layer metadata reads apart`.

Conflict (`rmk/src/storage/mod.rs`, three hunks): upstream 7797b0141 replaced
`FLASH_OPERATION_FINISHED` with the `Flush` barrier and reworded the
`write_peer_address` doc. Resolution: `read_layer_metadata` placed before the
new two-line doc comment (the earlier session had spliced it between the two
doc lines), `Flush` kept and the two `ReadLayerMetadata`/`LayerMetadata`
variants appended after it, both new test blocks kept. Also folded into the
feature commit: the `return Ok(())` → `Ok(())` clippy fix in the
`SetLayerMetadata` handler (left uncommitted by the earlier session) and
nightly rustfmt of `rmk-macro/src/codegen/orchestrator.rs` and `rynk/src/api.rs`,
which did not pass before the rebase either.

Task B (audit F2): `read_layer_metadata` now takes an
`embassy_sync::mutex::Mutex<RawMutex, ()>` for the whole request/response
exchange, so a second Rynk session queues instead of resetting and stealing the
shared signal. The storage task's reply is `(layer, Option<LayerMetadata>)`
and the waiter loops until the reply names its own layer, so a reply left
behind by a caller cancelled between send and wait is skipped. No allocation,
no new statics beyond the mutex; the response signal keeps one slot.
Tests in `storage::tests` (poll-interleaving, `Waker::noop`, in the style of
`peer_address_write_waits_for_its_own_flush`):

- `concurrent_layer_reads_keep_their_own_response` (the audit repro, adapted):
  two reads for layers 0 and 1, only one request reaches the channel, the
  layer-0 reply completes only the first future, the second then issues its own
  request and completes on its own reply.
- `layer_read_skips_a_cancelled_reads_reply`: a dropped read's late reply does
  not complete the next reader.

Residual edge, by design of the layer-keyed reply: a read of layer N cancelled
after sending, followed by a write to N and a fresh read of N before the storage
task drains, could accept the pre-write reply. Only a request-sequence token
would close that; not done since the task asked for the layer index.

Write path: `SetLayerMetadata` sends `FlashOperationMessage::LayerMetadata`
and returns before the write lands, with no completion acknowledgement. This
matches every other Rynk setter in `host/context.rs` (all fire-and-forget on
`FLASH_CHANNEL`); only the BLE profile manager and `write_peer_address` use
upstream's `flush()`. `FLASH_CHANNEL` is FIFO, so a later `GetLayerMetadata`
from any session observes the write. Upstream's `flush()` does not cover it,
and it cannot simply be added: `flush()` documents that `FLUSHED` has a single
waiter slot and calls must not overlap, which two Rynk sessions would violate
in exactly the F2 way. A write acknowledgement would need the same
serialisation or a per-request token.

Checks: rustfmt ok, diff --check ok, clippy ok for `rynk,_ble,storage`, `rynk`,
`rynk,storage`; nextest 3/3 (`concurrent_layer_reads`, `layer_read_skips`,
`layer_metadata_survives_flash_map_reopen`); rmk-types `host` tests for
`layer`/`wire_values_locked` ok.

## feat/ble-name-template

Commit: `ba90faee2 Add runtime BLE name templates`.

Conflict (`rmk/src/storage/mod.rs`, two hunks): same 7797b0141 shape as above;
`read_ble_name` kept using upstream's `request_read`, `Flush` kept and
`ReadBleName`/`BleName` appended. `rmk/src/ble/mod.rs` auto-merged cleanly
over upstream's `run_ble_keyboard` signature change; reviewed hunk by hunk.

Topic fixes folded into the commit: nightly rustfmt of three files
(`rmk-macro/src/codegen/chip/ble.rs`, `rmk-types/src/protocol/rynk/command.rs`,
`rynk/src/api.rs`), and the `vial,_ble` CI set did not compile because
`BleName` lived in the rynk-only `protocol::rynk::payload::connection` module
while `ble/name.rs` and storage use it without `rynk`. `BleName` and
`BLE_NAME_MAX_LEN` now live in `rmk_types::ble` (always compiled, next to
`BleStatus`) and are re-exported from the rynk payload module, so the protocol
API and wire snapshot are unchanged. Both problems predate the rebase.

Checks: rustfmt ok, diff --check ok, clippy ok for `rynk,_ble,storage`,
`vial,_ble`, `dongle,rynk,_ble,storage`; `cargo check` of rmk-types with only
`_ble` ok; nextest 3/3 (`ble::name` tests); rmk-types `host` tests
`ble_name`/`wire_values_locked` ok; rmk-config `ble_name` validation tests ok.

## fold/runtime-behavior-parity

Commits (oldest first): 505d06d1c, dc68b5a4c, 79115dacf, 6d835ef60,
fe25108da, 5b6cc59a5, 020440461, a51115761, 2e4cc2000,
`ac491eff2 fix(auto-mouse): accept runtime configuration after empty startup`
(head, preserved).

Conflicts (`rmk/src/keyboard/auto_mouse_layer.rs`) at commits 3 and 10.
Upstream deleted `assert_action_event_subscriber_available` and its three
tests, inlining the assert into `run()` with a shorter message. Resolution:
upstream's inline assert kept in `run()`; the runtime-reload handler
(`on_auto_mouse_layer_config_change_event`) carries the same inline assert
instead of the deleted helper; the three helper tests are gone, the branch's
`runtime_reload_releases_old_layer_and_replaces_entries` and
`empty_runner_accepts_first_runtime_configuration` kept; `action_sub` is
`(ACTION_EVENT_SUB_SIZE != 0).then(..)` as the head commit intended; the
`entries.is_empty()` early park removed by the head commit stays removed.
Commit 2 differs from the original only in a context line.

Topic fix folded into `5b6cc59a5 feat(rynk): persist sparse named morse
profiles`: the `MorseProfileName` import is only used under `storage`, so the
`rynk`-only CI set failed clippy; import now gated. Predates the rebase.

Checks: rustfmt ok, diff --check ok, clippy ok for
`rynk,_ble,split,storage,async_matrix`, `rynk`, `rynk,storage`; nextest
(`runtime_reload`, `empty_runner_accepts`, `survives_restart`, `auto_mouse`)
ok, re-run after the import fix; rmk-types `host` tests
(`wire_values_locked`, `wire_frames`, `morse`, `auto_mouse`, `behavior`) ok.

## fix/battery-power-transitions

Commits: `83d85756a fix(split): keep peripheral battery state coherent`,
`3fecb54c0 feat(ble): allow omitting split battery services`.

Redundancy check: upstream 772ac2a72 (peripheral battery event to the dongle)
and #1112 only forward `PeripheralBatteryEvent`; upstream still drops the level
on unplug (`level: None` in `input_device/battery.rs`) and never invalidates a
disconnected peripheral's cached battery. Both commits remain necessary.

Conflict (`rmk/src/ble/ble_server.rs`, three hunks): upstream collapsed the six
cfg'd `Server` structs into one struct with cfg'd fields. Resolution: upstream's
single struct, with the `peripheral_battery_services` field and its import
gated on `all(split, not(_no_split_peripheral_battery_service))`. `ble/mod.rs`,
`battery_service.rs`, `driver.rs`, `peripheral.rs` auto-merged; reviewed.

Task C (transport-review finding 7): confirmed on the rebased tree. The
disconnect path publishes `Unavailable`, but `BlePeripheralBatteryServer::run`
only notifies on `Available { level: Some(..) }`, so a connected host keeps
the last percentage. No supported host-visible representation exists: the
Battery Level characteristic is a plain `u8` with a `VALID_RANGE` descriptor
of `[0, 100]`, the peripheral slots are initialised to `0`, and upstream's own
central-battery path ignores `Unavailable` the same way. Reusing `0` would be
an in-range value that hosts render as an empty battery, and anything above
100 violates the declared range, so no change was made. Closing the gap needs
either the BAS 1.1 Battery Level Status characteristic (battery-present flag)
or a vendor characteristic carrying peripheral connectivity next to the level.

Checks: rustfmt ok, diff --check ok, clippy ok for
`split,vial,storage,async_matrix,_ble,steno`, `rynk,_ble,split,storage,async_matrix`,
`dongle,vial,split,_ble,storage`; nextest 8/8 (`charge_transitions`,
`charge_state_before`, `connection_state_tests`, `peripheral_battery`,
`battery_presentation`).

## fix/ephemeral-split-layer-state

Commit: `a28b33321 fix(split): send complete ephemeral layer state`.

Conflict (`rmk/src/keymap.rs`, tests tail): both sides appended tests; all
kept. The branch's no-op `let keymap = KeyMap {..}; keymap` in
`KeyMap::build` was dropped (upstream's `layout_option` field landed in the
same literal; the let/return was a clippy `let_and_return` failure the audit
already listed). `split/driver.rs` and `peripheral.rs` auto-merged over
upstream's `sleep_sub` relocation; reviewed.

Checks: rustfmt ok, diff --check ok, clippy ok for
`split,vial,storage,async_matrix,_ble,steno` and `split,vial,storage`;
nextest 4/4 (`split_layer_state_tracks_the_complete_active_set`,
`layer_state_snapshot_round_trips_through_the_split_message`,
`legacy_layer_message_keeps_its_wire_discriminant`,
`complete_layer_state_has_compact_layout`).

## fold/split-reliability

Commits: `497384ab0`, `67c8e2773`, `8158777f5` (earlier session's rebase,
kept after verification: range-diff, every hunk of `split/ble/central.rs`
read, compiled, tested).

Conflict shape (`rmk/src/split/ble/central.rs`): upstream #1081 turned the
connect timeout into a `let connected = match ..` with a sleep-aware branch.
The topic's re-arm timeout (`KNOWN_PEER_CONNECT_REARM_MS`), the per-peer
`KnownPeerRecovery` miss counter and the rescan-after-N-misses address clear
sit inside upstream's awake branch; the asleep branch keeps addresses as
upstream does.

Checks: rustfmt ok, diff --check ok, clippy ok for
`split,vial,storage,async_matrix,_ble,steno` and
`rynk,_ble,split,storage,async_matrix`; nextest 2/2 (`known_peer_recovery`).

## fold/connection-selection

Commits: `2b44bcc37 feat(connection): implement output selection keycodes`,
`99c4dcb76 feat(ble): prefer BLE when selecting profiles`,
`e2d265d37 fix(connection): skip persisting an unchanged preferred transport`.

Earlier session's rebase verified: the `profile.rs` hunk drops the branch's
`FLASH_OPERATION_FINISHED` import in favour of upstream's `flush()` and keeps
the three `set_preferred(ConnectionType::Ble)` calls.

Extra fix required: upstream #1103 (`bbda6c439`) removed
`KeyboardAction::OutputUsb`/`OutputBluetooth` (and `DebugToggle`,
`OutputAuto`) as unimplemented placeholders, so the branch, which is their
implementation, no longer compiled for `vial,_ble`. Folded into the first
commit: the two variants re-added at the end of `KeyboardAction` (append-only,
so the discriminants upstream just re-numbered stay put), doc comments, the
`kbctrl!` macro doc list and a row each in the keycodes docs table. Names are
resolved from `strum::VariantNames`, so keyboard.toml bindings work without
parser changes.

Checks: rustfmt ok, diff --check ok, clippy ok for `vial,_ble` and
`split,vial,storage,async_matrix,_ble,steno`; nextest 4/4 (`preferred_transport`,
`usb_preference_flip`, `flipping_away`, `unchanged_status`), re-run after the
fix; rmk-types `host` tests (`wire_values_locked`, `action`) ok.

## glove80-rmk/split-app (PR 984)

Commits: `89b39a3ce split: add a bounded application-message side channel`,
`1fd38ee61 fix(split): satisfy collapsible match lint`,
`edaedf3d1 refactor(split): dedupe link guard and flatten peripheral handling`,
`768fb8091 split: size the application RX channel for burst delivery`.

Dropped as redundant: `db53d6ec0 fix(split): preserve peripheral BLE stack
ownership`. Upstream `9447a6b12 refactor(ble): transports own their BLE stack;
split managers move inside BleTransport (#1026)` already gives
`run_rmk_split_peripheral` the `controller`/`address` signature and builds the
single-link stack itself; the commit was a no-op against the old base too
(the branch's earlier commits had re-applied the pre-#1026 signature and this
commit undid that). The Trouble-roles refactor (#1106) is orthogonal: it only
changes Cargo features and the notification MTU constant.

The earlier session's rebase was discarded: its first commit had lost the
`driver.rs`/`peripheral.rs` wiring and its refactor commit re-applied the old
`stack: &Stack` signature and the pre-subrating `LeSetPhy` bounds.

Conflicts: `driver.rs` at commits 1 and 3 (upstream moved `sleep_sub` out of
the `display` cfg; the application arm stays the last outgoing arm);
`peripheral.rs` at commits 1, 3 and 5. At commit 3 the branch's hunk carried
the old function head, so upstream's `run_rmk_split_peripheral` head/body was
restored verbatim from the base tree and the `display` cfg upstream removed
from `SleepState` was dropped in the new `handle_central_message`. Commit 5
resolved to the base and skipped. The net diff against the base equals the old
net diff except for upstream's own context changes.

Checks: rustfmt ok, diff --check ok, clippy ok for
`split,vial,storage,async_matrix,_ble,steno` and `split,vial,storage`. The
branch adds no tests.

## Unfinished / caveats

- No branch was compiled as firmware or for examples; `dfu_split` arms in
  `peripheral.rs` (split-app) were only checked as host `cfg`-gated code.
- The clippy sets run are the host-checkable subsets relevant to each branch,
  not the full CI matrix.
- The residual same-layer stale-reply edge on `persistent-layer-metadata` is
  described above and left as is.
- Finding 7 (battery) is documented, not fixed.
