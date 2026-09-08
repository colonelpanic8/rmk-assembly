# RMK personal-change audit: keyboard, Morse/combo, pointing, Unicode, and input configuration

## Scope and method

Audited the exact introduced range `f8da2742971d048e0080e57d69d98f00a859e4e9..9a88399366a01b344cf90fb3723cae68dc4dc949`, reading the pinned objects directly so later movement of the build worktree could not affect attribution. Scope was keyboard/Morse/combo/held-buffer behavior, pointing (including Cirque), Unicode, Ctrl/GUI swap, and host/config/macro integration, with particular attention to lost or stuck HID state.

This was a read-only static audit. Per instruction, I did not build the tree or run the full test suite. Findings below are limited to issues with a concrete code path and reproduction; speculative Cirque protocol concerns and behaviors already present in the upstream base were excluded.

## Findings

### 1. HIGH — `rynk` without `storage` no longer compiles

**Evidence:** `rmk/src/input_device/mod.rs:17-18` enables `pointing_config` for every `rynk` build, but `rmk/src/input_device/pointing_config.rs:14-16` unconditionally imports `FLASH_CHANNEL` and `FlashOperationMessage`. Those symbols are storage-only (`rmk/src/channel.rs:17-18,90-92`; `rmk/src/lib.rs:119-120`). The feature definition does not make `rynk` imply `storage` (`rmk/Cargo.toml:137-140`), and `examples/use_rust/qemu-riscv-rynk/Cargo.toml:13` is a concrete supported configuration using `default-features = false, features = ["rynk"]`.

**Reproduction:** compile the QEMU Rynk example, or any crate depending on RMK with `default-features = false, features = ["rynk"]`. Name resolution reaches storage-gated imports that do not exist.

**Impact:** a supported feature combination is a compile-time regression; it also prevents using nonpersistent Rynk even though the rest of the host setters generally gate persistence independently.

**Confidence:** High (directly implied by mutually inconsistent `cfg` gates).

### 2. HIGH — persisted pointing configuration and per-layer modes are never wired into generated firmware

**Evidence:** `pointing_config::init` is the only path that seeds live state and applies stored configuration (`rmk/src/input_device/pointing_config.rs:53-62`). `PointingLayerModes` is the only subscriber that updates `STATE.layer` and reapplies on layer changes (`:106-118`). `Storage::read_pointing_config` exists at `rmk/src/storage/mod.rs:1274-1281`. A tree-wide search at the audited commit finds no caller of either `pointing_config::init` or `read_pointing_config`, and no instantiation of `PointingLayerModes`. The generated orchestrator initializes keymap/storage, registered processors, and sensor processors (`rmk-macro/src/codegen/orchestrator.rs:365-370,550-565`), but does not initialize either pointing-config component; the built-in/custom processor registry similarly has no entry (`rmk-macro/src/codegen/registered_processor.rs:12-92`).

**Reproduction:** on macro-generated Rynk firmware with storage, set a device default and a layer override via `SetPointingConfig`, reboot, then move the device and change layers. The persisted record is never read, and `STATE.layer` remains 0 because its subscriber never runs. In the current boot session a `SetPointingConfig` applies against layer 0 only; later layer changes do nothing.

**Impact:** the advertised persistence and layer-sensitive behavior are nonfunctional in the standard generated firmware path.

**Confidence:** High (all possible call/instantiation sites were searched at the exact tree).

### 3. HIGH — a failed pointing sensor can busy-loop and starve the executor

**Evidence:** after three initialization failures, `try_init` permanently sets `InitState::Failed` (`rmk/src/input_device/pointing.rs:90-123`). The event loop now immediately goes back through `wait_for_low` and `poll_once` (`:206-256`); `poll_once` returns immediately for `Failed` (`:130-133`). The personal diff removed the base's explicit `if self.init_state == InitState::Failed { pending::<()>().await; }` guard from this loop.

**Reproduction:** use a PMW-style device with a motion GPIO held low (for example, an absent/miswired sensor whose line is pulled low) and make initialization fail three times. `wait_for_low` is immediately ready, `poll_once` performs no await, and the loop repeats continuously.

**Impact:** one optional failed sensor can consume the cooperative executor and make scanning, HID, BLE, and other tasks unresponsive.

