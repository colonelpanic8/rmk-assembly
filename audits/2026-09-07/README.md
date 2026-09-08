# Critical audit of the carried RMK stack

Audited 2026-09-07. Base `f8da2742971d048e0080e57d69d98f00a859e4e9`; assembled commit `9a88399366a01b344cf90fb3723cae68dc4dc949`; tree `ad36ded1d782d52aed0c5924569993f37ca91362`.

The scope is the pinned, combined stack: 36 entries (32 branches, two PRs, two standalone patches), eight entry fixups, and their recorded conflict resolutions. The net diff is 205 files, 41,172 added lines, and 1,763 removed lines. [Machine-readable scope](scope.json) and [offline stack status](stack-status.txt) preserve the inventory. The manifest matches the lock. This is a source and host-test audit, not a hardware qualification or a live-remote freshness check.

## Confirmed findings

### F1 — P1: the maintenance gate allows newer mutation commands

`rmk/src/host/rynk/mod.rs:163` manually lists commands requiring maintenance mode and returns false for everything else. `SetMorseProfileEntry`, `DeleteMorseProfile`, and `SetSplitTransportForce` are dispatched at lines 301, 302, and 337 but are absent from that list. With maintenance disabled, a host can still rewrite/delete a timing profile, remove its hold-trigger positions, or force the split transport on an automatic-transport board. This contradicts the documented guarantee that the lock blocks every host mutation.

The existing maintenance scenarios pass. New command-policy tests fail for all three omissions; protocol scenarios show profile edit/delete returning success instead of `NotReady`. Their assertion output says a seven-byte error response was expected but a six-byte success response arrived. This is an authorization omission, not a transport timeout.

Fix ownership: `feat/maintenance-mode` interacting with `fold/runtime-behavior-parity` and `feat/nrf-half-duplex-split`. Immediate fix: gate these commands. Structural fix: require an explicit access class in the endpoint declaration so adding a command cannot silently authorize it. Keep the configured unattended-operation default; correct enforcement when the owner engages the lock.

### F2 — P2: simultaneous layer-name reads share one reply slot

`rmk/src/storage/mod.rs:68,107–112,1340–1346` uses one global `Signal<Option<LayerMetadata>>` for every layer read. USB and BLE can each run a Rynk session. If both ask for metadata, both reset and wait on the same signal, which carries neither a layer nor a request identity and retains only one response. One reader can receive the other layer's metadata; a response can be overwritten or consumed, leaving another reader waiting.

A focused poll-interleaving test queues layer 0 and layer 1 reads, delivers the storage response for layer 0, and shows the layer 1 future consuming it. This reproduces the response-routing bug without requiring radio hardware.

Fix ownership: `feat/persistent-layer-metadata`. Give each request a bounded reply slot/token or serialize the complete request/response exchange with cancellation-safe ownership. An ID in a shared single-consumer signal alone is insufficient: callers must not steal and discard each other's responses. Use the same design for storage writes that need completion acknowledgements.

### F3 — P2: saved Unicode mode is never restored by the host boot path

`rmk/src/host/storage.rs:89–231` restores records in `read_boot_data`, but has no arm for `StorageKey::UnicodeMode`. `KeyMap::new_from_storage` calls this path at `rmk/src/keymap.rs:457–458`. The Unicode branch added restoration to `read_behavior_config` (`rmk/src/storage/mod.rs:875`), which this boot path no longer calls. `UnicodeModeCycle` still saves the new mode.

Reproduction: save Windows mode, construct default behavior, and run the actual single-pass boot loader. The mode remains Linux. A keyboard switched to another OS input method therefore reverts after reboot.

Fix ownership: integration of `fold/unicode-input` with `feat/single-pass-storage-read`. Add restoration to the authoritative boot path, eliminate the duplicate restoration logic, and test persistence through boot rather than codec round trips alone.

### F4 — P2: a supported hold-trigger table exceeds the flash scratch buffer

`rmk/src/storage/mod.rs:1643–1661` sizes its buffer only from `MACRO_SPACE_SIZE`, with a small minimum. The added `MorseHoldTriggerPositions` record scales independently. A configuration with 160 positions (four profiles × 40 positions), `macro_space_size = 256`, and `rynk_buffer_size = 1024` compiles but serializing the position record fails with `SerializationError(BufferTooSmall)`. The ordinary 256-byte macro setting gives a 288-byte scratch buffer; this table needs more than 480 bytes.

