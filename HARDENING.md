<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.0.1** was hardened automatically. 8 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The '🐦 Install Shorebird (MacOS / Linux)' step pipes a remote script directly to bash without first downloading and verifying it: `curl ... https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. If the remote URL is compromised or the connection is intercepted, arbitrary code will execute on the runner. The script should be downloaded to a file, inspected/verified, and then executed separately.

Locations:

- `action.yml:43`

### unsafe-shell (severity: high)

The '🐦 Install Shorebird (Windows)' step pipes a remote PowerShell script directly to `iex` (Invoke-Expression), the PowerShell equivalent of `curl | bash`: `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' | iex`. If the remote URL is compromised, arbitrary code will execute on the runner. The script should be downloaded to a file and then executed separately.

Locations:

- `action.yml:49`

### unpinned-uses (severity: high)

The '💾 Configure Shorebird Cache' step uses `actions/cache@v4`, which is a mutable tag reference. A tag can be moved to point to a different (potentially malicious) commit at any time. It must be pinned to a full 40-character commit SHA, e.g. `actions/cache@1bd1e32a3bdc45362d1e726936510720a7c6158d # v4`.

Locations:

- `action.yml:37`

### script-injection (severity: high)

Rule (a) violation: The '➕ Add Shorebird to Path (MacOS / Linux)' step interpolates `${{ env.install_path }}` directly inside a `run:` shell command: `install_path=${{ env.install_path }}`. Any `${{ ... }}` expression is substituted into the shell script before the shell parses it, allowing an attacker who controls the `install_path` env value to inject arbitrary shell commands. The value should be passed via an `env:` block and referenced as a quoted shell variable instead.

Locations:

- `action.yml:55`

### script-injection (severity: high)

Rule (a) violation: The '➕ Add Shorebird to Path (Windows)' step interpolates `${{ env.install_path }}` directly inside a `run:` PowerShell command: `Add-Content $env:GITHUB_PATH "${{ env.install_path }}\bin"`. The expression is substituted into the script before PowerShell parses it, enabling command injection. The value should be passed via an `env:` block and referenced as a PowerShell environment variable instead.

Locations:

- `action.yml:60`

### github-env-injection (severity: high)

The 'Determine Install Path (MacOS / Linux)' step writes `install_path` to `$GITHUB_ENV` without sanitization: `echo install_path=$install_path >> $GITHUB_ENV`. The value is derived from `${XDG_CONFIG_HOME}`, an inherited process environment variable that can be set by the calling workflow (untrusted). A newline embedded in `XDG_CONFIG_HOME` could inject additional key=value pairs into `$GITHUB_ENV`. The value must be sanitized with `printf '%s' "$install_path" | tr -d '\n\r'` before writing.

Locations:

- `action.yml:22`

### github-env-injection (severity: high)

The 'Determine Install Path (Windows)' step writes `install_path` to `$env:GITHUB_ENV` without sanitization: `Write-Output "install_path=$install_path" >> $env:GITHUB_ENV`. The value is derived from `$home`, which is an environment-controlled path. A newline embedded in the value could inject additional key=value pairs into `$GITHUB_ENV`. The value should be sanitized before writing.

Locations:

- `action.yml:30`

### github-env-injection (severity: high)

The '➕ Add Shorebird to Path (MacOS / Linux)' step writes `$install_path/bin` to `$GITHUB_PATH` without sanitization: `echo $install_path/bin >> $GITHUB_PATH`. The `install_path` variable is assigned from `${{ env.install_path }}` (itself derived from the calling workflow's environment), making it untrusted. A newline in the value could inject additional entries into `$GITHUB_PATH`. The value must be sanitized with `printf '%s' ... | tr -d '\n\r'` before writing.

Locations:

- `action.yml:56`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all 8 findings in hardened/action/action.yml:

1. unpinned-uses: Pinned actions/cache@v4 to actions/cache@0057852bfaa89a56745cba8c7296529d2fc39830 # v4

2. unsafe-shell (Linux/Mac): Replaced 'curl ... | bash -s -- --force' with: download script to temp file via curl -o, execute 'bash "$INSTALL_SCRIPT" --force' (dropped '--' per instructions since it was the shell's option terminator), then remove temp file.

3. unsafe-shell (Windows): Replaced 'iwr ... | iex' with: download script to temp file via Invoke-WebRequest -OutFile, execute with '& $install_script', then remove temp file.

4. script-injection (Linux/Mac add to path): Moved ${{ env.install_path }} to env block as INSTALL_PATH, referenced as $INSTALL_PATH in shell.

5. script-injection (Windows add to path): Moved ${{ env.install_path }} to env block as INSTALL_PATH, referenced as $env:INSTALL_PATH in PowerShell.

6. github-env-injection (Linux/Mac determine path): Added sanitization with 'printf | tr -d' before writing install_path to $GITHUB_ENV.

7. github-env-injection (Windows determine path): Added sanitization with PowerShell -replace before writing install_path to $env:GITHUB_ENV.

8. github-env-injection (Linux/Mac add to path): Sanitized INSTALL_PATH with 'printf | tr -d' before writing to $GITHUB_PATH.

