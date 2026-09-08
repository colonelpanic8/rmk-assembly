# Handoff to jay-lenovo: qualify the refreshed Glove80 firmware

Written 2026-09-07 by the Fable 5.1 session on ryzen-shine. Everything below
is already pushed; nothing here needs work on ryzen-shine.

## What changed and why you are being asked

The RMK assembly was refreshed onto live upstream `main`
(`8b4d1b312d8ef2eeed55705a5f33977ab13f0a55`) and carries the 2026-09-07 audit
repairs. That propagated down the whole chain. The Glove80 firmware builds and
links here, but **no hardware has been flashed or exercised**. You have a
Glove80 connected; the ask is qualification, not more building.

## Exact pins (all pushed)

| Repo | Commit |
|---|---|
| `colonelpanic8/rmk-assembly` master | `0e3d214` |
| `colonelpanic8/rmk` branch `assembled` | `ef9787302c52fe8a550f4d41b788602e63b73ad8` (tree `63685eca`) |
| `colonelpanic8/moergo-rmk` master | `02b82f1` |
| `colonelpanic8/moergo-config` master | `3b605fd4` |
| `colonelpanic8/nrf-sdc` branch `bt-hci-0.10-timeslot-flash` | `2dfeabc74b3ed423ace82e527a8bce9b221d6f17` |

To get there:

```sh
cd ~/Projects/glove80-config
git fetch origin && git checkout 3b605fd4
git submodule update --init --recursive
just firmware        # Glove80 left+right; succeeded on ryzen-shine
```

The last local build produced:

```
left:  0x26000-0xd9300, 0x9807b007, 02f888b630be236276855c45b6c5d71680b6de96c24f5fd3b01a4900acc65d63
right: 0x26000-0x8a200, 0x9808b007, 0ecff10ae079b9f52486aca46f9b48baf040a34cb7d1bcc4600043056d11efea
```

Compare your address ranges against those before flashing. A surprising range
change is a build-input warning, not proof of RAM exhaustion.

## What to qualify, in priority order

The refresh changed behaviour in exactly the areas the audit touched, so the
qualification should target those rather than a generic smoke test.

1. **Both halves, together.** The split wire enum changed
   (`TransportOverride` became a struct variant, `TransportOverrideAck` was
   added), so left and right must be flashed from the same build. Qualify the
   left/central half first and keep a known-good recovery UF2 at hand.
2. **Storage across the update.** Upstream renumbered `KeyboardAction`, so
   expect a one-time storage reset for keymap actions. After flashing, run
   `just apply` and then `just diff` to confirm the runtime config in
   `config/glove80.toml` is actually on the keyboard.
3. **Split reconnect and transport force.** A transport force is now applied
   only after the peripheral acknowledges it, with a 5 s timeout that leaves
   both halves where they were. Unplug/replug, force the transport both ways,
   and confirm neither half is stranded.
4. **Half-duplex serial timing.** Reply deadlines are now derived from the
   configured baud rate rather than a fixed 5 ms. Only reachable on a wired
   split; skip if you are on BLE.
5. **Lighting.** Per-layer scenes and conditional rules persist through a
   reboot; `powered_only_scope` now distinguishes authority VBUS from the
   local half's VBUS, so on battery the peripheral should follow whichever the
   configured scope names. Runtime conditional rules written by the *previous*
   firmware need one re-send from the host (storage keys moved); scene tables
   are unaffected.
6. **Pointing and keyboard reports.** Caret/keypad taps now go through the
   keyboard, so holding a modifier while the pad emits a key must keep the
   modifier. A pad button held while a keyboard mouse key repeats must not be
   released.

## Known open item: go60 does not fit

`just go60-firmware` fails to link:

```
rust-lld: error: section '.data' will not fit in region 'FLASH': overflowed by 11900 bytes
```

Upstream's larger BLE stack (trouble 0.24 plus the `p256-cortex-m4` security
dependency that its `security` feature pulls in) pushed the go60 image past
its flash budget. The Glove80 is unaffected. This is untouched and needs a
decision: trim features for the go60, or change its partition layout. Do not
treat it as a blocker for Glove80 qualification.

## Things worth knowing while you work

- The dev shell now ships `gcc-arm-embedded` and sets
  `CC_thumbv7em_none_eabihf`, because `p256-cortex-m4-sys` compiles C for the
  firmware target and cc-rs would otherwise reach for the host gcc.
- `just firmware` refuses to build from a dirty tree; commit or stash first.
- The full rmk host test suite passes on the assembled tree (862 tests). No
  hardware, RF, cable, or power measurement has been done — that is exactly
  the gap you are closing.
- Audit evidence, per-agent reports and the finding-by-finding disposition are
  in `rmk-assembly/audits/2026-09-07/`, notably `README.md` (the findings) and
  `fable-progress-ledger.md` (what was fixed, refuted, or left open).
