# Operator execution standard: one complete script per logical stage

**Adopted:** 2026-11-12, at the operator's explicit request.  
**Priority:** Use this workflow for future homelab, disaster-recovery, Ansible and infrastructure work.

## Primary operating rule

**Prefer ONE complete, robust, copy-pasteable PowerShell 7 script for an entire logical milestone**, even when it is long. The operator should not have to run a succession of small snippets that merely collect the next fact. Consolidate known checks and steps into the same execution. Avoid repeated back-and-forth for trivial preflights. Code quality, clarity, predictability and safety matter more than script brevity.

A stage script should:

1. **Be self-contained** and runnable from the Windows PowerShell 7 prompt. Use here-strings with Base64 for larger WSL/Bash payloads, so that Linux commands do not require manual console typing. Do not depend on previous shell variables or snippets.
2. **Verify target identity and environment first** (correct guest, architecture, OS, network isolation, expected mount/paths, pinned SSH host key, available credentials, expected baseline, available disk space, existing processes).
3. **Group all safe checks** needed for the stage, then perform the planned changes only if all preconditions hold. Include dependency detection, expected artifact checksums/signatures, and final verification when relevant.
4. **Use fail-fast behavior and clear output**, with named checkpoints, nonzero exit status on failure, concise diagnostic messages and a final PASS/FAIL summary. Report exactly what changed. Preserve logs privately and optionally copy a sanitized summary to the Windows clipboard.
5. **Be idempotent or safely resumable**. Inspect existing state before writing, skip already-completed work, and stop on partial changes that require human review rather than blindly retrying. The operator should not need to run the same action twice to discover that it already succeeded.
6. **Keep all infrastructure scopes explicit.** Separate test/recovery guest, existing recovery guest, and production. Never select a broader target silently. Do not modify an earlier DR inventory, checkpoints, production services or public credentials.
7. **Use one visible confirmation for material state changes** where appropriate. Destructive, irreversible, credential-sensitive or production-impacting work must be segregated behind an explicit approval, even if this requires a second script. Never use an all-in-one design to conceal risk.
8. **Never expose passwords, private keys or tokens in logs or Git.** Keep private and public documentation distinct. Public examples are sanitized.
9. **Document only observed results.** Mark each stage verified/partial/failed; never infer success from a script exit when its expected final validation markers are absent. Create/update documentation after substantive milestones, not after every tiny check.

## PowerShell / WSL / SSH lessons

- Entry point is PowerShell 7 on Windows. Prefer copy/paste over typing in the Hyper-V VMConnect console (which lacks convenient clipboard integration).
- Use WSL as Ansible controller; do not accidentally resolve/download Ubuntu 26.04 packages when preparing Ubuntu 24.04 `noble/amd64` target packages.
- Pass longer remote scripts with Base64 rather than fragile nested quoting; preserve exit codes through the PowerShell/WSL/SSH boundaries.
- **Use `ssh -n` when the caller Bash script is supplied via stdin**. Otherwise SSH may consume the remainder of the script; this actually caused a truncated Stage 4.5 test.
- Use separate pinned `known_hosts` and client SSH keys per disaster-recovery target.
- Show only actionable output, preferably with a copy-to-clipboard facility, but do not silently suppress failure details.
- Never automatically reissue an `lvextend` or another disk mutation after uncertain partial success.

## Standard full-script execution flow

```text
STEP 0  Target and explicit scope
STEP 1  Preflight: OS, tools, storage, network, SSH, expected state
STEP 2  Download/verify artifacts on trusted connected controller if required
STEP 3  Preparation: backups, inventory, temporary workspace, checksum manifest
STEP 4  Apply the stage change (only after scope/approval)
STEP 5  Verify runtime health and idempotency/repeatability
STEP 6  Negative/guardrail tests where safe
STEP 7  Final PASS/FAIL summary, paths to private logs, next dependency or blocker
```

For a read-only milestone, skip the mutation but still provide a comprehensive report. For risky milestones, separate the explicit approval gate from preparation, but keep the number of operator interactions minimal. Avoid five 20-line scripts when one tested 150-line script would achieve the same bounded stage more reliably.

## Documentation practice

After the operator supplies conclusive output, update both the private operations log and a sanitized public learning repository as appropriate, through draft PRs for review. Keep validation evidence and next-stage limits explicit. Do not merge PRs without operator instruction.

**Reference:** Stage 4.5 initial WSL script was prematurely consumed by SSH stdin; Stage 5.0–5.3 repeatedly used small scripts to gather predictable read-only facts. These are documented reasons for changing the operator workflow.
