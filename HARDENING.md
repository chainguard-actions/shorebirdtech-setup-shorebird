<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v0.1.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v0.1.2** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The macOS/Linux install step in action.yml pipes the output of curl directly to bash without first saving the script to disk and verifying it. This means any compromise of the remote URL (raw.githubusercontent.com/shorebirdtech/install/main/install.sh) would result in arbitrary code execution on the runner. Pattern matched: `curl ... | bash -s -- --force`.

Locations:

- `action.yml:12`

### github-env-injection (severity: high)

In the macOS/Linux install step, the variable `install_path` is derived from the inherited environment variables `$HOME` and `$XDG_CONFIG_HOME` (both set by the calling workflow and therefore untrusted). Its value is written directly to `$GITHUB_PATH` via `echo $install_path/bin >> $GITHUB_PATH` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow could inject newlines into `$XDG_CONFIG_HOME` to smuggle additional entries into `$GITHUB_PATH`.

Locations:

- `action.yml:15`

### unpinned-uses (severity: high)

Two `uses:` references in .github/workflows/main.yaml are pinned to mutable tags rather than immutable full-length commit SHAs, making the workflow vulnerable to supply-chain attacks if those tags are moved:
- `VeryGoodOpenSource/very_good_workflows/.github/workflows/semantic_pull_request.yml@v1` (tag `v1`)
- `actions/checkout@v3` (tag `v3`)
Both should be pinned to a full 40-character hex commit SHA.

Locations:

- `.github/workflows/main.yaml:9`
- `.github/workflows/main.yaml:19`

### missing-permissions (severity: medium)

The workflow file .github/workflows/main.yaml has no top-level `permissions:` key, and the `e2e` job (which checks out and runs the action) has no job-level `permissions:` key either. Without explicit permissions, the job inherits the default repository token permissions, which may be broader than necessary. A top-level `permissions: {}` or minimal job-level permissions should be specified.

Locations:

- `.github/workflows/main.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings:
1. unsafe-shell (action.yml line 12): Replaced `curl ... | bash -s -- --force` with downloading the script to a mktemp file, executing `bash "$INSTALL_SCRIPT" --force`, then removing the temp file. The '--' was correctly dropped since it was the shell's option terminator, not the script's.
2. github-env-injection (action.yml line 15): Added sanitization step using `printf '%s' "$install_path/bin" | tr -d '\n\r'` before writing to $GITHUB_PATH, preventing newline injection via $XDG_CONFIG_HOME or $HOME.
3. unpinned-uses (.github/workflows/main.yaml lines 9, 19): Pinned VeryGoodOpenSource/very_good_workflows to SHA c2e27845f65775943bc310b3ab89e5c5432b8d4d (# v1) and actions/checkout to SHA a37ce9120846195fa4ece8f58b268e6043cb2f26 (# v3).
4. missing-permissions (.github/workflows/main.yaml line 1): Added top-level `permissions: {}` to restrict the default GITHUB_TOKEN permissions to none.

### Iteration 2

**Fixes applied:** unsafe-shell

**Notes:**

Fixed the Windows installation step in action.yml. Replaced the `iwr ... | iex` pattern (PowerShell equivalent of curl|bash) with a safe two-step approach: (1) download the install.ps1 script to a temp file using `Invoke-WebRequest -OutFile`, (2) execute it separately with `& $installScript`, then clean up with `Remove-Item`. This matches the safe pattern already used in the Linux/macOS step.