This affects runtime saves and startup initialization, which writes the configured table at `rmk/src/storage/mod.rs:1164–1169`. The storage task logs write errors, while ordinary host mutations acknowledge queue admission, so a host can appear to save settings that will not survive reboot. Failed initialization marks storage disabled, leading to erasure/reinitialization on subsequent boots.

Fix ownership: `feat/morse-hold-trigger-positions` and its storage integration. Derive scratch capacity from the maximum serialized key-plus-value across all compiled record types, including alignment, or shard large tables. Validate the whole storage budget independently of the protocol budget. The protocol already has a compile-time payload-size check; raising its buffer does not repair the storage buffer.

### F5 — P1: brightness additions silently change existing Rynk key meanings

`rmk-types/src/keycode/consumer.rs:23–25` inserts three variants before `Play`; `rmk-types/src/keycode/hid.rs:339–344` inserts them before the mouse/modifier keys. Both enums derive Serde, whose enum tags follow declaration order. `#[repr(u8)]` and explicit HID values do not make derived serialization use those values.

Two frozen upstream payloads reproduce the break: `[0x04]` decoded as `ConsumerKey` was Play and is now BrightnessMinimum; `[0xd6,0x01]` decoded as `HidKeyCode` was Left Control and is now MouseAccel0. The identical tests pass against the pinned upstream base and fail against the assembly. These enums are nested in keymap/actions exposed by Rynk. `rynk/src/device.rs:61–87` accepts same-major protocol versions, so the 0.1/0.2 minor difference does not protect a host from silently reading or writing the wrong action. Persisted data is also affected when an upgrade preserves storage; ordinary build-hash erasure can hide that part of the problem.

Fix ownership: `feat/consumer-keycodes`. Preserve existing wire tags by appending variants, or define an explicit stable wire codec. A deliberate breaking codec change requires a distinct compatibility boundary, not a same-major handshake. Keep upstream-generated wire fixtures and test old-to-new decoding; updating a current snapshot alone cannot prove compatibility.

### F6 — P1: Unicode emission and user modifiers do not have independent ownership

`rmk/src/keyboard.rs:2001–2044` temporarily ORs the Linux Ctrl/Shift opener or macOS Alt into the physical modifier bitset, then clears those bits. `register_modifiers`/`unregister_modifiers` at lines 2343–2359 have no ownership count. If the user already holds the same modifier, Unicode completion clears it while the physical key remains pressed. A focused test holds physical Left Alt, emits macOS Unicode, and observes an empty modifier state afterward.

Separately, `build_keyboard_report` applies Control/GUI swap to the synthetic Linux opener at `rmk/src/keyboard.rs:2208–2219`. With swapping enabled, Unicode emits GUI+Shift+U instead of the required Ctrl+Shift+U. A simulator regression reproduces that interaction. The sequence is an OS input protocol; applying user modifier remapping changes the protocol itself.

Fix ownership: `fold/unicode-input`, with `feat/ctrl-gui-swap` exposing the second failure. Give synthetic protocol input scoped state, temporarily suspend or merge user state according to explicit semantics, emit protocol HID usages after user remapping, and restore the live user state on completion/cancellation.

### F7 — P1: the new pointing keypad path can release physically held keyboard keys

`rmk/src/input_device/pointing.rs:555–563` emits Keypad gestures through `tap_key`. At lines 648–672, that helper writes a complete keyboard report containing only the virtual key, then a completely empty report. A keyboard report describes the full pressed state. Holding Shift or another key while the pointing pad emits a key therefore tells the host that the held key was released; the keyboard engine still thinks it is held and may not send a corrective report until the next keyboard event. This path also bypasses the new Control/GUI remapping.

Provenance matters: the Caret helper already had this design in upstream. The carried pointing-runtime branch adds Keypad motion/tap use of it and increases the affected surface. This is inherited debt reused by a new feature, not an entirely new helper regression. Source tracing confirms the independent report writers; hardware behavior was not exercised.

Fix ownership: `fold/pointing-runtime-config-pinned`, coordinated with the input core. Pointing should submit virtual key events to the owner of keyboard report state instead of sending independent full snapshots.

### F8 — P2: deleting a position combo changes meaning after reboot

`rmk/src/host/context.rs:221–250` treats an empty position-combo definition as `None` in RAM but persists its tagged empty PositionCombo value. `rmk/src/host/storage.rs:131–137` restores that record as `Some(PositionCombo(empty))`. Before reboot, the legacy `GetCombo` returns an empty combo; after reboot it returns `Invalid`, and `GetComboBulk` rejects the entire page containing that slot (`rmk/src/host/rynk/handlers/combo.rs:16–61`). The combo engine ignores zero-length combos, so this finding concerns persistent/API semantics rather than stuck input.

