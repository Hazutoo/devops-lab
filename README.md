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
| Encrypted backup and recovery | [Restic over hardened Windows SFTPGo](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md) | Verified five-service archive restore and systemd timer dispatch; offline-aware scheduling installed, actual prune/recovery pending |
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

## Later November 2026 backup validation

After the original SFTP smoke test and failed automation preflight, a corrected setup successfully uploaded a five-service cold archive, restored and checksum-verified its payload, extracted mandatory sources and passed a full Restic data check. Backup/maintenance timers are enabled; actual timer-driven runs, pruning, interrupted-host recovery and application-level disaster recovery remain pending. Details are maintained in the existing [Restic/SFTPGo case study](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md).

## Follow-up: intermittent backup destination and observed timer execution

A later scheduling revision adapted the existing Restic backup to a Windows desktop powered on only occasionally. It preserves the encrypted repository and verified cold-backup workflow while checking backup eligibility every 5 minutes and maintenance eligibility every 10 minutes. **Actual systemd invocations and correct not-due skips were observed**, but a newly overdue upload after Windows returns online, real pruning and crash recovery have **not** been demonstrated. The [existing Restic/SFTPGo case study](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#h-operational-follow-up-offline-aware-scheduling-for-an-intermittent-destination) contains implementation rationale and verification boundaries.

## Capacity baseline and future platform plan

A read-only host audit on 5 November 2026 confirmed spare memory capacity, unallocated LVM extents and five running Docker services. The existing Wi-Fi and hardware remain in use by choice. A [sanitized baseline and phased roadmap](docs/2026-11-05-capacity-baseline-and-roadmap.md) separates verified observations from planned monitoring, documentation CI/CD, automation, NAS and workstation-based lab work.


## 6 November 2026: validated reliability fixes

- [Zigbee2MQTT Discovery mismatch and energy-statistics case study](docs/2026-11-06-home-assistant-mqtt-discovery-and-energy-statistics.md) — traced invalid group payload attributes, verified scoped MQTT filtering, preserved historical energy measurements during a guarded Recorder metadata repair, and confirmed resumed five-minute statistics. Hourly follow-up remains open.
- [Restic/SFTPGo follow-up](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md#j-follow-up-verified-off--on-encrypted-backup-and-isolated-retrieval-6-november-2026) — a real offline-to-online timer retry produced a new encrypted production snapshot and checksum-verified isolated restore. Actual prune, crash recovery, alerts and full clean-host application restoration are not claimed.
