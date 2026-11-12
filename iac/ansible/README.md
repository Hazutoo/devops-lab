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


## Stage 2: DR-only reusable guardrail role

Stage 2 adds `roles/dr_guardrails/tasks/main.yml` and `playbooks/10-dr-guardrails.yml` while preserving the known-good Stage 1 preflight. The role evaluates guest identity, network routes, active SSH/Docker services and recovery directory existence. The entry playbook refuses unexpected inventory targets using locally delegated `pre_tasks`; host-specific transport/address values come from `hostvars[inventory_hostname]`.

Because the playbook is stored in `playbooks/` but the role lives in a sibling `roles/` folder, configure its search path when invoking Ansible:

```bash
export ANSIBLE_ROLES_PATH="$PWD/roles"
ansible-playbook -i inventories/local/hosts.yml \
  playbooks/10-dr-guardrails.yml -e @/path/to/private-dr-vars.yml \
  --limit dr --syntax-check
ansible-playbook -i inventories/local/hosts.yml \
  playbooks/10-dr-guardrails.yml -e @/path/to/private-dr-vars.yml \
  --limit dr --check
```

The external vars file must define `dr_expected_hostname`, `dr_expected_ipv4_cidr`, `dr_network_interface` and `dr_required_directories`. The example inventory is deliberately nonfunctional and no actual private values are stored here.

**Operator-verified original version:** syntax PASS, positive `--check` run `ok=13 changed=0 unreachable=0 failed=0`, wrong-hostname rejection before remote fact gathering, and forced-local-connection rejection before remote fact gathering. The first attempt failed due to the delegated connection context and was corrected by reading the original target via `hostvars[inventory_hostname]`.

**Not yet independently verified:** direct source parity of the newly committed Stage 2 role/playbook and execution from this branch; repeat normal-mode Stage 2 runs, automated fresh-VM configuration and future write-capable roles. This guardrail assumes Docker is already active and is therefore **not** a pre-Docker bootstrap check.

### Stage 2.1 validation update (12 November 2026)

The operator subsequently validated the **committed Stage 2 files on public `main`**, rather than the deleted feature branch. Git blob hashes matched expected values and the checked-out files matched existing WSL files byte-for-byte:

- `roles/dr_guardrails/tasks/main.yml`: `df73289441f9bd02598cfe37dcb80ae62b0047cd`
- `playbooks/10-dr-guardrails.yml`: `e690f9659c2649dd1dcfad28778a405ead565f40`

Syntax check passed. Two normal-mode DR-only runs each returned `ok=13 changed=0 unreachable=0 failed=0`. A separate negative test with an incorrect expected target address failed before remote fact gathering. The earlier wrong-hostname and forced-local-transport tests also passed. This establishes **read-only guardrail repeatability**, not idempotence of future mutation roles.

No real host details, secrets or operator logs were added to this repository. The role still assumes installed, active Docker and existing recovery directories; separate `pre_bootstrap` checks will be required for a fresh blank VM.


## Stage 3: pre-Docker recovery guest validation

The `dr_prebootstrap` role and `playbooks/20-dr-prebootstrap.yml` check guest identity, SSH inventory target/user, offline IPv4/IPv6 routing and running SSH **without requiring Docker or recovery directories**.

To invoke this read-only role, configure `ANSIBLE_ROLES_PATH` (roles are siblings of `playbooks/`) and supply a **private** DR-only inventory and an external vars file containing `dr_expected_hostname`, `dr_expected_ipv4_cidr`, `dr_expected_ssh_user` and `dr_network_interface`:

```bash
export ANSIBLE_ROLES_PATH="$PWD/roles"
ansible-playbook -i inventories/local/hosts.yml \
  playbooks/20-dr-prebootstrap.yml \
  -e @/path/to/private-prebootstrap-vars.yml \
  --limit dr --check
```

This example is intentionally nonfunctional without private DR parameters and a reachable, Python-enabled Ubuntu guest. **It does not perform initial operating-system, network or SSH bootstrap.**

**Operator's verified original local source (12 November 2026):** dependency scan, syntax PASS, two normal runs (`ok=11 changed=0 unreachable=0 failed=0` each), wrong-IP, wrong-SSH-user and forced-local-transport rejections before guest fact gathering. The newly committed public version still requires its own source-parity and Git-checkout verification. Blank-host compatibility has **not** yet been tested.
