---
date: 2026-11-17
status: design-complete-dependency-validation-pending
topics: [disaster-recovery, ansible, orchestrator, audit, automation]
---

# DR Stage 9.2 — reuse existing automation with an audit-first contract

The earlier 17 November checkpoint PRs were **merged**, not left Draft. This new documentation checkpoint records the next architecture stage; it does not claim code deployment, snapshot restore or new acceptance.

## Evidence and scope

Read-back of earlier repo-stored Ansible examples established that the prebootstrap role validates OS, guest and isolation without requiring Docker, while the existing guardrail role requires an active Docker service and recovery directories. This ordering matters: prebootstrap must precede offline Docker provisioning; post-runtime guardrails cannot be used as a fresh-host check.

The existing playbooks for offline Docker, staging, application restore and isolated runtime were observed in an earlier operator inventory. They use distinct, stage-specific Ansible inventory groups and helper scripts. **Their live source was not available through the GitHub repository** and remains to be audited before wiring any real target. The earlier integrated test runner and its corrected-but-not-executed reporting variant should be reused where possible rather than rebuilt without need.

## Design

Use a single PowerShell 7 entry point coordinating WSL/Ansible with UTF-8 Base64 script transport and preserved exit codes. Mode `Audit` is default. `Restore` requires a separate explicit approval and revalidation of the immutable target/artifact plan. Mode `Verify` performs independent service and integrity checks with per-checkpoint evidence. Store operational evidence privately; commit only sanitized summaries.

The next operator stage is **one comprehensive read-only static dependency graph** of the real local roles and helpers. It inspects playbooks, inventory groups, task modules, privilege requirements, included files, source hashes and syntax checks. Static passes are not functional restore proof, and ambiguous dynamic helper dependencies must block automatic execution.

The execution must not create a new VM, alter production, change protected DR guests, restore secrets, touch the backup repository or access a live Zigbee coordinator. Retain previous protected artifacts and logs.

## Acceptance semantics

- Previous controlled rehearsal: `CONDITIONAL_ACCEPTANCE_POSTFLIGHT`, with its original final-report FAIL retained.
- New combined offline image archive: checksum and transfer verified, **current import NOT TESTED**.
- Unified runner: design only; **NOT IMPLEMENTED**.
- RTO: measured from recovery invocation to a declared set of service checks. RPO: restored snapshot age at simulated incident, not observed data loss. Both remain **NOT MEASURED** for this stage.

## Session checkpoint

**VERIFIED:** state of previous PRs and earlier repo-backed prebootstrap/guardrail assumptions. **PENDING:** live WSL dependency graph and execution evidence. **UNTOUCHED:** all protected infrastructure and repository backup state. Documentation remains in Draft PR until separately reviewed; no merge authorized.

## Stage 9.2 execution evidence (operator-observed)

The operator ran the read-only PowerShell 7/WSL dependency scanner and supplied its console result. All seven Ansible playbooks passed syntax checking, and the six expected roles were found. The offline Docker playbook uses play-level privilege escalation; application and isolated-runtime roles contain additional task-level privilege escalation. The offline Docker role references a guest-side Python helper.

**Acceptance:** `PASS_STATIC_SOURCE_ONLY`; restore gate deliberately blocked; no VM changes, no Restic write, no RTO/RPO measurement. These static results do not establish operational compatibility or validate the latest combined Docker image import. A new bounded source-code interface extraction remains necessary before cross-target orchestration. The private evidence stays on the operator's backup host.

## Next implementation input — prepared, not executed

A single operator-side read-only source-interface capture script is prepared in the private Draft PR. It will collect the existing playbook/role implementations for source-level analysis, record file hashes, and generate a review-only package with heuristic redaction and an explicit operator review requirement. It does not connect to a guest, run Ansible or modify any backup. The source capsule is **not yet executed or assessed**; it is the evidence needed before implementing a cross-target adapter. No new functional DR acceptance is claimed.

## Source capsule execution evidence

The operator ran the standalone Stage 9.3 PowerShell 7/WSL source capture. It enumerated **seven playbooks, six roles and fourteen selected source files** and created a locally retained source-review archive and manifest. Redaction output reported 4 address substitutions and 30 long-token substitutions, with no sensitive-identifier lines removed; these metrics **do not establish that the archive is safe for disclosure**. The operator must review it before sharing.

**Verified from operator-supplied console output:** source capture completed, no guest/VM or backup writes, and restore gate remained blocked. **Not yet verified:** actual package contents, role/helper dependency closure, new Docker image import, integrated restore and fresh RTO/RPO. Previous recovery acceptance remains conditional and unchanged.

## Source review and implementation milestone

The operator-uploaded source capsule and manifest passed independent checksum and internal manifest comparison. Source analysis established that the offline Docker provisioning role is strictly bound to a specific DR guest and pre-install package database state, and that later stages use six compressed Python worker invocations inside Ansible tasks. Sanitization damaged those encoded payloads, so the capsule cannot support complete worker review. Simple host substitution is **not** a safe integration technique.

A first actual orchestrator entrypoint was added privately, with **Audit implemented as a source-pinned, offline metadata and phase-plan generator**. It decodes embedded Python workers in memory for AST/contract metadata, without executing application code or exporting unredacted sources. `Restore` and `Verify` are explicit fail-closed placeholders pending target-specific adapter design and operator authorization. The new script is **prepared, not executed**; it is not evidence of successful integrated restoration. All prior recovery acceptance boundaries remain unchanged.
