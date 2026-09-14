# Runtime board-status lighting: one predicate encoding instead of a fourth generation

Design audit written 2026-09-14 against: rmk-assembly `master` (base
`8b4d1b31`, entry `feat/lighting-layer-indicator-conditions` at `94684603`),
`assembled` at `01178c2f9`, moergo-rmk `0255975`, glove80-config `53a7c03`.
Read-only: nothing here has been implemented. Line numbers refer to those
revisions.

## 1. Decision

**Delete `BoardStatus` and its two hard-coded indicators, and replace the
generation-versioned conditional-scene cell with one self-describing rule
encoding: a rule is an LED, an effect, and an ordered list of tagged,
length-prefixed predicates.** Maintenance state and split-transport state become
two new predicate tags and two new `LightingContext` fields. Every existing
predicate (layer, battery, output mode, connection, effects, layer set,
indicators) becomes a tag in the same list. One new capability bit says the rule
endpoints exist; a bitmask in the rule status says which tags this firmware
parses. After this, adding a predicate costs one wire struct, one tag constant,
one `ConditionSet` field with its match arm, one TOML field, and one editor
control. It costs no opcodes, no capability bit, no storage version, no split
packet, and no host client change.

Justification in one paragraph. The generation approach has now been paid for
three times, and the third payment was incomplete: the `layers` and `indicators`
predicates added by `feat/lighting-layer-indicator-conditions` never cross the
split link (`split_lighting.rs:1181-1185` sets both to `None`; `walk_snapshot`
at `:1762-1772` only emits the trailer packet for connection and effects) and
are not covered by the replica digest (`hash_conditions`, `:1468-1527`, hashes
five of the seven fields; `LIGHTING_REPLICA_DIGEST_SCHEMA_V2` was minted for
this and is used nowhere). That is what "each generation costs the same again"
looks like in practice: the cost is not only six opcodes, it is six places that
must each be widened by hand, and one of them was missed. A fourth generation
would cost roughly what the third did (the two commits `c3325cf73` and
`1cad3a6a8` are about 1,200 changed lines across rmk-types, rmk, rynk,
rynk-wasm, storage and docs, before moergo-config, moergo-control and
rynkbench) and would leave the fifth predicate at the same price. The
self-describing encoding costs about the same once, fixes the split-link and
digest gaps as a side effect because both consume the same canonical bytes, and
makes the marginal predicate nearly free. The in-RAM representation stays the
fixed `ConditionSet` struct, so the embedded evaluation path and its cost model
do not change; only the wire, storage, and split encodings do.

### 1.1 What is genuinely compile-time and what is merely conventionally compiled

The owner's constraint separates cleanly once "compiled" is split into three
categories.

**Hardware topology, correctly compiled.** These describe the board and cannot
be edited into existence at runtime: `[lighting] topology_revision`,
`[[lighting.zone]]`, `[[lighting.output]]`, `[[lighting.emitter]]` (stable LED
ids, key mapping, node/output/physical index), `BOARD_LEDS_PER_HALF`,
`BOARD_CHANNEL_CEILING`, `BOARD_KEEP_LED_POWER_*`, the SPI/PWM pins, and
`BOARD_SCENE_CAPACITY` (a RAM budget, see section 6). The `[event.*]` channel
sizes in `keyboard.toml` are in this category too. Nothing changes here.

**Factory defaults, acceptably compiled.** Values the firmware boots into only
until the host writes something, where the host can replace or clear them
completely: `[lighting.background]`, `[lighting.controls].wake_layers` (already
overridden by the persisted `LightingWakeLayers` record, `lighting.rs:709`),
`initial_output_mode` (already overridden by the persisted `LightingOutputMode`
record, `:712`), the PaletteFX `DEFAULT_EFFECT*` constants (`:589-592`, already
overridden by the persisted extension records), and `LayerPolicy::ActiveStack`.
These hold no behaviour a user cannot remove. They stay compiled. One gap worth
noting: brightness, `output_enabled`, and the background are runtime-settable
but not persisted (no storage record exists for them), so after a reboot they
revert to the compiled default until `just apply` runs. That is a separate
follow-up, not part of this design.

**Behaviour, wrongly compiled.** Anything a user can see on the board and cannot
delete from any configuration. Today that set is exactly `BoardStatus`
(`crates/moergo-rmk/src/lighting.rs:90-195`) with its `MAGIC_LAYER = 2`,
`BOARD_MAINTENANCE_LED`, `BOARD_SPLIT_TRANSPORT_LED`, and four colour
constants. The compiled output-mode indicator deleted today was the other
member. The compiled `[[lighting.conditional_scene]]` mechanism
(`LIGHTING_CONDITIONAL_SCENE_CELLS`, wired via `with_conditional_scenes` at
`central_lighting.rs:376` and `ConditionalScenes::new` at `lighting.rs:724`)
is now empty on both boards and should stop being wired on MoErgo boards so it
cannot quietly refill. `[lighting.controls].output_mode_indicator` is the
compiled twin of what was deleted today and is unused by both boards; leave the
RMK-generic schema alone but never set it. `wake_linger_ms` and
`powered_only_scope` are policy that remains compiled; neither board sets them
and neither has a user-visible undeletable effect, so they are out of scope
here and noted as follow-ups.

