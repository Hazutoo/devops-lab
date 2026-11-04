# Incremental SSH and Smart-Home Security Hardening

**Environment:** Ubuntu Server 24.04.5 LTS; OpenSSH; Windows PowerShell; Docker; Mosquitto; Zigbee2MQTT; Home Assistant  
**Period:** 4 November 2026  
**Evidence status:** **VERIFIED** key-only SSH and broker configuration; **IMPLEMENTED / operator-confirmed** Zigbee2MQTT frontend token; **PENDING** persistent firewall and external exposure checks.

## Overview

A personal Ubuntu server hosted network services and smart-home applications. A security inventory identified SSH password authentication, an unprotected Zigbee2MQTT web frontend, and services bound to all host interfaces. The challenge was improving authentication without accidentally disconnecting the administrator or breaking MQTT-dependent devices.

The scope was staged hardening, not a full security certification.

## Architecture and trust boundaries

~~~mermaid
flowchart TD
    A[Trusted local client] --> S[OpenSSH on host]
    A --> H[Home Assistant / host network]
    A --> Z[Zigbee2MQTT frontend / Docker bridge]
    H --> B[Mosquitto / Docker bridge]
    Z --> B
    U[Ubuntu and Docker] --> S
    U --> H
    U --> Z
    U --> B
    U --> P[Pi-hole / host network]
    U --> J[Jellyfin / Docker bridge]
~~~

The diagram is conceptual; exact addresses, device IDs, names and private network topology are intentionally withheld.

## Initial findings

| Component | Audit observation |
| --- | --- |
| SSH | Password authentication enabled; public keys supported; X11 forwarding enabled |
| User key access | No entries in authorized_keys at baseline |
| UFW | Inactive |
| Docker network filter | Host INPUT/FORWARD policy ACCEPT; DOCKER-USER available |
| MQTT | Mosquitto listener on TCP 1883; allow_anonymous=false; password_file configured |
| Zigbee2MQTT MQTT integration | Broker URL, username and password configured |
| Zigbee2MQTT web frontend | Enabled; auth_token missing |
| Published ports | Selected services on IPv4 and IPv6 wildcard interfaces |

A workstation on the same LAN successfully connected to eight tested TCP ports, including SSH, DNS/web management, MQTT, Zigbee2MQTT, Jellyfin and Home Assistant. **This was a LAN test, not an external security assessment.**

## SSH implementation

A new ED25519 key was generated on Windows. Only the public key was installed on the Ubuntu account, using appropriate .ssh and authorized_keys permissions. Crucially, key-only login was tested **before** disabling password authentication.

Representative, sanitized client test (same options were used in the actual session):

~~~powershell
ssh -i "$env:USERPROFILE\.ssh\homelab_ed25519" -o IdentitiesOnly=yes -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no <USER>@<LAN_HOST>
~~~

The effective server drop-in contained:

~~~text
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
~~~

The operator validated configuration syntax using sshd -t, checked effective values using sshd -T, and reloaded the SSH service. The post-change effective settings matched the intended values.

### Client troubleshooting

A subsequent bare SSH login returned 'Permission denied (publickey)' because the custom identity filename was not automatically selected. A short alias initially failed because Windows Notepad saved the SSH client config as config.txt rather than config. Renaming the file and assigning the correct IdentityFile resolved the issue. The user explicitly confirmed the shorthand login worked.

**VERIFIED:** key-based SSH functioned after the configuration change; password-based SSH and root login were disabled. Ordinary sudo password prompts were unaffected.

## MQTT and Zigbee2MQTT authentication

Mosquitto already had anonymous authentication disabled and referenced a password file, so the work did not unnecessarily replace client credentials or interrupt the message broker.

The Zigbee2MQTT frontend had no auth_token when first audited. A token was configured; the operator confirmed it worked. The token itself and any secret files are deliberately excluded.

**Limitations:** Password configuration presence does not prove ACL quality or TLS enforcement. Authentication on a plain HTTP page does not itself encrypt the connection.

## Firewall experiment and rollback

A broad IPv4 rule was briefly inserted in the DOCKER-USER chain and confirmed by an iptables listing. Its intended connectivity test **did not execute**, because a Windows PowerShell block had been pasted into the Linux Bash session. The original rule was deleted successfully.

This action is intentionally classified as **ROLLED BACK**, **not verified firewall hardening**. A narrower connection-tracked rule was discussed but not deployed in the visible record. No persistent policy protecting host-network and bridge-network services, IPv4 and IPv6, was proven.

The engineering issue is that Docker forwarding rules and host INPUT rules differ. Return packets, discovery traffic and remote SSH recovery must all be considered.

## Evidence and status

| Control | Status |
| --- | --- |
| SSH ED25519 access | VERIFIED |
| Password and root SSH login disabled | VERIFIED using effective sshd configuration |
| SSH client alias | VERIFIED by operator |
| Mosquitto anonymous clients disabled | VERIFIED in inspected configuration |
| Zigbee2MQTT frontend token | IMPLEMENTED / operator-confirmed |
| AppArmor | Enabled in host audit |
| External-facing service reachability | NOT TESTED |
| Router port forwarding / UPnP | NOT TESTED |
| Persistent Docker/host firewall | PENDING |

## Lessons learned and future work

1. Prove key-only SSH connectivity before removing password login, keeping a recovery path.
2. Verify effective daemon configuration rather than trusting one edited file.
3. Inspect existing MQTT controls before modifying credentials used by active clients.
4. Treat localhost, Docker bridge, host networking, IPv4, IPv6 and VPN as different trust boundaries.
5. Never infer Internet exposure from wildcard binding or successful local TCP tests.
6. Roll back unvalidated firewall experiments instead of declaring them successful.
7. Add HTTPS or equivalent trusted transport, MQTT ACL review and a carefully tested persistent firewall plan.

**Scope boundary:** This project documents authentication improvements and an honest failed firewall trial; it does not claim a fully hardened server.

## Subsequent follow-up: persistent firewall deployed

The firewall trial described above was **a historical intermediate step**, not the final status of this homelab. Later on 4 November 2026, a dedicated `nftables` `inet` table and systemd unit were deployed. After reboot, five Docker containers were running, eight selected TCP ports remained reachable from the trusted LAN, Pi-hole resolved DNS queries, and boot logs confirmed the firewall finished before Docker started.

This does **not** establish that traffic from untrusted subnets or external IPv6 sources is blocked. Those negative tests remain pending. See the separate case study: [Host and Docker Firewall Hardening with nftables and systemd](nftables-host-docker-firewall.md).

## Subsequent backup transport security work

Later the same day, the backup destination was moved from a failing native Windows OpenSSH/SFTP backend to SFTPGo. A dedicated non-admin Windows service account, loopback-only administrative UI with TOTP, host-key verification, key-only SFTP and a narrowly scoped inbound firewall rule were validated. A Restic 1 MiB backup/restore passed; scheduled production backups did not.

These measures protect a **backup transport and storage service**, not the existing Ubuntu SSH daemon or the smart-home protocols discussed above. See [the encrypted Restic/SFTPGo case study](../backup/encrypted-restic-sftpgo-windows-backup-and-restore.md).
