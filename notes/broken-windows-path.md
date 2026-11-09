# Broken Windows PATH: how to check and fix it

Nov 7, 2026 · @Marco

## The problem

Installers and uninstallers edit the Windows PATH, and some do it badly. A broken PATH can stop unrelated tools from working, even ones you have not touched.

PATH is a list of folders separated by `;`. When a program runs a command such as `cmd`, `git` or `python` without a full path, Windows searches these folders in order. There are two PATHs, merged at login:

- **System PATH**: stored in `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`, shared by all users, needs admin rights to change.
- **User PATH**: stored in `HKCU\Environment`, per user, no admin rights needed.

Common ways tools damage it:

- **Lost separator**: an uninstaller removes its own entry together with both `;` around it, so two neighbours merge into one non-existent folder (e.g. `...\Java\javapath%SystemRoot%\system32`).
- **Truncation**: old installers that use `setx` cut PATH at 1024 characters, silently dropping the tail.
- **Wrong registry type**: PATH saved as `REG_SZ` instead of `REG_EXPAND_SZ`, so `%SystemRoot%` and similar variables are no longer expanded.
- **Stale or duplicate entries**: folders left behind by removed software, harmless but noisy.

## Symptoms

Suspect the PATH when several tools fail at once with "not found" errors, right after installing or uninstalling software.

- Installers built on Node/Electron (e.g. MCUXpresso Installer, VS Code tooling) report `spawn cmd ENOENT` or "could not spawn command".
- PowerShell says `where.exe`, `ping` or `ipconfig` is "not recognized", although the files exist in `C:\Windows\System32`.
- Build tools fail to find `cmake`, `ninja`, `git`, `python` or a compiler that is installed.
- A command works in one terminal but not in another, or only after a reboot (a session started before or after the change).

## How to check

One command tells you in a second whether the PATH is healthy. Open a **new** PowerShell window (not as administrator, so it starts in your home folder) and run:

```powershell
cd $HOME
where.exe cmd
```

The answer must be `C:\Windows\System32\cmd.exe`. An error means System32 is missing from PATH.

For a full diagnosis, run these checks. They read the registry directly, so they show the saved PATH, not the one your session inherited.

1. Show both PATHs as stored, one entry per line:

   ```powershell
   [Environment]::GetEnvironmentVariable('Path','Machine') -split ';'
   [Environment]::GetEnvironmentVariable('Path','User') -split ';'
   ```
2. Find merged entries (a second drive letter or a `%variable%` in the middle of an entry):

   ```powershell
   foreach ($s in 'Machine','User') {
     [Environment]::GetEnvironmentVariable('Path',$s) -split ';' |
       Where-Object { $_ -match '.+([A-Za-z]:\\|%[A-Za-z]+%)' } |
       ForEach-Object { "$s : MERGED -> $_" }
   }
   ```
3. Find entries pointing to folders that no longer exist:

   ```powershell
   foreach ($s in 'Machine','User') {
     [Environment]::GetEnvironmentVariable('Path',$s) -split ';' | Where-Object { $_ } |
       ForEach-Object {
         $e = [Environment]::ExpandEnvironmentVariables($_)
         if (-not (Test-Path $e)) { "$s : MISSING -> $_" }
       }
   }
   ```
4. Check the registry type of the system PATH; it must be `ExpandString`:

   ```powershell
   (Get-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment').GetValueKind('Path')
   ```
5. Check the length; close to or above 2047 characters, older tools may truncate it:

   ```powershell
   ([Environment]::GetEnvironmentVariable('Path','Machine')).Length
   ```

Note that the Environment Variables editor shows entries unexpanded (`%SystemRoot%\system32`), while PowerShell shows them expanded (`C:\WINDOWS\system32`). They are the same entry.

## How to fix

Back up first, fix in the graphical editor, then start a fresh session. No reboot is needed.

1. **Back up both PATHs** (see Prevention below) so you can undo any mistake.
2. **Open the editor**: press Win+R, type `sysdm.cpl`, go to **Advanced** and click **Environment Variables...**. Changing the system PATH needs admin rights.
3. **Edit the right PATH**: select **Path** under *System variables* (or *User variables*, depending on where the check found the problem) and click **Edit...**.
4. **Repair entries** in the list view, one folder per line. The editor adds the `;` separators itself, so never type them here.
   - Merged entry: edit it to keep only the first folder, then click **New** and add the second one.
   - Missing core entries: make sure these exist, in this order, near the top of the system PATH: `%SystemRoot%\system32`, `%SystemRoot%`, `%SystemRoot%\System32\Wbem`, `%SystemRoot%\System32\WindowsPowerShell\v1.0\`.
   - Stale entries pointing to deleted folders can be removed with **Delete**.
   - Alternative: **Edit text...** shows the whole PATH on one line, where you can insert a missing `;` directly. The list view is safer.
5. **Save**: click OK in all three windows. Closing with the X discards the changes. Saving from this editor also restores the correct `REG_EXPAND_SZ` type.
6. **Start a fresh session**: close all terminals (Windows Terminal completely, since new tabs inherit the old environment) and any open installer or IDE, then reopen them. If a tool still sees the old PATH, sign out and back in.
7. **Verify** with `where.exe cmd` from `$HOME`, then rerun the installation that failed.

If you have no admin rights, adding the missing folder to the **user** PATH is a working stopgap; ask IT to repair the system PATH.

## Prevention

Keep a known-good backup and run the one-line check after every install or uninstall of toolchains, IDEs or SDKs.

- **Back up** both PATHs to `.reg` files (run from an admin PowerShell for the system one):

  ```powershell
  reg export "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" "$HOME\path-system-backup.reg" /y
  reg export "HKCU\Environment" "$HOME\path-user-backup.reg" /y
  ```

  Double-clicking a `.reg` file restores it; sign out and back in afterwards.
- **Keep a readable copy** too, useful for comparing before and after:

  ```powershell
  [Environment]::GetEnvironmentVariable('Path','Machine') -split ';' | Out-File "$HOME\path-system.txt"
  [Environment]::GetEnvironmentVariable('Path','User') -split ';' | Out-File "$HOME\path-user.txt"
  ```
- **Check after changes**: `where.exe cmd` from `$HOME`; if it fails, run the full diagnosis above.
- **Avoid `setx PATH`** in your own scripts: it truncates at 1024 characters and flattens the user and system PATH together.
- **On managed PCs**, tell IT when a corporate deployment breaks the PATH, since it will come back with the next push.
