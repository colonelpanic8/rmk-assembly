# Astra fix handoff — 2026-09-07

Completed the eight assigned topic branches. All commits below were pushed by fast-forward to `colonelpanic8/rmk`; no other topic branches were pushed. Fable owns the remaining audit fixes, upstream refresh, and reassembly.

| Remote branch | Published fix commit(s) | Fix and focused validation |
| --- | --- | --- |
| `feat/consumer-keycodes` | `26a1399d287f33e6ab1492a971dfaf148dff8bc8` | Append new enum variants to preserve existing Postcard wire ordinals. Compatibility fixture and 3 wire snapshots pass. |
| `feat/morse-retro-tap` | `8041066651464c683c7acc3f4a60e5ca4277852f` | Remember interruption before hold timeout; suppress phantom retro tap. 6 tests pass. |
| `fold/runtime-behavior-parity` | `d642d07488891bac9930bd9cf20570209a337464` | Keep initially empty auto-mouse-layer runner responsive to runtime configuration. 63 focused tests pass. |
| `feat/lighting-layer-indicator-conditions` | `2f0104f1ea2a42a6ba12522c6568198b31508f70` | Gate six lighting commands behind the lighting feature. Rynk without lighting/storage compiles and passes Clippy. |
| `fold/unicode-input` | `9547159112c403a22b4088aecb81ad280faf89fc` | Emit literal Unicode HID sequences without mutating held key/modifier state, then restore the ordinary report. 8 tests pass. |
| `fold/opposite-hand-hold` | `450c579517a9a014cfe8c8e6403e24ff984a114a` | Resolve mutually exclusive hand policies without inheriting the opposite policy; preserve the selected policy when disabling its alternative. 10 behavior and 17 type tests pass. |
| `fold/pointing-runtime-config-pinned` | `a73c2b7ab6565ba2ab32d793cf4fa3078c911de7` | Restore persisted pointing config at boot; apply startup/config/layer changes in each processor; honor remote layer payloads and explicit mode commands; compile without storage. 71 focused tests pass, including boot restoration; final revised runtime test also passes. |
| `glove80-rmk/lighting-v2` | `fc14c9a380e8b9c906c77ed5c5daa3420cac6002`, then `e909a63849c382793551eb821aaa38c9114f969f` | Preserve PoweredOnly policy across mutable-state round trips (41 tests); replace obsolete native bulk transport with async HID and correct framing (2 fragmentation/error tests). [PR #1031](https://github.com/rmk-rs/rmk/pull/1031) head verified at the latter commit. |

Changed Rust files pass nightly rustfmt checks; focused Clippy checks pass for each fix. Pointing additionally passes `rynk,storage` and `rynk` Clippy plus a no-default-features check. Consumer all-target Clippy encountered two pre-existing test lints; library and changed fixture Clippy pass. No hardware, Windows/macOS backend, or refreshed-assembly validation is claimed.

## Integration notes for Fable

- No `fork-assembler update/build` was run. Root `manifest.toml`, `manifest.lock.json`, `resolutions/`, and `patches/` were not edited. The generated branch was not committed to. Most topics have no open PR; their existing fork branches were updated, and the existing lighting PR was updated through its head branch.
- Pointing's boot call is on its standalone topic's legacy boot path. Reconcile it with the earlier single-pass storage topic during reassembly. Preserve the new config-invalidation subscription and configuration/layer/control priority when reconciling the later pointing processor priority change.
- Unicode's Ctrl/GUI-swap interaction needs the combined regression after reassembly; the standalone Unicode topic does not contain that swap feature. Its literal report path is designed to bypass logical modifier remapping.
- Maintenance gates for named Morse profile writes and split transport force, and Unicode restoration in the single-pass boot loader, were investigated but **not published or applied to the recipe**. Draft patches are in `.worktrees/critical-audit-20260907/integration-fixes/`. The pointing drafts there predate the final review adjustments and must not be applied blindly.
- `.worktrees/critical-audit-verification-20260907/` is an unpublished scratch archive with draft combined changes, not authoritative integration output. `.worktrees/critical-audit-20260907/AUDIT.md`, `architecture-review.md`, `fix-review.md`, and reproduction artifacts contain this session's additional audit evidence. Remaining findings and architecture work belong to Fable.

Astra is stopped after this handoff; the eight published topic heads above are ready for Fable to incorporate.