A focused storage/boot test reproduces the occupied representation after deleting the slot. Fix ownership: `feat/position-combos` with the boot loader. Normalize deletion to one tombstone representation, and enforce it on both writes and restore.

### F9 — P2: pointing configuration accepts records it cannot actually apply

The handler at `rmk/src/host/rynk/handlers/pointing.rs:19–22` forwards directly to `pointing_config::replace`. That function only checks revision (`rmk/src/input_device/pointing_config.rs:69–87`). The payload silently clamps over-capacity counts and chooses the first matching duplicate (`rmk-types/src/protocol/rynk/payload/pointing.rs:94–139`). Overrides for undeclared devices never reach `apply`, which iterates only declared devices.

A host can submit excessive counts, duplicate IDs, duplicate layer/device overrides, or orphan overrides; the write is acknowledged, persisted, and revisioned even though part of it is unreachable. This is a source-verified validation omission, not a hardware-tested failure. Validate counts, uniqueness, references, and layer indices before mutation; return Invalid without advancing revision if the complete candidate is invalid. Owner: `fold/pointing-runtime-config-pinned`.

### F10 — P1: half-duplex deadlines cannot accommodate valid traffic

`rmk/src/split/serial/mod.rs:510–520,596–628` gives the central five milliseconds for its outgoing burst, poll, peripheral turnaround, and incoming burst combined. It arms the deadline **before** transmitting up to four frames. The peripheral waits two milliseconds and may send four replies plus its terminator (`:729–769`). The generated nRF default is 115,200 baud (`rmk-macro/src/codegen/split/central.rs:137–143`). Four 32-byte frames alone take about 11.1 milliseconds at 8N1; split DFU permits individual payloads of 256 bytes.

The deadline can therefore expire during valid traffic. The central then permits a new exchange while the peripheral may still be driving. This is source/timing analysis; no cable or oscilloscope test was performed. Owner: `feat/nrf-half-duplex-split`. Separate transmit, turnaround, receive, and recovery phases. Start the response timer after the poll drains, budget the maximum encoded reply burst from the actual baud, and retain an independent cancellation-safe bus-quiet deadline.

### F11 — P1: a corrupt response frame permits an immediate turnaround collision

`drain_response_window` clears `response_deadline` on any frame-read error (`rmk/src/split/serial/mod.rs:560–591`). But the peripheral can still be transmitting subsequent frames and its final `HalfDuplexIdle`. The manager logs the error and loops (`rmk/src/split/driver.rs:341–349`), allowing the next exchange to drive into the remaining burst. A single damaged frame can thus corrupt the rest of an otherwise recoverable exchange.

Source-verified, without hardware fault injection. Owner: `feat/nrf-half-duplex-split`. Record the decode error but retain receive/drain ownership until a valid terminator or conservative end-of-burst bound. UART faults need an explicit recovery phase too; an error is not evidence that the peer released the wire.

### F12 — P1: transport forcing switches the central without proving peer agreement

`rmk/src/split/driver.rs:350–371` applies the local force after `send` returns. That send suppresses non-disconnection errors (`:240–250`); the BLE implementation also logs and suppresses most `write_characteristic_without_response` errors (`rmk/src/split/ble/central.rs:559–578`). There is no application acknowledgement from the peripheral. A locally accepted BLE write also does not establish peer application of the selector.

On error or a disconnect race, the central can leave BLE while the peripheral stays there, removing the very link needed to finish the command. Owner: `feat/nrf-half-duplex-split`. Propagate errors and implement desired `(generation, mode)` state with peer acknowledgement, bounded rendezvous/fallback, and resynchronization on reconnect. A GATT write response alone does not prove the peripheral applied the command. This is a source-verified failure path, not a radio reproduction.

### F13 — P2: lighting scene replacement is atomic in RAM but not in storage

`scene_mutation` commits the engine mutation, then `persist_scenes` reads it back and overwrites shard keys starting at zero (`rmk/src/host/rynk/lighting.rs:875–885,981–1021`). It writes the table header last. The loader trusts the current header and whatever values are now at those shard keys (`rmk/src/storage/mod.rs:939–973`). Writing a header last is insufficient when the previous table's shards have already been replaced in place. Runtime conditional scenes use the same scheme (`host/rynk/lighting.rs:1026–1066`).

A focused interruption test persists sixteen old cells across two shards, writes only the first replacement shard, and invokes the startup loader. It restores eight new cells plus eight old cells. This models power loss between successfully completed record writes; it does not simulate torn flash writes. The existing record-level storage protection cannot make a multi-record update atomic.

