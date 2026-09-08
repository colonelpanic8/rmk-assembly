# RMK assembled input-path audit

Scope: assembled commit `9a88399366a01b344cf90fb3723cae68dc4dc949`, compared with base `f8da2742971d048e0080e57d69d98f00a859e4e9`. This was a read-only source and history audit of keyboard, Unicode, Control/GUI swap, combo, Morse, and pointing paths. I did not duplicate the already-reported Unicode-mode boot restoration or Morse maintenance-gate findings.

## Concrete findings

### 1. High — Unicode emission corrupts modifier ownership, and Control/GUI swap rewrites the Linux protocol chord

**Evidence:** `rmk/src/keyboard.rs:2001-2044` injects the Linux opener with `register_modifiers(LCtrl | LShift)` and later `unregister_modifiers`, and does the same with `LAlt` for macOS. Those helpers mutate the same bitset used for physical modifiers (`rmk/src/keyboard.rs:2343-2359`); it records no per-source ownership or count. `build_keyboard_report` then rewrites every resolved Control/GUI bit when swap is active (`rmk/src/keyboard.rs:2208-2219`).

**Trigger and effect:**

* Hold physical Left Control or Left Shift and press a `UNICODE(n)` key in Linux mode. Registration cannot distinguish the already-held bit from the synthetic opener. Unregistration clears it, so the host receives an early release even though the physical key is still held. The later physical release cannot restore the lost state. The same failure affects a physically held Left Alt in macOS mode.
* Enable Control/GUI swap and invoke a Unicode key in Linux mode. The required `Ctrl+Shift+U` report becomes `GUI+Shift+U`, so the input method never opens. The swap is appropriate for user key semantics but is also being applied to an internal wire-protocol sequence.

This is an assembled-only cross-topic interaction. Unicode emission arrived in `2c04fa74b`; Control/GUI swap arrived in `7c6623be1`; neither exists at the base revision.

**Repair direction:** Give synthetic protocol inputs scoped ownership separate from physical modifier state and render the union into reports. Protocol-level chords should be expressed after user remapping, or explicitly marked as physical HID usages that bypass Control/GUI swap. On completion, restore the report derived from live physical/logical state rather than clearing shared bits.

### 2. High — Pointing-generated keyboard taps overwrite the keyboard engine's HID state

**Evidence:** `rmk/src/input_device/pointing.rs:548-563` calls `tap_key` for Caret and Keypad gestures. `tap_key`/`report_for_keycode` (`rmk/src/input_device/pointing.rs:644-671`) sends a complete keyboard report containing only the generated key, followed by a completely empty report. It does not consult `Keyboard`'s held keycodes, modifiers, one-shots, macro state, or Control/GUI mapping. USB HID keyboard reports are state snapshots, not incremental events.

**Trigger and effect:** Hold any normal key or modifier while a Caret/Keypad gesture emits a keyboard-page key. The pointing press report tells the host that every held keyboard key and modifier was released; the following empty report releases the generated key as well. Since `Keyboard` still believes the physical keys are held, it need not emit another report, leaving host and firmware state divergent until some unrelated keyboard event. Generated Control/GUI keycodes also bypass the configured swap.

The underlying Caret implementation is present at the base revision, so this is inherited architectural debt rather than a wholly new regression. The assembled tree broadens the trigger surface: `22be77335` adds Keypad motion and primary-button taps through the same path, while `7c6623be1` adds mapping state that this path bypasses.

**Repair direction:** Pointing code should publish virtual key press/release events to the keyboard state owner. Only that owner should construct and send `KeyboardReport`s. If latency requires a direct path, it still needs a shared report-state arbiter that merges sources and re-emits the authoritative aggregate on release.

### 3. Medium — Deleting an empty position combo is undone into a different runtime representation after reboot

**Evidence:** `ComboDefinition::is_empty` canonicalizes either empty variant as deletion (`rmk-types/src/combo.rs:127-155`). `set_combo_definition` therefore stores `None` in RAM, but persists the submitted tagged variant (`rmk/src/host/context.rs:221-250`). Boot restoration unconditionally turns a stored `PositionCombo` into `Some(Combo::new_positions(config))`, even if it is empty (`rmk/src/host/storage.rs:125-137`). The legacy single getter rejects every position variant, and the legacy bulk getter rejects the entire page if any position variant occurs (`rmk/src/host/rynk/handlers/combo.rs:16-27,41-61`). The engine itself ignores size-zero combos (`rmk/src/keyboard/combo.rs:110-115`), so this does not stall key processing.

**Trigger and effect:** Delete a slot through `SetComboDefinition` using `ComboDefinition::Positions(PositionCombo::empty())`. Before reboot the slot is `None`, `GetCombo` returns legacy `Combo::empty()`, and definition reads return the canonical empty action variant. After reboot it becomes `Some(Positions(empty))`: `GetCombo` returns `Invalid`, and any `GetComboBulk` page containing the slot fails in full. The newer getter also exposes a different variant before and after reboot.

This path is introduced by the position-combo topic (`85962a807` / assembled merge `0cedaae50`); the tagged persisted variant and boot arm do not exist at base.

