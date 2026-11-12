# DR Stage 5.0–5.2 — offline readiness and safe root-LVM expansion

**Date:** 12 November 2026  
**Target:** Separate newly installed Ubuntu Server 24.04 `amd64` recovery test guest.  
**Result:** Root LV + ext4 grew online by 10 GiB; prerequisites for Docker offline installation remain unfulfilled.

## Read-only audit

The second isolated recovery VM had a 32 GiB VHDX, but the installer allocated only about half of its 28.95 GiB LVM physical volume to root. Initial root was 14.47 GiB and the volume group had 14.47 GiB spare.

The guest was Docker-free, with no Docker Engine, Compose or containerd packages installed, no cached Debian packages and no default IPv4/IPv6 route. A controller-side search also found no previously downloaded Docker offline bundle. Read-only `vgs` required `sudo`; its initial failure as a nonroot user did not indicate a storage error.

## Controlled extension and evidence

The operator gave a dedicated interactive confirmation. The extension script verified hostname, expected nonproduction guest address, no default routes, ext4 and the expected root LV, plus minimum spare VG space.

```text
Before:
  root LV       14.47 GiB
  volume-group free 14.47 GiB
  root free     ~7.9 GB

Action:
  lvextend --resizefs --size +10G <expected-root-LV>

After:
  root LV       24.47 GiB
  volume-group free  4.47 GiB
  root free     ~18 GB
  filesystem    ext4 (online resize2fs)
  used          25%
  final SSH exit code: 0

ROOT_LVM_EXPANSION_PASS
STAGE 5.2 COMPLETE
```

The 32 GiB virtual disk was not enlarged, and no reboot was needed. Reserving 4.47 GiB unused space in the volume group was intentional. This operation is not meant to be rerun.

## Engineering boundaries

The verified change covers **guest root storage only**. It does not prove Docker installation, package dependency completeness, service recovery or measured disaster-recovery objectives.

The next step is a read-only availability/dependency audit, followed by a pinned Docker `noble/amd64` offline bundle with integrity checks and a gated Ansible installation plan. The isolated guest should not be given internet access for convenience.

No real inventory values, private hostnames, private addresses, usernames, backup names or private SSH keys are included here.


## Stage 5.3 — preflight for offline package preparation (verified)

Operator output confirmed that the WSL-based download controller runs Ubuntu 26.04 `amd64`, with `curl`, `gpg`, `gpgv`, `sha256sum`, `dpkg-deb` and Python available. HTTPS HEAD tests succeeded against Docker's official Ubuntu 24.04 `noble` Release endpoint and Ubuntu's official Noble InRelease endpoint.

Read-only checks against the separately isolated Ubuntu 24.04 `amd64` guest confirmed the expected hostname and architecture, locally installed base package dependencies (including `iptables`, `libseccomp2`, `libsystemd0`, `libdevmapper`, `libc6`, `apparmor` and CA certificates), approximately 18 GB available root space, and **no default IPv4/IPv6 route**.

```text
DOCKER_NOBLE_SOURCE_REACHABLE
UBUNTU_NOBLE_SOURCE_REACHABLE
GUEST_IDENTITY_PASS
NO_DEFAULT_ROUTES_PASS
GUEST PACKAGE AUDIT PASS
STAGE 5.3 COMPLETE
```

**Not yet proven:** package acquisition, GPG/repository signature verification, dependency closure, version pinning, archived bundle SHA-256 verification, offline Docker installation or recovery.

**Operator workflow change:** Future steps should use [a single, comprehensive script per bounded stage](OPERATOR_SCRIPT_STANDARD.md) rather than a succession of small snippets. Retain explicit confirmation for substantial write operations and fail-closed target checks.