So the constraint leads exactly where the task suspected: the whole board
status story becomes runtime data, and the compiled lighting surface reduces to
topology plus factory defaults. It does not lead to making the emitter table
or the capacity constant runtime, and this document does not propose that.

### 1.2 Things found on the way that shape the design

- **F-A. Split link drops two predicates.** Described above. A right-half rule
  gated on a layer set or a lock indicator renders on the peripheral as if
  that gate were absent. The current `config/glove80.toml` uses neither on a
  right-half LED (its F1-F5 layer indicators are plain layer scenes), so it is
  latent, not live. The design fixes it by carrying predicates as the same
  canonical bytes on the split link.
- **F-B. Digest does not cover two predicates.** Two replicas whose right-half
  rules differ only in `layers` or `indicators` attest identical digests.
  Fixed by hashing the canonical predicate bytes and reporting schema V2.
- **F-C. The peripheral evaluates maintenance from its own state.**
  `BoardStatus::contribution` calls `rmk::state::maintenance_mode_enabled()`
  on whichever half renders. The peripheral never runs
  `initialize_maintenance_mode` (only `host/rynk/mod.rs:111` does), so its
  answer is the uninitialized default. Harmless today only because both
  boards' indicator LEDs sit on the central half (Glove80 LED 12, Go60 LEDs 8
  and 6, all in node 0). The design replicates the state instead.
- **F-D. The split-transport indicator is Go60-only.** Glove80's
  `keyboard.toml` has `[split] connection = "ble"` and no `detect_pin`, so
  `selector::initialize` never runs and `auto_enabled()` is false; the
  `split_transport_visible` gate is permanently false there. The hardware
  observation (LED 12 reads rgb(0,128,0) with layer 2 active) is the
  maintenance indicator alone. The "two readings stacked on one LED" is a
  Glove80 coincidence of two board constants, not a rendering conflict.
- **F-E. Boot copies the whole persisted rule table onto the stack.**
  `crates/glove80-rmk/src/central.rs:71-76` builds a
  `heapless::Vec<LightingAdvancedConditionalSceneCell, SCENE_CAPACITY>` at boot.
  The new install path streams shards straight into the engine and removes
  this transient.

## 2. Wire design

All new types go in `rmk-types/src/protocol/rynk/payload/lighting.rs` (or a
sibling `lighting_rules.rs` re-exported from the same module) using the
existing `wire_type!` macro so they pick up `Serialize`, `Deserialize`,
`MaxSize`, `defmt::Format`, and `tsify`.

### 2.1 Constants

```
LIGHTING_RULE_MAX_PREDICATES   = 8      // per rule
LIGHTING_PREDICATE_BODY_MAX    = 16     // bytes; connection is the largest at 11
LIGHTING_RULE_PAGE_BYTES       = 224    // packed rule bytes per page or chunk
LightingFeatureFlags::RULES    = 1 << 17
```

### 2.2 Predicate tags

| tag | name | body (postcard of an existing or new wire type) | max body bytes |
|----:|------|--------------------------------------------------|---------------:|
| 1 | `layer` | `LightingLayerCondition { layer: u8, active: bool }` | 2 |
| 2 | `battery` | `LightingBatteryCondition` | 6 |
| 3 | `output_mode` | `LightingOutputMode` | 1 |
| 4 | `connection` | `LightingConnectionCondition` | 11 |
| 5 | `effects` | `LightingEffectsCondition` | 1 |
| 6 | `layers` | `LightingLayersCondition { active: u32, inactive: u32 }` | 10 |
| 7 | `indicators` | `LightingIndicatorCondition` | 6 |
| 8 | `maintenance` | `LightingMaintenanceCondition { unlocked: bool }` (new) | 1 |
| 9 | `split_transport` | `LightingSplitTransportCondition { link: Option<LightingSplitLink>, force: Option<LightingSplitForce> }` (new) | 4 |

`LightingSplitLink { Wired, Ble }`. `LightingSplitForce { Auto, Wired, Ble }`,
mirroring `selector::{FORCE_AUTO, FORCE_WIRED, FORCE_BLE}`. Tag 0 is invalid.
Tags 10..63 are unassigned. The tag namespace is a `u8` on the wire but the
status bitmask below is a `u64`, so the practical ceiling is 63 predicate
kinds, which is not a constraint anyone will meet.

`unlocked == rmk::state::maintenance_mode_enabled()`. The firmware calls the
permissive state "maintenance mode enabled" and the key action calls it
"MAINT_LOCK_TOG"; the wire and TOML use the lock vocabulary because that is
what the user sees.

### 2.3 Rule and predicate

```
wire_type! { pub struct LightingPredicate { pub tag: u8, pub body: Vec<u8, LIGHTING_PREDICATE_BODY_MAX> } }
wire_type! { pub struct LightingRule {
    pub led_id: LightingLedId,
    pub effect: LightingEffect,
    pub predicates: Vec<LightingPredicate, LIGHTING_RULE_MAX_PREDICATES>,
} }
```

Postcard already makes this a TLV: `Vec<u8, N>` encodes as a varint length
followed by the bytes, so an unknown tag's body can be skipped without knowing
its type. This is the property the nested-struct generations lacked and the
reason a postcard enum of predicates is not used: an unknown enum variant index
cannot be skipped.