Owner: PR 1031 lighting persistence and its conditional-scene extension. Store shards under a new generation, then commit a manifest containing generation, length, policy, and integrity information. Boot must select one complete committed generation and retain the prior one if the new write was interrupted. Report durable success only after the storage owner acknowledges the commit, or explicitly expose a separate pending/durable status. Current helpers return success after queue admission and silently return on readback failure.

### F14 — P2: compiled output-mode lighting conditions never match

The TOML resolver accepts `output_mode` on conditional scenes (`rmk-config/src/resolved/lighting.rs:638–646`), and code generation includes it in the rule (`rmk-macro/src/codegen/lighting.rs:354–371`). But the compiled `ConditionalScenes` source always evaluates conditions with no output-mode value (`rmk/src/lighting/source.rs:474,494`). Every rule requiring a mode consequently evaluates false, including a rule requiring the engine's current mode. Runtime-authored conditional rules have a separate evaluator that does receive the live mode.

A focused engine test installs a compiled green rule requiring AlwaysOn in an AlwaysOn engine. Rendering returns the gray background instead. The existing equivalent runtime-rule test passes. Owner: PR 1031 lighting configuration/runtime integration. Give every condition evaluator the same immutable render snapshot, including policy and extension-enabled state. Alternatively reject unsupported compiled predicates at configuration time, but that would remove behavior the configuration currently promises.

### F15 — P2: local powered-only lighting scope has no behavioral consumer

`PoweredOnlyScope` documents separate Authority/Local behavior (`rmk/src/lighting/source.rs:286–294`), is accepted by config/codegen, and appears in host readback. But `rmk/src/lighting/standard/engine.rs:1266–1297` makes the power decision solely from the one `context.powered` bit; it never reads the scope. With replicated authority context, a battery-powered peripheral can stay lit when only the central has USB power, despite Local being selected. A custom caller can substitute local context, but then the scope setting still cannot select between the two sources.

Source-verified, without a split power measurement. Owner: PR 1031, specifically local-powered-scope integration. Carry authority and local power separately in the renderer snapshot and apply the selected scope explicitly. The existing test sets Local but never gives authority and peripheral opposite power states.

### F16 — P1: supported Rynk feature combinations do not compile

`rmk/src/input_device/mod.rs:17–18` enables `pointing_config` for all Rynk builds, but that module unconditionally imports the storage-only flash channel and storage module (`pointing_config.rs:14–16`). `rynk` does not imply `storage` in Cargo, and the checked-in QEMU Rynk example uses exactly that combination. This makes a supported nonpersistent configuration fail name resolution.

Two compile checks confirm distinct omissions. `--features rynk` fails earlier in `rmk-types/src/protocol/rynk/command.rs:573–584`: six retained extended-lighting endpoints lack the `#[cfg(feature = "lighting")]` attributes used by neighboring endpoints and refer to unavailable payload types. Enabling lighting with `--features rynk,lighting` then reaches the two unresolved flash/storage imports in pointing configuration.

Owners: `fold/pointing-runtime-config-pinned` and the lighting compatibility integration (`7bf6292c20`). Gate persistence independently while retaining runtime configuration, and gate lighting endpoints consistently with their payloads. Include Rynk without lighting and without storage in the focused compile matrix.

### F17 — P1: persisted pointing defaults and layer changes are not wired into generated startup

`pointing_config::init` seeds the live configuration and `PointingLayerModes` updates its active layer (`rmk/src/input_device/pointing_config.rs:53–62,106–118`). `Storage::read_pointing_config` exists (`rmk/src/storage/mod.rs:1277`), but the audited tree contains no call to that loader or initializer, and no instantiation of the layer subscriber. The generated orchestrator does not install them.

In the standard generated firmware path, a live write applies for layer zero, later layer changes do not reapply it, and the saved configuration is not restored on reboot. Hand-written board code could install these public components; the missing standard integration is the finding. Owner: `fold/pointing-runtime-config-pinned`. Register initialization, restore, and the running layer task together, then test save/reboot/layer transitions through generated firmware.

### F18 — P1: failed sensor initialization can busy-loop the cooperative executor

After three failures, `PointingDevice::try_init` enters permanent `Failed` and returns immediately (`rmk/src/input_device/pointing.rs:90–133`). Its event loop still waits on the motion GPIO and immediately polls again (`:206–256`). If the failed/absent sensor's line stays low, that wait is immediately ready and the loop never yields. The carried change removed upstream's explicit pending wait for permanent failure.

