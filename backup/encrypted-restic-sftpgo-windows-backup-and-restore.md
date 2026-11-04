# Encrypted Cross-OS Backups with Restic, Windows SFTPGo and SSH Key Authentication

**Period:** November 2026  
**Environment:** Ubuntu Server 24.04.5 LTS, Restic 0.16.4, Docker Compose, Windows 11 Pro, SFTPGo Community 2.7.6, OpenSSH client, Windows Firewall, PowerShell  
**Evidence status:** **VERIFIED** encrypted repository initialization, key-only SFTP operations, 1 MiB smoke restore and subsequent five-service cold archive upload plus isolated archive restore and full-data check. **IMPLEMENTED** backup/maintenance timers. **NOT VERIFIED** real retention pruning, interruption recovery, complete application recovery or offsite resilience.

## Overview

A five-container homelab needed encrypted copies of persistent state stored on a second machine. The source ran Home Assistant, Zigbee2MQTT, Mosquitto, Pi-hole and Jellyfin on Ubuntu. The Windows target had spare storage but was not necessarily powered on continuously.

An earlier manual, filesystem-consistent archive had already been generated and copied to Windows with matching SHA-256 checksums. That was a valid **transfer-integrity test**, but not encryption at rest and not a recovery rehearsal. This case study covers a subsequent attempt to build an encrypted, eventually scheduled backup pipeline with Restic.

The implementation deliberately separates three questions:

1. Can the remote system accept SSH/SFTP connections without credentials or administrative privileges being exposed?
2. Can Restic write encrypted data and actually restore the same bytes?
3. Can an automated cold backup safely stop, restart and recover all applications even after interruptions?

The first two were demonstrated in the initial phase. A later corrected installer also completed a real five-service cold archive backup and isolated archive restore; crash recovery, application-level rebuild and unattended retention still require independent tests.

## Architecture and security boundaries

~~~mermaid
flowchart LR
  U[Ubuntu / five Docker services] --> C[Restic client]
  C -->|SSH public key / pinned host key| S[SFTPGo on Windows]
  S --> E[Encrypted Restic repository]
  W[Local Windows administrator] -->|loopback only / TOTP| A[SFTPGo WebAdmin]
  A -.manages.-> S
  F[Windows Firewall / trusted source + port] -.restricts.-> S
~~~

**No internal addresses, user names, hostnames, identifiers, private keys or server fingerprints are published here.** The virtual SFTP filesystem root maps to a dedicated storage directory; it is not the Windows disk root.

### Separation of privileges

- **Restic SSH client:** root-readable private client key and password file on Ubuntu; neither appears in scripts or repository documents.
- **Windows service identity:** a dedicated local non-admin account, replacing the installer default of `LocalSystem` after required NTFS ACLs were granted.
- **SFTPGo protocol identity:** a separate virtual account with an SSH public key and access only to the repository's virtual root.
- **Web administrator:** separate login with TOTP-based MFA and recovery codes; HTTP management bound to loopback, not the LAN.
- **Network:** a Windows Firewall inbound Allow scoped to the Ubuntu source address, the destination's LAN address, TCP SFTP port and private Ethernet interface.

These controls reduce scope but do not establish protection from every client or from compromise of the two hosts.

## A. What failed with the initial native Windows OpenSSH backend

The first target used native Windows OpenSSH Server, a dedicated local account, chroot and forced internal SFTP. Key-only login and ordinary file uploads were possible, but Restic repository initialization failed:

~~~text
OpenFile: sftp: "Bad message" (SSH_FX_BAD_MESSAGE)
~~~

A deliberate `sftp put -p` test also emitted:

~~~text
remote fsetstat: Bad message
~~~

This exposed a useful distinction: **a TCP handshake or even a successful upload does not prove that an SFTP backend supports the filesystem operations a backup application uses.** The exact OpenSSH/chroot root cause was not conclusively demonstrated. An incomplete repository from this attempt was preserved for inspection rather than silently reused.

The solution was to evaluate a separate SFTP implementation, not weaken the Restic repository or disable SSH host-key checks.