Canonical form rules, enforced by firmware on write and by every encoder:

- Tags strictly ascending within a rule. Rejected otherwise with
  `LightingError::InvalidRequest`. This forbids duplicates, makes byte equality
  equal semantic equality, and makes the digest canonical.
- A body must decode to exactly its declared length (no trailing bytes).
- An empty predicate list is a valid, unconditional rule.

A typed convenience view lives above the wire, in rmk-types under a `std` or
`wasm` feature and in the firmware conversion layer: `LightingRulePredicate`
enum with one variant per known tag plus `Unknown { tag: u8, body: Vec<u8, 16> }`,
and `LightingRule::predicates()` / `LightingRule::from_predicates()`
conversions. tsify turns the enum into a discriminated union, which is what the
editor wants.

### 2.4 Pages and chunks

Rules are variable-length, so pages carry bytes and a count, not a fixed
`Vec<Rule, N>`. A page holds as many whole rules as fit.

```
wire_type! { pub struct LightingRuleStatus {
    pub revision: u32,
    pub capacity: u16,          // rules the table can hold (SCENE_CAPACITY)
    pub rule_len: u16,          // rules installed
    pub page_bytes: u16,        // LIGHTING_RULE_PAGE_BYTES
    pub max_predicates: u8,     // LIGHTING_RULE_MAX_PREDICATES
    pub predicates: u64,        // bit N set: tag N parses on this firmware
} }

// Byte-bounded page: `count` postcard-encoded LightingRule values concatenated in `rules`.
pub struct LightingRulesPage { pub revision: u32, pub total_count: u16, pub offset: u16, pub count: u8, pub rules: Vec<u8, LIGHTING_RULE_PAGE_BYTES> }
pub struct PutLightingRuleChunkRequest { pub transaction_id: u32, pub offset: u16, pub count: u8, pub rules: Vec<u8, LIGHTING_RULE_PAGE_BYTES> }
```

Postcard maximum for either: 5 + 3 + 3 + 1 + 2 + 224 = 238 bytes, under
`LIGHTING_PAYLOAD_SIZE` (256). Add the same `const _: () = core::assert!(...)`
guard pattern `rmk/src/lighting/standard/mod.rs:25-29` uses. A worst-case rule
(eight predicates each with a 16-byte body, blink effect) is 3 + 15 + 1 +
8 × 18 = 163 bytes, so any single rule fits a page. A typical Magic-layer rule
(`layer` + `connection`) is about 20 bytes, so a page carries around ten. Today's
advanced page carries three, so `config apply` of a 47-rule table drops from
16 page round trips to about 5.

Requests reuse the existing transaction types:
`BeginLightingRuntimeConditionalSceneReplaceRequest`,
`LightingRuntimeConditionalScenePageRequest`,
`CommitLightingRuntimeConditionalSceneReplaceRequest`,
`AbortLightingRuntimeConditionalSceneReplaceRequest`, and
`LightingRuntimeConditionalSceneTransaction`. Paging is by rule index: a client
reads `offset = 0`, then `offset += page.count`, until `offset == total_count`.

### 2.5 Opcodes

Six opcodes in the next free block of `rmk-types/src/protocol/rynk/command.rs`:

| opcode | command | request | response |
|-------:|---------|---------|----------|
| `0x0950` | `GetLightingRuleStatus` | `()` | `LightingResult<LightingRuleStatus>` |
| `0x0951` | `GetLightingRules` | `LightingRuntimeConditionalScenePageRequest` | `LightingResult<LightingRulesPage>` |
| `0x0952` | `BeginLightingRuleReplace` | `BeginLightingRuntimeConditionalSceneReplaceRequest` | `LightingResult<LightingRuntimeConditionalSceneTransaction>` |
| `0x0953` | `PutLightingRuleChunk` | `PutLightingRuleChunkRequest` | `LightingUnitResult` |
| `0x0954` | `CommitLightingRuleReplace` | `CommitLightingRuntimeConditionalSceneReplaceRequest` | `LightingStateResult` |
| `0x0955` | `AbortLightingRuleReplace` | `AbortLightingRuntimeConditionalSceneReplaceRequest` | `LightingUnitResult` |

These are the last six conditional-scene opcodes that will ever be minted;
that is the point. Add the four mutating ones to the `Some(true)` arm of
`RynkService::maintenance_policy` (`rmk/src/host/rynk/mod.rs:174`) and the two
reads to the `Some(false)` arm; `every_endpoint_has_a_maintenance_policy`
(`:800`) fails the build otherwise. Update the endpoints table in
`docs/docs/main/docs/development/rynk_protocol.md` and add a short "Lighting
rules" paragraph explaining tags, ascending order, and the `predicates` mask.

### 2.6 Capability advertisement

`RULES` (`1 << 17`) in `LightingCapabilities.features` means the six endpoints
exist and `GetLightingRuleStatus.predicates` is authoritative for which tags
parse. It is set whenever `runtime_conditional_scene_capacity > 0`
(`handlers/lighting.rs:1207`). The four legacy bits (12, 14, 15, 16) keep
being advertised for as long as the legacy endpoints are served (section 5).
The `predicates` mask is "tags this firmware parses", not "tags this board can
evaluate": a `split_transport` rule parses on the Glove80 and is simply
unsatisfiable there, exactly as `effects` is unsatisfiable on an engine with no
extension state today (`source.rs:274-276`).

