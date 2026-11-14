# Offline application recovery with Ansible: an isolated DR exercise

**Exercise:** November 2026  
**Technology:** PowerShell 7, Hyper-V, WSL, Ubuntu Server 24.04, Ansible, Docker Engine 29, Compose, Restic over SFTP  
**Result:** **Conditional acceptance** after staged offline recovery and functional tests.  
**Evidence:** Observations from the operator's completion-marker transcripts; underlying private reports and full Ansible logs were not independently inspected for this article.

## Overview

The goal was to demonstrate recovery of an encrypted five-service home-lab backup onto a **separate isolated VM** without connecting the guest to a production network or modifying the live host. The recovery process had to be checksum-verified, safely resumable and explicit about what had not been tested.

The original services were a home-automation server, Zigbee gateway, MQTT broker, DNS/filtering service and media server. The prior milestone [provisioned Docker offline](2026-11-13-offline-docker-provisioning-ansible.md); this case study covers the subsequent application image/data recovery and validation.

## Architecture and safety model

```text
Production host ---- encrypted backup / exported images
        |                         |
        +------> trusted Windows/WSL controller
                               |
                          pinned SSH
                               |
                    isolated recovery VM
                         (no default route)
                               |
                 local-only application tests
```

- The controller supplied signed, verified artifacts; the isolated guest did not need registry or package-index access.
- Guest identity, SSH host trust, isolated Hyper-V networking, storage and target paths were checked before writes.
- A dedicated Ansible inventory restricted execution to one recovery guest.
- Existing files with unexpected content were treated as conflicts, never overwritten silently.
- No backup pruning, production reconfiguration or secondary recovery-environment mutations were part of this exercise.
- Secrets were obtained locally for authenticated checks and were not committed or printed.

## Implementation

### 1. Audit and pin the backup

The source was inventoried before restoration. The operator reported **three Compose projects, five containers and eight required persistent bind-mount areas**. Two excluded paths were explicit: the media library and an optional broker log volume.

The Restic repository was identified through pinned SFTP settings. The selected snapshot carried a wrapped TAR plus companion checksum and manifest, rather than a directly mounted file tree.

Observed milestones:
- **10 snapshots** indexed;
- selected archive with **407 TAR entries** and companion SHA-256 verified;
- **8/8** required persistence locations and **3/3** Compose files present;
- a later independent `restic check --read-data` finished successfully against all **10** then-current snapshots.

A correct archive checksum and a full Restic read-data check are different evidence levels; both were recorded separately.

### 2. Export and import exact container images

Five existing production images were exported offline and transferred to the guest. The resulting image archives totaled approximately **1.45 GiB**. Image identity checks covered the imported image configuration, not simply a mutable tag.

**Docker 29 lesson:** containerd-backed image identity, an OCI manifest digest, and the configuration referenced in a Docker save TAR need not be represented by the same SHA-256 string. A first validator compared incompatible identifiers; a corrected validator recovered the already downloaded partial image archive without discarding it.

### 3. Restore data safely

The backup's application files were restored to controller staging, rehashed, moved to private guest staging and verified again before deployment.

Observed validation:
- **243/243** regular files independently verified in staging;
- **8/8** required persistent areas and **3/3** Compose definitions covered;
- **230** files verified after controlled deployment to the target application paths;
- second Ansible deployment run required and achieved **`changed=0`**.

Archive processing checked unsafe paths and links. Directory entries mattered: an empty required directory must not be mistaken for missing backup data merely because it contains no regular files.

### 4. Activate only safe services

The MQTT broker, home-automation service and media server were started in the isolated environment without pulling images.

The DNS filtering service was tested separately, using a **private copy of its data**, a Docker `internal` network and local container-IP probes. This avoided interfering with the guest's system DNS resolver. The production Compose configuration for this DNS service was intentionally **not** started unchanged.

The Zigbee component's image and configuration were recoverable, but its physical USB coordinator was unavailable; a radio test was not claimed.

**Docker networking lesson:** on an internal-only bridge, a requested published host port may not result in an active published port. A validator must inspect effective bindings, not assume that `--publish` succeeded. The functional DNS/HTTP test instead used the isolated container address, retaining the no-exposure policy.

