# Offline Docker provisioning of an isolated recovery VM — Stages 5.4–5.5

**Date:** 13 November 2026  
**Platform:** Windows PowerShell 7 + Hyper-V, WSL controller, Ansible Core and Ubuntu Server 24.04 `noble/amd64` target  
**Status:** Operator reported `FINAL_STATUS=PASS` for both milestones. The detailed Ansible log files were not independently reviewed for this article.

## Goal and isolation model

Provision Docker Engine and Compose on a clean, **network-isolated** recovery VM without granting it internet access. A connected WSL controller acquires authenticated packages; the separate guest receives only the verified bundle through SSH.

The target was constrained through pinned SSH trust, hostname/OS/architecture checks, no-default-route checks and a dedicated **one-host Ansible inventory**. No production machine or previously restored recovery VM was an automation target.

## Stage 5.4 — signed package acquisition

A separate APT configuration targeted Ubuntu 24.04 `noble/amd64` even though WSL itself ran a newer Ubuntu release. Dependency resolution accounted for the target's captured dpkg status, rather than blindly installing controller-platform packages.

The requested Docker stack consisted of `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` and `docker-compose-plugin`. The bundle workflow verified signed repository indexes, downloaded packages, checked SHA-256 digests against authenticated package metadata, recorded a manifest and simulated an offline installation against local package sources.

The operator reported:

```text
GPGV_VERIFIED_RELEASES=4
DOWNLOADED_PACKAGES=5
FINAL_STATUS=PASS
STAGE_5.4_COMPLETE
```

Five downloads were consistent with the five requested components given the target's already-installed runtime dependencies. No guest installation occurred during this milestone.

## Stage 5.5 — apply through Ansible, without guest internet

After explicit operator approval, a single PowerShell script:

1. Revalidated the signed package evidence, manifest and pinned archive checksum.
2. Confirmed target identity, network isolation, package-manager health, sudo authentication and the expected dpkg baseline.
3. Copied the archived packages to a dedicated guest staging path and rechecked SHA-256.
4. Created a dedicated Ansible role, playbook and one-host inventory.
5. Performed the offline installation and verified the Docker runtime, Compose and Buildx.
6. Ran an **offline container smoke test** without network access.
7. Re-ran the playbook and required a successful `PLAY RECAP` with `changed=0`.
8. Rechecked guest isolation and runtime health.

The final operator report was:

```text
FINAL_STATUS=PASS
STAGE_5.5_COMPLETE
```

Inspection of the stage script confirmed that the completion marker is printed only after the first run, idempotence test and independent guest checks pass. The original complete play recap was not included in the operator's excerpt.

## Practical lessons from implementation

- **Locale matters:** recent Ansible versions require UTF-8. A controller running with `LC_ALL=C` failed before executing the playbook; using `C.UTF-8` resolved it.
- **Avoid permanent sudo exceptions:** the guest required a sudo password. Temporary password handling was used without changing sudoers to `NOPASSWD`.
- **Avoid fragile nested shell quoting:** a multiline WSL invocation accidentally expanded a temporary path to an invalid destination; the wrapper was changed to invoke the target executable directly.
- **Use a real Ansible configuration:** `ANSIBLE_CONFIG=/dev/null` was rejected because it had no supported configuration-file extension.
- **Keep every execution resumable:** artifact checksum checks and existing-file guards prevent silent overwrites and unexpected cross-target changes.

## What this does not prove

This result demonstrates **offline Docker infrastructure provisioning**, not complete disaster recovery. Application Compose stacks, persisted volumes, secrets, Zigbee USB/device passthrough, data restoration, end-to-end service functionality and measured RTO/RPO remain separate work.

The next planned task is a read-only application/recovery-input audit, followed by a separately authorized service-restoration stage. Actual infrastructure addresses, usernames, SSH paths, credentials, checkpoint identifiers and private backup details are intentionally omitted.
