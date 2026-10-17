<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.4.1** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

Two steps pipe remote content directly to a shell interpreter without first saving to disk:
(1) `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force` — fetches and executes a remote shell script via `curl | bash`.
(2) `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex` — fetches and executes a remote PowerShell script via `iwr | iex` (Invoke-Expression). Both patterns allow a compromised or man-in-the-middle remote server to execute arbitrary code on the runner. The script should be downloaded to a temporary file, its integrity verified (e.g. checksum), and then executed separately.

Locations:

- `action.yml:68`
- `action.yml:75`

### github-env-injection (severity: high)

Two steps write `$INSTALL_PATH` — sourced from `steps.install-path.outputs.path` / `steps.install-path-windows.outputs.path` (a `steps.*.outputs.*` value, which is workflow-controlled and therefore untrusted) — directly to `$GITHUB_PATH` without the required sanitization step (`printf '%s' "$INSTALL_PATH" | tr -d '\n\r'`). A calling workflow could inject newlines into the output value to smuggle additional entries into `GITHUB_PATH`, enabling PATH-hijacking attacks. The fix is to sanitize the value before writing: `safe=$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r'); echo "$safe/bin" >> "$GITHUB_PATH"`.

Locations:

- `action.yml:107`
- `action.yml:115`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

Fixed four issues in action.yml: (1) unsafe-shell Linux/macOS: replaced curl|bash with download to mktemp then bash execution, dropping the shell's '--' separator per instructions; (2) unsafe-shell Windows: replaced iwr|iex with Invoke-WebRequest -OutFile to temp file then & $install_script; (3) github-env-injection Linux/macOS: added safe=$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r') before writing to GITHUB_PATH; (4) github-env-injection Windows: added $safe = $env:INSTALL_PATH -replace backtick-r|backtick-n, '' before writing to GITHUB_PATH.

