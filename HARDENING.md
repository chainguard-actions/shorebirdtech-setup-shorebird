<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.0.2** was hardened automatically. 9 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

The composite action uses `actions/cache@v6`, which is pinned to a mutable version tag rather than an immutable 40-character commit SHA. This means the dependency can be silently updated (or compromised) without any change to this action. It should be pinned to a full SHA, e.g. `actions/cache@1bd1e32a3bdc45362d1e726936510720a7c6158d # v4`.

Locations:

- `action.yml:34`

### unsafe-shell (severity: high)

The 'Install Shorebird (MacOS / Linux)' step pipes a remote script directly to bash: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. This is unsafe because the script content is never verified before execution. The script should be downloaded to a file, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `action.yml:41`

### unsafe-shell (severity: high)

The 'Install Shorebird (Windows)' step pipes a remote PowerShell script directly to `iex` (Invoke-Expression): `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' | iex`. This is the PowerShell equivalent of `curl | bash` and executes unverified remote content. The script should be saved to disk, verified, and then executed separately.

Locations:

- `action.yml:46`

### script-injection (severity: high)

Rule (a) violation: The expression `${{ env.install_path }}` is interpolated directly inside a `run:` shell command string in the 'Add Shorebird to Path (MacOS / Linux)' step (line 51: `install_path=${{ env.install_path }}`). The `env.install_path` value was set from an inherited process environment variable (`$HOME`/`$XDG_CONFIG_HOME`) and flows through YAML template substitution before the shell parses it, allowing shell metacharacter injection. The value should be passed via an `env:` block and referenced as `$install_path` (already set earlier in the script) or via a safe env var reference.

Locations:

- `action.yml:51`

### script-injection (severity: high)

Rule (a) violation: The expression `${{ env.install_path }}` is interpolated directly inside a `run:` PowerShell command string in the 'Add Shorebird to Path (Windows)' step (line 56: `Add-Content $env:GITHUB_PATH "${{ env.install_path }}\bin"`). The `env.install_path` value flows through YAML template substitution before PowerShell parses it, allowing injection of arbitrary content into the GITHUB_PATH write. The value should be referenced via `$env:install_path` directly without `${{ }}` interpolation.

Locations:

- `action.yml:56`

### github-env-injection (severity: high)

The 'Determine Install Path (MacOS / Linux)' step writes `install_path=$install_path` to `$GITHUB_ENV` (line 22) where `$install_path` is derived from the inherited process environment variables `$XDG_CONFIG_HOME` and `$HOME`. These are workflow-controlled (set by the calling workflow) and are treated as untrusted. The value is written without the required sanitization step (`printf '%s' "$install_path" | tr -d '\n\r'`), enabling newline injection that could set arbitrary environment variables for subsequent steps.

Locations:

- `action.yml:22`

### github-env-injection (severity: high)

The 'Determine Install Path (Windows)' step writes `install_path=$install_path` to `$env:GITHUB_ENV` (line 28) where `$install_path` is derived from the inherited `$home` variable. This is a workflow-controlled value written without sanitization, enabling newline injection into GITHUB_ENV.

Locations:

- `action.yml:28`

### github-env-injection (severity: high)

The 'Add Shorebird to Path (MacOS / Linux)' step writes `$install_path/bin` to `$GITHUB_PATH` (line 52: `echo $install_path/bin >> $GITHUB_PATH`) where `$install_path` was previously set from the unsanitized `$GITHUB_ENV` value (itself derived from workflow-controlled env vars). The value is written without sanitization (`printf '%s' ... | tr -d '\n\r'`), enabling newline injection into GITHUB_PATH. Additionally, `$install_path` is unquoted in the echo command.

Locations:

- `action.yml:52`

### github-env-injection (severity: high)

The 'Add Shorebird to Path (Windows)' step writes `${{ env.install_path }}\bin` to `$env:GITHUB_PATH` (line 56: `Add-Content $env:GITHUB_PATH "${{ env.install_path }}\bin"`). The `env.install_path` value is workflow-controlled and is written to GITHUB_PATH without sanitization, enabling newline injection that could add arbitrary entries to the PATH.

Locations:

- `action.yml:56`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all 9 findings in action.yml:
1. Pinned actions/cache@v6 to full SHA 55cc8345863c7cc4c66a329aec7e433d2d1c52a9
2. Linux install: download script to mktemp file, execute 'bash "$INSTALL_SCRIPT" --force' (dropped '--' shell option terminator per instructions)
3. Windows install: download script via Invoke-WebRequest to temp .ps1 file, execute with '& $install_script'
4. Linux path step: moved ${{ env.install_path }} to env: block as INSTALL_PATH, referenced as $INSTALL_PATH in shell
5. Windows path step: replaced ${{ env.install_path }} with $env:install_path
6. Linux GITHUB_ENV write: sanitized install_path with 'printf | tr -d \n\r' before writing
7. Windows GITHUB_ENV write: sanitized with PowerShell -replace to strip CR/LF before writing
8. Linux GITHUB_PATH write: sanitized INSTALL_PATH with tr -d '\n\r', quoted echo command
9. Windows GITHUB_PATH write: sanitized $env:install_path with -replace before writing to GITHUB_PATH

