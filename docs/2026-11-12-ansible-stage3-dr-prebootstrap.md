# Case Study: Docker-independent DR prebootstrap assertions (Stage 3)

## Overview

Introduce a **read-only Ansible role** that can validate a Linux recovery guest **before** Docker and backup data are present. The previous post-recovery role checked active Docker and restored directories and therefore could not be used to gate an empty operating-system installation.

## Implementation

The `dr_prebootstrap` role and `20-dr-prebootstrap.yml` playbook use `become: false` and defer remote fact gathering until two locally delegated guardrail checks have passed:

- Four explicit parameters must be provided outside the public repository: expected hostname, CIDR, SSH user and interface.
- The `dr` inventory group must contain exactly one host. Original-host inventory attributes are read via `hostvars[inventory_hostname]` to avoid confusion with locally delegated connection context.
- Remote checks then confirm Ubuntu 24.04 and expected hostname, address on the intended interface, no IPv4 or IPv6 default route, and active SSH.

The role does **not** reference Docker, recovery directories or Restic, does not install software, and does not modify system configuration. This is a post-SSH **pre-Docker** check; a guest must already accept SSH and support Ansible's Python execution.

## Operator verification — 12 November 2026

The operator executed the local role and playbook against an existing isolated DR guest:

```text
PREBOOTSTRAP_INDEPENDENCE_PASS
Syntax: PASS
Normal run 1: ok=11 changed=0 unreachable=0 failed=0
RUN_1_PASS
Normal run 2: ok=11 changed=0 unreachable=0 failed=0
RUN_2_PASS
NEGATIVE_WRONG_IP_PASS
NEGATIVE_WRONG_USER_PASS
NEGATIVE_LOCAL_TRANSPORT_PASS
STAGE 3 VALIDATION COMPLETE
```

Each negative scenario rejected the manipulated expected target **before** the remote setup task. No guest or production change was reported in the two successful normal-mode runs.

**Validation boundary:** these results are for the original local WSL source. The role and playbook in this pull request are transcribed from the operator-provided source and need separate Git checkout/hash-parity validation. Tests against a completely fresh VM are **not** yet performed. A static dependency scan does not substitute for execution in that environment.

## Follow-up

Validate repository/source parity, then apply the role to a second newly installed VM attached only to a separately verified isolated Hyper-V switch. A separate guarded provisioning phase can later install offline packages and Docker. Do not repurpose the post-recovery role as an empty-host installer, and do not assume the Hyper-V checkpoint is an independent backup.

## Stage 3.1: independently verified public repository revision (12 November 2026)

The Stage 3 feature branch had already merged; the operator checked out the current public `main` branch and validated the **committed** files against the local WSL source. Both Git blob hashes and byte comparisons passed:

```text
ROLE_HASH_PASS
PLAYBOOK_HASH_PASS
Syntax: PASS
Normal run 1: ok=11 changed=0 unreachable=0 failed=0
RUN_1_PASS
Normal run 2: ok=11 changed=0 unreachable=0 failed=0
RUN_2_PASS
NEGATIVE_USER_TEST_PASS
STAGE 3.1 VALIDATION COMPLETE
```

**Verified:** the public role and playbook match the working WSL source and execute repeatably on the existing isolated DR guest, without reported changes. The wrong-SSH-user negative test again prevented remote fact gathering. This supersedes the earlier pending checkout/source-parity status.

**Not verified:** behavior on a fresh empty VM, first SSH/Python bootstrap, Docker installation, change-applying provisioning idempotency, application failover or measured RTO/RPO. Operator-specific addresses, names, credentials and unredacted logs remain outside the public repository.