This source-proven path lets one optional bad sensor starve scanning, HID, and BLE tasks. Owner: the carried pointing-device polling changes. Restore a parked failure state or a bounded retry timer that always yields, and make recovery/reset explicit. Test with a permanently asserted motion pin and a driver that fails initialization; an ordinary successful-sensor test will not exercise the defect.

### F19 — P1: mouse reports also lack a single button-state owner

Pointing owns device buttons, Drag latch, and Press touch state (`rmk/src/input_device/pointing.rs:374–400,460–546`) and emits a full mouse report. Keyboard mouse keys independently emit full reports from their own state (`rmk/src/keyboard.rs:1878–1906,2266–2273`). A keyboard mouse-key report does not include the pad's latched/held button, so it releases that button at the host. Separate sensor processors can likewise clear each other's held buttons. This affects holding a device button while keyboard mouse movement repeats, as well as Drag/Press.

This is introduced persistent pointing state interacting with existing keyboard mouse reports; it is source-verified rather than hardware-reproduced. Owner: the pointing/drag/Cirque integration. Aggregate all persistent button owners centrally and apply movement deltas there. The single-report-owner recommendation must cover both keyboard and mouse interfaces.

### F20 — P2: an absent opposite-hand setting erases explicit unilateral false

Generated Morse profiles call `MorseProfile::new(Some(false), ...)` and then `.with_opposite_hand_hold(None)` (`rmk-macro/src/codegen/action_parser.rs:108–112,144–151`). The latter preserves the unilateral-true bit pattern but clears the shared explicit-false pattern (`rmk-types/src/morse.rs:119–162`). `unilateral_tap()` becomes `None`, so a profile meant to disable a globally enabled unilateral policy instead inherits it.

A focused constructor/builder test reproduces this exact codegen chain. Owner: `fold/opposite-hand-hold` interacting with unilateral-profile configuration. Preserve explicit false when the other field is absent. Longer term, resolve mutually exclusive policies into one explicit semantic enum before packing; sequential setters for overlapping bit fields make configuration order change meaning.

### Other source findings and design risks

The [transport review](transport-review.md) includes the exact paths and triggers for additional issues:

- A force queued during BLE setup can outlive the failed session and override a newer disconnected selection on reconnect. Use generations instead of a sessionless one-element queue.
- After advertising times out, the split peripheral waits only for keyboard input. Selector changes and the forced-BLE liveness fallback no longer wake it, so unplugging or forcing BLE can leave it disconnected until a local keypress.
- `LinkEndpoint::reset` preserves the inbox and outgoing lanes. Restarted serial sessions can deliver old input/control messages. Define replay policy and clear transient state at an epoch boundary.
- Accepted multi-port nRF configurations default each port to the same TIMER/PPI resources and then fail in generated Rust. Allocate or validate the complete resource set once.
- Peripheral battery invalidation reaches internal/Rynk state but the connected host's GATT battery value retains the last percentage. Treat this as a host-contract gap: select a supported unavailable/connectivity representation rather than inventing an out-of-range battery percentage.
- Runtime advertising names differ from the static GAP Device Name. Either synchronize both surfaces or explicitly document advertising-only naming.

The [input review](input-review.md) also identifies two architectural risks requiring targeted behavioral tests: replacing a live combo definition discards its triggered/release state, and one all-device pointing processor shares button/drag/touch/remainder state across device IDs. The first extends existing runtime-setter debt; the second reuses upstream aggregation for richer gestures. Keep active gesture instances tied to their original definition generation, and keep transient pointing state per device.

The [additional input review](input-review-additional.md) strengthens those paths and identifies two further source-verified interactions: an incomplete combo intercepts a physical press before retro-tap demotion sees it, and unchecked pointing numeric values permit 255-tap storms for nonpositive Caret thresholds or `i16` overflow in Scroll/Sniper multiplication. These extend F9's validation problem beyond ignored configuration fields. Reject invalid thresholds and widen/saturate arithmetic before accumulation; process physical interruption facts before combo buffering. Test active-combo edits while its output remains held, because maintenance authorization does not itself quiesce input.

The [lighting review](lighting-review.md) also finds misleading presented-frame provenance during wake linger (pixels use the synthesized active-layer context while readback records the original context) and missing validation of the lighting subsystem's 64-layer limit. Its performance recommendations are to precompute physical routing and iterate matching conditional cells once: current route searches and indexed conditional scans can be quadratic. Measure actual board memory before changing fixed-capacity storage; style pools, transaction/replica copies, and per-LED deadline arrays all contribute. No topology-validation correctness defect was found in the reviewed duplicate, completeness, or mapping checks.

