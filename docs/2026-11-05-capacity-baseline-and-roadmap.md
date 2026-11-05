# Homelab capacity baseline and development roadmap

**Evidence date:** 5 November 2026  
**Status:** VERIFIED for the read-only baseline below; all future services are PLANNED unless explicitly described as already deployed.

## Scope and operating philosophy

This is a personal, single-node Ubuntu homelab providing useful services around the clock, plus an optional on-demand development lab on a separate workstation. Priorities are measured reliability, maintainability, practical automation, and recoverability—not maximum service count or enterprise-style high availability.

**Current decision:** retain the existing hardware and working Wi-Fi connection. No immediate SSD, RAM, Ethernet or additional server purchase is planned. Reassess hardware only after observing a real capacity, reliability, performance or recovery constraint.

## Observed host baseline

The following values are one-time command outputs, **not** continuous measurements or performance benchmarks. Identifying network details are intentionally omitted.

| Area | Observed on 5 November 2026 |
| --- | --- |
| OS | Ubuntu Server 24.04.5 LTS; bare metal |
| CPU | Intel Core i7-6700T, 4 cores / 8 threads, VT-x |
| RAM | 7.6 GiB total, 1.2 GiB used, 6.5 GiB available; 4 GiB swap unused |
| Memory PSI | `some` and `full` 10/60/300-second averages all 0.00 during the check |
| Storage | One SATA HDD, approximately 465.8 GiB; Ubuntu LVM |
| Root filesystem | 98 GiB ext4, approximately 25 GiB used and 69 GiB available |
| LVM | `ubuntu-vg` ~462.71 GiB, `ubuntu-lv` 100 GiB, ~362.71 GiB unallocated in the VG |
| Connectivity | 5 GHz Wi-Fi connected; Ethernet interface without carrier |
| Virtualization | KVM device available; no hypervisor was running the host |
| Docker | Five running services: Pi-hole, Jellyfin, Home Assistant, Zigbee2MQTT, Mosquitto |

The free LVM extents are **not currently mounted free space**. Do not interpret this table as a request to extend the root volume; the existing root has ample free space for current needs. A one-time memory snapshot does not guarantee capacity under a future load.

## Already in use versus planned

**Already in use:** Ubuntu, Docker-based home services, network configuration and SSH hardening, a persistent nftables policy, and Restic-to-Windows/SFTPGo backup work documented in the existing case studies. The repository distinguishes tested archive restoration from *unverified* full application disaster recovery.

**Planned / not claimed as deployed:**
1. Test outstanding backup scenarios (offline-to-online destination, interrupted process, retention and clean-host service recovery).
2. Add low-overhead availability/host monitoring, Telegram alerts and remote access (for example Uptime Kuma, Beszel or New Relic, and Tailscale).
3. Create a separate modern Docusaurus documentation project with PR checks, static publishing, container image builds, a safe deploy path and tested rollback.
4. Add day-to-day administration and automation (Cockpit/Homepage, Ansible with a light web runner such as Semaphore UI).
5. Add local Git mirroring (Gitea), keeping independent backup rather than treating a mirror as immutable history.
6. Expand into files/photos only after retention, recovery and permissions are designed; Samba and Immich are candidates, not installed services.
7. Evaluate a second DNS resolver on **a different physical device** if real redundancy becomes necessary.
8. Extend observability as justified (Prometheus/Grafana, Loki/Alloy, alerting and bounded retention).
9. Use the separate, more powerful workstation for resource-intensive Windows/Linux VMs, K3s, databases, OpenTofu and short-lived development environments.

## Definition of done for new capabilities

A new service is complete only when its reason for existing, configuration/versions, data location, access boundaries, healthcheck, backup scope (if stateful), recovery or rollback method, and observed validation are documented. A successful container start alone is insufficient.

## Explicit limitations

- One physical host is a single point of failure; additional containers or VMs on it do not create physical redundancy.
- A second DNS container on the same host is not a true fallback.
- A copy or matching checksum does not establish full application recovery.
- Monitoring running only on this host cannot independently alert on total host loss.
- Wi-Fi is an accepted current operational choice, not a promise of enterprise network availability.
- Public documentation excludes credentials, internal IP addresses, MAC addresses, private storage paths and confidential data.

This page records an early project milestone and intentions as of 5 November 2026. Update statuses only after actual implementation and tests.
