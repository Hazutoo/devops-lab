# Operator-controlled PowerShell SSH capture for Linux troubleshooting

**Date:** 6 November 2026  
**Status:** Workflow verified with ordinary remote commands and interactive sudo.

## Purpose

For an operator working in an ordinary chat interface, an SSH/clipboard helper can eliminate repetitive manual terminal selection while preserving explicit approval for each command. The assistant does **not** gain access to the terminal: the human runs a command and pastes the resulting text into the conversation.

## Architecture

Windows Terminal + PowerShell 7 → OpenSSH client → Ubuntu server → remote Bash → capture output to private temporary file → retrieve via SSH → Windows clipboard → manual paste into chat.

The target name is a **local SSH config alias**, not a public address.

## Reference implementation (generic)

The following is an adapted, **sanitized illustration** of the operator's tested wrapper. Replace `YOUR_SSH_ALIAS` with the operator's existing SSH alias. This code is intended for **Windows PowerShell 7**, not Ubuntu Bash:

~~~powershell
function Invoke-LinuxClipboard {
    param(
        [Parameter(Mandatory, Position=0)]
        [string]$Command,
        [string]$Target = 'YOUR_SSH_ALIAS'
    )

    $id = [guid]::NewGuid().ToString('N')
    $remoteFile = "/tmp/ssh-output-$id.log"
    $encoded = [Convert]::ToBase64String(
        [Text.Encoding]::UTF8.GetBytes($Command)
    )

    $remoteCommand = "umask 077; set -o pipefail; printf '%s' '$encoded' | base64 -d | bash 2>&1 | tee '$remoteFile'"

    & ssh -tt $Target $remoteCommand
    $remoteExit = $LASTEXITCODE

    $lines = & ssh $Target "cat '$remoteFile' && rm -f '$remoteFile'" 2>&1
    $fetchExit = $LASTEXITCODE

    if ($fetchExit -eq 0) {
        Set-Clipboard -Value ($lines -join "`n")
        Write-Host 'Remote output copied to clipboard.'
    } else {
        Write-Warning 'Could not retrieve output. A remote temporary log may remain.'
    }
    Write-Host "Remote exit code: $remoteExit"
}
~~~

### Simple and multiline examples

~~~powershell
Invoke-LinuxClipboard 'uptime && free -h'
Invoke-LinuxClipboard 'sudo docker ps'

$script = @'
set -e
echo "=== Failed systemd services ==="
systemctl --failed
echo "=== Container state ==="
sudo docker ps
'@

Invoke-LinuxClipboard $script
~~~

A single-quoted PowerShell here-string is useful for nested Bash/Python quoting and multi-line scripts. The wrapper deliberately uses `ssh -tt` so `sudo` can request a password interactively, rather than placing credentials into script input.

## Verified behavior and limitations

During a real session, the original working operator-specific helper returned `exit code 0`, displayed remote results, and copied output to the clipboard for `uptime`, Docker commands using `sudo`, `systemctl`, and host memory/disk checks. A first non-TTY variant failed on `sudo` password prompting, motivating the `-tt` change.

This portable example is **adapted** and is not separately shown to have been executed. It should be tested before use. It is not autonomous agent access, not a guaranteed noninteractive automation framework, and not a reliable wrapper for full-screen interactive terminal applications.

**Security:** Remote captured output is persisted temporarily (permission mask `077`); interrupted sessions may leave files behind. Captured output reaches the Windows clipboard; never print secrets, private keys, passwords, tokens or sensitive full configuration files. Use explicit checks, backups and post-change validation for mutations; keep manual execution and review.

**Working convention for assistant guidance:** output **Windows PowerShell** blocks invoking the helper; embed Bash inside a here-string if multiline, ask for pasted evidence between steps, and do not imply direct SSH access from the chat assistant.