### 2.7 Unknown predicates

- **Older firmware, newer host.** A chunk containing a tag outside the
  firmware's mask is rejected with a new, appended
  `LightingError::UnknownPredicate { tag: u8 }`. The transaction stays open so
  the host can abort it. Silently dropping the predicate is never acceptable:
  a conditional rule would become an unconditional one, which is the failure
  the split-link comments at `split_lighting.rs:736-740` already warn about.
  Hosts are expected to check the mask before writing, so this error is
  defensive.
- **Newer firmware, older host.** The host decodes unknown tags into
  `LightingRulePredicate::Unknown` and preserves them byte-for-byte on
  write-back. The editor shows such a predicate read-only as "unknown condition
  (tag N)". `moergo-control config pull` cannot express one in TOML and fails
  with a message naming the tag and asking for a newer CLI; `config diff`
  reports it as a difference it cannot describe rather than as "equal".
- **Evaluation.** Within firmware, a parsed but unevaluable predicate is
  unsatisfiable (the existing convention). `split_transport` is unsatisfiable
  when `context.split_transport.auto` is false, mirroring today's
  `auto_enabled()` gate.

## 3. Context and split replication

### 3.1 `LightingContext` gains two fields

In `rmk/src/lighting/context.rs`:

```
pub struct SplitTransportState { pub auto: bool, pub force: SplitForce, pub wired: bool }   // SplitForce { Auto, Wired, Ble }
pub struct LightingContext { ..., pub maintenance_unlocked: bool, pub split_transport: SplitTransportState }
```

Populate on the authority in `KeymapLightingState::snapshot`
(`rmk/src/lighting/rmk_state.rs:41-60`) from
`crate::state::maintenance_mode_enabled()` and, under `#[cfg(feature = "split")]`,
from `selector::{auto_enabled, forced_mode, wired_selected}`; without the
feature, `auto = false`. Lighting is fork-only code (every `rmk/src/lighting`
file is absent from upstream), so widening the struct has no upstream cost.
Constructor sites to update: `rmk_state.rs`, `context.rs` (`Default` derive
covers it), `source.rs` and `standard/tests.rs` test fixtures,
`crates/split-lighting-tests/src/lib.rs`, and the `PERIPHERAL_CONTEXT` static
at `crates/moergo-rmk/src/lighting.rs:821-837`.

`ConditionSet` (`source.rs:264-283`) gains `maintenance: Option<MaintenanceCondition>`
and `split_transport: Option<SplitTransportCondition>`, and `matches`
(`:353-445`) gains two arms reading the context. `RenderPolicy` is unchanged:
these are world state, not engine settings.

### 3.2 Replication of the new context

`ReplicatedContext` (`split_lighting.rs:120-140`) gains the two fields. The
`Context` and `ContextUpdate` packets carry them in the reserved zero bytes 7
and 8 (bytes 7..16 were the retired layer-state field; see the comment at
`:532-534`). Byte 7: `maintenance_unlocked` as 0/1. Byte 8: bit 0 `auto`,
bits 1-2 `force` (0 auto, 1 wired, 2 ble), bit 3 `wired`. Packet lengths stay
24. The peripheral's `merge_ephemeral` (`lighting.rs:855-858`) continues to
override only `layers` and `connection` from local state; the new fields come
from the authority so both halves show the same thing and F-C disappears.

The central replication loop (`central_lighting.rs:634-780`) marks
`context_dirty` on indicator and battery events but has no arm for maintenance
or transport edges. Add one static `Signal<RawMutex, ()>` (`CONTEXT_NUDGE`)
raised by `MaintenanceLightingState::on_maintenance_mode_event`
(`lighting.rs:737`) and by `SplitTransportLightingNudge`
(`central_lighting.rs:879-895`), and consume it in the loop by selecting it
together with the existing `indicators` arm so the `select4` shape does not
grow. The file's own comments (`:131-134`, `:1038-1041`) record two
hardfaults from adding arms to these large futures; keep new bodies
`#[inline(never)]` and out of line. Because the existing processor is the
only `MaintenanceModeEvent` subscriber, `[event.maintenance_mode] subs` in
`keyboard.toml` and `config/firmware.toml` stays at 1.

Context is excluded from the replica digests by design
(`payload/lighting.rs:1283-1287`); freshness comes from revision and ack.
Optionally add the same two bytes to `StatusReport` (`split_lighting.rs`
`STATUS_REPORT_LEN`) so `moergo-control lighting replica` can show the
peripheral's view; this is diagnostics, not correctness.

### 3.3 Replication of rules across the split link

`SPLIT_APP_MSG_MAX` is 26 bytes. The `ConditionalSceneCell` packet (tag 9, 26
bytes, `split_lighting.rs:683-741`) stays as is: it carries slot, effect,
and the three inline predicates `layer`, `battery`, `output_mode`, which is
the common case and needs no second packet. Replace `ConditionalSceneExt`
(tag 13) with:

```
ConditionalScenePredicates { generation: u8, revision: u32, len: u8, bytes: [u8; 18] }   // TAG 24, 26 bytes
```