### Additional configuration flaw

`UNICODE(n)` accepts any u16 at `rmk-macro/src/codegen/action_parser.rs:319–325`, even when the codepoint table is missing or shorter. Runtime lookup simply warns and does nothing (`rmk/src/keyboard.rs:2001–2005`). Validate these references when resolving the keymap, including nested actions, so a typo produces a location-specific build error instead of an inert key. This is lower priority than the state and wire failures above.

## Validation evidence

The isolated checkout is `.worktrees/audit-astra-20260907`. Only reproduction tests were added there. The generated assembly, topics, manifest, lock, and tracked resolutions were not edited by this audit.

- [Regression patch](regression-repros.patch): expected-correctness tests that intentionally fail on the audited tree.
- [Focused regression results](regression-results.log): 12 tests, five existing controls passed and seven new regression assertions failed. Three policy tests plus two protocol tests describe F1; the other failures describe F2 and F3.
- [Larger-capacity configuration](capacity.toml) and [its result](capacity-results.log): one additional failing test for F4. This test requires that configuration and is deliberately excluded from the default-capacity run.
- [Wire results on the assembly](wire-assembly-results.log) and [the upstream control](wire-upstream-results.log): the same two compatibility tests fail on the assembly and pass on upstream (F5).
- [Input integration results](input-regression-results.log): failing Control/GUI-swap Unicode scenario and position-combo boot deletion test (F6, F8).
- [Held-modifier result](unicode-held-results.log): physical Left Alt is lost after Unicode (F6).
- [Interrupted scene replacement](scene-interruption-results.log): mixed generations after interruption between shard writes (F13).
- [Compiled versus runtime lighting predicate](compiled-conditional-results.log): compiled-rule regression fails; existing runtime-rule control passes (F14).
- [Morse profile builder result](morse-profile-results.log): explicit unilateral false is cleared by the omitted opposite-hand option (F20).
- [Rynk-only compile result](no-storage-results.log) and [Rynk plus lighting without storage](lighting-no-storage-results.log): two distinct feature-gating failures (F16).
- [Clippy result](clippy-results.log): the scoped library/test check with `rynk,storage,lighting,split` failed with six diagnostics. Five involve changed areas: layer-metadata handler return, device-data type complexity, auto-mouse constant comparison, keymap return, and a large lighting command variant. The `Crc32::new` diagnostic is in unchanged upstream code. These lint failures are separate from the behavioral findings.

In total, sixteen expected-correctness regression tests fail across ten finding groups (F1–F6, F8, F13–F14, F20), and two feature compile checks fail (F16). Six existing controls and the two upstream compatibility controls pass. The count is tests, not sixteen independent bugs. Transport and remaining pointing findings are source analysis and must be validated on their target backends. The reproduction Rust files pass the repository's nightly rustfmt settings, and their diff passes whitespace checks.

RMK tests were run with nextest because the mock clock requires process isolation. The first plain cargo-test attempt was rejected by that guard; it was not counted as behavioral evidence. No full test suite, board flash, RF/cable/power measurement, or firmware activation was performed.

Reproduce from a fresh detached checkout of the audited commit after applying the patch:

```sh
nix shell nixpkgs#cargo-nextest --command cargo nextest run \
  --manifest-path rmk/Cargo.toml --config-file .config/nextest.toml --profile ci \
  --no-default-features --features rynk,storage,lighting,split \
  -E '(test(audit_) and not test(audit_configured_hold_trigger)) or (test(rynk_maintenance_mode) and not test(audit_)) or test(boot_read_restores_behavior_tables) or test(layer_metadata_survives)'

KEYBOARD_TOML_PATH=/absolute/path/to/capacity.toml \
  nix shell nixpkgs#cargo-nextest --command cargo nextest run \
  --manifest-path rmk/Cargo.toml --config-file .config/nextest.toml --profile ci \
  --no-default-features --features rynk,storage,lighting --lib \
  -E 'test(audit_configured_hold_trigger)'

cargo test --manifest-path rmk-types/Cargo.toml --test audit_wire_compatibility
cargo test --manifest-path rmk-types/Cargo.toml --test audit_morse_profile

cargo check --manifest-path rmk/Cargo.toml --no-default-features --features rynk --lib
cargo check --manifest-path rmk/Cargo.toml --no-default-features --features rynk,lighting --lib
```


## Architectural recommendations and repair order

