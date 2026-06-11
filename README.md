# Zabbix Template: Ubiquiti UniFi API

A comprehensive Zabbix 7.0 template set for monitoring Ubiquiti UniFi infrastructure via both the UniFi Site Manager cloud API and local controller endpoints. Provides unified visibility across gateways, access points, switches, UPS units, and redundant power supplies.

---

## Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Template Architecture](#template-architecture)
- [Getting Started](#getting-started)
  - [1. Obtain a Cloud API Key](#1-obtain-a-cloud-api-key)
  - [2. Prepare Local Controller Credentials](#2-prepare-local-controller-credentials)
  - [3. Import the Template](#3-import-the-template)
  - [4. Create the Root Host](#4-create-the-root-host)
  - [5. Configure Host Macros](#5-configure-host-macros)
  - [6. Run Discovery](#6-run-discovery)
  - [7. Configure Per-Host Macros](#7-configure-per-host-macros)
- [Macros Reference](#macros-reference)
- [Alarms Reference](#alarms-reference)
- [WAN Alarm Suppression](#wan-alarm-suppression)
- [High Availability (Shadow Mode)](#high-availability-shadow-mode)
- [Troubleshooting](#troubleshooting)

---

## Overview

This template set bridges the UniFi Site Manager cloud API (`api.ui.com`) with local UniFi OS controller polling. Discovery is automatic - a single root host with your API key discovers all consoles, devices, SD-WAN configs, and sites, and creates Zabbix hosts for each automatically.

**What gets monitored:**

- UniFi console cloud connectivity, firmware, backup age, and release channel
- Per-WAN uptime (24h rolling), latency, external IP changes, plugged state, and downtime periods
- Site-level statistics (offline devices by type, gateway count, critical notifications, IPS mode, combined WAN uptime)
- SD-WAN tunnel health with proportional severity, config errors, hub/spoke WAN issues
- Local gateway CPU, memory, temperature, WAN status, VPN tunnels, HA/Shadow Mode
- Per-AP radio health (band, channel, utilisation, client count, retries)
- Per-switch port status, errors, PoE power, and TX/RX rates
- Per-SFP port temperature, RX/TX optical power (dBm), voltage, current, and TX/RX fault state
- UPS battery level, runtime, mains state, and outlet relay state
- Redundant Power Supply failover and port redundancy state
- Cloud and local controller health checks for all discovered hosts

---

## Requirements

| Component | Minimum Version |
|-----------|----------------|
| Zabbix Server | 7.0 |
| Zabbix Agent | Not required (all HTTP_AGENT) |
| UniFi OS | 3.x or later |
| UniFi Network App | 8.x or later |

The Zabbix server (or proxy) must have HTTPS access to:
- `api.ui.com` (cloud API)
- Each UniFi console's management IP on port 443 (local polling)

---

## Template Architecture

The YAML file contains eight linked templates. You only assign one manually - everything else is created by LLD discovery.

```
Ubiquiti UniFi API          <- Assign this to a single Zabbix host
  |
  +-- [LLD] Discover Hosts
  |     +-- Ubiquiti UniFi API Host    (one host per console)
  |           +-- [LLD] Discover UAP Devices   --> Ubiquiti UniFi Device + Ubiquiti UniFi UAP
  |           +-- [LLD] Discover USW Devices   --> Ubiquiti UniFi Device + Ubiquiti UniFi USW
  |           +-- [LLD] Discover UPS Devices   --> Ubiquiti UniFi Device + Ubiquiti UniFi UPS
  |           +-- [LLD] Discover RPS Devices   --> Ubiquiti UniFi Device + Ubiquiti UniFi RPS
  |           +-- [LLD] Discover WAN Interfaces
  |           +-- [LLD] Discover Host WAN Interfaces (uptime/latency via local API)
  |           +-- [LLD] Discover Site Statistics
  |           +-- [LLD] Discover Host Controllers
  |
  +-- [LLD] Discover SD-WAN Configs
        +-- Ubiquiti UniFi SD-WAN      (one host per SD-WAN config)
```

**Ubiquiti UniFi Device** is a shared base template applied alongside all device-type templates (UAP/USW/UPS/RPS). It provides cloud-side firmware status, reboot detection, and managed state monitoring. Alarms from this base template apply to all device types.

---

## Getting Started

### 1. Obtain a Cloud API Key

The template authenticates to the UniFi Site Manager API using an API key.

1. Log in to [unifi.ui.com](https://unifi.ui.com)
2. Click your profile icon (top right) and select **API Keys** (or navigate to **Account > API Keys**)
3. Click **Create API Key**
4. Give it a descriptive name (e.g. `Zabbix Monitoring`)
5. Copy the key - it is only shown once

**Required permissions:** The API key inherits the permissions of the account that created it. The account must have at least **Viewer** access to all sites you wish to monitor. A dedicated read-only Ubiquiti account is recommended for production use.

> The API key is passed as an `X-API-KEY` header on all requests to `api.ui.com`. It does not grant access to local controller endpoints.

---

### 2. Prepare Local Controller Credentials

Local polling hits each console's UniFi OS controller directly over HTTPS (port 443). A local account with **read-only (Viewer)** permissions is sufficient and recommended.

**To create a local read-only account:**

1. Log in to the UniFi OS console web UI (e.g. `https://10.1.1.1`)
2. Navigate to **Settings > Admins & Users**
3. Click **Add Admin**
4. Set the role to **Viewer** (read-only)
5. Note the username and password

The account must exist on **each console** you want to poll locally. If you use a Ubiquiti SSO account linked to the console, the same credentials work - but a dedicated local account avoids dependency on cloud authentication.

**Permissions required:**
- Read access to the Network application (devices, health, port stats)
- Read access to UniFi OS system stats (gateway CPU/memory/temperature)

No write permissions are needed or recommended.

---

### 3. Import the Template

1. In Zabbix, navigate to **Data collection > Templates**
2. Click **Import**
3. Upload `UnfiZabbixAPI.yaml`
4. Ensure all checkboxes are ticked (Templates, Items, Triggers, etc.)
5. Click **Import**

All eight templates will appear in your template list.

---

### 4. Create the Root Host

Create a single Zabbix host to act as the API entry point. This host does not represent a physical device - it exists solely to run cloud discovery.

1. Navigate to **Data collection > Hosts > Create host**
2. Set:
   - **Host name:** `UniFi Site Manager` (or any name)
   - **Templates:** `Ubiquiti UniFi API`
   - **Host group:** Your preferred group (e.g. `UniFi`)
   - **Interfaces:** None required
3. Click **Add**

---

### 5. Configure Host Macros

On the root host, open the **Macros** tab and set:

| Macro | Value | Notes |
|-------|-------|-------|
| `{$APIKEY}` | `your-api-key-here` | From Step 1. Use **Secret text** type. |

Optional threshold overrides (defaults are suitable for most deployments):

| Macro | Default | Description |
|-------|---------|-------------|
| `{$WAN_UPTIME_WARN}` | `99` | Site-wide combined WAN uptime % for AVERAGE alarm |
| `{$WAN_UPTIME_HIGH}` | `95` | Site-wide combined WAN uptime % for DISASTER alarm |

---

### 6. Run Discovery

LLD runs automatically on the configured interval (default: 1 hour). To trigger immediately:

1. Navigate to the root host
2. Go to **Data collection > Discovery rules**
3. Click **Execute now** on **Discover UniFi Hosts** and **Discover SD-WAN Configs**

After a few minutes, new hosts will appear for each discovered console, pre-populated with:

| Auto-populated Macro | Source |
|----------------------|--------|
| `{$HOSTID}` | Console UUID from cloud API |
| `{$UNIFI.LOCAL.IP}` | First private IP from the console's reported addresses |
| `{$DEVICENAME}` | Console display name |

---

### 7. Configure Per-Host Macros

Each discovered console host requires local controller credentials. Open each console host, go to the **Macros** tab, and set:

| Macro | Value | Notes |
|-------|-------|-------|
| `{$UNIFI.USERNAME}` | `zabbix-viewer` | Local controller read-only username |
| `{$UNIFI.PASSWORD}` | `your-password` | Use **Secret text** type |

Common optional overrides:

| Macro | Default | When to change |
|-------|---------|----------------|
| `{$UNIFI.WAN.LATENCY.WARN}` | `100` (ms) | Raise to 200+ for high-latency links (e.g. Central America sites with 60-80ms base RTT to UK) |
| `{$UNIFI.WAN.ENABLED:WAN2}` | `1` | Set to `0` if WAN2 is not physically connected on this gateway |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | Adjust per device capability |
| `{$UNIFI.TEMP.WARN}` | `70` | Adjust for deployment environment |
| `{$BACKUP_INTERVAL}` | `8d` | Match your configured UniFi backup schedule |

---

## Macros Reference

### Global (Root Template — Ubiquiti UniFi API)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$APIKEY}` | *(secret)* | UniFi Site Manager API key |
| `{$WAN_UPTIME_WARN}` | `99` | Site-wide combined WAN uptime AVERAGE threshold (%) |
| `{$WAN_UPTIME_HIGH}` | `95` | Site-wide combined WAN uptime DISASTER threshold (%) |

### Per-Console Host (Ubiquiti UniFi API Host)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$APIKEY}` | *(secret)* | Inherited or overridden API key |
| `{$UNIFI.LOCAL.IP}` | *(auto)* | Console management IP - set by discovery |
| `{$UNIFI.USERNAME}` | *(empty)* | Local controller read-only username |
| `{$UNIFI.PASSWORD}` | *(secret)* | Local controller password |
| `{$UNIFI.API.AUTH.URI}` | `api/auth/login` | Login endpoint |
| `{$UNIFI.API.AUTH.TOKEN}` | `TOKEN` | Session cookie name |
| `{$UNIFI.API.URI}` | `proxy/network/api/s/default/stat` | Stats endpoint |
| `{$UNIFI.API.REST.URI}` | `proxy/network/api/s/default/rest` | REST endpoint |
| `{$HOSTID}` | *(auto)* | Console UUID from cloud - set by discovery |
| `{$BACKUP_INTERVAL}` | `8d` | Expected maximum time between backups |
| `{$WAN_UPTIME_NOTICE}` | `99.9` | Per-WAN 24h availability WARNING threshold (%) — ~1.4 min/day |
| `{$WAN_UPTIME_WARN}` | `99` | Per-WAN 24h availability AVERAGE threshold (%) — ~14 min/day |
| `{$WAN_UPTIME_HIGH}` | `98` | Per-WAN 24h availability HIGH threshold — 0% fires a separate alarm |
| `{$UNIFI.WAN.LATENCY.WARN}` | `100` | WAN latency warning threshold (ms) |
| `{$UNIFI.WAN.ENABLED}` | `1` | Master WAN alarm switch (1=enabled, 0=suppressed for all WANs) |
| `{$UNIFI.WAN.ENABLED:WAN2}` | `1` | WAN2-specific alarm switch — set to 0 to suppress all WAN2 alarms |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | CPU usage warning threshold (%) |
| `{$UNIFI.CPU.USAGE.HIGH}` | `90` | CPU usage critical threshold (%) |
| `{$UNIFI.MEM.USAGE.WARN}` | `80` | Memory usage warning threshold (%) |
| `{$UNIFI.MEM.USAGE.HIGH}` | `90` | Memory usage critical threshold (%) |
| `{$UNIFI.TEMP.WARN}` | `70` | Temperature warning threshold (°C) |
| `{$UNIFI.TEMP.HIGH}` | `80` | Temperature critical threshold (°C) |

### Per-Device (UAP / USW / UPS / RPS)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$UNIFI.UPTIME.WARN}` | `24` | Uptime warning threshold (hours) - alarms if uptime is below this after a reboot |
| `{$UNIFI.CPU.USAGE.WARN}` | `80` | CPU warning (%) |
| `{$UNIFI.CPU.USAGE.HIGH}` | `90` | CPU critical (%) — UAP/USW only |
| `{$UNIFI.MEM.USAGE.WARN}` | `80` | Memory warning (%) |
| `{$UNIFI.MEM.USAGE.HIGH}` | `90` | Memory critical (%) — UAP/USW only |
| `{$UNIFI.TEMP.WARN}` | `55` | Temperature warning (°C) — RPS only |
| `{$UNIFI.TEMP.HIGH}` | `65` | Temperature critical (°C) — RPS only |
| `{$UNIFI.UPS.BATTERY.WARN}` | `20` | UPS battery warning threshold (%) |
| `{$UNIFI.UPS.BATTERY.HIGH}` | `10` | UPS battery critical threshold (%) |
| `{$UNIFI.UPS.RUNTIME.HIGH}` | `300` | UPS runtime critical threshold (seconds) |

### Per-SFP Port (USW — discovered automatically for ports with `sfp_found = true`)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$UNIFI.SFP.TEMP.WARN}` | `70` | SFP module temperature warning threshold (°C). Active 10G SFP+ modules routinely run 60–65°C; 70°C indicates a genuinely elevated condition. |
| `{$UNIFI.SFP.TEMP.HIGH}` | `75` | SFP module temperature critical threshold (°C). Commercial SFP+ modules are typically rated to 70°C; above this degradation risk is high. |
| `{$UNIFI.SFP.RXPOWER.LOW.WARN}` | `-12` | RX optical power warning threshold (dBm) |
| `{$UNIFI.SFP.RXPOWER.LOW.HIGH}` | `-16` | RX optical power critical threshold (dBm). Near receiver sensitivity limit for 10GBase-LR. |
| `{$UNIFI.SFP.TXPOWER.LOW.WARN}` | `-6` | TX laser power warning threshold (dBm) |
| `{$UNIFI.SFP.TXPOWER.LOW.HIGH}` | `-9` | TX laser power critical threshold (dBm). Near minimum spec for 10GBase-LR. |

> **SFP temperature note:** SFP+ modules run significantly hotter than the switch chassis. 60–65°C is normal for an active 10G module in a warm rack. Raise `{$UNIFI.SFP.TEMP.WARN}` to `67` on a per-host basis if you see persistent warnings on healthy modules in a warm environment.

---

## Alarms Reference

Severities follow standard Zabbix conventions: INFO < WARNING < AVERAGE < HIGH < DISASTER.

### Cloud Connectivity and Console Health

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Console lost cloud connectivity | DISASTER | Console state is not `connected` via cloud API |
| Host is blocked by Ubiquiti | HIGH | Console account is suspended or blocked |
| Unadopted UniFi OS devices found | WARNING | One or more UniFiOS devices are pending adoption |
| No backup within expected interval | AVERAGE | Last backup older than `{$BACKUP_INTERVAL}` |
| UniFi OS firmware update available | INFO | A newer firmware version is available |
| UniFi OS version changed | INFO | Firmware version changed (post-upgrade tracking) |
| Console on non-production release channel | INFO | Console is on a beta or testing channel |
| Cloud system logging not enabled | INFO | Cloud log forwarding is disabled |
| Console device state changed | INFO | Device state transitioned (e.g. updating, setup) |

### Gateway - Local Polling

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| HA failover: gateway is now STANDBY | DISASTER | Primary failed; this unit is now serving traffic |
| WAN status is not OK | HIGH | Local WAN health reports a non-ok state |
| Shadow Mode link (eth6) disconnected | HIGH | HA interconnect cable is unplugged |
| Gateway CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Gateway CPU temperature critical | HIGH | CPU temp > `{$UNIFI.TEMP.HIGH}`°C |
| Gateway board temperature critical | HIGH | Board temp > `{$UNIFI.TEMP.HIGH}`°C |
| Gateway memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| Gateway CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Gateway CPU temperature high | WARNING | CPU temp > `{$UNIFI.TEMP.WARN}`°C |
| Gateway board temperature high | WARNING | Board temp > `{$UNIFI.TEMP.WARN}`°C |
| Gateway memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| WAN IP changed | WARNING | Local WAN IP changed from one value to another |
| ISP name changed | INFO | ISP name reported by the local controller changed |
| Shadow Mode peer IP changed | INFO | HA peer management IP changed |
| Local firmware version changed | INFO | Local controller firmware version changed |

### Per-WAN (discovered automatically — two LLD rules)

WAN uptime and latency are sourced from the **local controller** (`/stat/health`), which provides a true 24-hour rolling window and per-WAN latency readings. Downtime and packet loss period counts are sourced from the **cloud API** (`/v1/sites`), which tracks individual outage events at ~5-minute granularity.

Two sets of WAN items are discovered per gateway:

- **Discover WAN Interfaces** (cloud API) — plugged state, enabled state, speed type, IPv4 address. These work even if the local controller is unreachable.
- **Discover Host WAN Interfaces** (local API) — 24h uptime, latency, downtime periods, packet loss periods, external IP.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| {WAN} availability is 0% (completely down) | HIGH | 24h availability = 0% and `{$UNIFI.WAN.ENABLED:{WAN}}=1` |
| {WAN} physically unplugged | HIGH | Plugged = false for 3 consecutive checks and `{$UNIFI.WAN.ENABLED:{WAN}}=1` |
| {WAN} sustained downtime (3/3 periods) | HIGH | All 3 of the most recent outage periods show downtime |
| {WAN} 24h availability degraded | AVERAGE | 24h availability < `{$WAN_UPTIME_WARN}`% (~14 min/day down) |
| {WAN} intermittent downtime (2/3 periods) | AVERAGE | 2 of the last 3 outage periods show downtime |
| {WAN} 24h availability slightly below target | WARNING | 24h availability < `{$WAN_UPTIME_NOTICE}`% but ≥ `{$WAN_UPTIME_WARN}`% |
| {WAN} packet loss in 2/3 recent periods | WARNING | Packet loss in 2 or more of the last 3 measurement periods |
| {WAN} latency high | WARNING | 24h average latency > `{$UNIFI.WAN.LATENCY.WARN}` ms |
| {WAN} administratively disabled | WARNING | WAN interface disabled in UniFi config |
| {WAN} IPv4 address changed | WARNING | Local IPv4 address on this WAN changed |
| {WAN} external IP changed | INFO | Public (ISP-visible) IP changed |

> **Latency tip:** For sites where 60–80 ms to the hub is normal (e.g. Central America to UK), raise `{$UNIFI.WAN.LATENCY.WARN}` to `200` or higher on those gateway hosts to avoid false positives.

### Site Statistics (discovered automatically)

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Site has no gateway device | DISASTER | Gateway device count = 0 — all routing is down |
| Site aggregate WAN uptime critical | DISASTER | Combined WAN uptime < `{$WAN_UPTIME_HIGH}`% |
| Site has critical notification(s) in UniFi | AVERAGE | UniFi is reporting one or more critical notifications |
| Site aggregate WAN uptime degraded | AVERAGE | Combined WAN uptime < `{$WAN_UPTIME_WARN}`% |
| Site has 1 offline device | AVERAGE | One device has gone offline |
| Site has 1 offline WiFi device | AVERAGE | One AP has gone offline |
| Site has 1 offline wired device | AVERAGE | One switch/wired device has gone offline |
| Site has multiple offline devices | HIGH | More than one device offline simultaneously |
| Site has multiple offline WiFi devices | HIGH | More than one AP offline simultaneously |
| Site has multiple offline wired devices | HIGH | More than one wired device offline simultaneously |
| Site has offline gateway device(s) | HIGH | One or more gateways are offline |
| IPS/IDS disabled or not configured | WARNING | Threat management is not active on this site |
| Site has device(s) with firmware updates | INFO | One or more devices have firmware updates queued |

### SD-WAN

Tunnel severity is proportional to actual impact. A dual-WAN spoke losing one WAN drops some tunnels but remains connected via its other WAN — that is HIGH, not DISASTER. DISASTER only fires when a site is completely cut off or a hub loses all spoke connections.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Spoke(s) completely isolated — all tunnels down | DISASTER | One or more spokes have zero connected tunnels (all WANs lost) |
| Hub(s) have no active spoke connections | DISASTER | A hub has no spokes connected to it at all |
| SD-WAN config errors | HIGH | Configuration has errors that may prevent tunnels establishing |
| Hub WAN internet issues | HIGH | A hub site is reporting WAN internet issues |
| Spoke WAN internet issues | HIGH | A spoke site is reporting WAN internet issues |
| SD-WAN config generation failed | HIGH | Config generation status is not OK |
| SD-WAN tunnel(s) disconnected | HIGH | One or more tunnels are down (site may still have other active paths) |
| SD-WAN config warnings | WARNING | Configuration has warnings |

> **Example:** Guatemala has two WANs (WAN and WAN2), each with a tunnel to the UK hub. If WAN goes down, two tunnels drop → HIGH (`disconnectedTunnels`). WAN2 still has its tunnel, so Guatemala is not isolated. If both WANs go down → zero connected tunnels → DISASTER (`isolatedSpokes`).

### Controllers (discovered automatically)

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Controller not running | HIGH | A UniFi application (network, protect, etc.) has stopped |
| Controller unadopted device(s) found | WARNING | One or more devices are unadopted in this controller |
| Controller update available | INFO | A new software version is available for this application |
| Controller version changed | INFO | Application version changed (post-update tracking) |

### Access Points (UAP)

The following alarms come from the **Ubiquiti UniFi UAP** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline — not seen by local controller | HIGH | Device state ≠ 1 for 3 consecutive checks |
| Device offline (cloud) | HIGH | Cloud API reports status = offline |
| CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| Radio not running | HIGH | Radio state is not RUN |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Firmware upgrade available | WARNING | Local controller reports an upgrade is available |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold — recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

### Switches (USW)

The following alarms come from the **Ubiquiti UniFi USW** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline — not seen by local controller | HIGH | Device state ≠ 1 for 3 consecutive checks |
| Device offline (cloud) | HIGH | Cloud API reports status = offline |
| CPU usage critical | HIGH | CPU > `{$UNIFI.CPU.USAGE.HIGH}`% |
| Memory usage critical | HIGH | Memory > `{$UNIFI.MEM.USAGE.HIGH}`% |
| SFP TX fault active | HIGH | Module reports a transmit laser fault — likely failed SFP or broken TX fibre |
| SFP RX fault active | HIGH | Module reports a receive fault — likely broken RX fibre or failed remote laser |
| SFP temperature critical | HIGH | SFP module temp > `{$UNIFI.SFP.TEMP.HIGH}`°C |
| SFP RX power critical low | HIGH | Received optical power < `{$UNIFI.SFP.RXPOWER.LOW.HIGH}` dBm — near receiver limit |
| SFP TX power critical low | HIGH | Transmit laser output < `{$UNIFI.SFP.TXPOWER.LOW.HIGH}` dBm — laser likely failing |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Port down | WARNING | Port transitions from up to down after previously being up |
| Port TX errors increasing | WARNING | TX error counter is actively incrementing |
| Port RX errors increasing | WARNING | RX error counter is actively incrementing |
| SFP temperature high | WARNING | SFP module temp > `{$UNIFI.SFP.TEMP.WARN}`°C |
| SFP RX power low | WARNING | Received optical power < `{$UNIFI.SFP.RXPOWER.LOW.WARN}` dBm |
| SFP TX power low | WARNING | Transmit laser output < `{$UNIFI.SFP.TXPOWER.LOW.WARN}` dBm |
| Firmware upgrade available | WARNING | Local controller reports an upgrade is available |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold — recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |

> **Port down alarms** resolve immediately if the port comes back up. If the port remains down, the alarm auto-resolves after 24 hours. Operators can also close it manually for ports that are intentionally unused.

> **SFP monitoring** is discovered automatically. Items are only created for ports where `sfp_found = true` (i.e. a module is physically inserted). Voltage and laser current are logged for trending but have no alarm thresholds by default. All SFP alarms are suppressed if the device is offline. Supported models: any USW with SFP/SFP+ ports (confirmed on USW-Pro-Aggregation `USAGGPRO`).

### UPS (Uninterruptible Power Supply)

The following alarms come from the **Ubiquiti UniFi UPS** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| Offline — not seen by local controller | HIGH | Device state ≠ 1 for 3 consecutive checks |
| Device offline (cloud) | HIGH | Cloud API reports status = offline |
| Battery level critical | HIGH | Battery < `{$UNIFI.UPS.BATTERY.HIGH}`% |
| Battery runtime critical | HIGH | Remaining runtime < `{$UNIFI.UPS.RUNTIME.HIGH}` seconds |
| Running on battery — mains power failed | HIGH | UPS is in battery mode |
| Battery level low | WARNING | Battery < `{$UNIFI.UPS.BATTERY.WARN}`% |
| Outlet lost power | WARNING | Outlet relay state = off |
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold — recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

### Redundant Power Supply (RPS)

The following alarms come from the **Ubiquiti UniFi RPS** template (local controller polling) and the **Ubiquiti UniFi Device** base template (cloud API), both applied together.

| Alarm | Severity | Fires when |
|-------|----------|-----------|
| RPS delivering 12V — connected device PSU failed | DISASTER | RPS is actively supplying 12V to a device |
| RPS delivering 54V — connected device PSU failed | DISASTER | RPS is actively supplying 54V (PoE) to a device |
| RPS port ACTIVE — primary PSU failed on connected device | DISASTER | RPS port has taken over from a failed PSU |
| Offline — not seen by local controller | HIGH | Device state ≠ 1 for 3 consecutive checks |
| Device offline (cloud) | HIGH | Cloud API reports status = offline |
| Temperature critical | HIGH | Temp > `{$UNIFI.TEMP.HIGH}`°C |
| Temperature high | WARNING | Temp > `{$UNIFI.TEMP.WARN}`°C |
| RPS port disconnected — redundancy lost | WARNING | Port is not connected; device has no redundant power path |

> **RPS port discovery** only creates items for ports that have a connected device. Ports with a generic name (e.g. `Port 1`, `Port 2`) are filtered out — this is by design, as those ports have nothing plugged in yet. Once a device is connected, the RPS names the port after the device's hostname (e.g. `FRN1CORESW01`), and it is automatically discovered on the next LLD cycle.
| CPU usage high | WARNING | CPU > `{$UNIFI.CPU.USAGE.WARN}`% |
| Memory usage high | WARNING | Memory > `{$UNIFI.MEM.USAGE.WARN}`% |
| Uptime less than {$UNIFI.UPTIME.WARN}h | WARNING | Device uptime below threshold — recent reboot |
| Device no longer managed | AVERAGE | Device has been removed from management |
| Firmware update available (cloud) | INFO | Cloud API reports a newer version is available |
| Device rebooted | INFO | Startup timestamp changed by more than 5 minutes |
| Firmware version changed | INFO | Firmware version changed |

---

## WAN Alarm Suppression

For gateways where a WAN port is intentionally unused, set a host-level context macro to suppress both the unplugged and 0% uptime alarms for that WAN:

| Macro | Value | Effect |
|-------|-------|--------|
| `{$UNIFI.WAN.ENABLED:WAN2}` | `0` | Suppresses all WAN2 alarms (uptime and plugged) |
| `{$UNIFI.WAN.ENABLED:WAN1}` | `0` | Suppresses all WAN1 alarms (not recommended on a primary) |
| `{$UNIFI.WAN.ENABLED}` | `0` | Suppresses WAN alarms for all WANs on this host |

**To add a host macro:**
1. Go to **Data collection > Hosts** and open the console host
2. Click the **Macros** tab
3. Add the macro name and value, then click **Update**

Changes take effect on the next trigger evaluation (within one poll cycle).

---

## High Availability (Shadow Mode)

For deployments using UniFi HA (two gateway units in Shadow Mode):

- **HA failover: gateway is now STANDBY** fires at DISASTER severity when the primary unit fails and this unit takes over active routing
- **Shadow Mode link (eth6) physically disconnected** fires at HIGH when the HA interconnect cable is unplugged, meaning redundancy is lost even if the primary is still running
- **Shadow Mode peer IP changed** fires at INFO if the management IP of the standby unit changes unexpectedly
- Trigger dependencies suppress downstream device alarms when the console itself goes offline

No additional configuration is required - HA status is detected automatically from the local controller's Shadow Mode API.

---

## Troubleshooting

**Discovery runs but no hosts are created**
- Verify `{$APIKEY}` is set correctly on the root host
- Check the Zabbix server can reach `api.ui.com`
- Check the `unifi.api.hosts` item for errors under **Monitoring > Latest data**

**Local items show connection errors or timeouts**
- Confirm `{$UNIFI.LOCAL.IP}` is populated on the console host (set automatically by discovery)
- Confirm the Zabbix server/proxy can reach that IP on port 443
- Check `{$UNIFI.USERNAME}` and `{$UNIFI.PASSWORD}` are set and the account exists locally on that console
- Some consoles require the account to have logged in via the UI at least once before API auth succeeds
- Brief timeouts (10s) are expected if the console temporarily loses management connectivity — these clear automatically

**WAN uptime or latency shows 0 / incorrect values**
- Both items are sourced from the local controller (`/stat/health`). If the local controller is unreachable, they fall back to safe defaults (100% / 0ms) until the connection recovers
- Confirm `Local Health Raw` is collecting successfully under **Monitoring > Latest data**

**WAN2 alarms on a single-WAN gateway**
- Set host macro `{$UNIFI.WAN.ENABLED:WAN2}=0` — see [WAN Alarm Suppression](#wan-alarm-suppression)

**False reboot alarms on newly-discovered devices**
- The startup time trigger requires the stored timestamp to change by more than 5 minutes, filtering polling jitter
- If still noisy on first discovery, acknowledge and close — the alarm only fires on genuine changes thereafter

**Port error alarms that won't clear automatically**
- Port TX/RX error triggers use cumulative counters and are marked `manual_close`
- Acknowledge and close the alarm in Zabbix once the underlying issue is resolved

**HTTP 429 errors from the local controller**
- All local device polls default to 4-minute intervals to stay within the controller's rate limits
- If 429s persist, increase the interval on the affected master item via a host-level item override
