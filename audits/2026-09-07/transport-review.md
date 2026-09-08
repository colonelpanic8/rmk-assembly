# RMK transport-focused critical audit

**Compared:** assembled snapshot `9a88399366a01b344cf90fb3723cae68dc4dc949` against base `f8da2742971d048e0080e57d69d98f00a859e4e9`.

**Scope:** BLE, split serial, nRF half-duplex operation, reconnect and transport selection, peripheral battery propagation, BLE naming, and nRF code generation/interrupt setup. This was a read-only static audit of the pinned trees. Findings below are regressions or newly unsafe interactions in the assembled delta. I intentionally omit the separately reported missing maintenance-mode gate on `SetSplitTransportForce`.

I did not identify a separate delta-specific defect in the new nRF BLE GPIOTE/time priority setup at `rmk-macro/src/codegen/chip/chip_init.rs:126-143`; the code-generation defect found is the resource collision described below.

## Findings

### 1. High — the half-duplex response deadline is shorter than valid traffic at the generated default baud rate

**Trigger.** Use the generated nRF half-duplex configuration at its default 115,200 baud and exchange one or more nontrivial frames. A burst can contain four frames; a reply can also contain four frames.

**Evidence.** `rmk/src/split/serial/mod.rs:510-520` fixes the entire response window at 5 ms and the peripheral reply gap at 2 ms. `rmk/src/split/serial/mod.rs:596-628` starts that 5 ms deadline *before* the central transmits as many as four queued frames and its poll. The peripheral then waits 2 ms and emits as many as four replies plus the idle terminator (`rmk/src/split/serial/mod.rs:729-769`). The generated nRF default is 115,200 baud (`rmk-macro/src/codegen/split/central.rs:137-143`), while `FRAMES_PER_EXCHANGE` is four (`rmk/src/split/serial/mod.rs:183-193`). Even a 32-byte 8N1 frame takes about 2.78 ms on the wire; one such reply cannot finish inside a 5 ms window after the mandatory 2 ms gap, and the deadline has already been consumed by the central's preceding writes. With `dfu_split`, a single payload can contain 256 bytes (`rmk/src/split/mod.rs:205-243`), making the mismatch much larger.

**Impact.** The central can time out before a valid peripheral burst starts or finishes, clear its receive window, and begin another exchange while the peripheral is still driving the bus. That turns routine load into collisions, retransmissions, dropped input, failed control acknowledgements, and unreliable split DFU. The comment that four frames keep an exchange below the timeout is not true at the default generated baud.

**Required repair.** Model transmission and response phases separately. Start the response allowance after the poll has completely left the wire, and calculate the receive deadline from baud rate, encoded maximum frame size, maximum reply burst, reply gap, and margin. Cancellation safety needs a separate conservative bus-quiet guard rather than borrowing this response deadline.

### 2. High — one corrupt reply frame makes the central drive into the remainder of the peripheral burst

**Trigger.** Any COBS, CRC, postcard, empty-read, or UARTE error occurs on a response frame that is not the final idle terminator.

**Evidence.** `drain_response_window` treats every frame-read error as proof that the window is over: it sets `response_deadline = None` and returns (`rmk/src/split/serial/mod.rs:560-591`). A decoded bad frame has already been consumed through its sentinel and could safely be followed by more valid frames (`rmk/src/split/serial/mod.rs:408-449`). The peripheral is allowed to continue sending the rest of its burst and only relinquishes the bus after `HalfDuplexIdle` (`rmk/src/split/serial/mod.rs:759-769`). The manager merely logs a read error and immediately loops (`rmk/src/split/driver.rs:341-349`), so the next read starts a new central exchange with no remaining quiet-window guard.

**Impact.** A recoverable damaged frame becomes a deterministic turnaround collision. Subsequent frames, including the terminator, can be corrupted as well, prolonging or deadlocking recovery exactly when the link is already noisy.

**Required repair.** Record the bad frame, but keep receiving until a valid idle terminator or a conservative end-of-burst deadline based on maximum remaining wire time. Transport-level RX errors should enter an explicit recovery/drain phase before the central is allowed to drive again.

### 3. High — BLE transport force reports delivery when the write failed or was never acknowledged by the peripheral

**Trigger.** A force request is sent over BLE during a transient GATT/controller error, packet loss, or disconnect race.

**Evidence.** `PeripheralManager::send` converts every write error except `Disconnected` into success (`rmk/src/split/driver.rs:240-250`). The BLE writer specifically logs every `write_characteristic_without_response` error other than `NotFound`, then returns `Ok` (`rmk/src/split/ble/central.rs:559-578`). Even its success only means the command was accepted locally for a write without response; there is no application acknowledgement from `rmk/src/split/peripheral.rs:258-261`. Nevertheless, the manager immediately applies the force to the central after `send` returns (`rmk/src/split/driver.rs:350-371`).

