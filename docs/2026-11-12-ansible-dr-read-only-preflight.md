# Case Study: Establishing a Read-Only Ansible DR Preflight

## Overview

A Linux-based Ansible Core controller was introduced on a workstation using WSL2, leaving a continuously running production homelab unchanged. The first stage assessed a separate, offline recovery guest that had already passed a limited disaster-recovery rehearsal.

## Environment

- Workstation: PowerShell 7, Hyper-V and Ubuntu in WSL2
- Controller: isolated Python virtual environment with **Ansible Core 2.21.6**
- Managed guest: Ubuntu Server 24.04 with Docker Engine and SSH
- Communication: operator-controlled SSH with separately trusted host key and dedicated ED25519 client identity
- Safety boundary: an internal-only Hyper-V switch, offline guest and **no default route** in the guest

No real hostnames, internal IPs, recovery archive identifiers, private paths or credentials are included in this public record.

## Problem

The previous DR exercise proved limited recovery and service tests but did not prove reproducible preparation of a replacement machine. Configuration drift, manual offline installation steps and accidentally crossing the production/DR boundary needed to be addressed before attempting automated provisioning.

## Implementation

1. Confirmed that the WSL controller could reach the isolated guest's SSH port while the DR guest had no default route.
2. Repaired a missing Ubuntu Python virtual-environment dependency; the initial `venv` creation had failed because `ensurepip` was unavailable.
3. Installed Ansible Core 2.21.6 inside a dedicated virtual environment, not system-wide or on the production machine.
4. Validated the local Ansible controller with `ansible.builtin.ping`.
5. Established guest SSH authentication using a dedicated ED25519 key and host-key pinning; an existing guest SSH key authorization file was backed up before adding the new public key.
6. Executed a DR-only read-only Ansible playbook checking guest identity, expected address, absent default routes, SSH/Docker service activity and restored-data directory existence.
7. Published a **parameterized reference playbook** under [`iac/ansible`](../iac/ansible/README.md). The public adaptation was subsequently validated by the operator against a separate private inventory and temporary parameter file.

## Validation

**Verified operator results, 12 November 2026:**

```text
Ansible connection test: ping=pong, changed=false
Local playbook syntax check: passed
DR preflight: ok=14 changed=0 unreachable=0 failed=0
```

The private original playbook's 14 tasks were run with `--check`. There was no host provisioning, service restart or production configuration change during this Ansible preflight.

**Additional verified result, 12 November 2026:** the operator checked out the feature branch at commit `4aa6d8dff554b987d4319508c76872074502406a` (`COMMIT_VERIFIED`) and tested **the public parameterized playbook** with a private DR-only inventory and temporary non-committed variables. Syntax check passed, and an isolated guest `--check` run reported:

```text
PLAY RECAP
DR guest: ok=12 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

The public sample inventory contains placeholders and was not directly executed. The detailed Ansible output included verbose registered `stat` loop items; future changes should suppress full metadata in public-facing logs.

**Not demonstrated:** idempotency of any future change-applying provisioning roles, privileged guest container inventory, elevated Hyper-V switch verification, offline Docker installation automation, end-to-end service restoration, genuine failover, or RTO/RPO measurement.

## Follow-up: Stage 1.1 normal-mode repeatability (12 November 2026)

After validating the public parameterized playbook in check mode, the operator pinned its Git blob hash and ran that **unchanged read-only playbook twice in normal mode** with an external private inventory and temporary variables.

```text
PLAYBOOK_HASH_PASS
NORMAL RUN 1: ok=12 changed=0 unreachable=0 failed=0
RUN_1_PASS
NORMAL RUN 2: ok=12 changed=0 unreachable=0 failed=0
RUN_2_PASS
NORMAL-MODE REPEATABILITY PASS
```

**Verified:** both sequential ordinary runs succeeded without reported changes or unreachable hosts. This provides repeatability evidence for this preflight, not an idempotency demonstration of provisioning or mutation roles. No private host identifiers, network addresses, snapshot names, recovery paths or raw metadata logs are published.

## Lessons Learned

- WSL2 can be a suitable test controller while the Windows host retains responsibility for Hyper-V creation; a confirmed TCP socket does not prove SSH authentication.
- A separate virtual environment limits interference with the system Python installation.
- Offline DR assumptions should be encoded as **fail-closed assertions**, not treated as prose-only safeguards.
- `changed=0` in a check-mode run is evidence of a non-changing preflight, not a complete idempotence test. Two subsequent normal-mode runs confirmed repeatability **only of the read-only preflight**, not of future provisioning tasks.
- Public portfolio code can demonstrate the method without publishing identifiable private infrastructure details.

## Next Steps

Keep the validated public preflight as a known baseline, reduce verbose loop output, move network and identity checks into a dedicated DR guardrail role, define checksum-pinned offline package artifacts, and use PowerShell for guarded VM creation. Keep production changes and unmodified production Compose execution outside this scope.


## Follow-up: independent Hyper-V configuration audit (12 November 2026)

An **elevated, read-only** Windows PowerShell inspection succeeded after an earlier non-elevated/insufficiently checked query had not returned usable VM properties.

**Verified:** Hyper-V management service running; one matching Generation 2 DR VM running with four vCPUs; exactly one VM network adapter connected to the intended **Internal** virtual switch; one Standard-type baseline checkpoint present. The audit reported `HYPERV_DR_AUDIT: PASS`.

This is independent host-side configuration evidence, beyond the Linux guest's no-default-route assertions. It is **not** a packet-level isolation/leak test, independent VM backup or proof of full recovery. No VM, network, switch or checkpoint settings were changed.
