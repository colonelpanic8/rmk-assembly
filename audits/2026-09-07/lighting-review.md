# RMK lighting audit

Scope: read-only review of `9a88399366a01b344cf90fb3723cae68dc4dc949` against base `f8da2742971d048e0080e57d69d98f00a859e4e9`, focused on `rmk/src/lighting`, lighting config/codegen, topology, output, conditional evaluation, and wake behavior. All reviewed lighting files are absent from the base tree, so every finding below is introduced by the assembled change set; blame is included to identify the narrower topic commit where useful. I did not review the Rynk transaction/persistence paths owned by the main audit.

## Findings

### F1 — P1: `powered_only_scope = "local"` is accepted and advertised but has no runtime effect

**Locations:**

- `rmk/src/lighting/source.rs:286-294` defines the documented Authority/Local behavior.
- `rmk-config/src/resolved/lighting.rs:332-335` accepts and preserves the TOML choice.
- `rmk-macro/src/codegen/lighting.rs:240-245` emits the choice into `LIGHTING_CONTROLS`.
- `rmk/src/lighting/standard/engine.rs:1266-1297` decides `PoweredOnly` solely from `context.powered` and never reads `self.controls.powered_only_scope`.
- Repository-wide references to `powered_only_scope` outside config/codegen are readback only (`rmk/src/host/rynk/lighting.rs:424-428`); there is no behavioral consumer.

**Trigger:** configure a split renderer with `initial_output_mode = "powered_only"` and `powered_only_scope = "local"`, then put authority and peripheral on different VBUS states (for example, central attached to USB and a wireless half on battery).

**Actual:** every engine evaluates the one `LightingContext.powered` bit it is handed. The scope enum cannot select local VBUS, because the engine has neither a branch on the enum nor separate authority/local power inputs. With the replicated authority context described by `StandardReplicaState`, a battery-powered peripheral can remain lit whenever the authority is powered. This defeats the setting and can materially increase peripheral battery drain. A caller could manually substitute another context, but then the emitted/advertised setting still does nothing and Authority cannot be selected from the same input model.

**Expected:** Authority makes replicas follow authority VBUS; Local makes each renderer use its own node's VBUS, as the public enum says.

**Evidence/ownership:** `rg powered_only_scope` finds only definition, config lowering, codegen, tests, and Rynk readback—no renderer behavior. Narrow blame points to `faac84f7f5` (`feat(lighting): support local powered-only scope`). **Introduced**, not inherited.

**Recommendation:** model both authority and local power in the renderer snapshot (or provide an explicit local-power callback) and select between them in one engine helper used by render and `state()`. Add a split test with opposite authority/local states; the existing test at `rmk/src/lighting/standard/tests.rs:572-635` sets `PoweredOnlyScope::Local` but never distinguishes the two sources.

### F2 — P1: compiled conditional scenes with `output_mode` silently never match

**Locations:**

- `rmk-config/src/lib.rs:761-768` exposes `output_mode` on TOML conditional scenes.
- `rmk-config/src/resolved/lighting.rs:638-647` resolves it, and `rmk-macro/src/codegen/lighting.rs:354-379` emits it into a compiled `ConditionSet`.
- `rmk/src/lighting/source.rs:358-362` deliberately rejects an output-mode predicate unless the source supplies `Some(current_mode)`.
- `rmk/src/lighting/source.rs:466-503` hard-codes `None, None` in all compiled `ConditionalScenes` matching paths.
- `rmk/src/lighting/standard/engine.rs:1315-1320,1364-1370` passes live mode/effect state only to the runtime conditional table; the generic compiled status source has no path to receive either.

**Trigger:** put `output_mode = "always_on"` (or another mode) in `[[lighting.conditional_scene]]` in `keyboard.toml`, wire the generated `LIGHTING_CONDITIONAL_SCENE_CELLS` through `ConditionalScenes` as the standard engine's status source, and render while that mode is active.

**Actual:** `ConditionSet::matches` receives `output_mode == None`, so `None != Some(wanted)` and the cell is filtered out on every render. Config resolution and code generation succeed; the rule simply never lights. Runtime-authored conditional scenes do work, which makes the same logical condition behave differently depending on whether it came from TOML or Rynk.

**Expected:** a compiled conditional scene should observe the same engine-owned mode as its runtime counterpart, or config resolution should reject conditions the compiled source cannot evaluate.

**Evidence/ownership:** this is a direct control-flow proof; the current compiled-conditional test at `rmk/src/lighting/standard/tests.rs:2523-2621` covers only a layer predicate, while `output_mode_conditions_select_between_runtime_rules` at `:2288` covers only the runtime table. Narrow blame points to `0514ac89c` for config support and `3439c9cf4` for the unconditional `None` call. **Introduced**, not inherited.

**Focused regression test:** instantiate `StandardLightingEngine<EmptySource, ConditionalScenes<...>>` with one compiled cell gated on `Some(OutputMode::AlwaysOn)`, leave the engine in its default AlwaysOn mode, render, and assert that cell's color. It currently falls through to the lower band.

**Recommendation:** make compiled conditional scenes engine-owned, or add a render-policy wrapper that injects `output_mode` and `effects_enabled` into both compiled and runtime sources. The generic `LightingSource` boundary cannot currently observe engine policy, which is the architectural cause.

