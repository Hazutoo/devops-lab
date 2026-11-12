# Stage 4.5 — Docker-free Ansible prebootstrap on a fresh isolated guest

**Date:** 12 November 2026  
**Status:** VERIFIED for a newly installed Ubuntu guest after manual OS, SSH and Python availability; full unattended provisioning is pending.

## Scope

The operator created a second, distinct Hyper-V guest with a new Ubuntu Server 24.04 installation on a separate Internal-only switch. This guest had OpenSSH and Python but **no Docker** and no restored application data. The existing recovered DR guest and production system were left untouched.

The primary goal was to show that the existing `dr_prebootstrap` role is independent of the post-recovery role and can run against a newly installed, Docker-free target.

## Guest and SSH preparation

- Ubuntu 24.04 and Python 3.12 were verified; OpenSSH was active.
- The guest had static IPv4 connectivity within its own isolated point-to-point test subnet; no default IPv4 or IPv6 route was present.
- Windows SSH first-contact trust was compared against the WSL-observed host key. A dedicated Ed25519 controller key and pinned host-key file were created for the new guest.
- A subsequent key-only SSH test succeeded (`BatchMode=yes`, strict host-key checking, exit code 0).

**Trust limitation:** cross-checking a prior Windows TOFU record against WSL is not independently authenticated out-of-band host-key verification.

## Execution and observed results

The initial Bash wrapper terminated after its SSH command because SSH consumed remaining stdin. This was corrected by using `ssh -n` before running Ansible. The incomplete first run is excluded from the verified playbook results.

The operator then created a **separate private** inventory for the new guest and checked the source Git blobs against the previously validated Ansible files.

```text
SOURCE_HASH_PASS
SSH_NO_STDIN_PASS
DR02_INVENTORY_CREATED
Syntax check: PASS
Check mode: ok=11 changed=0 unreachable=0 failed=0
CHECK_MODE_PASS
Normal run 1: ok=11 changed=0 unreachable=0 failed=0
RUN_1_PASS
Normal run 2: ok=11 changed=0 unreachable=0 failed=0
RUN_2_PASS
Negative wrong-host test: rejected before remote facts
NEGATIVE_HOST_PASS
Original DR inventory SHA unchanged
DR01_INVENTORY_UNCHANGED_PASS
STAGE 4.5 COMPLETE
```

The source blobs of the existing `dr_prebootstrap` role and `20-dr-prebootstrap.yml` playbook matched their known hashes. The role passed on a genuinely new guest without Docker. Neither the recovered DR guest nor production environment was changed.

## Interpretation and limits

**Demonstrated:** read-only Ansible prebootstrap guardrails and target checks on a newly installed Linux guest without Docker, repeatable normal runs with zero reported changes, independent inventory/SSH credentials and an early fail-closed negative target test.

**Not demonstrated:** unattended OS installation, initial account/SSH/Python installation, offline package acquisition or signature validation, Docker/Compose deployment, restored services, packet-level network isolation, production cutover or recovery time objectives.

This stage is a prerequisite for the next offline provisioning phase, not a finished disaster-recovery procedure.

## Next stage

Before any write-capable playbook, collect read-only guest storage/package facts and audit actual offline installation artifacts and trusted checksums. Keep the prebootstrap safety gate separate from the Docker-dependent post-recovery checks.
