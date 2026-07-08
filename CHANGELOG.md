# Changelog

All notable changes to this template set are documented here. Versioning starts at 1.1.0 (2026-07-08); everything before that was tracked by date only, so those older entries are kept as-is rather than renumbered retroactively.

## [1.1.0.1] - 2026-07-08

A large expansion of per-port monitoring: SFP module metadata, full gateway SFP support (previously switch-only), port renaming/QoS visibility, and several new Ethernet-level fields - plus a template-wide `DISCARD_UNCHANGED_HEARTBEAT` consistency pass.

### Added
- SFP module metadata items: `Compliance`, `Part Number`, `Serial Number`, `Vendor`, `Revision` on switches; `Part Number`, `Serial Number`, `Vendor`, `EEPROM Readable` on gateways (gateways don't expose compliance/revision). An INFO alarm fires when a module's serial number changes, indicating a physical swap rather than a reseat.
- Gateway SFP monitoring, previously switch-only: `Discover Gateway SFP Ports` (TX fault, RX loss-of-signal - the gateway-side equivalent of a switch's RX fault) and `Discover Gateway SFP Optical Ports` (temperature, RX/TX power, voltage, current). Gateways use different field names than switches (`sfp_tx_fault`/`sfp_rx_los` vs. `sfp_txfault`/`sfp_rxfault`) and don't expose an `sfp_compliance` field, so the optical/DAC split uses a part-number-prefix filter instead of the compliance field switches use.
- `Port Name` item for every port (switch and gateway, not just SFP ones), with an INFO alarm when it changes - UniFi auto-renames a port after the connected device's hostname, so this tracks patching changes over time. Previously the port name only existed as a discovery-time label with no history.
- New per-port Ethernet items, switch and gateway, informational unless noted: `Full Duplex`, `Autoneg`, `Is Uplink`, `Link Down Count` (cumulative flap counter), `MAC Table Count` (learned MAC addresses - spot an unexpected hub/switch downstream), `RX/TX Dropped` packets (**new WARNING alarm** if actively increasing, mirroring the existing TX/RX Errors pattern), `STP State` (**new WARNING alarm** if stuck on `blocking`, an active loop-prevention signal), `STP Edge Port`, and `QoS Mode` (surfaces Pro AV/traffic-control profiles like `aes67_audio`).

### Fixed
- Optical-only SFP items (`Temperature`, `RX/TX Power`, `Voltage`, `Current`) were previously created for every SFP-found port, including DAC (direct-attach copper) cables, which structurally can't report any of those values (no laser or photodiode). Confirmed live by comparing the same physical DAC cable from both ends (a gateway-to-switch uplink): identical missing fields on both sides. These items are now only discovered on non-DAC (fibre) modules; DAC ports still get the fault/info tier (TX fault, part number, serial, vendor, etc.), just not the optical measurements that can never populate.
- Extended `DISCARD_UNCHANGED_HEARTBEAT` to 9 more slow-changing metadata items that were missing it inconsistently with their siblings (e.g. `UniFiOS Version` didn't have it while the equivalent per-device `Firmware Version` did): Hardware Type, Public IP Address, Hardware Serial Number, Unifi Stack status, UniFiOS Version, WAN IPv4 Address, WAN Speed Type, per-controller Version, and Radio Band. Items that directly drive an alarm as their primary state input (connection state, blocked, WAN enabled, outlet relay, RPS state, etc.) were deliberately left alone, so they keep a live timestamp at all times.

---

## [1.1.0] - 2026-07-08

Mostly about false-positive alerts caused by transient local/cloud API failures, the UniFi API returning consoles outside a key's configured scope, and an easier way to set shared local credentials. No template architecture changes.

### Added
- Per-port `Anomalies` item for switch ports and gateway ports (`unifi.usw.port.anomalies[{#PORTID}]` / `unifi.gw.port.anomalies[{#PORTID}]`), alongside the existing per-port `Satisfaction` item. Both are plain monitored values with no trigger attached.

### Fixed
- UniFi's Site Manager API does not enforce an API key's configured site restrictions on `/v1/hosts` or `/v1/sites` - both endpoints always return every console on the account regardless of the key's configured scope (confirmed via direct testing with a site-restricted key). Added `{$UNIFI.SITE.EXCLUDE}`, a client-side workaround macro that filters excluded consoles (matched by hostname, full host ID, or host ID prefix) out of discovery before they're turned into Zabbix hosts. ([#1](../../issues/1))
- Local controller raw items (gateway, UAP, USW, UPS, RPS) no longer treat a failed login/request (timeouts, `429` rate limiting, etc.) as "device not found." Previously, any request failure was silently converted into an empty/fake successful response, which downstream items read as the device being offline, causing false "offline - not seen by local controller" and false `0`-value readings during ordinary network blips or controller rate limiting. A failed poll is now simply skipped (last known value is kept) rather than misreported.
- Confirmed via live testing against a real console: 10 concurrent local controller logins returned `429` on 7 of them, explaining why consoles with many devices (each device independently logging in every poll cycle) were seeing periodic false-offline flapping lasting close to the old 5-poll persistence window.
- Fixed the trigger-prototype dependency on the offline triggers, which requires matching the target's `recovery_expression` exactly, not just name and expression, or Zabbix reports the dependency as "does not exist" on import.
- Per-port `Satisfaction` items (switch and gateway) had no `value_type` set, defaulting to Numeric (unsigned), which can't store the `-1` fallback used when a port doesn't expose this field. Every device-level `Satisfaction` item already correctly used `FLOAT`; the port-level ones just missed it. Confirmed live that some switch models (e.g. `US24PRO`) never expose per-port satisfaction at all, regardless of link state, so this fallback path is hit routinely, not just on edge cases.

### Changed
- **"{HOST.NAME} offline - not seen by local controller"** triggers (UAP/USW/UPS/RPS) now alarm on the first genuinely bad poll and recover on the first good poll, instead of requiring 5 consecutive bad/good polls in each direction. This is safe now that transient failures no longer produce fake bad readings (see Fixed, above): a bad reading only ever comes from the controller genuinely reporting the device down.
- Local controller credentials (`{$UNIFI.USERNAME}` / `{$UNIFI.PASSWORD}`) can now be set **once** on the root host and are automatically inherited by every discovered console, instead of requiring manual entry on every console individually. A per-console override still works for sites needing different credentials.

### Removed
- Two unused root-level cloud API items (`Unifi API Raw: Devices` - `GET /v1/devices`, and `Unifi API Raw: Sites` - `GET /v1/sites`) that were polled every minute with zero downstream consumers anywhere in the template.

### Investigated, no change made
- Tried to fix the redundant polling at its source: have each device read the console's already-fetched local/cloud data instead of re-fetching it itself. Not possible with Zabbix's built-in item types. `DEPENDENT` items require the master item on the same host, and `CALCULATED` items need a literal host name baked into the formula at save time, before any macro is resolved, but console and device host names only exist once LLD has discovered them. Reverted the attempt. The only way to genuinely fix this is turning per-device metrics into LLD item prototypes living on the console host instead of giving each device its own Zabbix host, which is a real architecture change, not a drop-in fix. Documented as a known limitation in the README instead.
- Also considered just merging each device's local and cloud raw items into a single item, purely to cut the item count. Rejected: it doesn't reduce the number of actual HTTP requests at all (still one login, one local fetch, one cloud fetch, regardless of how many Zabbix items they're wrapped in), and it would couple two failure modes that are usefully independent today.

---

## 2026-06-18

### Fixed
- `WAN status is not OK` no longer writes the literal value `unknown` into item history on a transient read failure (changed error handling from `CUSTOM_VALUE` to `DISCARD_VALUE`), so a momentary local API hiccup doesn't leave a misleading value behind.
- Raised the "offline - not seen by local controller" persistence window from 3 to 5 consecutive bad polls, reducing false positives from brief local API blips.

### Docs
- Corrected the UniFi UI navigation steps for creating a cloud API key and a local read-only account - the API key is in the left sidebar of unifi.ui.com, not under a profile icon, and local credentials are created via the People icon in the Network app.

## 2026-06-11

### Added
- SFP port monitoring for switches: temperature, RX/TX optical power (dBm), voltage, current, and TX/RX fault state, discovered automatically for ports with an SFP present.
- Support for UniFi aggregation switches (`USAGG`) in switch device discovery matching.

### Docs
- Added RPS (Redundant Power Supply) naming instructions to the README.

## 2026-06-01

### Added
- Initial public release: UniFi Site Manager cloud API + local UniFi OS controller polling for consoles, access points, switches, UPS units, and redundant power supplies, with automatic LLD-based discovery of hosts, devices, WAN interfaces, SD-WAN configs, and site statistics.
