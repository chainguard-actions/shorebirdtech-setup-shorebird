<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.4.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The '🐦 Install Shorebird (MacOS / Linux)' step pipes remote content directly to bash: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. The script is fetched from a mutable `main` branch URL and executed immediately without being saved to a file first. Similarly, the '🐦 Install Shorebird (Windows)' step uses the PowerShell equivalent: `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' | iex`, which is the same unsafe pattern. Both should download the script to a temporary file, verify its integrity (e.g. checksum), and then execute it.

Locations:

- `action.yml:54`
- `action.yml:61`

### github-env-injection (severity: high)

The '➕ Add Shorebird to Path (MacOS / Linux)' and '➕ Add Shorebird to Path (Windows)' steps write `$INSTALL_PATH` (sourced from `steps.install-path.outputs.path` / `steps.install-path-windows.outputs.path`, which are `steps.*` outputs and therefore workflow-controllable values) directly to `$GITHUB_PATH` without the required sanitization step (`printf '%s' "$INSTALL_PATH" | tr -d '\n\r'`). A malicious calling workflow could inject newlines into the step output to smuggle additional entries into `GITHUB_PATH`, enabling PATH-hijacking attacks. The fix is to sanitize the value before writing: `safe=$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r'); echo "$safe/bin" >> "$GITHUB_PATH"`.

Locations:

- `action.yml:88`
- `action.yml:96`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

Fixed four issues in action.yml:
1. MacOS/Linux install step: replaced `curl ... | bash -s -- --force` with download-then-execute pattern using mktemp. Dropped the `--` (it was the shell's option terminator, not the script's argument) so the script receives `--force` as $1.
2. Windows install step: replaced `iwr ... | iex` with `Invoke-WebRequest -OutFile` to a temp file, then `& $install_script`.
3. MacOS/Linux path step: sanitized INSTALL_PATH with `safe=$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')` before writing to GITHUB_PATH.
4. Windows path step: sanitized INSTALL_PATH with `$safe = $env:INSTALL_PATH -replace '[\r\n]', ''` before writing to GITHUB_PATH.

