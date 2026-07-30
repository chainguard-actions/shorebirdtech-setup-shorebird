<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.0.1** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Install Shorebird (MacOS / Linux)' step downloads a remote shell script from the `main` branch of shorebirdtech/install and pipes it directly to bash without first saving it to disk: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. This is unsafe because the content of the remote script is executed immediately without any integrity verification, and the `main` branch reference is mutable.

Locations:

- `action.yml:41`

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable tags rather than immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved:
- `action.yml` line 37: `actions/cache@v4` (mutable tag `v4`)
- `.github/workflows/main.yaml` line 10: `VeryGoodOpenSource/very_good_workflows/.github/workflows/semantic_pull_request.yml@v1` (mutable tag `v1`)
- `.github/workflows/main.yaml` line 21: `actions/checkout@v4` (mutable tag `v4`)

Locations:

- `action.yml:37`
- `.github/workflows/main.yaml:10`
- `.github/workflows/main.yaml:21`

### script-injection (severity: high)

Two `run:` blocks in action.yml interpolate `${{ env.install_path }}` directly inside shell command strings (sub-rule a), allowing the value — which is set from a prior step and flows through the YAML template engine before the shell sees it — to inject shell metacharacters:
- Line 52 (bash): `install_path=${{ env.install_path }}` — the expression is expanded by the template engine before bash parses the script.
- Line 58 (pwsh): `Add-Content $env:GITHUB_PATH "${{ env.install_path }}\bin"` — same issue in PowerShell.

Additionally, line 53 (sub-rule b): `echo $install_path/bin >> $GITHUB_PATH` — the shell variable `$install_path` is unquoted, allowing word-splitting and glob expansion on its value.

Locations:

- `action.yml:52`
- `action.yml:53`
- `action.yml:58`

### github-env-injection (severity: high)

Multiple steps write values derived from inherited (workflow-controlled) environment variables to GitHub's special environment files without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):

- Line 23 (bash): `echo install_path=$install_path >> $GITHUB_ENV` — `$install_path` is derived from `$XDG_CONFIG_HOME` or `$HOME`, both of which are inherited from the calling workflow and could contain newlines that inject additional key=value pairs into GITHUB_ENV.
- Line 31 (pwsh): `Write-Output "install_path=$install_path" >> $env:GITHUB_ENV` — same issue on Windows; `$install_path` is derived from `$home` without sanitization.
- Line 53 (bash): `echo $install_path/bin >> $GITHUB_PATH` — the unsanitized `$install_path` value (itself read back from GITHUB_ENV via `${{ env.install_path }}`) is written to GITHUB_PATH without sanitization.
- Line 58 (pwsh): `Add-Content $env:GITHUB_PATH "${{ env.install_path }}\bin"` — the template-expanded value of `env.install_path` is written to GITHUB_PATH without sanitization.

Locations:

- `action.yml:23`
- `action.yml:31`
- `action.yml:53`
- `action.yml:58`

### missing-permissions (severity: medium)

The workflow file `.github/workflows/main.yaml` has no top-level `permissions:` key and no job-level `permissions:` key on any of its jobs (`semantic-pull-request` and `e2e`). Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., `write` on all scopes for repositories configured with permissive defaults).

Locations:

- `.github/workflows/main.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 5 findings across action.yml and .github/workflows/main.yaml:

1. unsafe-shell: Changed MacOS/Linux install step to download install.sh to /tmp/shorebird_install.sh first, execute it, then delete it. Changed Windows install step similarly to download to a temp file before executing. Eliminated curl-pipe-to-bash and iwr-pipe-to-iex patterns.

2. unpinned-uses: Pinned actions/cache@v4 → SHA 0057852bfaa89a56745cba8c7296529d2fc39830, actions/checkout@v4 → SHA 11d5960a326750d5838078e36cf38b85af677262, VeryGoodOpenSource/very_good_workflows@v1 → SHA c2e27845f65775943bc310b3ab89e5c5432b8d4d. All retain tag comments for readability.

3. script-injection: Moved ${{ env.install_path }} expressions out of run: blocks into step env: blocks (as INSTALL_PATH), then referenced via plain shell variable $INSTALL_PATH / $env:INSTALL_PATH.

4. github-env-injection: Added sanitization (printf '%s' | tr -d '\n\r' for bash; -replace for pwsh) before writing install_path to GITHUB_ENV and GITHUB_PATH in all four affected locations.

5. missing-permissions: Added top-level permissions: {} to main.yaml, plus minimal job-level permissions (pull-requests: read for semantic-pull-request, contents: read for e2e).

