# DevOps Lab

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
| Encrypted backup and recovery | [Restic over hardened Windows SFTPGo](backup/encrypted-restic-sftpgo-windows-backup-and-restore.md) | Key-only SFTP, least privilege, verified five-service encrypted archive upload/isolated restore; timers enabled, real pruning pending |
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