**Impact.** The central leaves BLE while the peripheral remains on BLE, or the central leaves wired while the peripheral remains wired. Because the split link itself is then gone, it cannot carry a corrective command. This defeats the rendezvous property claimed by `request_transport_force` (`rmk/src/split/mod.rs:91-103`).

**Required repair.** Propagate every transport write failure and add a peripheral-level acknowledgement containing the force generation. Apply the central selector only after that acknowledgement. A write-with-response improves immediate error reporting but does not replace the application acknowledgement needed across disconnect races.

### 4. Medium — a queued force can survive a failed session and override a later, newer selection

**Trigger.** Request a force after a transport marks a peripheral connected but before its `PeripheralManager` starts consuming the force channel; let discovery/session setup fail, then make a different force request while disconnected and later reconnect.

**Evidence.** BLE sets the slot connected immediately after receiving a raw connection, before GATT discovery and manager startup (`rmk/src/split/ble/central.rs:376-398`). `request_transport_force` uses any slot's connected bit to decide whether to enqueue rather than apply locally (`rmk/src/split/mod.rs:91-103`). The global capacity-one channel has no session or generation (`rmk/src/channel.rs:110-114`). If setup fails before the manager's receive arm at `rmk/src/split/driver.rs:309-332` runs, the old item remains queued. A later request made when disconnected applies directly to the local selector and does not replace that item. On a future session, the old item is consumed and applied to both sides.

**Impact.** A force that the caller believes failed with the dead session can unexpectedly revert a newer selection on reconnect. It can also leave the channel full, causing intervening connected requests to return `false`.

**Required repair.** Represent force as desired state plus generation, not as a sessionless work queue. Define “connected” for rendezvous as an application-ready manager, and discard or supersede generations when a session ends.

### 5. Medium — advertising timeout parks the split BLE peripheral on keyboard input only, bypassing selector wakeups and its liveness fallback

**Trigger.** Let split-peripheral advertising time out after 300 seconds, select wired, and later unplug or issue a force back to BLE without generating a local keyboard event.

**Evidence.** The normal loop races advertising against wired selection and `forced_ble_liveness_fallback` (`rmk/src/split/ble/peripheral.rs:149-179`). After the advertising timeout, however, it awaits only a newly subscribed `KeyboardEvent` (`rmk/src/split/ble/peripheral.rs:201-207`). It does not wait for selector changes, pointing activity, or the liveness fallback. The fallback future exists only inside the already-completed `select3`.

**Impact.** The selector can say wireless while the BLE task does not advertise. In the force-to-BLE case, the central has already left wired and the peripheral's eight-second fallback is not running, so the halves remain disconnected until a key is pressed locally.

**Required repair.** Make the timeout sleep race keyboard/pointing activity with selector changes. If BLE is forced while parked, start advertising immediately and keep the forced-BLE liveness timer active for the whole forced state.

### 6. Medium — the half-duplex reset handshake retains messages from the previous session

**Trigger.** Switch away from serial after an exchange has filled the inbox or outgoing lane queues, then switch back and complete the reset handshake.

**Evidence.** `LinkEndpoint` owns unacknowledged frames, three outgoing lanes, peer credit, and an inbox (`rmk/src/split/serial/mod.rs:226-253`). Its `reset` clears only sequence numbers, unacknowledged frames, resend state, and the stuck-ack counter (`rmk/src/split/serial/mod.rs:345-355`). `begin_session` merely sets `session_fresh` (`rmk/src/split/serial/mod.rs:551-556`), and the auto manager reuses the same driver across transport selections (`rmk/src/split/serial/mod.rs:114-152`).

**Impact.** After a reset described as a fresh session, the manager can receive old key/application messages from the retained inbox or transmit outdated control/application messages from retained lanes. A canceled read can readily leave multiple accepted frames in the inbox because each exchange accepts a burst but returns messages one at a time.

**Required repair.** Define which data survives an epoch. Clear the inbox and transient outgoing queues on reset; rebuild authoritative snapshots after link-up. If any payload must survive, tag it with an epoch/generation and reject stale values explicitly.

### 7. Medium — disconnect invalidation never reaches an already-connected host's battery characteristic

**Trigger.** A split peripheral reports a battery percentage, the keyboard remains connected to a BLE host, and that peripheral disconnects.

