# Recovering Network Readiness, NTP and Boot Time on Ubuntu Server

**Environment:** Ubuntu Server 24.04.5 LTS; NetworkManager; Netplan; systemd-networkd; systemd-timesyncd; Docker  
**Period:** 3–4 November 2026  
**Evidence level:** **VERIFIED** post-change system state. Startup-time improvement is observational, not a controlled benchmark.

## Overview

A small Ubuntu homelab remained reachable over Wi-Fi but experienced long boot delays and inconsistent NTP synchronization. The issue involved a disconnected wired interface, NetworkManager managing the live Wi-Fi connection and an unnecessary systemd-networkd wait-online dependency.

The goal was to remove the boot blockage while preserving the sole SSH access path and verify time synchronization after reboot.

## Environment and initial problem

The live Wi-Fi interface was managed by NetworkManager; Ethernet had no carrier. The wait-online service reached an approximately 120-second timeout even though IP connectivity was available. systemd-networkd's offline state also interfered with the intended time synchronization path.

The presence of an IP address did not guarantee that systemd considered the system online.

## Technical approach

1. Explicitly assign the unused Ethernet interface to NetworkManager and mark it optional in a Netplan overlay.
2. Validate changes with a timeout-based rollback window rather than applying network settings blindly over SSH.
3. Disable unused systemd-networkd and wait-online units while keeping NetworkManager active.
4. Restart time synchronization and validate with timedatectl.
5. Reboot during the subsequent kernel maintenance window and measure the resulting boot.

The applied Netplan overlay used this structure, with the interface identifier sanitized:

~~~yaml
network:
  version: 2
  ethernets:
    <ETHERNET_INTERFACE>:
      renderer: NetworkManager
      dhcp4: true
      optional: true
~~~

## Implementation

The following commands were executed during troubleshooting (their exact shell ordering is simplified to show the verified sequence):

~~~bash
sudo netplan generate
netplan get
sudo netplan try --timeout 120
# Accept only after verifying that SSH remains connected.
sudo systemctl disable --now systemd-networkd-wait-online.service
sudo systemctl disable --now systemd-networkd.service systemd-networkd.socket
sudo systemctl stop systemd-timesyncd
sudo rm -f /run/systemd/netif/state
sudo systemctl start systemd-timesyncd
systemctl --failed
timedatectl status
timedatectl timesync-status
~~~

**Host-specific decision:** Disabling networkd is appropriate here because NetworkManager owned the actual management connection. This is not general advice for hosts using systemd-networkd.

## Challenges and troubleshooting

The core challenge was diagnosing a *readiness* dependency rather than an actual connectivity outage. Another risk was performing Netplan changes remotely; using netplan try gave an opportunity to revert without physical access.

Removing stale runtime network state and restarting timesyncd was part of the actual recovery. Those files are ephemeral, so the permanent fix depended on consistent interface ownership and systemd units.

## Verification and observed results

| Check | Observed result |
| --- | --- |
| NetworkManager | Active and enabled |
| Unneeded networkd and wait-online units | Inactive and disabled |
| Time synchronization | Synchronized to an Ubuntu NTP server |
| Failed systemd units | 0 after cleanup |
| NTP after reboot | System clock synchronized: yes; NTP service active |
| Kernel after maintenance | 7.0.0-42-generic |
| Post-reboot startup time | **54.538 seconds** |

An earlier boot had been reported near 2 minutes 40 seconds due to the wait-online delay. The later 54.538-second result supports removal of that bottleneck, but the kernel and packages were also updated during maintenance. This is not a controlled A/B benchmark.

## Lessons learned

- Identify the network manager that actually owns each interface.
- An optional disconnected NIC should not prevent a host from completing boot.
- Use rollback-capable network configuration testing when connected only through SSH.
- Verify NTP and service readiness after reboot, not just after applying a unit change.
- Avoid universalizing a host-specific networkd disablement.

## Potential improvements

- Alert on clock synchronization failures and unexpected boot regressions.
- Revisit the Netplan assumptions if management moves from Wi-Fi to wired Ethernet.
- Establish a physical or out-of-band recovery path before future firewall changes.

**Scope:** No initial Ubuntu provisioning tutorial, ongoing NTP monitoring or production availability guarantee is claimed.