**Confidence:** High (direct async control-flow path; the trigger is exactly the case documented by the removed guard).

### 4. HIGH — Keypad-mode taps release all physically held keyboard keys/modifiers on the host

**Evidence:** the newly added Keypad mode calls `tap_key` for its tap gesture and motion directions (`rmk/src/input_device/pointing.rs:555-563`). `tap_key` bypasses `Keyboard` and sends standalone boot-keyboard reports; for ordinary keys those reports hard-code `modifier: 0`, include only the synthetic key on press, and then send an all-up report (`:644-670`). It therefore knows nothing about `Keyboard`'s `held_keycodes`, modifiers, one-shots, or Ctrl/GUI resolution.

**Reproduction:** hold `A` or physical Ctrl, then move/tap a pad configured in Keypad mode with an ordinary HID output such as an arrow. The synthetic press report omits the held state, and the synthetic release report clears the entire keyboard report. RMK's internal held state is unchanged, so it does not reassert the physical key until some later keyboard event happens.

**Impact:** normal chord state is lost at the host and can remain logically down only inside firmware, producing missed modifiers/keys and release/order anomalies. (Caret used this unsafe helper in the base; the regression here is extending it to the newly introduced Keypad mode.)

**Confidence:** High (HID keyboard reports are absolute state, and both emitted reports are explicit).

### 5. HIGH — independent full mouse reports break device-button, Drag, and Press ownership

**Evidence:** the pointing processor now owns device-originated buttons, a drag latch, and touch presence (`rmk/src/input_device/pointing.rs:374-400`) and emits a complete `MouseReport` from that state plus `keymap.mouse_buttons()` (`:460-546`). Keyboard mouse keys independently own another complete report (`rmk/src/keyboard.rs:1878-1906,2266-2273`; `rmk/src/keyboard/mouse.rs:299-311`). The keyboard report has no access to the pointing processor's `device_buttons`, `drag_latched`, or `touching`. Generated multi-sensor setups also instantiate a separate processor per sensor (`rmk-macro/src/codegen/input_device/pmw3610.rs:162-179`), so the same collision exists between sensors.

**Reproduction:** latch Drag on a pad (or hold a Cirque/device button, or maintain touch in Press mode), then press/release a keyboard mouse key or let a held mouse-movement key repeat. The keyboard's next full mouse report omits the pad-owned bit and releases it at the host. A later pad event may reassert it, creating a broken drag/click. With two processors, an event from sensor B similarly omits sensor A's held button.

**Impact:** persistent mouse-button state is spuriously released, causing dropped drags, fragmented clicks, and button flicker.

**Confidence:** High (all producers send absolute reports to the same HID interface; ownership is asymmetric and unmerged).

### 6. MEDIUM — the documented all-device pointing processor shares per-device state and mode

**Evidence:** `PointingProcessorConfig::default()` selects `ALL_POINTING_DEVICES` (`rmk/src/input_device/pointing.rs:341-361`), but one processor has only one accumulator, `current_mode`, `device_buttons`, `drag_latched`, and `touching` (`:374-400`). Every accepted sensor packet overwrites that shared state (`:460-471`), and a mode event for any device overwrites the one shared mode (`:568-589`). The new Cirque driver explicitly describes paired pads (`rmk/src/input_device/cirque_pinnacle.rs:1-8`), making the default API a realistic multi-device path.

**Reproduction:** attach two devices to one default/all-device processor. Configure different modes for IDs 0 and 1: whichever mode event arrives last applies to both. Hold a button/touch on device 0, then deliver a zero-button packet from device 1: device 0's state is overwritten and the combined report releases it.

**Impact:** the public/default all-device mode cannot safely implement per-device live configuration or simultaneous device state.

**Confidence:** High for the state collision; Medium severity because macro-generated PMW configurations normally avoid the default by creating one processor per ID.

### 7. MEDIUM — Unicode input loses already-held modifiers; Linux mode also breaks under Ctrl/GUI swap