**Evidence.** The new disconnect path changes the cached battery to `Unavailable` and publishes a `PeripheralBatteryEvent` (`rmk/src/split/driver.rs:50-56,83-97`). The GATT battery task only handles `Available { level: Some(level) }` both in its initial snapshot and event loop (`rmk/src/ble/battery_service.rs:287-316`). It ignores the new `Unavailable` event, leaving the characteristic at the previously notified percentage.

**Impact.** Host-visible battery state remains stale indefinitely after a peripheral disconnect, even though the internal status and Rynk surface say unavailable. The assembled change's invalidation guarantee therefore stops at the internal cache.

**Required repair.** Define a host-visible unavailable representation supported by the selected Battery Service schema and update it on disconnect, or expose connectivity alongside the level so consumers can invalidate the cached percentage. Add a connection-lifetime test that reads the GATT value before and after a peripheral disconnect.

### 8. Medium — valid multi-port nRF configurations generate duplicate TIMER/PPI ownership

**Trigger.** Define two nRF central serial ports and omit `timer`/`ppi_channels` on both, or explicitly reuse the same resources.

**Evidence.** Every nRF port defaults independently to `TIMER2`, `PPI_CH0`, and `PPI_CH1` (`rmk-macro/src/codegen/split/central.rs:122-129`) and moves those peripheral tokens into `BufferedUarte::new` (`rmk-macro/src/codegen/split/central.rs:145-169`). The validator permits multiple central ports and only checks that timer and PPI are specified together; it neither assigns unique defaults nor rejects duplicates (`rmk-config/src/board.rs:173-205`).

**Impact.** A configuration accepted by `rmk-config` expands to Rust that attempts to move the same singleton peripheral resources more than once and fails to compile. Explicit duplicate assignments fail in the same opaque generated-code location.

**Required repair.** Require explicit timer/PPI resources when more than one nRF port exists and validate uniqueness, or allocate a known-disjoint resource tuple by port index and reject configurations that exceed the chip's available resources.

### 9. Low — the mutable advertised BLE name disagrees with the GAP Device Name

**Trigger.** Configure or persist a BLE name template different from `device_config.product_name`, especially one containing `{slot}`, then scan and connect.

**Evidence.** The runtime name module restores and renders the template per profile (`rmk/src/ble/name.rs:41-82`). Advertising uses that rendered value (`rmk/src/ble/mod.rs:289-306`), but the GATT server's GAP configuration is constructed once with the static product name (`rmk/src/ble/mod.rs:229-244`).

**Impact.** The same device advertises as, for example, `Board 2` while a connected client reading GAP Device Name sees the static product name. OS UI and caches can show inconsistent names, and `SetBleName` cannot fulfill the usual expectation that the device name itself changed.

**Required repair.** Use the same rendered value for advertising and GAP Device Name, updating the characteristic on profile/name changes if the stack permits it. If the feature is deliberately advertising-only, rename/document the API to make that contract explicit.

## Architecture proposals

1. **Make transport selection replicated desired state.** Store `(generation, mode)` centrally, have the peripheral session report its applied generation, and only switch the central after acknowledgement. Reconnecting peers receive the latest generation as part of initial synchronization. This removes stale queued commands and the assumption that a transport write equals delivery, and it naturally extends if automatic selection later supports more than one peripheral.

2. **Give half-duplex one phase-owning I/O state machine.** The state machine should own TX, turnaround, RX burst, corruption drain, and cancellation recovery. Derive timing from actual encoded byte counts and configured baud; do not let callers transmit until the state machine has observed the terminator or a worst-case bus-quiet deadline. A dedicated pump can also accept outbound messages with backpressure, avoiding the peripheral writer's acknowledged permanent drop when a lane is full (`rmk/src/split/serial/mod.rs:774-785`).

3. **Define session epochs and snapshot policy.** On every serial/BLE reconnect, clear ephemeral inputs and old commands, then send replaceable authoritative snapshots (connection, layer, lighting/status, battery availability). Epoch-tagged application traffic can opt into replay explicitly. This makes reconnect behavior testable instead of depending on which futures were canceled with local queues populated.

4. **Centralize nRF transport resource allocation.** Code generation should allocate or validate UARTE, timer, PPI channels/groups, pins, and interrupt priorities as one resource set per port. Emit generic `TX_BUF`/`RX_BUF` only for backends that use them; the nRF branch currently allocates those buffers at `rmk-macro/src/codegen/split/central.rs:35-47,186-189` and then uses separate 4096/512-byte rings at `rmk-macro/src/codegen/split/central.rs:145-169`.

5. **Test host-visible identity and availability surfaces end to end.** For each profile, assert that scan name, GAP Device Name, and the Rynk getter agree. During one host connection, drive a split battery from unavailable to available and back to unavailable and assert both notifications and subsequent reads. These tests cover the contract at the transport boundary rather than only internal cache transitions.
