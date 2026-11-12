# DevOps Lab

## Operator-controlled terminal workflow

The [PowerShell SSH clipboard workflow](docs/2026-11-06-powershell-ssh-clipboard-workflow.md) describes how a human operator can run remote Linux commands through a local PowerShell helper and paste the captured results into an ordinary chat. It includes a sanitized example, interactive sudo support, multiline here-string guidance, and privacy boundaries. It does **not** give an assistant autonomous terminal access.


A collection of evidence-based DevOps case studies and infrastructure experiments from a personal Ubuntu homelab.

The projects emphasize operational troubleshooting, deliberate security changes, measurable validation and reproducible engineering decisions. They document **what was actually implemented**, including failures and unresolved work.

## Focus areas

- Linux administration, boot analysis and systemd
- NetworkManager, Netplan and time synchronization
- Docker and persistent state
- SMART diagnostics and HDD power-management measurements
- SSH and smart-home service security
- Backups, integrity verification and recovery planning

## Case studies

| Area | Case study | Evidence and scope |
| --- | --- | --- |
| Linux and storage | [Ubuntu Server Health Audit: SMART, APM and System Services](linux/ubuntu-server-health-audit.md) | Observed hardware counters, self-tests, boot and services |
| Networking and Linux | [Recovering Network Readiness, NTP and Boot Time](linux/networkmanager-ntp-boot-recovery.md) | NetworkManager/Netplan remediation; synchronized time after reboot |
| Backup engineering | [Docker Cold Backup and Verified Off-Host Transfer](backup/docker-cold-backup-integrity-and-transfer.md) | Earlier five-service cold archive; SHA-256 matched on Ubuntu and Windows; original archive not restored |
| Encrypted backup and recovery | [Restic over hardened Windows SFTPGo](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md) | Verified encrypted multi-service backup, isolated clean-VM recovery and limited service protocol tests; full failover and actual retention prune pending |
| Security engineering | [Incremental SSH and Smart-Home Security Hardening](security/ssh-and-iot-service-hardening.md) | Key-only SSH and frontend authentication; documents the earlier firewall rollback |
| Network security | [Host and Docker Firewall Hardening with nftables and systemd](security/nftables-host-docker-firewall.md) | Persistent firewall verified after reboot; trusted-LAN TCP and DNS tested; untrusted-source blocking still pending |

## Technical approach

Case studies follow [the repository template](docs/CASE_STUDY_TEMPLATE.md) and distinguish:

- **VERIFIED:** evidenced by actual command output, logs, tests or explicit operator confirmation.
- **IMPLEMENTED:** a change was applied, but not all independent checks were captured.
- **PROPOSED / PENDING:** suggestions or next actions not demonstrated as deployed.
- **FAILED / ROLLED BACK:** unsuccessful work, including changes reverted before validation.

Numerical results are dated snapshots, not long-term promises. A matching backup checksum is not a recovery test; an open local port is not proof of Internet exposure.

## Security and privacy

The published material omits internal and external infrastructure addresses, specific hostnames and network identifiers, credentials, secrets, private keys, device serial numbers and employer information. Command examples may use explicit placeholders. Source archives and full private operational logs are not published.

## Status

Maintained as a technical portfolio. New case studies are added as separate, reviewable pull requests. Changes are not automatically merged.

## Early November 2026 backup validation (historical checkpoint)

At the **4 November** checkpoint, after the original SFTP smoke test and failed automation preflight, a corrected setup uploaded a five-service cold archive, checksum-verified its restored payload, extracted mandatory sources and passed a full Restic data check. **At that date**, actual timer-driven runs, pruning, interrupted-host recovery and application-level disaster recovery had not yet been verified. Subsequent dated sections below document the successful offline-to-online backup and **10 November limited clean-VM application recovery**. Actual pruning, interrupted-operation recovery and complete production failover remain unverified. Details are in the [Restic/SFTPGo case study](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md).

## Follow-up: intermittent backup destination and observed timer execution