`bytes` is a slice of the rule's canonical predicate TLV **excluding** tags
1-3 (already inline). The central emits `ceil(n / 18)` such packets after
each cell whose remaining predicates are non-empty; the peripheral appends
each into a small per-cell staging buffer and, on the next cell or the commit,
decodes it into the last staged cell through `last_conditions_mut()`
(`conditional.rs:151-157`). A rule with `connection` and `effects` needs one
packet, as it does today. `snapshot_counts`/`walk_snapshot` (`:1650-1780`)
must count these packets so the whole-transaction reservation
(`try_queue_snapshot`, `:1793-1821`) stays exact. Bump `VERSION`
(`split_lighting.rs:42`) from 11 to 12; the peripheral already discards
mismatched versions (`:925`). Both halves must be flashed from one build,
and the "flash only the halves whose UF2 hash changed" shortcut in the headless
flash notes must not be applied to this release.

### 3.4 Digest

`hash_conditions` (`:1468-1527`) is replaced by hashing the rule's full
canonical TLV (all tags, including the inline three, re-encoded canonically)
after `slot`. `Fnv32::domain` seeds with `LIGHTING_REPLICA_DIGEST_SCHEMA_V2`
and `replica_digests` reports `schema: V2`. This closes F-B and covers every
future tag with no further edits. Both halves compute it from the same
firmware, so the central's expected/attested comparison
(`central_lighting.rs:800-816`) needs no change. `moergo-control lighting
replica` and rynkbench show the schema byte; nothing on the host interprets
it.

## 4. TOML surface (`crates/moergo-config/src/config.rs`)

Two optional fields, added to both `KeyConditionConfig` (`:660-668`, the
`when = { ... }` table) and `ConditionalSceneConfig` (`:1195-1230`), spelled
to match the neighbours: enum strings like `output_mode = "always-on"`, tables
like `connection = { ... }`.

```toml
# Glove80 Magic layer, replacing BoardStatus' maintenance indicator on R.
[[layer.key]]
key = [2, 4]                  # R
action = "MAINT_LOCK_TOG"

[[layer.key.rule]]
color = "#008000"             # unlocked: host mutations allowed
when = { maintenance = "unlocked" }

[[layer.key.rule]]
color = "#800000"             # locked
when = { maintenance = "locked" }
```

```toml
# Go60, replacing BoardStatus' split-transport indicator (colours from lighting.rs:99-109).
[[layer.key.rule]]
color = "#008000"
when = { split_transport = { force = "auto", link = "wired" } }

[[layer.key.rule]]
color = "#0040a0"
when = { split_transport = { force = "auto", link = "ble" } }

[[layer.key.rule]]
color = "#800000"
when = { split_transport = { force = "ble" } }

[[layer.key.rule]]
color = "#a000a0"
when = { split_transport = { force = "wired" } }
```

Because a `[[layer.key.rule]]` lives under a `[[layer]]` addressed by `id`,
the layer gate is implied and lowered by `LayerConfig::scenes`
(`config.rs:497-571`) as `layer = { layer = <resolved index>, active = true }`.
That is what dissolves the original bug: the indicator follows the layer named
"magic" wherever the layout puts it, and a layout with Autoshift in slot 2
simply has no such rule.

Config types: `MaintenanceStateConfig { Unlocked, Locked }` (kebab-case),
`SplitTransportConditionConfig { link: Option<SplitLinkConfig>, force: Option<SplitForceConfig> }`
with `SplitLinkConfig { Wired, Ble }` and `SplitForceConfig { Auto, Wired, Ble }`.
`validate_conditional_scene` (`:3539`) rejects a `split_transport` table naming
no field, as it does for `connection` (`:3605-3613`). Lowering:
`conditional_scene_to_rule` / `conditional_scene_from_rule` replace the
`_to_advanced_wire` / `_from_advanced_wire` pair as the primary path; the
advanced pair is kept for the fallback in section 5.3.
`require_layer_conditions_capability` (`moergo-control/src/config.rs:1571`)
generalises to "every tag the file uses is set in `LightingRuleStatus.predicates`",
with an error naming the first missing predicate.

Semantics to document in the field comments: `link` is the link carrying the
halves right now (`wired_selected()`), `force` is the volatile override; a
`split_transport` rule never matches on a board without automatic selection.

## 5. Migration and compatibility

### 5.1 Firmware table and storage

The engine keeps one `RuntimeConditionalSceneTable` (renamed or not; it is an
internal name). The three legacy endpoint families are served from it for
exactly one pinned release, translating each cell to their cell type as today.
A page that would need a predicate the family cannot express answers
`LightingError::Unsupported` for that page (today the advanced-to-extended
`TryFrom` surfaces as `InvalidRequest`, `payload/lighting.rs:2199-2210`;
`Unsupported` is the honest word and older hosts already decode it). Reads of
`GetLightingConditionalSceneStatus`/`GetLightingConditionalScenes` (compiled
cells, `0x091E/0x091F`) keep working and report zero cells;
`COMPILED_CONDITIONAL_SCENES` is already only advertised when non-empty
(`handlers/lighting.rs:1204`).

