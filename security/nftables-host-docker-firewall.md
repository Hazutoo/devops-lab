# Host and Docker Firewall Hardening with nftables and systemd

**Environment:** Ubuntu Server 24.04 LTS, Docker Engine, nftables (`iptables-nft` compatibility), systemd, Windows test client  
**Period:** November 2026  
**Evidence status:** **VERIFIED** persistence, boot ordering and trusted-LAN TCP/DNS access; **NOT TESTED** hostile-source blocking and external IPv6 reachability.

## Problem

A small Docker homelab hosted two different networking models: DNS and home automation in host networking, plus a message broker, device frontend and media server on user-defined Docker bridges. Several services listened on wildcard IPv4/IPv6 addresses. Disabling UFW or simply adding a Docker FORWARD rule does not provide a complete host ingress strategy, and altering Docker-owned chains carelessly can interrupt active services.

The goal was to introduce a reproducible, scoped firewall without losing SSH, service discovery or LAN functionality.

## Architecture

~~~mermaid
flowchart LR
    LAN[Trusted LAN] --> NIC[Ubuntu network interface]
    NIC --> HOST[Host INPUT services: SSH / DNS / Home Assistant]
    NIC --> BR[Docker bridge forwarding]
    BR --> APP[MQTT / device UI / media]
    FW[nftables inet table] -.filters INPUT and selected FORWARD paths.-> NIC
    SYS[Dedicated systemd unit] --> FW
    DOC[Docker service] --> BR
~~~

Topology is schematic; hostnames, addresses, hardware identifiers and private network configuration are omitted.

## Incremental implementation

1. Inventoried open TCP/UDP sockets, five running containers, host-vs-bridge networking and the Docker `DOCKER-USER` chain.
2. Checked the gateway UI: its firewall was enabled, port-forwarding list empty, UPnP disabled and DMZ disabled. This **did not prove** external inaccessibility or determine the carrier NAT state.
3. Applied temporary connection-tracked IPv4 INPUT and Docker bridge rules with rollback planning; trusted-LAN probes succeeded.
4. Tested IPv6 using a dedicated chain and a scheduled systemd rollback. Removed an orphaned chain from an early attempt; later tested explicit cleanup. An ordinary Windows client lacked a suitable local ULA IPv6 address, so IPv6 reachability remained **unverified**.
5. Tested a standalone nftables `inet` table with a 10-minute rollback timer. Confirmed all eight selected TCP ports remained reachable on the trusted LAN.
6. Saved the table into a dedicated configuration file and enabled a custom oneshot systemd service with `Before=docker.service`. Did **not** enable the default nftables service or flush the whole ruleset.
7. Removed the duplicate temporary IPv4 iptables rules, rebooted, and checked the resulting table, Docker containers, Windows TCP reachability and a real DNS query.

## Policy design

- Separate `inet` table owned by the homelab; `input` and `forward` base chains use priority `-10` and default `accept`.
- Restrict **new ingress from outside the trusted LAN CIDRs**, on a named Wi-Fi uplink.
- Apply the FORWARD restriction only toward designated Docker bridge interfaces. Docker NAT/forwarding chains remain Docker-managed.
- Permit ordinary IPv6 control traffic and trusted ULA/link-local sources; do not disable IPv6 or globally drop ICMPv6.
- Replacing the custom table on reload is narrower than flushing all nftables tables.

Illustrative, sanitized rules (not a full standalone copy/paste configuration):

~~~nft
table inet homelab {
    chain input {
        type filter hook input priority -10; policy accept;
        iifname "UPLINK_IFACE" ct state new ip saddr != 192.0.2.0/24 counter drop
        iifname "UPLINK_IFACE" meta nfproto ipv6 ct state new jump trusted_v6
    }

    chain trusted_v6 {
        ip6 saddr fd00::/8 return
        ip6 saddr fe80::/10 return
        meta l4proto ipv6-icmp return
        counter drop
    }
}
~~~

Here `192.0.2.0/24` is RFC 5737 documentation space, **not** the actual trusted network. The sample deliberately omits the bridge chain and is **not** the deployed policy. In production, use narrow, explicitly assigned trusted prefixes rather than the broad illustrative `fd00::/8`. The actual deployed rules were verified using `nft list table`.

## Operational proof

After a controlled reboot:

| Validation | Observed result |
| --- | --- |
| Custom firewall service | `active`, `enabled` |
| nftables table | Present after reboot |
| Original temporary iptables overrides | Removed; Docker's chain preserved |
| Docker containers | 5 of 5 running; two health-checked containers healthy |
| Windows TCP connectivity | 8/8 selected LAN ports successful |
| DNS | External hostname resolved through local Pi-hole |
| Startup sequence | Firewall finished at 16.212 seconds; Docker started at 31.031 seconds |

Thus the firewall finished **14.82 seconds before Docker began starting**, as demonstrated by the boot journal, rather than inferred from unit declarations.

## Rollback and operational guardrails

Temporary changes were scheduled for removal with `systemd-run`. Once the persistent service had been enabled, rollback was scoped to the custom nft table only:

~~~bash
sudo nft delete table inet homelab
~~~

This is not a permanent disablement: the enabled service will recreate the table at the next start/boot. A full rollback requires disabling the dedicated systemd unit as well. Avoid `nft flush ruleset` when Docker manages its own firewall rules.

## Limitations

- All successful port checks came **from the trusted LAN**; no negative test from an untrusted subnet was performed.
- No comprehensive IPv6 LAN/WAN probe was performed; ULA addresses are not proof of Internet exposure.
- A TCP handshake is not an end-to-end MQTT, Zigbee or media playback test.
- The policy trusts devices already inside the selected local subnet, allows ICMPv6, and only covers matching ingress/bridge interfaces. It is **not default-deny or IoT lateral-movement isolation**.
- The upstream router's firewall, empty NAT forwarding list and disabled UPnP/DMZ reduce obvious exposure paths, but WAN carrier NAT and external reachability were not independently assessed.
- Future work: controlled negative tests, app-level checks, service-by-service LAN access review, IPv6 route changes, and network segmentation.

## Lessons learned

Firewall engineering for Docker requires a distinction between host INPUT and container forwarding, independent rollback, operator-preserving LAN checks, explicit systemd boot-order evidence and honest verification boundaries. A persistent service plus eight successful LAN probes is not the same as a proven deny policy against untrusted origins.
