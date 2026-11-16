---
date: 2026-11-16
status: conditional-acceptance
topics: [disaster-recovery, hyper-v, restic, docker, ansible, infrastructure-as-code]
---

# Offline Docker Disaster Recovery: Evidence-Based Clean-VM Rehearsal

## Objective

Test whether a multi-service containerized homelab could be recovered to a *new*, isolated Ubuntu virtual machine from a verified encrypted Restic snapshot and pre-staged Docker artifacts, without container image registry access from the guest.

This was a controlled **disaster-recovery rehearsal**, **not** production failover or a real outage. Evidence came from operator-run PowerShell, Linux and application-protocol checks on 16 November 2026.

## Recovery design

- **Controller:** Windows PowerShell 7, Hyper-V and WSL Ubuntu.
- **Guest:** new Ubuntu 24.04 VM attached to an internal-only virtual network, without a default route.
- **Base:** prebuilt, checked Packer disk image, cloned byte-for-byte for the test.
- **Authentication:** pinned SSH host keys and a dedicated non-production guest account; no trust-on-first-use enrollment.
- **Restore inputs:** encrypted Restic snapshot, signed/offline Ubuntu dependency closure, five pinned Docker image archives and application configuration/data.
- **Application tests:** Home Assistant, Mosquitto MQTT, Zigbee2MQTT configuration, Jellyfin, and Pi-hole.

The recovery imported five exact image archives and restored an application-data archive containing 245 files over eight data mounts and three Compose definitions. The guest installed Docker 29.9.3 from pre-staged packages without internet connectivity.

## Observed outcomes

| Evidence | Result |
| --- | --- |
| New offline VM and pinned SSH | **PASS** |
| Docker installation, image loading and source-data restore | **PASS** |
| Home Assistant restored configuration + HTTP challenge | **PASS** |
| Zigbee2MQTT configuration | **PASS** — no radio hardware |
| Jellyfin restored database + public HTTP info | **PASS** — no media playback |
| Authenticated MQTT publish/subscribe | **PASS** |
| Pi-hole isolated DNS allow/deny + authenticated API | **PASS** |
| Scoped Ansible verification | **PASS** — two runs, `ok=4 changed=0` each |
| Guest isolation and protected-VM readback | **PASS** |
| Original final orchestrator exit | **FAIL** — reporting bug after all functional checks |
| Independent evidence postflight | **CONDITIONAL_ACCEPTANCE_POSTFLIGHT** |

### Recovery objectives

| Objective | Measured | Target |
| --- | ---: | ---: |
| RTO | **2.978 minutes** | ≤120 minutes |
| RPO | **0.771 hours** | ≤24 hours |

**Definitions matter:** RTO was measured from starting the recovery command until core application, MQTT and isolated Pi-hole functionality had been verified. It excludes incident detection and subsequent Ansible reporting. RPO is the *age of the restored snapshot at the simulated incident time*, not independently observed real-world data loss. These values are observations from this run, not an SLA.

## Engineering lessons

**Fail-closed checks must validate types, not only values.** The initial procedure successfully restored services but terminated while passing a timestamp string to a PowerShell Boolean validation parameter. An independent postflight recomputed RTO/RPO from timestamped evidence, verified the actual VM and isolated network, and created separate supplementary reports. The original orchestrator failure was deliberately preserved instead of relabeled as PASS.

**Image integrity is not application readiness.** Tests included authenticated MQTT payload round-trips and real DNS answers inside an isolated container namespace. Merely seeing an open HTTP port or a container image hash would not prove these behaviors.

**Container volume ownership is part of recovery fidelity.** A DNS test initially failed because restored file ownership was not preserved. Keeping numeric UID/GID and modes enabled service startup, while the final synthetic denylist test used the authenticated Pi-hole v6 API and verified DNS allow/deny responses.

**Defensive automation needs realistic failure paths.** The process encountered stale backup pins, an overly rigid Netplan layout expectation, first-boot SSH-key provisioning, argument-list quoting, and an API readiness race. Each fix was tested in a bounded stage before the complete integrated rehearsal.

## Explicitly excluded or not independently verified

- Physical Zigbee coordinator, device radio traffic and automation execution
- Jellyfin media library and playback
- Authenticated Home Assistant/Jellyfin API flows
- Full convergence of all external infrastructure-as-code roles
- Controlled production outage, real failover, and measured real data loss
- A later full recovery run using the prepared, statically verified report-validation source fix

**Status:** The application-level recovery objectives were conditionally accepted based on a verified independent postflight. The original execution result remained FAIL due solely to the final reporting type-conversion error. This distinction is essential for accurate operational evidence.

No infrastructure addresses, identifiers, hostnames, credentials, keys or private evidence paths are included in this public case study.


## Follow-up: bounded source patch and safe lab cleanup

The reporting-only failure was addressed in a separate **source patch**. Operator output confirmed its parsing and source validation passed, with `PATCHED_NOT_EXECUTED` and no new VM. This should not be confused with an end-to-end run of the patched version.

A later clean-up used an explicit two-step approval process: read-only discovery and disk dependency checks, followed by a guarded audit and a separately confirmed destructive action. It verified identifiers, checkpoint absence, disk parents/children, exclusive virtual-switch use, and preserved recovery evidence before deletion. The operator-observed final result was **PASS**:

| Cleanup scope | Confirmed removed |
| --- | ---: |
| Superseded test virtual machines (gracefully shut down first) | 2 |
| Explicitly targeted virtual disk files | 4 |
| Exclusive unused internal virtual switches | 2 |

The removed files totaled **23.766 GiB nominally**; actual net recovered free space was not separately measured. Protected recovery VMs, the Packer base image, the accepted recovery instance and the evidence logs passed post-deletion preservation checks. No production services, Restic repository data or complete application backup were deleted.

**Evidence boundary:** All results describe the observed, operator-run lab session, not unattended routine cleanup or guaranteed future recovery behavior.
