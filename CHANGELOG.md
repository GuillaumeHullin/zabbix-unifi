# Changelog

All notable changes to this template set are documented here. Versioning starts at 1.1.0 (2026-07-08); everything before that was tracked by date only, so those older entries are kept as-is rather than renumbered retroactively.

## [1.1.0] - 2026-07-08

Mostly about false-positive alerts caused by transient local/cloud API failures, the UniFi API returning consoles outside a key's configured scope, and an easier way to set shared local credentials. No template architecture changes.

### Added
- Per-port `Anomalies` item for switch ports and gateway ports (`unifi.usw.port.anomalies[{#PORTID}]` / `unifi.gw.port.anomalies[{#PORTID}]`), alongside the existing per-port `Satisfaction` item. Both are plain monitored values with no trigger attached.

### Fixed
- UniFi's Site Manager API does not enforce an API key's configured site restrictions on `/v1/hosts` or `/v1/sites` - both endpoints always return every console on the account regardless of the key's configured scope (confirmed via direct testing with a site-restricted key). Added `{$UNIFI.SITE.EXCLUDE}`, a client-side workaround macro that filters excluded consoles (matched by hostname, full host ID, or host ID prefix) out of discovery before they're turned into Zabbix hosts. ([#1](../../issues/1))
- Local controller raw items (gateway, UAP, USW, UPS, RPS) no longer treat a failed login/request (timeouts, `429` rate limiting, etc.) as "device not found." Previously, any request failure was silently converted into an empty/fake successful response, which downstream items read as the device being offline, causing false "offline - not seen by local controller" and false `0`-value readings during ordinary network blips or controller rate limiting. A failed poll is now simply skipped (last known value is kept) rather than misreported.
- Confirmed via live testing against a real console: 10 concurrent local controller logins returned `429` on 7 of them, explaining why consoles with many devices (each device independently logging in every poll cycle) were seeing periodic false-offline flapping lasting close to the old 5-poll persistence window.
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
