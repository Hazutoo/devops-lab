# Repository automation instructions

Read [the operator script standard](docs/OPERATOR_SCRIPT_STANDARD.md) before proposing or executing infrastructure, DR, homelab or Ansible tasks.

**Operator requirement:** For each logical stage, provide **one comprehensive, self-contained, copy-pasteable PowerShell 7 script** (with necessary WSL/SSH tasks embedded). Do not split predictable read-only checks into many small command blocks. Aim for one preflight, action (if authorized), all checks, and an explicit final report per stage. Prefer comprehensive/reliable scripts even if long.

Safety overrides batching: require explicit authorization for state-changing or potentially destructive operations; fail closed on target mismatch; keep DR, recovery and production isolated; preserve current configs, SSH trust, data and credentials. Never assume guest internet connectivity. Stop and report partial failures; do not blindly retry disk changes.

For Bash scripts piped on stdin, use `ssh -n` for nested SSH operations. This previously caused an incomplete run.

Only document observed validations. Keep private operations data out of public documentation. Submit GitHub documentation changes as draft PRs and do not merge without explicit instruction.