## Functional validation

| Component | Observed test result | Limits |
| --- | --- | --- |
| Home automation | Authenticated API access, **283** restored entity states and history queries passed | No real Zigbee hardware or automation actuation test |
| Media server | Authenticated API and **2** registered libraries passed | Media files themselves were excluded from the backup |
| MQTT broker | Authenticated CONNECT and complete **SUBSCRIBE -> non-retained PUBLISH -> message receipt -> UNSUBSCRIBE** passed | No physical-device traffic validation |
| DNS filter | DNS resolution, Gravity database and an actual blocked-domain response passed | Isolated test container, not production-equivalent LAN DNS |
| Restic | Full repository read-data integrity check passed | Not a measured RPO or application failover |
| Network boundary | Isolated guest networking verified at checkpoints | No production cutover attempted |

A first full application-test run was **PARTIAL** because the manually entered MQTT credentials were rejected. A separate diagnostic identified MQTT CONNACK return code 5 (not authorized). A subsequent single-purpose test read existing credentials *on the isolated guest* and passed the complete publish/subscribe flow. The original partial record was preserved rather than rewritten, and the final acceptance audit separately rechecked MQTT.

The final audit reported **four of four** core application test families covered and **conditional acceptance**, not production readiness.

## Defects found and fixed during validation

1. Incorrect equality assumptions for Docker/containerd image identities.
2. Empty directory entries rejected by a file-only archive validator.
3. Ansible role search path not explicitly configured.
4. A report lookup path that did not match the previous stage's output directory.
5. Strict image-tag validation rejecting an implicit `:latest` name, even though image identity was verified separately.
6. Incorrect assumptions about published ports on Docker's internal-only network.
7. A generic MQTT authentication failure hiding the actual CONNACK return code.
8. Test gates that initially inferred too much from container `running` or unauthenticated HTTP responses.

These are reasons to separate **syntax/unit testing**, **integration tests on the real environment**, and **operator-observed acceptance**. A successful locally simulated test does not guarantee end-to-end execution.

## Acceptance boundaries

**Demonstrated:** isolated infrastructure provisioning, exact image transfer, encrypted-backup recovery, checksum verification, scoped Ansible restore with repeatability, application API/message/DNS tests, and full Restic data integrity verification.

**Not demonstrated:**
- Zigbee coordinator passthrough, radio communication or recovery of live smart devices;
- the DNS service's original host-network Compose stack acting as production LAN DNS;
- restoration of the deliberately excluded media-library files;
- full user workflows, all integrations and every automation;
- automatic recovery from a truly blank VM in one uninterrupted run;
- production outage and controlled failover;
- **measured RTO and RPO** against objectives.

RTO cannot be obtained reliably by adding times from separate interactive recovery stages. RPO cannot be claimed from one snapshot timestamp without a declared recovery-point objective and measurement method.

## Next exercise: measure RTO and RPO honestly

1. Define expected service set, allowable exclusions and measurable acceptance criteria.
2. Define the start and end of RTO (e.g. new guest provisioning start -> accepted business functions), and a target time objective.
3. Define RPO as the maximum acceptable data age; record the newest verified available snapshot at disaster simulation time and the timestamps of recoverable application data.
4. Preserve the existing working recovery VM; perform destructive reset/rebuild only with separate authorization and a verified rollback option.
5. Run a repeatable fresh-target recovery with automatic checkpoint timestamps, transfer metrics and stop-on-conflict safety checks.
6. Test dependent services under the permitted isolated network conditions and report untested hardware explicitly.
7. Archive evidence privately and publish only a sanitized performance/coverage summary.

## Reproducibility and security notes

- Keep the one-script-per-logical-stage operator pattern: full preflight, bounded action, independent validation, failure diagnostics and a clear PASS/PARTIAL/FAIL marker.
- Prefer pinned identities and immutable content verification to mutable image tags.
- Preserve private guest and controller staging only under access restrictions and a reviewed retention policy.
- Do not publish IPs, hostnames, local usernames, key locations, private Restic snapshot/repository identifiers, API tokens, password files or decrypted application data.
- This case study is **a sanitized narrative of operator-reported results**, not a substitute for the private, generated runbook and original evidence files.