**Repair direction:** Normalize empty values at both storage boundaries. Prefer an explicit deletion/tombstone operation, or treat empty `Combo` and empty `PositionCombo` as `None` during boot. Persisting a delete in a type-preserving tagged payload conflicts with the runtime's canonical `Option` model.

### 4. Medium — `SetPointingConfig` accepts and persists structurally invalid configurations

**Evidence:** The handler forwards directly to `pointing_config::replace` (`rmk/src/host/rynk/handlers/pointing.rs:19-22`). `replace` checks only the optimistic revision before committing to RAM and flash (`rmk/src/input_device/pointing_config.rs:69-87`). The payload helpers silently clamp over-capacity counts and resolve duplicates with first-match searches (`rmk-types/src/protocol/rynk/payload/pointing.rs:94-139`). `apply` iterates only declared devices (`rmk/src/input_device/pointing_config.rs:90-103`).

**Trigger and effect:** A host can submit `device_count`/`override_count` beyond capacity, duplicate device IDs, duplicate `(layer, device_id)` overrides, or an override for a device absent from `devices`. The device acknowledges and persists the write, but silently truncates counts, makes later duplicates unreachable, or never publishes the orphan override. `GetPointingConfig` then returns an accepted configuration whose serialized fields do not describe the policy actually applied. This is particularly hard for an editor to recover from because the revision advances and the invalid value survives reboot.

The runtime pointing-config payload, handler, and state module are all absent at base and were introduced by `447c569ce` and `8b95f9259`.

**Repair direction:** Validate the complete payload before changing state or flash: counts within capacities, unique device IDs, unique `(layer, device_id)` overrides, and override references to declared devices. Return `RynkError::Invalid` without incrementing the revision on any failure. If orphan overrides are intended, document and implement how they become active without an associated device entry.

### 5. Medium — `UNICODE(n)` configuration permits dangling references that compile into dead keys

**Evidence:** The key-action parser accepts every syntactically valid `u16` and emits `Action::Unicode(index)` (`rmk-macro/src/codegen/action_parser.rs:319-325`). Behavior resolution validates codepoint strings but does not cross-check keymap references (`rmk-config/src/resolved/behavior.rs:349-363,384-391`). If `[behavior.unicode]` is absent, codegen emits the default empty table (`rmk-macro/src/codegen/behavior.rs:639-655`). Runtime lookup then warns and returns without a key event (`rmk/src/keyboard.rs:2001-2005`).

**Trigger and effect:** `UNICODE(0)` with no Unicode section, or `UNICODE(12)` with fewer than thirteen codepoints, produces valid firmware but a permanently inert key. The parser's own diagnostic says the argument is an index into that table, so accepting a provably absent element is inconsistent with the configuration contract.

Unicode actions and behavior tables are absent at base and were introduced by `2c04fa74b`.

**Repair direction:** During resolved-config validation, walk all keymap actions recursively (including tap-hold and similar wrappers), find the maximum Unicode index, and reject missing or out-of-range references with the layer/key position in the diagnostic.

## Architectural proposals tied to observed state paths

1. **Use one event-driven keyboard reducer and one report owner.** Physical matrix input, pointing-generated taps, Unicode protocol sequences, macros, combos, and remapping should submit typed events carrying source identity. A reducer should own physical state, virtual/synthetic state, and user transformations, then emit the aggregate report. This directly removes the split ownership behind findings 1 and 2.

2. **Separate durable definitions from active gesture instances.** Runtime setters replace `Combo` objects in place (`rmk/src/host/context.rs:178-250`) even though those objects contain live `state` and `is_triggered` fields (`rmk/src/keyboard/combo.rs:20-29`). Maintenance mode gates the command but does not quiesce keyboard processing. Replacing or deleting a triggered combo before its physical releases discards the only state that knows which output must be released, risking a stuck output. Keep the active instance/snapshot until its gesture unwinds, or reject definition changes while the affected input state is active. Apply the same generation/snapshot rule to Morse profiles and pointing modes where a press begins under one definition and releases under another.

3. **Keep per-device pointing state even for an all-device subscriber.** `PointingProcessorConfig::default` subscribes to all device IDs (`rmk/src/input_device/pointing.rs:341-360`), but one processor stores only one accumulator, `device_buttons`, drag latch, and touch flag (`rmk/src/input_device/pointing.rs:374-400,460-471`). Alternating events from two devices therefore treat device B's buttons as transitions from device A's prior buttons and combine motion remainders across devices; Keypad rising-edge taps and Drag/Press state make this especially visible. Index this transient state by device ID, or instantiate one processor per device and reserve aggregation for finalized mouse reports.

4. **Centralize validate-normalize-commit for host configuration.** Each Rynk write should validate the full candidate, normalize empty/tombstone representations, preserve active-state invariants, then atomically update RAM and queue persistence. The current paths perform different subsets of those steps, which is why pointing accepts unreachable records and position-combo deletion changes meaning across reboot.

## Validation notes

I verified each introduction claim with exact `git show`/`git diff` comparisons against the requested base and inspected the relevant assembled commits. I did not modify source or run the full suite. The findings above follow directly from state transitions visible in the pinned source; there were no existing focused tests that exercise concurrent physical modifiers during Unicode emission, mixed keyboard/pointing reports, empty position-combo reboot restoration, or rejected malformed pointing payloads.