Storage adds `StorageKey::LightingRuleShardV4(u8)` and its B-generation twin
(the A/B pattern of `LightingGeneration`, `storage/mod.rs:640-654`), with
`StorageData::LightingRuleShardV4(heapless::Vec<u8, LIGHTING_RULE_SHARD_BYTES>)`
holding packed rules, and reuses `LightingRuntimeConditionalSceneCommitRecord`
(generation, len, digest) where the digest folds each rule's canonical bytes.
Choose `LIGHTING_RULE_SHARD_BYTES` after checking the storage scratch buffer
(audit 2026-09-07 F4: 288 bytes at the default macro space); 192 is safe if
224 is not. Boot: if a V4 commit exists, stream its shards and install rule by
rule (removes the F-E stack transient); otherwise fall through to the existing
V3, V2, V1 readers and convert each cell to a rule. The first host commit after
upgrade writes V4 and empties the legacy headers, as
`commit_lighting_runtime_conditional_scenes` (`:1258-1305`) already does for
older generations, so a downgrade sees an empty table rather than stale cells.
Fewer, larger shards also mean fewer flash writes per persist: a 100-rule
table is 34 three-cell V3 records today and about 10-12 packed V4 records.

### 5.2 Deleting `BoardStatus`

In `crates/moergo-rmk/src/lighting.rs`: delete `MAGIC_LAYER`, the seven colour
and slot constants, `split_transport_color`, `BoardStatus`, and its
`LightingSource` impl; change `Engine`'s `Status` parameter to
`rmk::lighting::EmptySource` and `engine()` to pass it; drop
`with_conditional_scenes(...)` from `central_lighting.rs:376`. Delete
`BOARD_MAINTENANCE_LED` and `BOARD_SPLIT_TRANSPORT_LED` from both `central.rs`
and `peripheral.rs` of both board crates. Keep `MaintenanceLightingState` and
`SplitTransportLightingNudge`; they now also raise `CONTEXT_NUDGE`. Have the
`rmk_lighting_config!` codegen additionally emit
`LIGHTING_CONDITIONAL_SCENE_CELL_COUNT: usize` so `moergo-rmk` can
`const _: () = assert!(LIGHTING_CONDITIONAL_SCENE_CELL_COUNT == 0)`; that is
the guard that keeps compiled behaviour from returning without a deliberate
edit. `keyboard.toml` needs no change on either board; `config/firmware.toml`
therefore stays identical and `just firmware-config-check` is unaffected.

### 5.3 Hosts

