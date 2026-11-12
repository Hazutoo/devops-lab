# Read-only DR preflight with Ansible Core

This directory contains a **sanitized, parameterized reference** derived from a successful, independently operated, isolated disaster-recovery preflight on 12 November 2026. It is **not** a claim that these generalized public files were run exactly as committed.

## Goal

Validate a deliberately offline Ubuntu recovery guest before considering any write-capable provisioning tasks. The private environment was first recovered from an encrypted Restic archive and remained separate from the running production services. Ansible runs from a Linux controller under WSL2; VM creation and virtual switch management belong to Hyper-V PowerShell, not this playbook.

## Scope and guardrails

- The play targets the **`dr` group only** and has `become: false`; it never edits guest configuration.
- It verifies Ubuntu version/host identity, an expected NIC address, absence of IPv4/IPv6 default routes, active SSH/Docker services and expected recovery directory existence.
- `ansible.builtin.command` is explicitly marked as non-changing and allowed in check mode to preserve actual read-only inspection.
- It intentionally does **not** inspect Docker containers using elevated privileges or run any production Docker Compose file.
- The target must remain on a separately verified isolated virtual switch. Guest default-route absence alone is **not** proof of complete Hyper-V isolation.
- The real inventory, actual host address, SSH key, recovery paths and private artifacts must stay outside this public repo. The example is deliberately nonfunctional until parameters are replaced.

## Example layout

```text
iac/ansible/
  .gitignore
  README.md
  inventories/example/hosts.yml
  playbooks/00-preflight.yml
```

## Usage (not yet tested against the public example)

1. Install a supported Ansible Core version in a controller-only Python virtual environment.
2. Verify the SSH host key over an independently trusted channel, use an explicit dedicated SSH client key, and test `ansible.builtin.ping` first.
3. Copy the example inventory to a **private, ignored** path such as `inventories/local/hosts.yml`, substitute the real DR-only values, and provide SSH authentication using your local SSH configuration or command-line options.
4. From `iac/ansible` run:

```bash
ansible-playbook -i inventories/local/hosts.yml playbooks/00-preflight.yml --syntax-check
ansible-playbook -i inventories/local/hosts.yml playbooks/00-preflight.yml --limit dr --check
```

These commands are an example for the **generalized version**, not additional verification evidence. Do not run them against production.

## Evidence boundary

On 12 November 2026, the operator's **private, hardcoded original** read-only playbook and inventory passed syntax validation and executed on an isolated DR VM:

```text
ansible.builtin.ping: SUCCESS, changed=false
PLAY RECAP: ok=14 changed=0 unreachable=0 failed=0
```

The generalized public version was adapted afterward; it has **not** received a separately recorded syntax or end-to-end execution test. A second normal-mode execution/idempotence comparison, fully automated fresh-host provisioning, Hyper-V switch readback under adequate permissions, complete application failover and measured RTO/RPO remain outstanding.

Do not merge these status categories: verified **preflight** is not verified **provisioning** or **failover**.
