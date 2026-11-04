# Ubuntu Server Health Audit: SMART, HDD Power Management, and Service Checks

**Environment:** Personal Linux homelab  
**Review period:** 3–4 November 2026  
**Status:** Historical point-in-time assessment; not a continuously monitored health claim  
**Platform:** Ubuntu Server 24.04.5 LTS, Linux kernel `7.0.0-42-generic`

## Overview

I audited a small Ubuntu homelab server to establish a reliable baseline for system health and to investigate an HDD power-management issue. The key finding was that the disk was parking its heads frequently with one Advanced Power Management (APM) setting. I tested a different setting, checked the load-cycle counter, and confirmed that the chosen setting was still active after a reboot.

The aim was to distinguish real observations from assumptions, not to produce a generic one-command "server is healthy" verdict.

## Scope

- OS, kernel, uptime and resource-usage snapshot
- Failed systemd units, boot duration and core services
- NetworkManager and system clock synchronization
- Docker service and running container count
- HDD SMART attributes and completed self-tests
- HDD APM behavior and load-cycle counter

**Privacy:** Hostnames, addresses, disk serial numbers, service endpoints, user details and internal infrastructure identifiers are intentionally omitted. The disk name below reflects the observed Linux device path and may differ on other hosts.

## Observed system snapshot

The following figures were captured during a login on **4 November 2026**. They are a snapshot, not long-term averages.

| Check | Observed result |
| --- | --- |
| OS | Ubuntu 24.04.5 LTS |
| Kernel | `7.0.0-42-generic` |
| System load | `0.04` |
| Root filesystem | `24.7%` used of `97.87 GB` |
| Memory utilization | `12%` |
| Swap utilization | `0%` |
| Reported temperature | `37.0 °C` |
| Processes | `204` |
| Boot duration | `54.538 s` |
| Failed systemd units | `0` |
| Clock synchronization | Synchronized |
| Network / remote access | NetworkManager and SSH active |
| Container runtime | Docker active; five containers running |

The temperature reported in the login summary is not a substitute for dedicated per-sensor thermal monitoring.

## Reproducible inspection commands

These are representative read-only commands for repeating the assessment. They are **not** presented as a verbatim transcript of every command entered during the original session.

```bash
# OS and system resources
cat /etc/os-release
uname -r
uptime
df -h /
free -h

# Boot and service state
systemd-analyze
systemctl --failed
systemctl is-active NetworkManager docker ssh

# Clock
timedatectl status

# Container state (inspect names privately before sharing output)
docker ps --format 'table {{.Status}}\t{{.Image}}'

# HDD health and power management (adjust device path as needed)
sudo smartctl -A /dev/sda
sudo smartctl -l selftest /dev/sda
sudo hdparm -B /dev/sda
```

Some utilities (`smartctl`, `hdparm`) require the appropriate packages and permissions. The commands above read state; they do not start a new SMART self-test or modify APM settings.

## Investigation: excessive HDD head parking

An HDD's `Load_Cycle_Count` tracks head load/unload events. Frequent changes can be a clue that power-management behavior is too aggressive for the workload. It is only one indicator; a high count on its own does not prove imminent drive failure.

### Evidence

| Item | Observed result |
| --- | --- |
| Device evaluated | `/dev/sda` |
| Previously observed load-cycle count | `520736` |
| APM level `128` | Counter increased by **6** over approximately **10 minutes** |
| APM level `254` | Counter increased by **0** over the observed comparison interval |
| After reboot | APM level `254` was confirmed active again |

A local startup/service configuration was used to retain the chosen setting across reboot. The exact unit file is not reproduced here because it was not independently reviewed for public reuse.

**Interpretation:** For this disk and short observation window, APM 254 substantially reduced head parking. This does not establish that 254 is the best setting for all drives. APM support and semantics vary by device and firmware.

### SMART validation

- `Reallocated_Sector_Ct = 0` in the inspected attributes.
- `Current_Pending_Sector = 0` in the inspected attributes.
- Prior short and extended self-tests reported completion without error.
- The latest reviewed extended self-test showed **Completed without error** at approximately **10,050 power-on hours**.
- No first failing LBA was reported for that test.

These results did **not** indicate an immediate SMART self-test failure at review time. They do not eliminate the possibility of future failure; regular backups and ongoing checks remain necessary.

## Validation summary

| Check | Outcome | Evidence level |
| --- | --- | --- |
| Systemd failed-unit check | No failed units | Observed |
| Clock synchronization | Synchronized | Observed |
| Core services | NetworkManager, Docker and SSH active | Observed |
| Docker workloads | Five running containers | Observed |
| SMART self-tests | Short and extended tests passed | Observed |
| APM persistence | Level 254 after reboot | Observed |
| Load-cycle comparison | +6 vs +0 during short intervals | Observed; limited sample |
| Long-term disk reliability | Not established | Requires longitudinal monitoring |
| Disaster recovery / backup restore | Not evaluated in this case study | Out of scope |

## Lessons learned

1. Check the actual hardware counters rather than inferring disk health from a single overall SMART status.
2. Validate a power-management change with **before/after measurements** and another check after reboot.
3. Record scope and observation windows: a ten-minute comparison is useful evidence but not a long-term reliability test.
4. Keep raw infrastructure data private; publish only sanitized observations and commands that can be independently understood.
5. A healthy service snapshot does not replace alerting, backup verification, or disaster-recovery tests.

## Follow-up work

- Track `Load_Cycle_Count` over a longer period to validate the APM behavior.
- Add periodic, non-invasive SMART and disk-capacity monitoring.
- Document backup restore testing separately after it has actually been performed.
- Recheck the service and disk baselines after major OS/kernel changes.

---

*This document describes an actual historical homelab troubleshooting session. The numerical observations are point-in-time results; the reusable diagnostic command list is illustrative. No employer systems or sensitive configuration details are included.*