**Evidence:** Linux and macOS Unicode temporarily OR modifiers into `held_modifiers`, then clear those same bits (`rmk/src/keyboard.rs:2001-2036`). Modifier storage is a bitset, not reference-counted (`:2335-2359`), so the temporary owner cannot distinguish its bit from an already-held physical or action-owned modifier. Windows has the same problem through `tap_hid_key(RAlt)` (`:2038-2043,2063-2071`). Separately, every keyboard report swaps Ctrl/GUI after resolving modifiers (`:2208-2219`), so Linux's semantic Ctrl+Shift+U opener is emitted as GUI+Shift+U whenever swapping is active.

**Reproduction:** (a) physically hold LCtrl or LShift and invoke a Linux Unicode action, hold LAlt for macOS, or RAlt for Windows; when the sequence removes its temporary modifier, it also removes the physical owner and reports it up even though it is still held. (b) enable Ctrl/GUI swap and invoke Linux Unicode; its opener becomes GUI+Shift+U and the OS does not enter Unicode input.

**Impact:** Unicode actions can lose a physical modifier until it is released/re-pressed, and Linux Unicode is unusable with the new swap option.

**Confidence:** High for both code paths.

### 8. MEDIUM — generated Morse profiles erase an explicit `unilateral_tap = false`

**Evidence:** code generation correctly passes `Some(false)` into `MorseProfile::new` (`rmk-macro/src/codegen/action_parser.rs:108-112,144-151`), which encodes it as the shared explicit-false bit pattern (`rmk-types/src/morse.rs:330-343`). The immediately chained `.with_opposite_hand_hold(None)` then clears that pattern: it preserves only the `Some(true)` unilateral pattern and maps the shared explicit-false pattern to zero/`None` (`rmk-types/src/morse.rs:119-162`). The runtime subsequently falls back to the global default when it reads `None` (`rmk/src/keyboard/morse.rs:413-432`).

**Reproduction:** set global/default `unilateral_tap = true`; define a named profile with `unilateral_tap = false` and omit `opposite_hand_hold`; assign it to a tap-hold. The generated profile reads back as `unilateral_tap() == None`, so the key inherits global `true` instead of overriding it with `false`.

**Impact:** a documented per-profile false override silently reverses behavior. The packed representation's shared false code is specifically documented as supporting this override (`rmk-types/src/morse.rs:140-145`).

**Confidence:** High (deterministic constructor/builder bit transition).

### 9. MEDIUM — combo interception lets a Retro Tap survive another physical key press

**Evidence:** `process_inner` asks `process_combo` first and calls `process_key_action` only if a key action is returned (`rmk/src/keyboard.rs:397-423`). A press that is a component of an incomplete combo is stored as `WaitingCombo` and returns `None` (`:1301-1312,1341-1343`). Retro-tap candidates are demoted only inside `process_key_action` (`:426-443`), so such a physical press is invisible to the new "any other key" rule. Releasing a still-undemoted candidate retracts its already-emitted hold and emits the tap (`rmk/src/keyboard/morse.rs:253-263`).

**Reproduction:** hold a retro-tap home-row modifier beyond its hold timeout, press one key that participates in a combo but do not complete/release that combo, then release the home-row modifier. Despite another physical key being down, RMK releases the modifier hold and emits the tap.

**Impact:** Retro Tap produces the wrong key and retracts a modifier in exactly the chorded case where the hold should be retained.

**Confidence:** High (the incomplete-combo return bypasses the sole demotion loop).

### 10. MEDIUM — Rynk accepts pointing values that can stall or panic/wrap the processor

**Evidence:** `pointing_config::replace` validates only the revision before persisting and applying the entire host-provided structure (`rmk/src/input_device/pointing_config.rs:69-87`). Two concrete unsafe value families are exposed:

* `CaretConfig.threshold` is signed and unconstrained (`rmk-types/src/pointing.rs:95-135`). For `threshold <= 0`, `compute_caret_taps` cannot make normal progress and runs until its saturating 255-iteration escape (`rmk/src/input_device/pointing.rs:680-737`); its caller then emits 255 taps, each with two 5 ms delays (`:548-552,644-653`), monopolizing that processor for about 2.55 seconds per motion event.
* Scroll/Sniper multiplication uses ordinary `i16` multiplication before `saturating_add` (`rmk/src/input_device/pointing.rs:288-337`). The wire permits a multiplier of 255, and Cirque produces deltas through +/-255 (`rmk/src/input_device/cirque_pinnacle.rs:158-172`), so `255 * 255 = 65025` overflows `i16`: debug/overflow-checked firmware panics, while unchecked firmware wraps and can reverse/corrupt motion. Cursor mode already uses `saturating_mul` (`rmk/src/input_device/pointing.rs:593-605`).

