# Read-only DR preflight with Ansible Core

This directory contains a **sanitized, parameterized reference** derived from a successful, independently operated, isolated disaster-recovery preflight on 12 November 2026. The public playbook was subsequently validated on the isolated guest with private runtime variables; the example inventory still contains placeholders and has not been executed as provided.

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

## Usage with a private inventory

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

A subsequent operator run **did validate this public parameterized playbook** (feature-branch commit `4aa6d8dff554b987d4319508c76872074502406a`). The checkout was verified by Git SHA; syntax checking passed and `--check` on the isolated guest with private external vars reported:

```text
PLAY RECAP: ok=12 changed=0 unreachable=0 failed=0
```

The example inventory itself is placeholder-only and **was not the inventory used in this run**. The verbose `stat` loop results exposed private target paths **in the operator's local log**, not in Git. Avoid publishing raw operator logs and improve loop output in a subsequent reviewed change. A **subsequent** Stage 1.1 exercise pinned the public playbook Git blob and ran it twice in ordinary mode (not check mode). Both runs passed: `ok=12 changed=0 unreachable=0 failed=0`, with `RUN_1_PASS`, `RUN_2_PASS` and `NORMAL-MODE REPEATABILITY PASS`. This verifies repeatability of the **read-only** preflight, not the idempotence of future configuration-changing roles. Fresh-host provisioning, elevated Windows Hyper-V switch readback has since **passed** (Internal switch, one DR NIC, Standard checkpoint). Full application failover, measured RTO/RPO and deliberate packet-level isolation testing remain outstanding.

Do not merge these status categories: verified **preflight** is not verified **provisioning** or **failover**.