A later scheduling revision adapted the existing Restic backup to a Windows desktop powered on only occasionally. It preserves the encrypted repository and verified cold-backup workflow while checking backup eligibility every 5 minutes and maintenance eligibility every 10 minutes. **Actual systemd invocations and correct not-due skips were observed**, but a newly overdue upload after Windows returns online, real pruning and crash recovery have **not** been demonstrated. The [existing Restic/SFTPGo case study](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#h-operational-follow-up-offline-aware-scheduling-for-an-intermittent-destination) contains implementation rationale and verification boundaries.

## Capacity baseline and future platform plan

A read-only host audit on 5 November 2026 confirmed spare memory capacity, unallocated LVM extents and five running Docker services. The existing Wi-Fi and hardware remain in use by choice. A [sanitized baseline and phased roadmap](docs/2026-11-05-capacity-baseline-and-roadmap.md) separates verified observations from planned monitoring, documentation CI/CD, automation, NAS and workstation-based lab work.


## 6 November 2026: validated reliability fixes

- [Zigbee2MQTT Discovery mismatch and energy-statistics case study](docs/2026-11-06-home-assistant-mqtt-discovery-and-energy-statistics.md) — traced invalid group payload attributes, verified scoped MQTT filtering, preserved historical energy measurements during a guarded Recorder metadata repair, and confirmed resumed five-minute statistics. Hourly follow-up remains open.
- [Restic/SFTPGo follow-up](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#j-follow-up-verified-off--on-encrypted-backup-and-isolated-retrieval-6-november-2026) — a real offline-to-online timer retry produced a new encrypted production snapshot and checksum-verified isolated restore. Actual prune, crash recovery, alerts and full clean-host application restoration are not claimed.

## 8 November 2026: MQTT action deduplication and bounded container logs

- [Case study: duplicate Zigbee2MQTT switch events and Docker JSON log rotation](docs/2026-11-08-zigbee2mqtt-duplicate-events-and-docker-log-rotation.md) — traced paired JSON and action-topic messages, removed redundant Home Assistant triggers, verified two switch actions, and scoped 10 MiB × 3 Docker log rotation to Zigbee2MQTT. Post-change warning recurrence and actual log rollover remain pending.
- [Restic/SFTPGo follow-up](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#k-8-november-follow-up-verified-offline-to-online-backup) — verified a new timer-driven encrypted snapshot after the intermittently powered Windows destination returned online. Actual prune and clean-host recovery remain pending.


## 9 November 2026: lightweight monitoring and Companion App alerts

- [Case study: five-minute systemd health monitoring and Home Assistant mobile notifications](docs/2026-11-09-systemd-health-monitor-and-mobile-alerting.md) — deployed 18 host, application, backup-freshness and SMART checks with incident deduplication, a persistent retry queue and a local-only webhook. Healthy production timer runs, a separate mobile push test and isolated incident/queue/recovery tests passed. Real-failure delivery, rollback execution and independent host-down detection remain unverified.

## 10 November 2026: SMART false-positive fix and external host heartbeat

- [Updated monitoring case study: SMART bitmask recovery and independent Healthchecks.io heartbeat](docs/2026-11-09-systemd-health-monitor-and-mobile-alerting.md#10-november-2026-follow-up-smart-false-positive-remediation-and-off-host-heartbeat) — 15/15 isolated SMART policy tests passed; the real incident automatically cleared after hourly cache refresh. A systemd HTTPS heartbeat now reaches a hosted checker every five minutes, with email notifications configured. Controlled total-host outage and DOWN-email delivery remain untested.


## 10 November 2026: isolated disaster-recovery rehearsal

- [Cross-platform Restic clean-VM recovery (case study, section L)](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#l-10-november-2026-isolated-clean-vm-disaster-recovery-rehearsal) — recovered the real encrypted five-service archive onto an independently installed, offline-isolated Ubuntu VM, checked archive integrity on both machines, imported exact production images and validated restored application startup. Local DNS resolution and authenticated MQTT publish/subscribe passed; HA-to-broker TCP connectivity and guest reboot persistence were confirmed. **Not demonstrated:** Zigbee radio, full HA MQTT/automation behavior, media-library restoration, production failover, measured RTO/RPO or actual prune. This public case study omits all private infrastructure identifiers.

## 12 November 2026: Ansible DR preflight (read-only)

[Read the sanitized Stage 1 case study](docs/2026-11-12-ansible-dr-read-only-preflight.md) and [generalized Ansible reference](iac/ansible/README.md). The operator's private original read-only playbook passed a 14-task check-mode run with zero changes; the public parameterized adaptation passed a 12-task `--check` run and two separate normal-mode runs (`ok=12 changed=0` each), using external private inventory and variables. Repeatability was shown for this read-only preflight, not configuration-changing provisioning roles. The placeholder example inventory remains untested as provided. An independent elevated Windows Hyper-V readback also confirmed an Internal-only VM switch attachment and Standard baseline checkpoint; no packet-level leak test was performed. No real inventory or private infrastructure identifiers are published.
