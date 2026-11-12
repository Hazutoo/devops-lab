# Stage 4.2 — new isolated Hyper-V test guest (creation only)

**Date:** 12 November 2026  
**Environment:** Windows 11 Pro, Hyper-V Gen2, Ubuntu Server 24.04.5 ISO  
**Status:** VM created and verified **Off**; operating-system installation pending.

## Goal

Prepare a separate blank virtual machine for testing the Ansible `dr_prebootstrap` role on a genuinely fresh Linux guest. The existing restored DR machine and production services must remain untouched.

## Read-only capacity and topology audit

The Windows host had sufficient available CPU and RAM for a small additional VM. The limiting resource was storage, so a 32 GiB **dynamic** VHDX was chosen. The existing DR VHD parent/differencing chain was inspected read-only. An Internal Hyper-V switch, no detected WinNAT entries and disabled IPv4 forwarding were verified for the original DR network. This **does not** constitute a packet-level isolation test.

## Created resources

The operator ran a PowerShell script with explicit interactive confirmation. It refused an existing VM/switch name, conflicting addresses and low free disk space. Its final assertions returned:

```text
PREFLIGHT_PASS
SWITCH_CREATED
DR02_CREATION_PASS
VM state: Off
CPU: 2
RAM GB: 4
VHDX GB: 32
Forwarding: Disabled
Existing DR VM was not modified.
```

The new Gen2 VM has fixed 4 GiB RAM, 2 virtual processors and a 32 GiB dynamic VHDX. It is attached to a **separate Internal vSwitch** and its host-side IPv4 forwarding is disabled. Ubuntu Server install media is attached as the first boot device. Autostart and VM checkpoints are disabled.

All source IP addresses, VM names, internal infrastructure paths and image storage locations are omitted from this public case study.

## Evidence boundary

**Verified:** Hyper-V creation output, VM powered off, CPU/RAM/VHDX configuration, new Internal virtual switch attachment, host-side forwarding Disabled, and existing DR VM left unchanged.

**Not verified:** fresh Ubuntu OS installation, guest static address/SSH, absence of guest default routes, initial SSH host-key trust, Python installation, `dr_prebootstrap` validation on the fresh guest, end-to-end packet isolation, application restore or failover/RTO/RPO.

## Next phase

Power on **only** the new VM and install Ubuntu Server from ISO, assigning it a distinct static IP without gateway or DNS. Inspect guest routes and establish pinned SSH trust before running the Docker-independent Ansible role. Maintain a separate DR-02 inventory; never change an existing production or DR-01 inventory in-place.
