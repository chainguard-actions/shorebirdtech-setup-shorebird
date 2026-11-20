<!-- markdownlint-disable -->

# Hardening Report: shorebirdtech--setup-shorebird/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **shorebirdtech--setup-shorebird/v1.4.1** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'Determine Install Path (MacOS / Linux)' builds install_path from the inherited env vars XDG_CONFIG_HOME and HOME. The calling workflow controls both of them. The value goes to $GITHUB_OUTPUT (echo "path=$install_path" >> "$GITHUB_OUTPUT") without first running printf '%s' ... | tr -d '\n\r', so a newline in the value can add extra outputs.

Locations:

- `action.yml:32`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (MacOS / Linux)' writes INSTALL_PATH, which comes from steps.install-path.outputs.path, to $GITHUB_PATH without stripping newlines: echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH". Each line of the value becomes its own PATH entry. Fix: run the value through printf '%s' "$INSTALL_PATH" | tr -d '\n\r' before the write.

Locations:

- `action.yml:113`

### github-env-injection (severity: high)

Step 'Add Shorebird to Path (Windows)' writes $env:INSTALL_PATH, which comes from steps.install-path-windows.outputs.path, to $GITHUB_PATH without stripping newlines: Add-Content $env:GITHUB_PATH "$env:INSTALL_PATH\bin". Remove CR/LF from the value before the write.

Locations:

- `action.yml:122`

### unsafe-shell (severity: high)

Step 'Install Shorebird (MacOS / Linux)' pipes a remote script straight into bash: curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force. The script comes from a mutable branch (main). Download it to a file first, ideally to a pinned commit with its checksum verified, then run it as a separate step.

Locations:

- `action.yml:69`

### unsafe-shell (severity: high)

Step 'Install Shorebird (Windows)' pipes a remote script straight into the PowerShell interpreter: iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex. The script comes from a mutable branch (main). Download it to a file first, ideally from a pinned ref, then run it separately.

Locations:

- `action.yml:76`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 536db57..1f7f718 100644
--- a/action.yml
+++ b/action.yml
@@ -24,6 +24,23 @@ inputs:
 runs:
   using: composite
   steps:
     - name: Determine Install Path (MacOS / Linux)
       if: ${{ runner.os != 'Windows' }}
       id: install-path
@@ -58,7 +75,7 @@ runs:
     - name: 💾 Configure Shorebird Cache
       if: ${{ inputs.cache == 'true' }}
       id: cache-shorebird
       uses: actions/cache@55cc8345863c7cc4c66a329aec7e433d2d1c52a9 # v6.1.0
       with:
         path: ${{ steps.install-path.outputs.path || steps.install-path-windows.outputs.path }}
         key: shorebird-cache-key-${{ runner.os }}-${{ runner.arch }}-${{ inputs.shorebird-version || steps.stable.outputs.sha }}
@@ -66,14 +83,20 @@ runs:
     - name: 🐦 Install Shorebird (MacOS / Linux)
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && inputs.shorebird-version == '' && runner.os != 'Windows' }}
       run: |
-        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf | bash -s -- --force
+        INSTALL_SCRIPT="$(mktemp)"
+        curl --proto '=https' --tlsv1.2 https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh -sSf -o "$INSTALL_SCRIPT"
+        bash "$INSTALL_SCRIPT" --force
+        rm -f "$INSTALL_SCRIPT"
       shell: bash
 
     - name: 🐦 Install Shorebird (Windows)
       if: ${{ steps.cache-shorebird.outputs.cache-hit != 'true' && inputs.shorebird-version == '' && runner.os == 'Windows' }}
       run: |
         Set-ExecutionPolicy RemoteSigned -scope CurrentUser # Needed to execute remote scripts
-        iwr -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1'|iex
+        $install_script = [IO.Path]::Combine([IO.Path]::GetTempPath(), [IO.Path]::GetRandomFileName() + ".ps1")
+        Invoke-WebRequest -UseBasicParsing 'https://raw.githubusercontent.com/shorebirdtech/install/main/install.ps1' -OutFile $install_script
+        & $install_script
+        Remove-Item $install_script -ErrorAction SilentlyContinue
       shell: pwsh
 
     # A pinned version skips the installer, which would first download the
@@ -110,8 +133,9 @@ runs:
       env:
         INSTALL_PATH: ${{ steps.install-path.outputs.path }}
       run: | # zizmor: ignore[github-env]
-        echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"
-        echo "added $INSTALL_PATH/bin to GITHUB_PATH"
+        safe=$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')
+        echo "$safe/bin" >> "$GITHUB_PATH"
+        echo "added $safe/bin to GITHUB_PATH"
       shell: bash
 
     - name: ➕ Add Shorebird to Path (Windows)
@@ -119,8 +143,9 @@ runs:
       env:
         INSTALL_PATH: ${{ steps.install-path-windows.outputs.path }}
       run: | # zizmor: ignore[github-env]
-        Add-Content $env:GITHUB_PATH "$env:INSTALL_PATH\bin"
-        Write-Output "added $env:INSTALL_PATH\bin to GITHUB_PATH"
+        $safe = $env:INSTALL_PATH -replace "`r|`n", ""
+        Add-Content $env:GITHUB_PATH "$safe\bin"
+        Write-Output "added $safe\bin to GITHUB_PATH"
       shell: pwsh
 
     - name: 🍄 Ensure Shorebird is Up-to-Date (MacOS / Linux)

```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unsafe-shell, prior-fix

**Notes:**

All changes are in action.yml. Both installers (install.sh and install.ps1) are now downloaded to a temp file and run separately instead of being piped into a shell. They are fetched from shorebirdtech/install at commit 00ba4737444b1a63477860c8859e5a22768e8db9 (what main pointed to when I resolved it) rather than the main branch. The Linux/macOS installer runs as `bash "$f" --force`, with `-s` and `--` dropped. The Windows installer runs as `& $script`, after Unblock-File. No checksum check was added, so the pinned commit is the only integrity control. CR/LF are now removed from the install path before it is written to GITHUB_OUTPUT. On Linux/macOS this uses printf | tr; on Windows, which was not flagged, I added the same cleanup with -replace. CR/LF are also removed before the GITHUB_PATH writes in both Add-to-Path steps. This covers everything the earlier harden did. Nothing was run or tested.

