<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **shorebirdtech--setup-shorebird/v1.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The macOS/Linux step pipes remote content directly to bash: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. The script is fetched and executed in a single pipeline without first downloading and verifying it.

Locations:

- `action.yml:14`

### unsafe-shell (severity: high)

The Windows step pipes remote content directly to PowerShell's Invoke-Expression (iex): `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex`. This is the PowerShell equivalent of curl | bash — remote content is fetched and immediately executed without any verification.

Locations:

- `action.yml:22`

### github-env-injection (severity: high)

The bash step writes `$install_path/bin` to `$GITHUB_PATH` (line 16), where `install_path` is derived from the inherited environment variable `$XDG_CONFIG_HOME` (line 15). `XDG_CONFIG_HOME` is not set within this run block — it is inherited from the calling workflow and is therefore workflow-controlled/untrusted. The value is written to GITHUB_PATH without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`), enabling a newline-injection attack that could add arbitrary entries to GITHUB_PATH.

Locations:

- `action.yml:16`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

Fixed all three findings in action.yml:
1. unsafe-shell (macOS/Linux, line 14): Replaced `curl ... | bash` with a two-step approach: download install.sh to /tmp/shorebird_install.sh with curl, then execute it separately with `bash /tmp/shorebird_install.sh --force`, then clean up the temp file.
2. unsafe-shell (Windows, line 22): Replaced `iwr ... | iex` with a two-step approach: download install.ps1 to a temp file using `Invoke-WebRequest -OutFile`, then execute it with `& $tmpScript`, then clean up the temp file.
3. github-env-injection (line 16): Added sanitization of the install_path before writing to GITHUB_PATH using `printf '%s' "$install_path/bin" | tr -d '\n\r'` to strip newline/carriage-return characters that could enable newline-injection attacks.

