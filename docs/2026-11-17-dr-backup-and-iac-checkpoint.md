---
date: 2026-11-17
status: partial-current-restore-not-tested
topics: [disaster-recovery, restic, docker, hyper-v, ansible, offline-recovery, operations]
---

# Disaster-recovery checkpoint: protected artifacts, image portability and IaC reuse

## Scope and evidence

This 17 November 2026 checkpoint records **operator-observed terminal outputs** from a live homelab backup and an isolated Hyper-V rehearsal. It is not a new successful end-to-end recovery. The previously documented 16 November integrated offline DR exercise remains **conditional acceptance**, with the original end-of-run reporting failure retained.

The session preserved backup artifacts and catalogued existing automation before attempting a unified recovery entry point.

## Observed results

| Workstream | Status | Evidence boundary |
| --- | --- | --- |
| Encrypted system configuration bundle | **PASS** | Cross-host copy, checksum verification and decrypt/list round trip |
| Encrypted monitoring and backup-state supplement | **PASS** | Artifact transfer and verification; restored timers not activated |
| Offline export of five running Docker images | **PASS** | Gzip integrity, Docker save manifest count and checksum |
| Transfer to separate backup host | **PASS** | Destination checksum matched expected hash |
| Isolated VM copy and host network guardrails | **PASS** | A clone was created without modifying the original source disk |
| New archive import via Docker load | **NOT TESTED** | Guest lacked Docker and noninteractive privileged access |
| Test-VM graceful shutdown | **PASS** | Clone and copied disk preserved, no cleanup performed |
| Existing Ansible inventory | **PASS** | Seven playbooks and six roles inventoried read-only |
| Unified end-to-end DR orchestrator | **NOT IMPLEMENTED** | Dependency review remains outstanding |

### Backup design

The backup host now contains a dedicated unencrypted offline container-image archive (five services) and two separately encrypted system-state packages. Application data remains in an encrypted Restic repository. These are complementary recovery inputs, not a complete bootable system image.

Each encrypted configuration bundle was copied to the backup host and round-tripped for verification. The offline image archive was gzip-tested, checked for exactly five entries in the saved image manifest and verified after transfer. The test did not establish that mutable Docker tags still resolve to these images in public registries; retained offline image files are the recovery source of truth.

The archive is **not automatically safe to publish**: container layers can embed configuration or secrets, even when the ordinary runtime stores secrets externally.

### Incomplete import test

An isolated test VM was cloned from an existing recovery guest on a standalone disk. Its virtual switch was internal; host IPv4 forwarding, configured NAT and Internet Connection Sharing were absent in the observed checks. The guest had no default IPv4 route, and the separate original recovery VM remained powered off.

A planned all-in-one import halted at preflight because the selected guest lacked the Docker binary/service; passwordless sudo was also unavailable. **No Docker image import occurred.**

This failure exposed two operator-script defects: a false success marker printed after an earlier PowerShell exception, and corrupted Bash heredoc termination after cross-platform newline handling. A later attempt to gracefully stop the test VM also used an unsupported parameter and was corrected. The verified final state was **test VM off, image import untested**. A prior successful import in a different clean-VM rehearsal does not validate the current archive.

### Existing infrastructure-as-code

The controller already has six Ansible roles and seven playbooks covering prebootstrap checks, target guardrails, offline Docker provisioning, staged application-data restore and limited runtime activation. Several playbooks target *different stage-specific inventory groups* and invoke pinned, target-specific helper scripts. They cannot simply be concatenated into a generic workflow without validating source dependencies and privilege requirements.

The controller had Ansible available but lacked the Restic executable; separately retained, verified offline Docker package sets were found. The inventory commands made no repository, hypervisor or production changes.

## Engineering decisions for the next iteration

- Reuse existing Ansible and operator scripts, rather than building another disposable VM.
- Build one PowerShell 7 entry point coordinating WSL and Ansible, with **audit mode as the default** and a separate, explicitly approved restore mode.
- Pin the target and artifact identities. Check host/guest network isolation, SSH host trust, credentials availability and storage before making changes.
- Transport cross-shell payloads safely; preserve exit codes and never print success after an error. Protect SSH stdin if piping Bash scripts.
- Verify restored Docker image identities independently of mutable tag names and ambiguous registry/image digests.
- Keep application validation differentiated: configuration recovery is not the same as authenticated MQTT round trip, isolated DNS filtering or physical Zigbee actuation.
- Measure recovery timing with declared scope; snapshot age is not observed real-world data loss.
- Record concrete operator results at each substantive session and submit private and sanitized public documentation as **draft PRs**, without treating documentation commits as execution evidence.

## Acceptance

**Current checkpoint:** encrypted artifacts and image copies verified; current offline image import **not tested**; new unified orchestrator **not created**. The previous integrated DR acceptance remains **CONDITIONAL_ACCEPTANCE_POSTFLIGHT** for the independently verified earlier rehearsal only.

No private hostnames, IP addresses, SSH identities, repository paths, keys, secrets or recovery tokens are included here.
