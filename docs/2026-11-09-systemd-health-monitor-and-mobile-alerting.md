# Lightweight Homelab Health Monitoring with systemd and Home Assistant Mobile Alerts

## Overview

A small Ubuntu homelab needed meaningful failure notifications without the operational cost of a full metrics stack. The solution uses an existing Home Assistant instance for mobile delivery and a host-side Python health checker scheduled with systemd.

**Status as of 9 November 2026:** the monitor was deployed, enabled, and observed performing healthy production checks; the local webhook push path reached the selected mobile app during a separate operator-confirmed test; deduplication, persistence, retry and recovery logic passed isolated synthetic tests. Full real-failure delivery, rollback execution and off-host alerting remain unverified.

## Environment

- Ubuntu Server, Docker / Compose and systemd.
- Five existing application containers, including Home Assistant and a message broker.
- SMART disk monitoring and encrypted Restic backups already maintained independently.
- Python 3, systemd oneshot/timer, a local-only Home Assistant webhook, and the Home Assistant Companion App.
- Operator-controlled PowerShell-to-SSH workflow; no autonomous terminal execution.

## Problem

A cron-style check that emits notifications on every failed poll creates alert fatigue. A simple push script also loses alerts when its Home Assistant dependency is temporarily unavailable. Installing a separate database, metrics scraper, dashboard and alerting container for a few workloads would add moving parts without solving the independent-host failure problem.

The design requirements were: no additional containers, no HA administrator token, no credentials in command output, two consecutive unhealthy checks before alerting, one recovery message, persistent retry for temporarily unavailable delivery, safe checks during short maintenance windows, and guarded rollback.

## Implementation

### 1. Local notification gateway

An existing Home Assistant configuration was extended with one dedicated automation: POST-only, `local_only: true` webhook with a randomly generated identifier held in a root-readable secret file. The automation calls the registered Companion App notification action.

A root-only host script sends a small JSON body with `title` and `message` to the loopback webhook endpoint. The destination was narrowed to **one mobile device** after an initial two-device configuration; other HA automations and notification preferences were not changed.

HTTP 200 only confirms webhook acceptance. A separate operator-observed push test confirmed delivery to the selected phone; it does not prove every future delivery.

### 2. Host checks

A Python checker with `dry-run`, `run` and `status` modes is executed by a systemd oneshot service on an approximately five-minute inactivity-based timer. The implementation performs 18 checks in the observed baseline:

| Area | Detection rule |
| --- | --- |
| Memory | Available memory below 500 MiB |
| Root filesystem | Usage at or above 85% |
| Monitoring dependencies | SMART daemon and two Restic timers active |
| Backup job state | Last systemd results not failed |
| Backup freshness | Locally recorded last successful backup older than 72 hours |
| Docker engine | Engine reachable |
| Containers | Five instances running, with health state considered when present |
| HTTP | Three application endpoints return 2xx or 3xx |
| Disk SMART | SMART health and critical sector attributes, sampled at most hourly |

The Docker and application checks are skipped while the cold-backup service is active, avoiding false alarms during its expected container stop/start window. A disabled or crashed monitor itself is not externally supervised by this implementation.

### 3. Incident state and retry

The checker maintains root-restricted, atomically replaced JSON state. It alerts after **two consecutive bad observations**, and sends one recovery notification after **two consecutive good observations**. This avoids sending repeat alerts for a persistent issue.

When the webhook sender fails, the event is retained in a persistent queue and retried on subsequent runs. The current implementation sends up to five queued messages per invocation and bounds the queue at 100 events. HTTP 200 marks the item as accepted; this is **not** an end-device delivery acknowledgement, and a bounded queue can discard the oldest events after prolonged outage.

### 4. Deployment and rollback safeguards

Installation used a preflight, a private backup directory, file-existence conflict checks, Python syntax validation, a health dry-run and `systemd-analyze verify` before enabling the timer. The webhook change separately used a Home Assistant configuration check and service HTTP verification. A guarded rollback script compares deployed file hashes to avoid silently overwriting later edits.

The host monitoring rollout did **not** replace existing SMARTD rules, Restic backup scheduling, Docker Compose or unrelated Home Assistant automations. Rollback helpers were installed but not exercised.

## Validation

### Observed production results

