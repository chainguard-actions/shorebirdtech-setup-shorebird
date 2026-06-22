<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v0.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **shorebirdtech--setup-shorebird/v0.1.2** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The macOS/Linux install step pipes a remote shell script directly to bash without first downloading and verifying it: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. This allows the remote server to execute arbitrary code on the runner without any integrity check.

Locations:

- `action.yml:14`

### unsafe-shell (severity: high)

The Windows install step uses PowerShell's `iwr` (Invoke-WebRequest) piped directly to `iex` (Invoke-Expression): `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex`. This is the PowerShell equivalent of `curl | bash` — it downloads and immediately executes a remote script without any integrity verification.

Locations:

- `action.yml:23`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

Fixed both unsafe-shell findings in action.yml:
1. macOS/Linux (line 14): Replaced `curl ... | bash` with a two-step approach: download the script to `/tmp/shorebird_install.sh` using curl's `-o` flag, execute it with `bash /tmp/shorebird_install.sh --force`, then remove the temp file.
2. Windows (line 23): Replaced `iwr ... | iex` with a two-step approach: download the script to `$env:TEMP\shorebird_install.ps1` using `Invoke-WebRequest -OutFile`, execute it with `& $installerPath`, then remove the temp file with `Remove-Item`.
Both fixes prevent arbitrary remote code execution by separating the download and execution steps.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed both the macOS/Linux and Windows steps in action.yml to sanitize values before writing to $GITHUB_PATH:
1. macOS/Linux: Added `safe_install_path=$(printf '%s' "$install_path/bin" | tr -d '\n\r')` and used the sanitized variable when appending to $GITHUB_PATH.
2. Windows (PowerShell): Added `$safeInstallBin = ("$installDirectory\bin" -replace "`r","" -replace "`n","")` to strip newlines/carriage returns before writing to $env:GITHUB_PATH via Add-Content.