## B. Windows SFTPGo hardening

SFTPGo Community was installed and configured with distinct listeners: an SFTP endpoint on the target LAN interface and an admin UI on Windows loopback only. An initial inbound quarantine block remained active while configuration was audited.

A dedicated Windows service account was created, verified not to be an administrator, and given Modify permission to the application's config/SQLite/log directory and repository destination, with read/traverse permission on the parent. Existing SYSTEM and Administrators rights were preserved. The Windows service then successfully restarted and continued running under the less-privileged account. Its autostart configuration and local admin UI connectivity were also observed.

SFTPGo used an SQLite data provider. Administrative TOTP enrollment was completed using a standards-based authenticator and recovery codes were saved outside the repository.

A key implementation detail: the installer had created a broad inbound Windows Firewall Allow rule with `Any` address, protocol, port and profile. That rule was **disabled**, and a narrower private-interface TCP rule was added. The temporary quarantine was subsequently disabled after verification.

The SFTPGo virtual account was configured with its home mapped to the repository directory and an ED25519 client public key. Full read/write/delete rights **inside this virtual root** are needed for Restic's repository objects and pruning. Additional application-level allowed-IP, protocol and denied-login-method policies were recommended but were **not separately proven as saved**. The firewall's source/port configuration was verified; a negative scan from an untrusted source was not recorded.

### SFTP attribute compatibility

SFTPGo was configured to ignore unsupported SETSTAT operations. This was useful in the observed environment, but a succeeding `put -p` call with this setting should not be interpreted as remote preservation of all file attributes. Restic's encrypted content and its own metadata format, not SFTP timestamp preservation, are the relevant invariants for this use case.

## C. Independent host verification and end-to-end SFTP tests

The source first verified the destination SSH server host-key fingerprint on Windows. Ubuntu then retrieved the presented ED25519 host public key from the actual SFTP port, compared the fingerprints and only then pinned the matching key in the dedicated root-owned `known_hosts` file.

A new SSH client alias separated the new SFTP server/port from the older native Windows OpenSSH server. Relevant settings were:

~~~sshconfig
Host <BACKUP_SFTP_ALIAS>
    HostName <WINDOWS_LAN_ADDRESS>
    Port <SFTP_PORT>
    User <SFTP_VIRTUAL_ACCOUNT>
    IdentityFile /etc/restic/ssh/id_ed25519
    IdentitiesOnly yes
    UserKnownHostsFile /etc/restic/ssh/known_hosts
    StrictHostKeyChecking yes
    HostKeyAlgorithms ssh-ed25519
    PreferredAuthentications publickey
    PasswordAuthentication no
    BatchMode yes
~~~

The syntax above is intentionally illustrative: replace the bracketed placeholders before use. Do not accept an SSH host key solely because `ssh-keyscan` printed one; verify it independently.

Observed verification:

| Check | Result |
| --- | --- |
| TCP connection to SFTPGo | Successful |
| Windows host fingerprint vs Ubuntu scan | Exact match |
| Strict host-checking / key-only batch SFTP login | Succeeded; exit 0 |
| Remote virtual root listing | Succeeded |
| SFTP `put -p` / rename / download / delete | Succeeded |
| Downloaded random test file | 32 KiB; SHA-256 matched source |
| Leftover remote test file | Removed |

The initial file-test script accidentally generated an invalid download command due to a positional-formatting mistake; that attempt did not validate restore. After correction, all operations and SHA comparison succeeded. Reporting the corrected retest is materially more informative than treating the first exit status as a pass.

## D. Encrypted Restic repository and proven restore

A fresh encrypted Restic v2 repository was initialized through the SSH alias:

~~~bash
sudo restic \
  -r 'sftp:<BACKUP_SFTP_ALIAS>:/' \
  -o 'sftp.args=-F /etc/restic/ssh/config' \
  --password-file /etc/restic/password \
  init
~~~

The path `/` is the SFTPGo **virtual root** assigned to the protocol user. The repository password is a separate encryption credential from the SSH private key and must be preserved independently for recovery.

