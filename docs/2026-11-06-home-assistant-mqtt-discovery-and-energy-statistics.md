# Case Study: Fixing MQTT Light Capability Mismatches and Home Assistant Energy Statistics

**Date:** 6 November 2026  
**Status:** Two MQTT group fixes and short-term Recorder recovery VERIFIED; a later 7 November audit also VERIFIED resumed hourly statistics. Broader historical continuity remains PENDING.

## Overview

A Dockerized Home Assistant / Zigbee2MQTT / Mosquitto installation emitted repeated MQTT light warnings and stopped compiling long-term statistics for two energy sensors. The work emphasized tracing event payloads through the stack and validating narrow changes instead of resetting devices or upgrading everything at once.

## Environment

- Ubuntu Linux host, Docker Compose
- Home Assistant Core, Zigbee2MQTT, Mosquitto MQTT broker
- PowerShell-to-SSH command runner for controlled, operator-authorized commands
- Home Assistant Recorder using SQLite

No public hostnames, LAN addresses, credentials, unique device identifiers or private backup paths are included.

## Problem 1: The group publishes a color mode not advertised by Discovery

Home Assistant logs contained:

~~~text
Invalid color mode 'color_temp' received for entity light.bedroom_ceiling
Invalid color mode 'color_temp' received for entity light.entryway
~~~

An investigation mapped the two HA light entities to Zigbee2MQTT **groups**, each containing one warm-white, brightness-only bulb. MQTT Discovery advertised `supported_color_modes: ['brightness']`, while captured live group state included `color_mode: color_temp` and `color_temp: 454`. Individually published bulb states did **not** contain those fields.

This distinction matters: changing the bulb itself or asserting a mixed-model group bug would have missed the demonstrated source of the mismatch.

### Implemented change

Scoped Zigbee2MQTT filters removed the two unwanted fields from group state publications:

~~~yaml
groups:
  '5':
    friendly_name: g_bedroom_ceiling
    filtered_attributes:
      - '^color_mode$'
      - '^color_temp$'
  '6':
    friendly_name: g_entryway_ceiling
    filtered_attributes:
      - '^color_mode$'
      - '^color_temp$'
~~~

The configuration was backed up before the change; Zigbee2MQTT was restarted. A duplicate execution was blocked by the script's existing-filter preflight.

### Verification

- A post-change live MQTT subscription received messages from **both groups** after light toggles and brightness changes.
- Both groups continued publishing `state` and `brightness`.
- **Zero** captured post-change messages contained `color_mode` or `color_temp`; the checker reported **PASS**.
- A subsequent 15-minute Home Assistant log check had no `Invalid color mode` entries.

**Boundary:** This proves the narrow message-level fix and short-window log result; not all future device behavior.

## Problem 2: Energy statistics unit mismatch

Two daily-energy entities reported that their current `kWh` measurements could not be converted to historical `None` units. Inspection of `statistics_meta` found:

~~~text
unit_of_measurement = NULL
unit_class = unitless
source = recorder
has_sum = 1
~~~

Historical numeric states matched the existing energy dashboard, so deleting or converting numbers would have thrown away valid observations.

### Implemented change

With Home Assistant stopped, a separate consistent SQLite backup was created and its integrity verified. A guarded transaction changed **only the two affected statistics metadata records**:

~~~sql
UPDATE statistics_meta
SET unit_of_measurement = 'kWh',
    unit_class = 'energy'
WHERE statistic_id = ?
  AND source = 'recorder'
  AND unit_of_measurement IS NULL
  AND unit_class = 'unitless';
~~~

The original run validated exact row counts and both resulting metadata records before restarting Home Assistant. Historical numeric states, sums and measurement rows were not altered. Direct database modification is a higher-risk, exceptional maintenance approach; prefer supported application repair flows when acceptable, and never edit a live SQLite Recorder database without appropriate safeguards.

### Verification

- Consistent pre-change SQLite backup passed `PRAGMA quick_check`.
- Home Assistant restarted and responded HTTP **200**.
- Both affected entities reported numeric `kWh` states and `total_increasing` state class.
- New 5-minute statistics rows were observed for **both** sensors with correctly labeled `kWh / energy` metadata.
- No matching unit-conflict warning appeared in the immediate post-restart log sample.

An initial check incorrectly showed zero new records because it compared the statistics **window-start** timestamp with the precise repair second. A follow-up correctly recognized the new 5-minute window. This is an example of why verification queries must respect sampling windows.

**Pending:** confirmation of the next hourly aggregate, dashboard continuity and a later review of daily-reset sensor semantics. Reassigning metadata does not itself rebuild historical sums.

## Additional check: Wi-Fi energy socket

Earlier TP-Link/Kasa polling timeouts were not reproduced in the reviewed 24-hour HA logs. The energy socket's state was available, and a voltage measurement had a fresh update. No device configuration change was made, and intermittent failures are not categorically ruled out.

## Lessons learned

1. **Compare Discovery with live MQTT data.** Entity names alone obscure whether an integration entity represents a bulb or a Zigbee group.
2. **Trace bugs to the producing layer.** Here the group payload, not the individual warm-white bulb, carried invalid color fields.
3. **Use narrow changes with observable tests.** Two filtered attributes fixed the observed group mismatch without resetting the Zigbee mesh.
4. **Protect valid telemetry history.** Backing up the database and changing only statistics metadata preserved historical numeric values.
5. **Respect time-bucket semantics.** A five-minute bucket starting before a change may still be the first new record after it.
6. **Separate verified outcomes from pending work.** Short-term statistics success does not automatically prove the next hourly rollup or full disaster recovery.

See the separate [encrypted backup and restore case study](../backup/encrypted-restic-sftpgo-windows-backup-and-restore.md) for the host's operational backup evidence.

## Later hourly-statistics checkpoint (7 November 2026)

The operator's later read-only Recorder audit observed that **hourly and five-minute statistics continued advancing** after the guarded two-sensor metadata repair. This closes the immediate "no hourly aggregate yet observed" limitation stated in the original 6 November troubleshooting window. It does **not** constitute a complete audit of historical dashboard continuity, rolling daily sensor semantics or all future aggregates.
