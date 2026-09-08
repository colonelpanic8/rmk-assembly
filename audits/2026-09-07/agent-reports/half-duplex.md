# feat/nrf-half-duplex-split — refresh and audit fixes

Agent report, 2026-09-07. Topic branch `feat/nrf-half-duplex-split` on
`fork` (colonelpanic8/rmk), worked in
`.worktrees/source/.worktrees/half-duplex` (branch `work/half-duplex-refresh`).

## Head

Pushed head: `f46e7337daa0443a3c5120db75cf393c2794a310` (confirmed by `git ls-remote fork refs/heads/feat/nrf-half-duplex-split` after the push).

Base: `8b4d1b312d8ef2eeed55705a5f33977ab13f0a55` (`upstream/main`, "Merge pull
request #1121 from rmk-rs/fix/ble-runner-reboot").

## Step 1 — the resumed rebase

The resume note said nothing had been pushed. That was stale: the
remote-tracking reflog shows `ffe45e47d` reached `fork` by push, and
`git ls-remote fork refs/heads/feat/nrf-half-duplex-split` returned
`ffe45e47d`. The previously pinned topic head (manifest lock, and
`refs/fork-assembler/feat-nrf-half-duplex-split`) is `572e4b0c0`, so the
range-diff below compares against that.

Checks made:

- `git log --oneline 8b4d1b312d8e..HEAD` listed the topic's 20 commits, in
  the original order, same subjects as `f8da2742971d..572e4b0c0`.
- `git range-diff f8da2742971d..572e4b0c0856 8b4d1b312d8e..HEAD`: 17 commits
  identical (`=`); three rewritten only where upstream context moved:
  - `b3fc23fe6 feat(split): support nRF half-duplex serial links` — the
    BLE central scan loop now waits for wireless selection and then keeps the
    upstream sleep-idle loop (#1081) and the 30 s scan timeout; the peripheral
    and split-peripheral imports pick up upstream's `subrating` gating
    (#1008/#1077). The topic's `wait_wired_selected` arms are kept in the
    connect, all-connected, and scan waits.
  - `3f3d1328a feat(split): host-forceable transport selection` — the
    manager's `select_biased` arm order follows upstream's move of the sleep
    arm.
  - `4ec9ba6be fix(split): drain parked transports and revert dead forced
    links` — same subrating import context in `ble/peripheral.rs`.
- Read the resulting hunks in `rmk/src/split/ble/central.rs`,
  `rmk/src/split/ble/peripheral.rs`, `rmk/src/split/driver.rs`, and
  `rmk/src/split/peripheral.rs`: both the topic's intent and upstream's
  changes are present.
- `cargo check --features split,storage --lib` on `ffe45e47d`: pass.

No checkpoint push was needed; the remote already held `ffe45e47d`.

## Step 2 — commits

All on top of `ffe45e47d`, in this order.

1. `4841fa376 fix(split): budget half-duplex reply deadlines from the baud rate` — F10 and F11.
   `HalfDuplexTiming::from_baud` derives, from 8N1 byte time and
   `SPLIT_MESSAGE_MAX_SIZE`: a first-frame allowance (poll bytes in flight +
   2 ms reply gap + one frame + 3 ms slack), a per-further-frame allowance,
   and a burst cap (a full `FRAMES_PER_EXCHANGE` window plus terminator). The
   allowance starts after the poll's write returns. A dropped exchange leaves
   `reply_pending` set so the next exchange listens the bus quiet before
   driving it. `listen_reply` skips a corrupt frame and keeps listening until
   the terminator or the burst cap; a transport fault is drained the same way
   and returned only afterwards. The baud reaches the driver through a new
   `baud` parameter on `run_half_duplex_peripheral_manager` and
   `run_auto_half_duplex_peripheral_manager`, generated from
   `serial[].baudrate` with `HALF_DUPLEX_DEFAULT_BAUD` (115200) when unset.
   Tests: `slow_baud_reply_burst_is_received_in_one_exchange`,
   `silent_peripheral_bounds_the_listen_phase` (mock-clock wire model at
   9600 baud), `corrupt_reply_frame_keeps_the_listen_phase_open`,
   `transport_fault_is_reported_after_the_bus_goes_quiet`. All four were
   run against a temporary re-implementation of the old semantics and failed
   there.
2. `9f6e50661 fix(split): apply a transport force only once the peripheral acknowledges it` — F12 and transport-review (3), (4).
   `selector::request_force` records `(generation, mode)` in a `Watch`;
   `request_transport_force` applies locally only with no peripheral
   connected. The manager sends `TransportOverride { generation, mode }` at
   session start (initial sync) and on every new request, arms a 5 s
   `FORCE_ACK_TIMEOUT`, and calls `set_forced` only on
   `TransportOverrideAck(generation)` matching the current desired
   generation; a stale ack is ignored; timeout logs and leaves both halves
   alone. Write errors on this path are logged rather than treated as
   delivery. The peripheral acks, then `flush`es (new `SplitWriter::flush`
   default no-op): the half-duplex peripheral serves polls until the ack is
   link-acknowledged; the BLE peripheral waits 500 ms for its notification
   to go on air. `SPLIT_TRANSPORT_FORCE_CHANNEL` is removed; the Rynk
   handler no longer returns `NotReady`.
   Tests (`split::driver::force_tests`):
   `force_is_applied_only_after_the_peripheral_acknowledges_it`,
   `force_is_not_applied_when_the_acknowledgement_never_comes`,
   `acknowledgement_of_a_superseded_generation_is_ignored` (also covers
   re-send on reconnect), `force_applies_locally_at_once_without_a_peripheral`;
   serial: `peripheral_flush_waits_for_the_control_frame_to_be_acked`,
   `peripheral_flush_gives_up_on_a_central_that_never_acks`.
3. `db2abbc6d fix(split): report a peripheral connected only once its session is usable` — transport-review (4)'s "define connected".
   The half-duplex central driver takes the slot id, reports connected on the
   handshake echo, and disconnected after `PEER_LOST_EXCHANGES` (64)
   unanswered polls, at which point it also renegotiates the session. The BLE
   session reports connected after discovery, right before the manager
   runs. Without this, the ack design would have removed the only way to
   steer a central alone on a dead wire (the manager runs whenever wired is
   selected). Test: `wired_peripheral_is_connected_only_while_it_answers`.
4. `f5cac02ff fix(split): start every wired session from empty queues` — transport-review (6).
   `LinkEndpoint::clear_queues` drops lanes and inbox; both drivers'
   `begin_session` call it (the peripheral loop now calls `begin_session`
   before `run`), and the peripheral also clears on the central's reset
   poll. The doc comment states that nothing survives an epoch. A peripheral
   flush that observes a reset reports failure. Tests:
   `session_start_drops_the_previous_sessions_queues`,
   `reset_poll_drops_the_peripherals_stale_queues`,
   `peripheral_flush_fails_when_a_reset_drops_its_frame`.
5. `f46e7337d fix(split): wake a parked BLE peripheral on selector changes` — transport-review (5).
   The selector and forced-BLE liveness logic live on this topic
   (`3f3d1328a`, `4ec9ba6be`), so fixed here: the post-timeout park is a
   `select4` of keyboard input, pointing input, the new
   `selector::wait_selection_changed`, and `forced_ble_liveness_fallback`.
   No unit test: it needs the trouble BLE stack.

`Cmd::SetSplitTransportForce` is dispatched at `rmk/src/host/rynk/mod.rs:129`
on this branch; its handler is `impl Handle<SetSplitTransportForce>` in
`rmk/src/host/rynk/handlers/connection.rs`. `mod.rs` was not edited (the
maintenance gate belongs to another topic); the handler only lost its
`NotReady` branch.

## Findings challenged

- F10, F11: confirmed against the code as described. The old code armed one
  5 ms deadline before its own writes; `FRAMES_PER_EXCHANGE`'s comment claimed
  the opposite. At 115200 baud one full frame is 87 µs × `SPLIT_MESSAGE_MAX_SIZE`.
- F12: confirmed. One consequence the audit does not mention: the old
  "switch after send" was also the only escape for a central that is alone
  on a dead wire, because the wired manager reported the peripheral
  connected whenever wired was selected and a force therefore never applied
  locally. Rather than switching on timeout (which the task rules out), commit
  3 makes "connected" mean an answering session, so that case takes the
  local-apply path.
- Transport-review (4), (5), (6): confirmed. (5) is on this topic.
- No finding was judged wrong. One design limit worth stating: the BLE
  peripheral's `flush` is a timed settle, not proof of delivery. If the ack
  is still lost, the central stays and the peripheral switches; the
  peripheral's existing liveness fallbacks (`forced_wired_liveness_fallback`,
  `forced_ble_liveness_fallback`) revert a dead forced link within 4–8 s, and
  a host retry then hits the no-peripheral local-apply path. During that
  500 ms the BLE peripheral does not read the link (its event queue holds 2
  entries), so central-to-peripheral messages in that window can be dropped;
  the link is about to be torn down anyway.

## Commands run

All from the worktree; `<abs>` is its absolute path. Pass/fail as observed.

- `git log --oneline 8b4d1b312d8e..HEAD` — 20 topic commits — pass.
- `git range-diff f8da2742971d..572e4b0c0856 8b4d1b312d8e..HEAD` — pass (see above).
- `git ls-remote fork refs/heads/feat/nrf-half-duplex-split` — `ffe45e47d`.
- `cargo check --manifest-path rmk/Cargo.toml --no-default-features --features split,storage --lib` — pass (before and after each commit).
- `cargo check ... --features rynk,_ble,split,async_matrix,storage --lib` — pass.
- `cargo check ... --features split,storage,dfu_split --lib` — fails before any topic code (`dfu/split/peripheral.rs` needs `dfu_nrf`/`cortex_m`); pre-existing, so DFU paths in `driver.rs` were reviewed by reading only.
- `nix shell nixpkgs#cargo-nextest --command cargo nextest run --manifest-path <abs>/rmk/Cargo.toml --config-file <abs>/.config/nextest.toml --no-default-features --features split,storage --lib -E 'test(split::)'` — 28 passed on the final tree (17 serial tests after commit 1, 24 after commit 2, 25 after 3, 28 after 4).
- Same with `--features rynk,_ble,split,async_matrix,storage` — 29 passed.
- The four commit-1 tests against a temporary old-semantics patch — 4 failed, as intended; patch reverted.
- `rustfmt +nightly --edition 2024 --check <every changed .rs file>` — pass.
- `git diff --check ffe45e47d..HEAD` — pass.
- `cargo clippy --manifest-path rmk/Cargo.toml --no-default-features --features split --lib -- -D warnings -A clippy::new_without_default` — pass; same for `split,storage` and `rynk,_ble,split,async_matrix,storage`. With plain `-D warnings` all three fail on one diagnostic only: `Crc32::new` in `rmk/src/crc32.rs` (`clippy::new_without_default`), which `git diff upstream/main -- rmk/src/crc32.rs` shows is untouched upstream code (the audit README records the same).
- Cross-compile of a throwaway copy of `examples/use_config/nrf52840_ble_split` reconfigured to `connection = "auto"` with a half-duplex UARTE port on both halves, for `thumbv7em-none-eabihf` (`cargo check --bin central --bin peripheral`): pass for both binaries (`Finished`, exit 0). Needed `nix shell nixpkgs#gcc-arm-embedded`, `LIBCLANG_PATH` from `nixpkgs#llvmPackages.libclang.lib`, and `BINDGEN_EXTRA_CLANG_ARGS="--target=arm-none-eabi -nostdinc -isystem <arm>/arm-none-eabi/include -isystem <arm>/lib/gcc/arm-none-eabi/15.3.1/include"` for the Nordic `*-sys` crates; the three earlier attempts failed on those toolchain gaps, not on RMK code. Copy lived in `/tmp/nrf-auto-split`, never in the repository.
- `git push --force-with-lease fork HEAD:refs/heads/feat/nrf-half-duplex-split` — pass: `ffe45e47d..f46e7337d  HEAD -> feat/nrf-half-duplex-split`.

## Not done / caveats

- No hardware was exercised; every timing statement above is from the code
  and the mock-clock tests.
- Commit 5 (BLE parking) has no test.
- The wire enum changed (`TransportOverride` became a struct variant and
  `TransportOverrideAck` was added; `MaxSize` derives cover it), so both
  halves must be flashed together, as the task allowed.
- Public API change: `request_transport_force` returns `()` instead of
  `bool`; `run_half_duplex_peripheral_manager` and
  `run_auto_half_duplex_peripheral_manager` take `baud: u32` after the serial
  port. Only generated code called them.
- Assembly impact: this topic's coherence fixup
  (`patches/nrf-half-duplex-split-coherence.patch`) touches only
  `rmk/src/ble/mod.rs`, which these commits do not change, but the tracked
  rerere pairs for this entry will need re-resolution wherever later entries
  conflict in `rmk/src/split/driver.rs`, `rmk/src/split/serial/mod.rs`, or
  `rmk/src/split/selector.rs`. The maintenance repo was not touched.
- Build directories `target-ble/` and `target-dfu/` created inside the
  worktree for parallel checks were deleted afterwards.