**Reproduction:** use `SetPointingConfig` with Caret threshold 0 or -1 and move the pad; separately set Scroll/Sniper multiplier to 255 and deliver delta 255.

**Impact:** an accepted, persisted host configuration can cause multi-second report storms or a firmware panic/wrong-direction motion.

**Confidence:** High for the arithmetic/control flow; Medium severity because it requires an extreme or invalid host configuration.

### 11. MEDIUM — replacing an active combo can strand its output down

**Evidence:** the new Rynk combo setters directly replace the runtime `Combo`, including its `state` and `is_triggered` fields, without quiescing or releasing the old instance (`rmk/src/host/context.rs:178-200,221-249`; endpoints at `rmk/src/host/rynk/handlers/combo.rs:31-38,79-86,105-124`). Combo output release is generated only when the live instance's pressed-state reaches zero while `is_triggered` is still true (`rmk/src/keyboard/combo.rs:137-166` and `rmk/src/keyboard.rs:1345-1390`).

**Reproduction:** trigger and continue holding a combo whose output is an ordinary key or modifier; send `SetCombo` or `SetComboDefinition` for that slot (even replacing it with an identical definition); then release the physical combo keys. The new instance starts with state 0/`is_triggered = false`, so no release for the already-reported old output is emitted.

**Impact:** the host key/modifier remains stuck until another reset/recovery report. This mutation hazard exists in the newly exposed Rynk configuration path even though analogous dynamic mutation patterns may exist in older host protocols.

**Confidence:** High for the state transition; Medium severity because it requires reconfiguration while the combo is active.

## Architecture recommendations

1. **Give each HID interface one state owner/composer.** Keyboard keys/modifiers and mouse buttons should be owned by central arbiters. Producers (matrix actions, Unicode, macros, pointing sensors, Drag/Press, keyboard mouse keys) should submit owner-scoped presses/releases or deltas; only the arbiter should emit absolute reports. Owner tokens/refcounts also solve same-bit modifier overlap and make cleanup deterministic.

2. **Make pointing state per device, then aggregate.** Keep mode, accumulator, raw buttons, touch, and latch keyed by device ID. Combine all persistent button owners once, while summing/clamping transient motion. Either remove `ALL_POINTING_DEVICES` for stateful processing or implement it as a collection of per-device slots.

3. **Treat feature lifecycle wiring as generated, testable metadata.** A feature that declares a service should register its required initializer, stored-state loader, and processor task in one place. Make feature dependencies explicit (`rynk + pointing config` either requires storage or compiles a nonpersistent path) and compile-check the feature matrix, especially `rynk` with and without storage.

4. **Validate and normalize wire configuration before mutation/persistence.** Reject nonpositive Caret thresholds, use widened/saturating arithmetic for all ratios, validate button masks/device uniqueness as appropriate, and only persist after the full structure passes validation. Strong bounded types would make invalid ratios/thresholds unrepresentable downstream.

5. **Define a quiescence protocol for live behavior mutation.** Before replacing combos/Morse/keymap state, either reject mutation while affected inputs are active or ask the HID arbiter to release outputs owned by the old definition and clear matching held-buffer/runtime state atomically.

6. **Test interactions at absolute-report boundaries.** Add focused tests for held keyboard key + Keypad tap, device button/Drag + keyboard mouse repeat, two simultaneous pointing devices, Unicode + already-held modifiers + Ctrl/GUI swap, retro-tap + incomplete combo, active-combo replacement, no-storage Rynk compilation, and reboot/layer application of stored pointing config.

## Prioritization

Fix 1-5 before relying on this revision for daily firmware: they cover a supported build break, a wholly unwired configuration feature, executor starvation, and deterministic loss of held HID state. Findings 6-11 are narrower but should be resolved before exposing live pointing/combo configuration broadly.