An initial `restic check --read-data` on the empty repository passed (zero snapshots/packs). A **1,048,576-byte random file** was then backed up as a tagged smoke test, restored to a different local directory and compared byte-for-byte using `cmp` and SHA-256.

Observed results:

~~~text
Files: 1 new
Added to the repository: 1.001 MiB (1.001 MiB stored)
PASS: Backup and restore SHA256 verified

1 / 1 index files loaded
1 / 1 snapshots
2 / 2 packs
no errors were found
PASS: Repository integrity verified
~~~

The smoke-test snapshot remained in the repository. **This is real evidence of encrypted remote storage and successful restore of the test file.** It does not validate restoring Docker service databases, Compose deployments or the entire host.

## E. Application backup scope and consistent snapshot design

The five Docker services use bind mounts for application state and three Compose project files. The original cold backup strategy stopped all five before archiving, avoiding concurrent writes to service databases, then restarted them. A media library mounted into Jellyfin was intentionally excluded; the application's configuration and cache were included.

An inventory before automating the new Restic path recorded all five containers running, all using `unless-stopped` restart policy, the required bind mounts present, and approximately 220 MiB across the main service directories. The Ubuntu root filesystem had about 69 GiB free for staging.

The proposed staged backup flow was:

~~~mermaid
flowchart TD
  P[Verify remote repository and local capacity] --> M[Persist restart/recovery marker]
  M --> S[Stop selected running containers]
  S --> A[Archive application and host configs locally]
  A --> R[Restart containers immediately]
  R --> U[Upload archive to encrypted Restic repo]
  U --> V[Restore test / integrity checks]
  V --> T[Retention and health monitoring]
~~~

The ordering matters. Network transfer should happen **after** restarting the services, and availability of the Windows destination should be checked **before** stopping them. The intended retention target was seven daily, four weekly and six monthly snapshots, scoped to a dedicated tag so unrelated test data would not be pruned.

### Why a recovery marker matters

Containers configured with `unless-stopped` are not guaranteed to restart automatically after a host reboot if they were manually stopped immediately before power loss. A recovery marker plus a boot-time systemd recovery service was therefore proposed and a recovery unit was created/enabled.

However, the recovery helper script was pasted through a terminal with garbled echo output. Although a `bash -n` check passed and the file existed with root-only permissions, the installed helper's real behavior was **not tested**. A syntactically correct shell script is not a verified disaster-recovery mechanism.

## F. Historical automation failure before successful v2 follow-up

A later all-in-one setup script was intended to validate preconditions, install service/timer units, perform a real Docker cold backup, restore its uploaded archive, check the repository, and only then enable scheduled jobs.

The installer was copied successfully using the operator's custom SSH client alias. A direct address-based SCP command had failed with `Permission denied (publickey)` because it bypassed the configured identity-selection alias — a client-side issue, not an SFTPGo failure.

The subsequent installer printed only its setup banner and the beginning of the `PRE-FLIGHT` stage, then exited. Inspection of the supplied Bash source found an optional-source discovery loop whose final absent-path test could return status 1 and terminate the script under `set -e` without a useful message. This was a plausible source-level defect, **not independently confirmed with a full execution trace**.

At the end of that **earlier failed attempt**, the following were **NOT VERIFIED as completed** (later progress is documented in section G):

- Automatic daily Restic backups and their systemd timer.
- Weekly maintenance timer, 7/4/6 retention execution or `forget --prune`.
- A production snapshot containing all five Docker applications.
- A complete archived restore and functional application restart from Restic.
- Recovery after forced interruption or reboot.
- Monitoring and alerts for a Windows target that is powered off.

The earlier 1 MiB synthetic-file restore remains the strongest verified Restic result.


## G. Follow-up: verified five-service archive backup and isolated restore

**Evidence:** Operator-provided installation transcript from 4 November 2026, after the earlier installer failure documented above. The corrected setup completed successfully. This is a later, distinct validation event; the earlier failure remains part of the engineering history.

### Installation and preflight

