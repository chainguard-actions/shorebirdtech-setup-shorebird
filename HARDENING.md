<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.3.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.3.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Install Shorebird (MacOS / Linux)' step pipes remote content directly to bash without first downloading to a file: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. If the remote URL is compromised or the connection is intercepted, arbitrary code executes immediately in the runner.

Locations:

- `action.yml:44`

### unsafe-shell (severity: high)

The 'Install Shorebird (Windows)' step pipes remote content directly to PowerShell's Invoke-Expression (iex) without first downloading to a file: `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex`. This is the PowerShell equivalent of `curl | bash` and carries the same supply-chain risk.

Locations:

- `action.yml:51`

### github-env-injection (severity: high)

The 'Add Shorebird to Path (MacOS / Linux)' step writes the env var INSTALL_PATH (sourced from `steps.install-path.outputs.path`, a `steps.*.outputs.*` value treated as untrusted) directly to $GITHUB_PATH without the required sanitization step (`printf '%s' "$INSTALL_PATH" | tr -d '\n\r'`). A newline embedded in the output value could inject arbitrary entries into PATH. The `# zizmor: ignore[github-env]` comment is a suppression hint for a different tool and does not sanitize the value.

Locations:

- `action.yml:60`

### github-env-injection (severity: high)

The 'Add Shorebird to Path (Windows)' step writes the env var INSTALL_PATH (sourced from `steps.install-path-windows.outputs.path`, a `steps.*.outputs.*` value treated as untrusted) directly to $env:GITHUB_PATH via Add-Content without sanitization. A newline embedded in the output value could inject arbitrary entries into PATH. The `# zizmor: ignore[github-env]` comment is a suppression hint for a different tool and does not sanitize the value.

Locations:

- `action.yml:68`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

Fixed all four findings in action.yml:
1. MacOS/Linux install: replaced `curl ... | bash -s -- --force` with download-then-execute pattern using mktemp; dropped the `--` (it was the shell's stdin option terminator, not a script argument).
2. Windows install: replaced `iwr ... | iex` with Invoke-WebRequest -OutFile to a temp .ps1 file, then `& $install_script`.
3. MacOS/Linux PATH injection: sanitized INSTALL_PATH with `printf '%s' "$INSTALL_PATH" | tr -d '\n\r'` before writing to $GITHUB_PATH.
4. Windows PATH injection: sanitized INSTALL_PATH with PowerShell's `-replace "`r|`n", ""` before writing to $env:GITHUB_PATH.

