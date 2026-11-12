# Case Study: Fail-closed Ansible guardrails for an offline DR guest

## Objective

Following an isolated read-only Ansible recovery preflight, extract its safeguards into a reusable role. Before any future write-capable provisioning, the orchestration should prove it targets **one expected DR inventory host**, not the production server or an accidental local connection.

This is a **guardrails demonstration**, not automated fresh-machine installation or service restoration.

## Implementation

A new `dr_guardrails` role performs read-only checks of the Linux guest: Ubuntu release and hostname, expected interface/CIDR, absence of both IPv4 and IPv6 default routes, active SSH and Docker, and existence of designated recovery directories.

The entry playbook deliberately starts with `gather_facts: false` and `become: false`. Its locally delegated pre-tasks require explicit DR-only variables and assert the DR inventory group contains exactly one host, with expected alias, SSH address and transport. Only then does the role gather guest facts and run the remaining checks.

The initial playbook used `ansible_connection | default('ssh')` in a task delegated to localhost. This read the delegated connection context and erroneously rejected the intended DR host. The operator captured the failure, backed up the original playbook, and corrected host-specific lookups to `hostvars[inventory_hostname]`. The public role and corrected playbook committed here are transcribed from the operator-provided successful script; checksum comparison against local WSL copies is not yet independently recorded.

## Verification — 12 November 2026

```text
Syntax: PASS
Positive DR-only --check: ok=13 changed=0 unreachable=0 failed=0
Wrong expected host: rejected before guest fact gathering
Forced local transport: rejected before guest fact gathering
STAGE 2 FIX VALIDATED
```

**VERIFIED:** the corrected role and playbook passed the controlled positive check-mode run and two distinct negative fail-closed tests. This validates these cases, **not** arbitrary inventory corruption or provisioning safety.

**PENDING:** independent checkout and source-hash parity of these GitHub files, repeated Stage 2 normal-mode runs, wrong-IP negative test, change-applying role idempotency, unattended offline provisioning, packet-level leak tests, application failover, and RTO/RPO.

## Scope and security boundaries

- Runs only on the isolated DR guest; no production host was touched.
- Read-only `command` inspections use `changed_when: false`; the role includes no Docker lifecycle, network reconfiguration, package installation or copy tasks.
- Directory metadata checks disable log emission with `no_log`.
- Parameters such as expected host, address and recovery paths are provided outside the public repository.
- The current role requires Docker to be active. A new blank VM without Docker will **fail** this validation by design; a separate pre-Docker bootstrap role or separate guardrail stage is necessary.
- The Hyper-V Standard checkpoint provides convenience rollback, not independent backup. Windows-side switch configuration and DR route checks were previously verified, but this Stage 2 role does not exercise packet-level isolation.
