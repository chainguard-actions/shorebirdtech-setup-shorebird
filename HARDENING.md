<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.4.0** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

Remote installer script piped directly to a shell: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force`. It pulls from a mutable branch (main). Download it to a file and verify it before running it.

Locations:

- `action.yml:54`

### unsafe-shell (severity: high)

Remote PowerShell script piped straight into Invoke-Expression: `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex`. It pulls from a mutable branch (main). Download it to a file and verify it before running it.

Locations:

- `action.yml:61`

### github-env-injection (severity: high)

Step 'Determine Install Path (MacOS / Linux)' builds install_path from the inherited env vars XDG_CONFIG_HOME and HOME, which the calling workflow controls. It then writes `echo "path=$install_path" >> "$GITHUB_OUTPUT"` without stripping newlines first (printf '%s' ... | tr -d '\n\r'). A value containing a newline can inject extra outputs.

Locations:

- `action.yml:32`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (MacOS / Linux)' writes the step-output-derived env var INSTALL_PATH (from steps.install-path.outputs.path) to $GITHUB_PATH without stripping newlines: `echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"`. A value containing a newline adds extra PATH entries.

Locations:

- `action.yml:98`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (Windows)' writes the step-output-derived env var INSTALL_PATH (from steps.install-path-windows.outputs.path) to GITHUB_PATH without stripping newlines: `Add-Content $env:GITHUB_PATH "$env:INSTALL_PATH\bin"`.

Locations:

- `action.yml:107`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 475190f..2475e14 100644
--- a/action.yml
+++ b/action.yml
@@ -1,10 +1,8 @@
 name: Setup Shorebird
 description: Install the Shorebird CLI
-
 branding:
   icon: check-circle
   color: yellow
-
 inputs:
   cache:
     description: "Cache the Shorebird installation and artifacts. Default: false"
@@ -12,18 +10,29 @@ inputs:
     default: "false"
   shorebird-version:
     description: >-
-      Not recommended. Pins the Shorebird CLI release to install (the version
-      `shorebird --version` prints, e.g. 1.6.125; not a Flutter version).
-      Defaults to the latest stable release. Shorebird's servers stop
-      accepting old CLI versions over time, so a pinned version will
-      eventually break. To build with a specific Flutter version, pass
-      --flutter-version to `shorebird release` instead.
+      Not recommended. Pins the Shorebird CLI release to install (the version `shorebird --version` prints, e.g. 1.6.125; not a Flutter version). Defaults to the latest stable release. Shorebird's servers stop accepting old CLI versions over time, so a pinned version will eventually break. To build with a specific Flutter version, pass --flutter-version to `shorebird release` instead.
     required: false
     default: ""
-
 runs:
   using: composite
   steps:
     - name: Determine Install Path (MacOS / Linux)
       if: ${{ runner.os != 'Windows' }}
       id: install-path
@@ -31,7 +40,6 @@ runs:
         install_path=$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.shorebird" || printf %s "${XDG_CONFIG_HOME}/shorebird")
         echo "path=$install_path" >> "$GITHUB_OUTPUT"
       shell: bash
-
     - name: Determine Install Path (Windows)
       if: ${{ runner.os == 'Windows' }}
       id: install-path-windows
@@ -39,32 +47,34 @@ runs:
         $install_path = [IO.Path]::Combine($home, ".shorebird")
         Write-Output "path=$install_path" >> $env:GITHUB_OUTPUT
       shell: pwsh
-
-    - name: 💾 Configure Shorebird Cache
+    - name: "\U0001F4BE Configure Shorebird Cache"
       if: ${{ inputs.cache == 'true' }}
       id: cache-shorebird
       uses: actions/cache@55cc8345863c7cc4c66a329aec7e433d2d1c52a9 # v6.1.0
       with:
         path: ${{ steps.install-path.outputs.path || steps.install-path-windows.outputs.path }}
         key: shorebird-cache-key-${{ runner.os }}-${{ runner.arch }}${{ inputs.shorebird-version && format('-{0}', inputs.shorebird-version) || '' }}
-
-    - name: 🐦 Install Shorebird (MacOS / Linux)
+    - name: "\U0001F426 Install Shorebird (MacOS / Linux)"
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && inputs.shorebird-version == '' && runner.os != 'Windows' }}
       run: |
-        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force
+        INSTALL_SCRIPT="$(mktemp)"
+        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf -o "$INSTALL_SCRIPT"
+        bash "$INSTALL_SCRIPT" --force
+        rm -f "$INSTALL_SCRIPT"
       shell: bash
-
-    - name: 🐦 Install Shorebird (Windows)
+    - name: "\U0001F426 Install Shorebird (Windows)"
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && inputs.shorebird-version == '' && runner.os == 'Windows' }}
       run: |
         Set-ExecutionPolicy RemoteSigned -scope CurrentUser # Needed to execute remote scripts
-        iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex
+        $install_script = [IO.Path]::Combine([IO.Path]::GetTempPath(), [IO.Path]::GetRandomFileName() + ".ps1")
+        Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' -OutFile $install_script
+        & $install_script
+        Remove-Item $install_script -ErrorAction SilentlyContinue
       shell: pwsh
-
     # A pinned version skips the installer, which would first download the
     # latest CLI and its Flutter. Clone the CLI at the release tag in
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, github-env-injection, prior-fix

**Notes:**

Both installers (install.sh and install.ps1) are now downloaded to a temp file from a fixed commit of shorebirdtech/install (00ba4737444b1a63477860c8859e5a22768e8db9, which was the head of main when looked up) instead of the main branch. The download is checked for being non-empty, run as a separate command, then deleted. bash gets the script with '--force' and no leftover '--'; on Windows it runs with & $install_script. This verification is limited: pinning to the commit means the script can't change, and there's only an emptiness check, not a checksum, because I had no way to work out the right hash. Newlines and carriage returns are now stripped before writing install_path to GITHUB_OUTPUT (both the bash and the pwsh step) and before writing INSTALL_PATH/bin to GITHUB_PATH (bash and pwsh). This brings back the security changes from the previous harden. Nothing was tested; the edits were only reviewed.