The recurring problem is that features add a new path through shared state without updating every other path. A large rewrite would make these interactions harder to verify. Consolidate the ownership boundaries incrementally, preserving the existing pure engine/service split in lighting and the bounded, allocation-free storage and transport approach.

1. **One report owner per HID interface, with source-aware state.** Route physical keys, pointing taps, macros, combo outputs, and Unicode through one keyboard reducer; aggregate pointing and keyboard mouse-button owners in a mouse reducer. Track each source's contributions separately; apply user remapping to logical user actions, then compose explicitly physical protocol usages. Completion and cancellation remove only the emitting source's contribution. Keep active combo/Morse instances separate from mutable definitions so releases remain valid across configuration changes. Start by moving pointing taps and Unicode through this owner; do not rewrite all Morse decision logic in the same patch.
2. **One declarative protocol schema.** Declare stable wire tags, access class, capacity constraints, and handler identity together. Generate exhaustive dispatch/policy matches and preserve compatibility fixtures produced by prior releases. Internal enum ordering must never silently become an external ABI. Maintenance policy should be exhaustive at compile time. Keep domain validators explicit rather than hiding validation in generated serialization code.
3. **One configuration restore and commit path.** Introduce typed validate/normalize/apply/persist operations. Use the same normalization during runtime mutation and boot. Register a feature's initializer, stored-state loader, and required running task together so codegen cannot silently omit its lifecycle. Give every storage request a safe reply route; distinguish acceptance into a queue from completed durable storage. For multi-record tables, use committed generations with a bounded rollback policy. Compute storage capacity across all record types. A deterministic firmware build seed is useful for reproducible builds but is not a substitute for an explicit storage schema/migration version.
4. **One split I/O phase owner, plus replicated desired state.** Treat half-duplex as a state machine owning transmit, turnaround, receive, corruption drain, and cancellation recovery. Supply baud/encoded-length timing parameters rather than unrelated timeout constants. Represent transport force as a desired generation with acknowledged application and a reconnect snapshot. Define session epochs and replay policy for queues. This addresses timing, stale force, stale input, and uncertain delivery together.
5. **One lighting evaluation snapshot and commit boundary.** Keep the engine authoritative, but feed compiled and runtime rules the same context. A mutation should produce a validated revisioned snapshot suitable for persistence and replication; paged readback of a moving live model is not an atomic snapshot. Rendering should consume one coherent snapshot. Preserve logical topology and effect composition as independent components, and prevent generic source/render APIs from accumulating unrelated host-control and replication responsibilities.
6. **Validate whole-board resources and budgets before codegen.** Resolve Unicode indices, pointing device/layer references, timer/PPI ownership, wire sizes, flash scratch size, shard addressing, and table capacities in one configuration validation stage. Prefer a location-specific configuration error over clamping, an inert action, or a Rust error deep in generated tokens. Keep resource allocation backend-specific while its ownership model is shared.

Recommended sequence: (a) repair the wire ABI, maintenance omissions, HID ownership, and half-duplex/force safety paths; (b) repair boot restoration, request routing, and persistent table commits; (c) unify the configuration/lighting evaluation paths and tighten validation. Make topic-local fixes on their topic branches, and put genuine cross-topic repairs on the first affected entry's coherence fixup. Never patch the generated assembled branch directly.

The acceptance suite should emphasize boundaries: old-host/new-firmware wire fixtures; lock policy for every mutating endpoint; hold/release across Unicode, pointing, and runtime edits; save/reboot for every record type; interruption between every multi-record write; and split exchanges with byte-time delays, corruption, cancellation, reconnect, and transport force. Add a small compile matrix for representative nRF single/multi-port and maximum-capacity configurations. Hardware qualification then needs both halves, unplug/replug, forced transport changes, noisy cable/retransmission conditions, and power cuts. Existing unit tests are valuable, but the confirmed failures are mostly interactions those isolated tests do not exercise.


## Coverage ledger

Every carried entry was included in the pinned aggregate review. Depth varied: source tracing across each feature family, focused execution for the reproduced findings, and static inspection for hardware paths. “No distinct defect” means none established in this pass, not a proof of correctness or a complete feature-matrix qualification. The input, transport, and lighting companion reports preserve the subsystem detail.

