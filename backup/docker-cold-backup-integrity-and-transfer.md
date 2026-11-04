# Docker Cold Backup and Verified Transfer to a Second Computer

**Period:** 3–4 November 2026  
**Environment:** Ubuntu Server 24.04.5 LTS; Docker Compose; Bash; tar/gzip; SHA-256; SCP; Windows PowerShell  
**Outcome:** **VERIFIED** backup archive and transfer; **PENDING** restore testing and automation.

## Overview and objective

Five persistent Docker services shared a single local HDD: Home Assistant, Zigbee2MQTT, Mosquitto, Pi-hole and Jellyfin. The immediate operational objective was to make a consistent, independent copy of their state before additional system maintenance, without claiming disaster-recovery readiness prematurely.

## Architecture

~~~mermaid
flowchart LR
    D[Docker services and host configs] --> I[Mount and config inventory]
    I -->|Stop services| A[Cold tar.gz archive]
    A --> H[SHA-256 and archive read test]
    H -->|SCP| W[Windows computer]
    W --> V[Independent hash comparison]
~~~

The source script used Docker metadata to locate bind mounts and a managed volume. It included Compose files and selected Ubuntu configuration alongside the application data. The Jellyfin **media library was deliberately excluded**, while its configuration and cache were included. Container images, writable layers and a full OS image were outside scope.

Stopping the five containers before tar creation avoided copying databases while they were actively being modified. The script restarted the previously running containers using a shell EXIT trap.

## Implementation

The actual backup helper was a session-specific Bash script. The archive-creation and validation portion used:

~~~bash
tar --create --gzip --acls --xattrs --numeric-owner --directory=/ --file="$ARCHIVE.partial" --null --files-from="$OUT/$BASE.paths"
gzip -t "$ARCHIVE.partial"
tar -tzf "$ARCHIVE.partial" >/dev/null
mv "$ARCHIVE.partial" "$ARCHIVE"
~~~

The manifest, archive and checksum were stored together. After service restart, the operator checked container status and verified the SHA-256 file on Linux:

~~~bash
sha256sum -c <ARCHIVE_BASENAME>.sha256
docker ps -a
~~~

The same three files were transferred with OpenSSH SCP to Windows. Windows PowerShell recomputed the archive's SHA-256 and compared it to the recorded checksum:

~~~powershell
$archive = "<ARCHIVE_BASENAME>"
$dir = "$env:USERPROFILE\Documents\HomelabBackup"
$expected = (Get-Content "$dir\$archive.sha256" -Raw).Trim().Split(' ')[0]
$actual = (Get-FileHash "$dir\$archive.tar.gz" -Algorithm SHA256).Hash
$expected -ieq $actual
~~~

This PowerShell example is sanitized; it is not a replacement for the complete original backup script.

## Troubleshooting

**Backup directory permissions:** The original root-run script created the destination with mode 0700 owned by root, preventing a normal user from inspecting the files. Ownership was corrected without making the directory world-readable. The script's directory-creation line was then edited locally.

**Wrong shell context:** The initial Windows directory-creation command was mistakenly run within the Ubuntu SSH Bash session. Bash rejected PowerShell syntax. Running the transfer in Windows PowerShell succeeded.

Both errors exposed practical runbook concerns that a one-line 'backup completed' statement would hide.

## Verification and actual results

| Test | Observed result |
| --- | --- |
| Archive creation | Completed |
| Archive byte size | **81,559,172 bytes** |
| gzip / tar archive readability | Passed |
| SHA-256 verification on Ubuntu | OK |
| SCP transfer to Windows | Archive, manifest and checksum transferred |
| SHA-256 verification on Windows | Matching checksum: OK |
| Five Docker containers after backup | All running; Pi-hole and Jellyfin showed healthy |
| Restore to new host | **NOT TESTED** |
| Automated retention | **NOT IMPLEMENTED** |
| Encrypted backup repository | **NOT IMPLEMENTED** |

The source backup was created first on the same HDD. Protection from source-HDD failure only existed after transfer to another computer. Matching SHA-256 checksums verify byte integrity of the copied archive, **not** the ability to reconstruct all services from it.

## Lessons learned

- Inventory live mounts instead of assuming where Docker persists data.
- Copy application state consistently, accounting for active databases.
- Include Compose infrastructure definitions and required host settings.
- Preserve ownership and restrictive permissions while allowing verification.
- Test the backup in both locations; distinguish hash validation from recovery.
- Avoid exposing archives or manifests that may contain credentials and tokens.

## Potential improvements

- Replace the manual run with encrypted, deduplicated off-host backups and retention (for example, Restic).
- Use a non-interactive, application-aware consistent backup strategy with less downtime.
- Version and document all container images for reproducible restoration.
- Test restoration to an isolated machine and record actual recovery time.
- Monitor backup age and failed jobs.

**Scope boundary:** No disaster recovery exercise, restore benchmark, RTO/RPO claim or scheduled transfer was performed during the documented work.

## Subsequent follow-up: encrypted Restic transport and synthetic-file recovery

A later session on 4 November 2026 successfully deployed an encrypted Restic repository over Windows SFTPGo using a non-admin Windows service identity, key-only SSH and restricted firewall access. A **1 MiB synthetic test file was backed up, restored byte-for-byte and validated with a full-data repository check**. That successful test is **not** a restore of the earlier manual five-service archive, which remains untested.

The subsequent attempted production-backup installer failed during preflight, so **automated cold backups, scheduled retention and full application recovery remain pending**. The `NOT IMPLEMENTED` findings in the historical verification table above refer to the original manual archive at the time it was created, not to the later, separately tested Restic repository.

See [Encrypted Cross-OS Backups with Restic, Windows SFTPGo and SSH Key Authentication](encrypted-restic-sftpgo-windows-backup-and-restore.md).