- Inspected existing backup scripts and unit files before replacing them; prior versions were preserved for rollback.
- Checked that all five Docker services were running and their persistent bind mounts matched the expected inventory.
- Discovered 20 available data/configuration paths; excluded the media library by design.
- Confirmed more than sufficient temporary disk space and noninteractive reachability of the **existing** encrypted Restic repository. Repository initialization was **not repeated**.
- Installed dedicated backup, recovery, maintenance, status and alert helper programs and systemd units. Recovery was enabled before the first service stop.
- The recovery helper passed a limited no-stop marker test; no power-loss/reboot or forced-kill recovery experiment was performed.

### Verified upload and restore evidence

| Test | Observed outcome |
| --- | --- |
| Cold archive of five services | Uploaded to encrypted Restic repository |
| Containers after backup | All five reported running |
| Repository snapshot | New production snapshot created, distinct from the earlier 1 MiB smoke test |
| Archive restore | 210.245 MiB restored to an isolated local directory |
| Integrity | Restored tar SHA-256 check: OK |
| Archive content | Eleven mandatory sources successfully extracted |
| Restic repository full-data check | 2/2 snapshots, 8/8 packs, no errors |

**Verified scope:** successful creation/upload of a five-service cold archive, return of the archive from encrypted storage, byte-integrity check and extraction of required content, and application containers running after the backup.

**Not demonstrated:** service-by-service startup using the restored files, recovery on a clean host, complete disaster recovery, an independently measured downtime/RPO/RTO or recovery under crash/power-loss conditions.

### Scheduling, retention and monitoring state

- Hourly backup eligibility/retry timer and weekly maintenance timer were **installed and enabled only after** upload, restore, checksum and repository checks passed.
- Tagged retention policy of **7 daily / 4 weekly / 6 monthly** passed a **dry-run** while the earlier independent smoke snapshot remained outside its scope.
- **Not yet verified:** an actual scheduled invocation, real `forget --prune`, unattended retry while the Windows destination is offline, or a failure notification reaching an external channel.
- Local diagnostics are available via the installed status helper, systemd and journald. External notification delivery was not configured.

The destination workstation must be powered on to receive the backups. Both computers remain at the same physical site, so this is off-host redundancy, **not offsite resilience**.

## Engineering lessons

1. **Test application-level SFTP operations.** A successful login does not mean repository initialization, rename, attributes or delete work.
2. **Use least privilege and explicit network boundaries.** Service identity, virtual user, admin UI and firewall are separate controls.
3. **Pin verified SSH host keys.** Keep a port-specific identity when replacing an SFTP backend.
4. **Prove restoration, not just upload.** The smoke-test restore and pack checks made the new repository demonstrably usable.
5. **Do not conflate test snapshots with production backups.** Restoring a 1 MiB file is not restoring a five-service homelab.
6. **Be precise about automation state.** The initial script failed at preflight; the later verified archive backup and enabled timers still do not prove crash recovery, real retention pruning or app-level disaster recovery.
7. **Design for offline targets and power loss.** A home Windows workstation may be unavailable at the scheduled time; failures need alerts and a second recovery path.
8. **Protect staged archives.** Raw tar files may expose service credentials even when the final Restic repository is encrypted.
9. **Keep a second off-host copy distinct from a true offsite copy.** Two computers in one building do not address theft, fire or other site-wide loss.

## Next steps

- Observe an actual timer-driven backup, including a supervised test with the Windows destination temporarily unavailable.
- Perform an isolated service-by-service application restore from the successfully retrieved cold archive.
- Exercise controlled interruption/reboot recovery, real retention pruning and alert delivery with the destination online/offline.
- Conduct a clean-host restore rehearsal and verify each application at the functional level.
- Independently verify all SFTP account restrictions and firewall denies from an untrusted origin.
- Add a third offsite destination if the recovery objectives require site-loss protection.

**Publication boundary:** this case study contains no private network addressing, SSH key material, secrets, account names, host fingerprints, device serials or privileged backup archives. It documents successful encrypted transport, an initial failed installer, and a later verified full cold archive backup with isolated restoration. Scheduled pruning, crash recovery, external alerting and application-level disaster recovery are not yet proven.