- **`moergo-control`** (`crates/moergo-control/src/config.rs`): the
  read path (`:330-390`) and apply path (`:1500-1560`) gain a first branch on
  `RULES`. Fallback chain when `RULES` is absent: advanced if the file uses no
  tag above 7, else a clear error ("firmware predates the maintenance and
  split-transport conditions; flash first"); then the existing extended and
  legacy branches unchanged.
- **`rynk` host crate** (`rynk/src/api.rs`): `get_lighting_rule_status`,
  `read_all_lighting_rules` (variable-count pager), and
  `replace_all_lighting_rules` following the shape of
  `replace_all_lighting_advanced_runtime_conditional_scenes` (`:1955-1999`) but
  packing rules into byte pages. `rynk-wasm/src/client.rs` exports the six
  endpoints plus two codec helpers, `decode_lighting_predicate(tag, body)` and
  `encode_lighting_predicate(predicate)`, so TypeScript never hand-rolls
  postcard. The Rynkbench wasm lock must stay on wasm-bindgen 0.2.126 (see
  memory note).
- **rynkbench**: add `RULES` to `src/session/lighting-features.ts`; add
  `readLightingRules`/`replaceLightingRules` to `link-session.ts` and route
  `readLightingRuntimeConditionalStatus` (`:647-657`) through `RULES` first;
  change the UI model `StatusRule` (`statusPresets.ts:7`) from
  `LightingAdvancedConditionalSceneCell` to a decoded `LightingRule`, with an
  adapter for firmware without `RULES`; teach `firmwareRules.ts` preview to
  evaluate `maintenance` from `GetMaintenanceMode` (`0x000D`) and
  `split_transport` from `GetSplitTransport` (`0x070A`), both already exposed,
  so the preview is exact rather than "unsatisfiable"; add two condition
  controls to the rules editor; add the R-key maintenance rules to
  `stockMagicLayer.ts` (the installer today sets keys, scenes, wake layers, and
  connection indicators but nothing for maintenance, so a stock install would
  otherwise lose the indicator the firmware used to draw).

### 5.4 Boards that ship no runtime rules (question 3)

After deletion, stock firmware shows no maintenance or transport indicator
until a configuration supplies one. This is acceptable and is the owner's
stated preference; the mitigation is to make the indicator part of every
artifact that already defines the Magic layer, none of which is compiled:

1. `config/glove80.toml` and `config/go60.toml` (section 4 snippets).
2. `presets/magic-glove80.toml` and `presets/magic-go60.toml`.
3. rynkbench's stock Magic installer.
4. A `moergo-control config validate` warning, not an error, when a layer binds
   `MAINT_LOCK_TOG` and no rule on the same key names `maintenance`. The lock's
   visibility has a mild security role (it tells the user whether unattended
   host mutation is possible), so losing it silently deserves a nudge.

Nothing compiled is added. A board flashed with stock firmware and never
configured is dark on that key, which is honest: the firmware has no
opinion about which key is Magic+R.

### 5.5 Ordering and precedence (question 5)

Today `BoardStatus` is applied at `priority::STATUS` first and the runtime
table second (`engine.rs:1398-1411`), so a runtime rule on LED 12 already
outranks the compiled indicator. After deletion the runtime table is the only
status-band source besides the (unused) compiled output-mode indicator. Within
the table order is file order, later wins on a shared slot, and
`LayerConfig::scenes` lowers `[[layer.key.rule]]` arms in reverse so a key's
rules read first-match-wins (`config.rs:521-527`). Nothing about that changes.
The indicators simply take the place in the file their author gives them,
next to the other Magic-layer status keys. The engine's compositor order
(background, extension, layer scenes, runtime scenes, host overlay, status)
is untouched.

## 6. RAM and flash cost (question 4)

Measured with a scratch program replicating the exact field layouts (rustc
1.97.1, 4-byte alignment matches thumbv7em because the widest field is `u32`).
`BOARD_SCENE_CAPACITY` is 100 on the Glove80 and 80 on the Go60; the 160 in the
`lighting.rs:73-77` comment is the value that faulted and was reduced.

| struct | today | after | delta |
|--------|------:|------:|------:|
| `ConditionSet` | 32 B | 36 B | +4 |
| interned table cell (`conditions` + `slot` + `style`) | 36 B | 40 B | +4 |
| `RuntimeConditionalSceneCell` (chunk/page, effect inline) | 52 B | 56 B | +4 |
| `LightingContext` | not measured | +4 B (one `bool`, one 3-byte state) | +4 |

Copies of the table per binary: live table, staging
`RuntimeConditionalSceneReplace`, and one `StandardReplicaState` in
`REPLICA_SLOT` on the central; live, `REPLICA_SLOT`, and the split `Stage`
on the peripheral. That is three tables each side.

| board | per table | static RAM per half | stack transient on replace |
|-------|----------:|--------------------:|---------------------------:|
| Glove80 (100) | +400 B | +1.2 KB | +400 B |
| Go60 (80) | +320 B | +960 B | +320 B |

The `StandardCommand` mailbox is unchanged: its largest variant is the 64-cell
overlay batch, not the seven-cell conditional chunk, so `COMMAND_CAPACITY`
does not multiply this growth. The stack transient is the one that matters:
the central faulted opening a replace transaction at capacity 160, so a
4-byte-per-cell growth is real but bounded at 11 percent. If the measurement
in section 8 shows pressure, a packed `ConditionSet` (one `u16` present-mask
plus fields without their `Option` wrappers) measured 32 bytes **with** both
new predicates, so the growth can be reclaimed entirely without touching the
wire.

Flash per stored rule (V4 shard): a typical `layer` + `connection` rule packs
to about 20 bytes, comparable to the roughly 22 bytes V3 spends on the same
cell as a postcard `LightingAdvancedConditionalSceneCell` (nine `Option`
tags at one byte each). Rules with only the implied layer gate are about 10
bytes. Code size: roughly one generation's worth of handlers and codec added
(estimate 3-5 KB of `.text`), minus the three legacy families when they are
removed in the following release. The Go60 has been within about 12 KB of its
flash ceiling (handoff of 2026-09-07), so removal should not slip; gating the
legacy families behind a cargo feature lets the Go60 drop them a release
earlier than the Glove80 if needed.

Protocol cost per predicate added later: zero opcodes, zero capability bits,
one bit in `predicates`, one `ConditionSet` field (usually 1-4 bytes per cell,
measurable the same way), and nothing on the split link, digest, or storage
formats.

## 7. Rollout order (question 6)

Order chosen so that at every step a newer host talks correctly to an older
keyboard and vice versa.

1. **rmk fold branch** `fork:feat/lighting-rules` in
   `dependencies/moergo-rmk/dependencies/rmk`: rmk-types (section 2 types,
   tags, opcodes, `RULES`, `UnknownPredicate`, and a locked wire-frame snapshot
   for a rule carrying every tag, alongside `lighting_wire_frames_locked`),
   rmk (`ConditionSet` and `LightingContext` fields, `rmk_state.rs`
   population, `matches` arms, handlers for `0x0950-0x0955`, legacy pagers
   answering `Unsupported`, storage V4 with migration and downgrade tests,
   maintenance policy entries), rynk and rynk-wasm, protocol docs. Append to
   `~/Projects/rmk-assembly/manifest.toml` after
   `feat/lighting-layer-indicator-conditions` (an append is an incremental
   build), `fork-fold build`, `fork-fold build --locked`, commit manifest,
   lock, and any resolutions together, push `assembled`.
2. **moergo-rmk**: pin the new `assembled`; moergo-config (section 4);
   moergo-control (section 5.3); firmware (sections 3 and 5.2:
   `split_lighting.rs` VERSION 12, context bytes, TAG 24, digest V2;
   `central_lighting.rs` nudge; `lighting.rs` deletions; board crates;
   streaming boot install); `crates/split-lighting-tests` round-trip and
   digest tests, including a regression that a right-half `layers` rule
   survives the link. Run the project's `just check`/parity check inside
   `nix develop` (bare cargo fails on this machine; see memory).
3. **rynkbench**: regenerate the vendored wasm from the pinned rmk, then the
   TypeScript changes in section 5.3, with `firmwareRules` and stock-installer
   tests.
4. **glove80-config**: pin moergo-rmk; add the rules to both configs and both
   presets; `just firmware-config-check` (unchanged `keyboard.toml`, so it
   passes trivially); `just firmware` and `just go60-firmware`; flash **both**
   halves; `just apply`; `just diff`.

Install order in the field: new `moergo-control` and rynkbench first (they
fall back to the advanced endpoints on old firmware), then firmware on both
halves (it still serves the legacy endpoints for old hosts), then the
configuration that uses the new predicates (the CLI refuses it on old
firmware with a clear message). The one hard coupling is left half with right
half, because of the split `VERSION` bump.

Hardware acceptance, on the Glove80 with layer 2 held: LED 12 dark before
`just apply` on the new firmware (proves the compiled indicator is gone);
green after, red after Magic+R, green after Magic+R again; `moergo-control
lighting replica` shows schema 2 and `Healthy`. On the Go60 additionally
force the transport both ways and unplug the cable, checking the four colours
on LED 6 and that the right half shows them identically if a rule targets a
right-half LED.

## 7a. Measured: risk 1 is not a blocker (added 2026-09-14 after the design)

The RAM prototype the design asks for in risk 1 was run. `ConditionSet`
(`rmk/src/lighting/source.rs:259`) was widened by two placeholder fields
(`Option<bool>` and `Option<u8>`) and the Glove80 central was rebuilt with the
same command and configuration as the baseline.

| section | baseline | widened | delta |
| --- | ---: | ---: | ---: |
| `.text` | 637372 | 637612 | +240 |
| `.data` | 9652 | 9652 | 0 |
| `.bss` | 157000 | 158232 | +1232 |

Static RAM moves from about 166.7 KiB to about 167.9 KiB of the nRF52840's
256 KiB, so roughly 88 KiB of headroom remains. **The plain `ConditionSet`
layout is affordable; the packed layout in section 6 is not required for this
change.** Real predicates a little larger than the placeholders would not
change that conclusion.

Two incidental findings from the prototype:

- Only **four** non-test sites construct `ConditionSet` exhaustively:
  `rmk/src/host/rynk/lighting.rs:2030`,
  `rmk/src/lighting/standard/conditional.rs:19` and `:43`, and
  `crates/moergo-rmk/src/split_lighting.rs:1179`. The last is the split-link
  decoder this design indicts, which is a useful confirmation that the
  hand-maintained widening surface is small but includes exactly the place that
  was previously missed.
- `just firmware` refuses to build while `dependencies/rmk` has local changes
  (`xtask: dependencies/rmk has local changes`). All rmk work must therefore go
  through a fold-branch worktree; local edits to the assembled tree cannot be
  built through the normal path.

## 8. Risks, and what to prototype first

Ranked by how likely they are to force a redesign.

1. **Central RAM and stack at capacity.** The board has faulted on exactly this
   axis before. Prototype first, before any wire work: widen `ConditionSet` by
   the two new fields on a branch, build the Glove80 central, compare `.bss`
   and `.data` against the last pinned build, then run the capacity scenario
   from `audits/2026-09-07/capacity.toml` (fill the table, open and commit a
   replace, export a replica) on hardware. If it faults or the margin looks
   thin, adopt the packed `ConditionSet` layout from section 6 as part of the
   same change. Evidence that settles it: the build's section sizes and a
   clean capacity run.
2. **Split replication regressions.** Adding a message tag and changing
   `walk_snapshot`'s packet accounting touches the code with the documented
   miscompile history. Prototype second: land the split-link half alone
   (TAG 24, context bytes, VERSION 12) behind the existing `ConditionSet`, run
   `split-lighting-tests`, and exercise a reconnect storm on hardware with a
   right-half rule that carries `layers`, which fails today and must pass.
3. **Storage record size versus the scratch buffer.** Compute before choosing
   `LIGHTING_RULE_SHARD_BYTES`; a serialization error here marks storage
   disabled and erases it on the next boot (audit F4 mechanism). A host
   storage test that persists a full table at the chosen size settles it.
4. **Go60 flash.** Measure `.text` after step 1 of the rollout. If the Go60 no
   longer links, gate the legacy endpoint families behind a cargo feature the
   Go60 disables immediately, accepting that old hosts cannot talk to a new
   Go60 for one release.
5. **Editor model migration.** Changing `StatusRule` touches presets, the stock
   installer, preview, and their tests. The adapter to and from
   `LightingAdvancedConditionalSceneCell` keeps every existing test meaningful;
   add one test that a rule with an unknown tag survives read, edit of another
   rule, and write.
6. **Losing the lock indicator silently on unconfigured boards.** Mitigated by
   section 5.4; the validate warning is cheap and should ship with the config
   change.

Where I am unsure. Whether the storage scratch buffer at the boards'
`macro_space_size` accepts a 224-byte record is a calculation not yet done.
Whether the `predicates` mask should also exclude tags a build cannot evaluate
(so a `split_transport` rule is refused on the Glove80 rather than silently
never matching) is a product choice; this document chooses parse-only because
it keeps one configuration valid across both boards and matches how `effects`
behaves, but the opposite choice is one `if` in the status handler. Whether to
keep the legacy families for one release or zero depends on whether any host
other than the two in this stack ever wrote rules to these boards; I found
none, but the flash and compatibility trade is the owner's call.
