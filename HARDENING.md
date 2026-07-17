<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v0.1.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v0.1.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

In action.yml, the install step fetches a remote shell script and pipes it directly to bash: `curl ... https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. This executes arbitrary remote content without any integrity verification, making it vulnerable to supply-chain attacks if the remote URL is compromised.

Locations:

- `action.yml:12`

### github-env-injection (severity: high)

In action.yml, the run block writes `$install_path/bin` to `$GITHUB_PATH` via `echo $install_path/bin >> $GITHUB_PATH`. The variable `$install_path` is derived from inherited environment variables `$HOME` and `$XDG_CONFIG_HOME`, which are workflow-controlled (set by the calling workflow). These values are written to $GITHUB_PATH without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`), allowing a malicious calling workflow to inject newlines and arbitrary entries into GITHUB_PATH.

Locations:

- `action.yml:14`

### unpinned-uses (severity: high)

The workflow file uses mutable tag references instead of pinned full 40-character SHA digests, making the workflow vulnerable to supply-chain attacks if those tags are moved or overwritten. Failing references:
- `uses: VeryGoodOpenSource/very_good_workflows/.github/workflows/semantic_pull_request.yml@v1` (mutable tag `v1`)
- `uses: actions/checkout@v3` (mutable tag `v3`)

Locations:

- `.github/workflows/main.yaml:9`
- `.github/workflows/main.yaml:16`

### missing-permissions (severity: medium)

The workflow file `.github/workflows/main.yaml` has no top-level `permissions:` key and no job-level `permissions:` key on any of its jobs (`semantic-pull-request`, `e2e`). Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (write access to contents, pull-requests, etc.). Explicit minimal permissions should be declared.

Locations:

- `.github/workflows/main.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings: (1) unsafe-shell in action.yml: replaced `curl ... | bash` with downloading the script to /tmp/shorebird_install.sh then executing it separately; (2) github-env-injection in action.yml: sanitized install_path using `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_PATH, and quoted all variable expansions; (3) unpinned-uses in .github/workflows/main.yaml: pinned actions/checkout to SHA f43a0e5ff2bd294095638e18286ca9a3d1956744 (v3) and VeryGoodOpenSource/very_good_workflows to SHA c2e27845f65775943bc310b3ab89e5c5432b8d4d (v1); (4) missing-permissions in .github/workflows/main.yaml: added `permissions: {}` at the top level and on both jobs (semantic-pull-request and e2e).