| Entry | Inspected scope | Result / limitation |
|---|---|---|
| `fold/macro-hooks` | Macro hooks, generated task/init order | No distinct defect; lifecycle architecture recommendation |
| `fold/split-reliability` | BLE reconnect/session handling | Transport review: advertising wake/reconnect risks |
| `pr-1031` | Lighting engine/config/output/topology, Rynk, persistence, split integration | F13–F15; lighting/transport appendices |
| `fold/connection-selection` | Preferred host output and profile switching | Transport review; no separate selection defect retained |
| `pr-984` | Peripheral BLE stack ownership | Source review; no distinct new ownership defect retained |
| `feat/single-pass-storage-read` | Authoritative boot traversal versus individual loaders | F3, F8, F17 |
| `feat/pointer-layer-scaling` | Pointing ratios and layer behavior | F9; numeric validation/arithmetic finding |
| `feat/combo-hold-output` | Combo hold/release state | F7/F19 input ownership; live-combo edit risk |
| `feat/morse-retro-tap` | Retro-tap arming/demotion and combo interception | Additional input review: incomplete-combo interaction |
| `feat/morse-per-profile-prior-idle` | Per-profile timing fallback | No distinct prior-idle defect; configuration normalization recommendation |
| `feat/morse-tap-unless-interrupted` | Interruption and tap/hold state | No distinct defect retained; interaction coverage remains needed |
| `feat/morse-hold-trigger-positions` | Position table config, storage, host writes | F1, F4 |
| `feat/morse-hold-trigger-on-release` | Hold-trigger release decisions | No distinct defect retained; active-instance recommendation |
| `feat/consumer-keycodes` | Keycode enums and protocol codec | F5; upstream compatibility controls |
| `fold/unicode-input` | Unicode emission, configuration, boot persistence | F3, F6; dangling-index validation |
| `fold/cirque-pinnacle` | Cirque packet/state handling and pointing integration | F19; driver hardware/register timing not qualified |
| `feat/pointing-drag-mode` | Drag/Press persistent state and report construction | F19; per-device state review |
| `fold/pointing-runtime-config-pinned` | Runtime config, numeric/structural validation, feature/lifecycle wiring | F7, F9, F16–F19 |
| `fold/runtime-behavior-parity` | Runtime Morse/behavior host and storage paths | F1, F3, F4; lifecycle consistency |
| `feat/maintenance-mode` | Policy classifier and dispatch | F1; existing lock controls and new regressions |
| `fix/battery-power-transitions` | Battery cache transitions and consumer propagation | Transport review: GATT unavailable-state contract |
| `feat/ble-name-template` | Name persistence, rendering, advertising and GAP | Transport review: scan/GAP mismatch |
| `fix/ephemeral-split-layer-state` | Ephemeral layer replication and reconnect state | Session epoch/snapshot recommendation |
| `feat/morse-unilateral-mod-chording` | Morse packed policy and chording interactions | F20; input review |
| `fold/opposite-hand-hold` | Opposite-hand profile bit packing and input state | F20 reproduced |
| `feat/position-combos` | Combo deletion, persistence, legacy getters, active state | F8; live-combo replacement risk |
| `feat/nrf-half-duplex-split` | Serial phases, deadlines, corruption, force, nRF resources | F1, F10–F12; transport appendix |
| `feat/device-data` | Host discovery, capacities and device data | Scoped compilation/lint; no separate behavioral defect retained |
| `runtime-lighting-wake-layers` | Wake mask, linger, persistence and render/readback | Lighting review: provenance and layer-bound validation |
| `feat/persistent-layer-metadata` | Layer metadata storage and request routing | F2 reproduced |
| `fix/mpsl-interrupt-priorities` | nRF interrupt/resource configuration | Static transport/codegen review; on-device priority behavior not qualified |
| `fix/pointing-mode-event-priority` | Pointing mode scheduling versus motion | F18; per-device mode/state concerns |
| `feat/deterministic-build-hash` | Build hash seed and storage invalidation contract | No distinct defect; explicit schema version recommendation |
| `feat/lighting-layer-indicator-conditions` | Condition wire types, feature flags, compiled/runtime predicates | F14, F16; missing lighting cfg on retained endpoints |
| `final-formatting-coherence` | Final combined patch and integration tree | Assembly/fixup inspection; no independent feature |
| `feat/ctrl-gui-swap` | Report remapping and synthetic input interaction | F6 reproduced; F7 bypass |

The eight coherence fixups were included through the assembled diff and their owning entry: lighting/Rynk, runtime behavior parity, maintenance, BLE name, position combos, nRF half-duplex, persistent layer metadata, and pointing-mode event priority. Findings identify combined-tree interactions rather than assuming independently correct branches remain correct after assembly. The audit did not rebuild the stack or revalidate every intermediate entry tree; it assessed the locked final tree.
