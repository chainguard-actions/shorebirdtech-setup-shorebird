<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.3.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.3.0** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'Determine Install Path (MacOS / Linux)' builds install_path from inherited process env vars ${XDG_CONFIG_HOME} and ${HOME}. The calling workflow sets these, and the script does not compute them. The value goes straight into $GITHUB_OUTPUT via `echo "path=$install_path" >> "$GITHUB_OUTPUT"` without first running `printf '%s' ... | tr -d '\n\r'`. A newline in XDG_CONFIG_HOME could inject extra outputs.

Locations:

- `action.yml:22`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (MacOS / Linux)' writes INSTALL_PATH (from steps.install-path.outputs.path, which comes from the env var XDG_CONFIG_HOME/HOME) to $GITHUB_PATH without sanitizing it: `echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"`. A newline in the value adds arbitrary PATH entries. Fix: strip CR/LF with `printf '%s' "$INSTALL_PATH" | tr -d '\n\r'` before writing.

Locations:

- `action.yml:61`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (Windows)' writes $env:INSTALL_PATH (from steps.install-path-windows.outputs.path) to $env:GITHUB_PATH without removing newline characters: `Add-Content $env:GITHUB_PATH "$env:INSTALL_PATH\bin"`.

Locations:

- `action.yml:70`

### unsafe-shell (severity: high)

Remote script piped directly to a shell: `curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force` (also fetched from the mutable 'main' branch). Download it to a file, verify it, then run it.

Locations:

- `action.yml:44`

### unsafe-shell (severity: high)

Remote PowerShell script piped directly to Invoke-Expression: `iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex`. Download it to a file, verify it, then run it.

Locations:

- `action.yml:51`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 420362e..398dd2f 100644
--- a/action.yml
+++ b/action.yml
@@ -1,19 +1,33 @@
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
     required: false
     default: "false"
-
 runs:
   using: composite
   steps:
     - name: Determine Install Path (MacOS / Linux)
       if: ${{ runner.os != 'Windows' }}
       id: install-path
@@ -21,7 +35,6 @@ runs:
         install_path=$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.shorebird" || printf %s "${XDG_CONFIG_HOME}/shorebird")
         echo "path=$install_path" >> "$GITHUB_OUTPUT"
       shell: bash
-
     - name: Determine Install Path (Windows)
       if: ${{ runner.os == 'Windows' }}
       id: install-path-windows
@@ -29,54 +42,53 @@ runs:
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
         key: shorebird-cache-key-${{ runner.os }}-${{ runner.arch }}
-
-    - name: 🐦 Install Shorebird (MacOS / Linux)
+    - name: "\U0001F426 Install Shorebird (MacOS / Linux)"
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && runner.os != 'Windows' }}
       run: |
-        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force
+        INSTALL_SCRIPT="$(mktemp)"
+        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf -o "$INSTALL_SCRIPT"
+        bash "$INSTALL_SCRIPT" --force
       shell: bash
-
-    - name: 🐦 Install Shorebird (Windows)
+    - name: "\U0001F426 Install Shorebird (Windows)"
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && runner.os == 'Windows' }}
       run: |
         Set-ExecutionPolicy RemoteSigned -scope CurrentUser # Needed to execute remote scripts
-        iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex
+        $install_script = [IO.Path]::Combine([IO.Path]::GetTempPath(), [IO.Path]::GetRandomFileName() + ".ps1")
+        Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' -OutFile $install_script
+        & $install_script
       shell: pwsh
-
     # Adding the CLI to PATH is this action's job, so the GITHUB_PATH writes
     # below are intentional.
     - name: ➕ Add Shorebird to Path (MacOS / Linux)
       if: ${{ runner.os != 'Windows' }}
       env:
         INSTALL_PATH: ${{ steps.install-path.outputs.path }}
-      run: | # zizmor: ignore[github-env]
-        echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"
-        echo "added $INSTALL_PATH/bin to GITHUB_PATH"
+      run: |
+        safe_install_path="$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')"
+        echo "${safe_install_path}/bin" >> "$GITHUB_PATH"
+        echo "added ${safe_install_path}/bin to GITHUB_PATH"
       shell: bash
-
     - name: ➕ Add Shorebird to Path (Windows)
       if: ${{ runner.os == 'Windows' }}
       env:
         INSTALL_PATH: ${{ steps.install-path-windows.outputs.path }}
-      run: | # zizmor: ignore[github-env]
-        Add-Content $env:GITHUB_PATH "$env:INSTALL_PATH\bin"
-        Write-Output "added $env:INSTALL_PATH\bin to GITHUB_PATH"
+      run: |
+        $safe_install_path = $env:INSTALL_PATH -replace "`r|`n", ""
+        Add-Content $env:GITHUB_PATH "$safe_install_path\bin"
+        Write-Output "added $safe_install_path\bin to GITHUB_PATH"
       shell: pwsh
-
-    - name: 🍄 Ensure Shorebird is Up-to-Date (MacOS /
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unsafe-shell, prior-fix

**Notes:**

action.yml: CR/LF are now stripped from install_path before each write to $GITHUB_OUTPUT (bash uses tr -d, pwsh uses -replace) and before each write to $GITHUB_PATH, on both MacOS/Linux and Windows. The two curl|bash and iwr|iex installs now download the script to a temp file, run it and then delete it. On the bash side the '-s --' was dropped so --force still reaches the script as $1. This re-applies the security changes from the previous harden, minus its emoji-name re-encoding. The scripts are still downloaded from upstream's 'main' branch with no checksum check. I left that as is because shorebirdtech/install publishes no versioned releases or checksums to check against.