Two successful oneshot invocations on 9 November reported:

```text
Checks: 18, current issues: 0
Pending notifications: 0
```

The timer was `active` and `enabled`, and the status command showed no active incidents or pending notifications. The separate webhook test was confirmed received on the intended mobile device.

### Isolated incident tests

A first attempt to load the extensionless Python executable with `spec_from_file_location` failed in the **test harness** before any synthetic checks ran. The corrected harness used `SourceFileLoader`, a temporary isolated state directory and a mocked sender. Assertions passed for:

1. An alert only after two failures.
2. No duplicate incident alert.
3. Queue persistence.
4. Retaining an item after simulated gateway failure.
5. Clearing it after simulated HTTP acceptance.
6. A recovery message after two good checks.
7. Recovery deduplication.
8. No writes to production monitor state.

**Not verified:** a deliberately caused actual container failure, real queued notification replay after Home Assistant downtime, operating-system failure detection, actual mobile delivery during a failure, successful rollback execution, or independent notification if the host itself is offline. The existing backup maintenance/prune and disaster recovery work have separate verification requirements.

## Lessons Learned

- Reuse existing services where they meet the need; more observability components do not automatically mean better reliability.
- Separate detection, incident state, delivery and end-device acknowledgement.
- Two-check debouncing plus recovery notifications offers more useful signals than frequent repeated alerts.
- Design monitoring around expected maintenance, especially cold backups that intentionally stop containers.
- Keep notification secrets local, avoid administrator API tokens, validate configurations and retain hash-guarded rollbacks.
- A host-local monitor that relies on Home Assistant cannot independently report a total host or Home Assistant outage. Add an external watcher if this failure mode matters.
- Keep **IMPLEMENTED**, **VERIFIED** and **PENDING** evidence boundaries distinct; simulated success is not production disaster-recovery evidence.

Related work: [encrypted Restic backups](../backup/encrypted-restic-sftpgo-windows-backup-and-restore.md) and [home-automation MQTT incident analysis](2026-11-08-zigbee2mqtt-duplicate-events-and-docker-log-rotation.md).

## 9 November 2026 follow-up: guarded monitor hardening

A second, operator-controlled maintenance stage hardened the existing Python monitor without installing new services or changing application containers, the notification gateway, backup automation or systemd scheduling.

### Findings

A source-level preflight found three resilience gaps: direct concurrent monitor invocations lacked an explicit interprocess lock; malformed JSON caused the state loader to raise an unhandled exception; and the SMART overall-health check did not inspect its command exit code. These findings were reproduced or detected in read-only/isolated tests, not observed as production incidents.

### Implementation and test evidence

The revised monitor uses a nonblocking file lock for normal state-mutating runs, maintains a last-known-good JSON checkpoint and quarantines corrupt primary JSON before attempting recovery. If the checkpoint is also invalid, it fails closed instead of discarding stored incident and notification state. The SMART result now requires a successful command exit and an explicitly recognized healthy status. The original atomic state-write logic and existing alert deduplication/retry semantics were preserved.

The installer validated baseline state and expected source structure, backed up the executable and state, generated a separately guarded rollback, ran **12 isolated assertions**, and only then replaced the executable atomically. All 12 tests passed, covering checkpoint persistence, malformed-state recovery, pending-event preservation, fail-closed behavior, concurrency lock behavior and positive/negative SMART responses.

A subsequent **automatic** systemd timer invocation completed successfully: **18 checks, zero current issues and zero queued notifications**. Both live and checkpoint JSON states parsed, matched and remained restricted to the privileged service account. The timer was active/enabled; no service restart or additional mobile push was needed for this validation.

The operator also confirmed receiving both explicitly generated test notification types (alert and recovery) on the selected phone. This independently validates the intended mobile push path, but not automatic delivery during a real container outage.

**Evidence boundaries:** hardening tests, deployment and one post-update automatic run are VERIFIED. The new rollback was installed and syntax-checked but **not exercised**. Real workload failure/recovery, actual queue replay through Home Assistant downtime, total-host failure detection and clean-host disaster recovery remain PENDING. A last-good checkpoint can lag the latest state; it is not an exactly-once delivery guarantee.

No private backup locations, executable hashes, webhook identifiers, internal addresses, device identifiers or proprietary configuration are included in this public report.