### F3 — P2: a lingered wake frame records the unmodified context as its provenance

**Locations:**

- `rmk/src/lighting/standard/engine.rs:1268-1293` synthesizes a snapshot whose layer set includes released wake layers during the linger window.
- `rmk/src/lighting/standard/engine.rs:1307-1310` records `context: *context`, where `context` is the original snapshot captured before synthesis.
- `rmk/src/lighting/standard/engine.rs:1353` renders from `snapshot`, which may be the synthesized linger snapshot.
- `rmk/src/lighting/standard/command.rs:204-215` promises that `PresentedFrame.context` is the context the frame was rendered from and specifically uses it for split stale-replica diagnosis.

**Trigger:** use a wake layer with nonzero `wake_linger_ms`, press it, release it, and let the post-release frame present during the linger interval.

**Actual:** layer scenes and conditional sources render as if the released layer remains active, but `state().presented.context.layers` says it is inactive. Host diagnostics and split replica-health comparisons therefore cannot explain the visible frame and may report a misleading provenance match/mismatch.

**Expected:** `PresentedFrame.context` contains `*snapshot.lighting_context()`—the possibly linger-adjusted context actually supplied to sources—or the API explicitly carries authoritative and effective-render contexts separately.

**Evidence/ownership:** the existing linger test at `rmk/src/lighting/standard/tests.rs:2231-2285` verifies pixels and deadlines but never calls `on_presented` or checks provenance. This is an interaction between provenance commit `a8155976b` and linger commit `7e7ed9a3d`. **Introduced**, not inherited.

### F4 — P3: lighting config can panic while lowering wake layers instead of rejecting the unsupported layer count

**Locations:**

- `rmk-config/src/resolved/lighting.rs:312-321` checks only `layer < keymap.layers`, then evaluates `1 << layer` into a `u64`.
- `rmk/src/lighting/rmk_state.rs:23-30` establishes that standard lighting supports at most 64 layers.
- `rmk-config/src/resolved/lighting.rs:537-578` likewise permits static scene layers based on the keymap's `u8` count without enforcing the lighting limit.

**Trigger:** configure more than 64 keymap layers and include a wake layer numbered 64 or above. The layer passes `layer < keymap.layers`; the `u64` shift is out of range and panics under checked overflow during macro/config resolution. With no such wake layer, codegen can still emit lighting that `KeymapLightingState::new` later rejects.

**Expected:** lighting resolution returns a clear `Err` that its active-layer representation is limited to 64 layers.

**Evidence/ownership:** direct numeric-bound mismatch. **Introduced**, not inherited. This is low priority because keyboards rarely approach 64 layers, but config handling should not turn valid `u8` keymap metadata into a procedural-macro panic.

## Architecture, memory, and performance

The service/output ownership split is sound: rendering is synchronous and deterministic, a frame is committed only after successful presentation (`rmk/src/lighting/service.rs:456-474,506-540`; `rmk/src/lighting/standard/engine.rs:1398-1401`), and zero brightness suppresses animation deadlines (`rmk/src/lighting/output.rs:248-263`). Topology validation also covers the important bijection invariants—duplicate semantic IDs/routes/physical addresses, missing logical routes, and holes in complete outputs (`rmk/src/lighting/topology.rs:402-612`). I found no source-verified topology-validation correctness defect.

The main design problem is that engine-owned policy is split from a generic status `LightingSource`; F2 is a direct consequence. Conditions that depend only on the snapshot work in both compiled and runtime tables, while engine state works only in the runtime path. One engine-owned conditional pipeline with immutable and mutable tables would remove that behavioral fork.

Hot-path complexity is unnecessarily quadratic in configured cells/routes:

- `ValidatedRouting::visit_slice` loops physical pixels and linearly searches every route (`rmk/src/lighting/output.rs:74-105`), making each presentation O(outputs × pixels × routes). Store routes in physical order, or emit a direct physical-index-to-slot table during codegen.
- `Compositor::apply` asks each source for its length, validates every slot, then asks for every slot again (`rmk/src/lighting/compositor.rs:277-309`). `ConditionalScenes::cell_at` rescans from the start for each index and is invoked in both slot and contribution phases (`rmk/src/lighting/source.rs:466-503`), yielding multiple O(m²) scans per animated frame. A cursor/iterator returning `(slot, contribution)` in one pass, with static sources prevalidated at installation, would reduce CPU and simplify the contract.

Memory is also front-loaded into fixed generic capacities. Every render constructs `[Option<u64>; N]` deadlines on the task stack (`rmk/src/lighting/compositor.rs:231-245`), in addition to working and committed RGB frames. Each runtime scene family carries a 64-entry `BuiltinEffect` style pool (`rmk/src/lighting/standard/scenes.rs:65-100`), and transaction staging/replica snapshots copy whole fixed-capacity tables (`rmk/src/lighting/standard/engine.rs:79-86`; `rmk/src/lighting/standard/command.rs:443-464`). On small Cortex-M targets these costs should be measured with the board's actual `N`, `SCENE_CAP`, task stack, mailbox, and replica-slot configuration. Prefer caller-owned/shared staging, compact deadline deltas where time horizon permits, and board-specific capacities with compile-time size reporting.

No full test suite was run. Findings are source-verified against the named commit; the focused conditional test above is the highest-value immediate regression test.
