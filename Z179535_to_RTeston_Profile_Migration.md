## Future Laptop Migration with USMT (User State Migration Tool)

  ### Requirement

  When IT uses USMT to backup and restore the laptop, the profile must be restored to `C:\Users\RTeston` instead of `C:\Users\Z179535`. Many applications and configurations depend on the `RTeston` folder path.

  ---

  ### Option 1: Pre-Create Profile Using Known SID

  **Step A: Get SID from old machine (before migration)**

  Run on old laptop while logged in as Z179535:
  ```cmd
  whoami /user
  ```
  Save the SID (e.g., S-1-5-21-4258776639-1271169360-244890835-423316). Domain SIDs are the same across machines.

  **Step B: On new machine (before first login as Z179535)**

  1. Log in as a local admin account
  2. Create the target folder:
  ```cmd
    mkdir C:\Users\RTeston
  ```
  4. Pre-create the registry entry using the SID from Step A:  
     (SID: **S-1-5-21-4258776639-1271169360-244890835-423316**)
  ```cmd
  reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\<SID>" /v ProfileImagePath /t REG_EXPAND_SZ /d "C:\Users\RTeston" /f
  reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\<SID>" /v State /t REG_DWORD /d 0 /f
  ```
  5. Run USMT LoadState to restore data into C:\Users\RTeston
  6. Log in as Z179535 — Windows uses the existing registry entry and loads from RTeston

  ---
  ### Option 2: Restore First, Then Rename (Fallback)

  Use this if USMT already restored to C:\Users\Z179535:

  1. Log in as a temporary local admin (not Z179535)
  2. Rename the profile folder:
  ``` ren "C:\Users\Z179535" "RTeston" ```
  3. Find and update the registry:
    - Open regedit
    - Navigate to: HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList
    - Find the SID key where ProfileImagePath = C:\Users\Z179535
    - Change it to: C:\Users\RTeston
  4. Update Shell Folders registry (see Shell Folders fix section below)
  5. Log in as Z179535

  ---
  Shell Folders Registry Fix (Required for Both Options)

  After profile is set up, verify Shell Folders point to RTeston:

  ```cmd 
  reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" | findstr Z179535
  ```

  If any show Z179535, run these commands to fix:
  ```cmd
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" /v "AppData" /t REG_SZ /d "C:\Users\RTeston\AppData\Roaming" /f
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" /v "Desktop" /t REG_SZ /d "C:\Users\RTeston\Desktop" /f
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" /v "Personal" /t REG_SZ /d "C:\Users\RTeston\Documents" /f
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" /v "Local AppData" /t REG_SZ /d "C:\Users\RTeston\AppData\Local" /f
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" /v "{374DE290-123F-4565-9164-39C4925E467B}" /t REG_SZ /d "C:\Users\RTeston\Downloads" /f
 ```
  (See full list of Shell Folders commands in main runbook)

  ---
  ### Key Message for IT

  "My account is Z179535 but my profile folder must be C:\Users\RTeston — not C:\Users\Z179535. Many applications and configurations depend on the RTeston folder path. Please ensure the USMT restore targets the RTeston profile folder."

  "My SID is: S-1-5-21-xxxx-xxxx-xxxx-xxxxxx (provide actual SID)"

  ---
  ### Post-Restore Checklist

  After USMT restore, verify:

  - echo %USERPROFILE% returns C:\Users\RTeston
  - Shell Folders registry points to RTeston (not Z179535)
  - OneDrive linked to RTeston folder
  - Docker Desktop working
  - WSL working
  - No Explorer.exe file association errors
  ---
  ### Key Message for IT

  "My account is Z179535 but my profile folder must be C:\Users\RTeston — not C:\Users\Z179535. Many applications and configurations depend on the RTeston folder path. Please ensure the USMT restore targets the RTeston profile folder."

  ---
  ### Post-Restore Checklist

  After USMT restore, verify:

  - echo %USERPROFILE% returns C:\Users\RTeston
  - Shell Folders registry points to RTeston (not Z179535)
  - OneDrive linked to RTeston folder
  - Docker Desktop working
  - WSL working
  - No Explorer.exe file association errors

---  

# Windows User Profile Migration Runbook  
**Z179535 → RTeston (Pragmatic / Least-Bad Approach)**

> ⚠️ DISCLAIMER  
>  
> This is a **controlled profile path reassignment** intended to preserve a required home folder name (`RTeston`) when rebuilding the profile is not an option.  
>  
> Expect some post-migration cleanup (VPN, OneDrive, Edge, Store apps).

---

## Objective

- Log in as `z179535`
- `%USERPROFILE%` resolves to `C:\Users\RTeston`
- Existing tooling that depends on `RTeston` continues to work
- Minimize long-term Windows instability

---

## Preconditions (Do Not Skip)

- BitLocker recovery key backed up
- OneDrive paused
- VPN disconnected
- Endpoint protection left enabled
- Confirm current identity:
  ```cmd
  whoami
  ```

---

## PHASE 1 — Create Temporary Local Admin

> You cannot migrate your own profile while logged in.

1. Open **Windows Terminal (Admin)**
2. Create a temporary local admin account:
   ```cmd
   net user LocalAdm_Strict P@ssw0rd!234 /add
   net localgroup Administrators LocalAdm_Strict /add
   ```
3. Sign out of `z179535`
4. Log in as:
   ```
   .\LocalAdm_Strict
   ```

---

## PHASE 2 — Prepare Target Folder

> This does **not** create a Windows profile.  
> It only stages the folder that the existing SID will later use.

1. Create the target folder:
   ```cmd
   mkdir C:\Users\RTeston
   ```
2. Pre-seed ownership:
   ```cmd
   icacls C:\Users\RTeston /setowner Administrators /T
   ```

---

## PHASE 3 — Controlled Data Copy (Critical)

> ⚠️ Do **NOT** use `/COPYALL`  
> ⚠️ Do **NOT** copy `ntuser.dat`

Run the following command:

```cmd
robocopy "C:\Users\Z179535" "C:\Users\RTeston" ^
 /E /XJ /R:1 /W:1 ^
 /XF ntuser.dat ntuser.dat.log* ^
 /XD "AppData\Local\Temp" "AppData\Local\Packages"
```

### What this copies safely
- Desktop
- Documents
- Downloads
- Pictures
- Most Roaming app data

### What this intentionally avoids
- SID-bound registry hive
- UWP / Store app corruption
- Temporary and volatile state

---

## PHASE 4 — Reset and Rebuild ACLs

```cmd
icacls "C:\Users\RTeston" /reset /T
icacls "C:\Users\RTeston" /inheritance:e /T
icacls "C:\Users\RTeston" /grant ^
  "Z179535:(OI)(CI)F" ^
  "SYSTEM:(OI)(CI)F" ^
  "Administrators:(OI)(CI)F" /T
```

---

## 🚧 GUARDRAIL — STOP & VERIFY (DO NOT SKIP)

Before touching the registry or logging in:

1. Verify folder contents:
   ```cmd
   dir C:\Users\RTeston
   ```
2. Verify permissions:
   ```cmd
   icacls C:\Users\RTeston | findstr Z179535
   ```

### You must confirm:
- Files are present
- `Z179535` has **Full Control**

❌ If this is wrong → **STOP** and fix it  
✔ If correct → proceed

---

## PHASE 5 — Update Registry Mapping (Single Value Only)

1. Open **Registry Editor**
2. Navigate to:
   ```
   HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList
   ```
3. Locate the SID key where:
   ```
   ProfileImagePath = C:\Users\Z179535
   ```
4. Change **only** this value to:
   ```
   C:\Users\RTeston
   ```

⚠️ Do not modify any other registry values.

---

## PHASE 6 — First Login Validation

1. Sign out of `LocalAdm_Strict`
2. Log in as `z179535`
3. Open Command Prompt:
   ```cmd
   echo %USERPROFILE%
   whoami /user
   ```

### Expected output
```
C:\Users\RTeston
```

---

## PHASE 7 — Post-Migration Stabilization

1. Reboot **twice**
2. Allow system to idle for 5–10 minutes
3. Re-sign into:
   - OneDrive
   - Edge
   - Teams
4. Reinstall per-user applications as needed
   - VPN clients
   - Security agents
   - Developer tools

> Some app reconfiguration is normal.

---

## PHASE 8 — Cleanup (After 48–72 Hours)

Only after full confidence:

1. Log in as `LocalAdm_Strict`
2. Remove old profile folder:
   ```cmd
   rd /s /q "C:\Users\Z179535"
   ```
3. Delete temporary admin account:
   ```cmd
   net user LocalAdm_Strict /delete
   ```

---

## Known Side Effects (Expected)

| Component | Impact |
|--------|-------|
| VPN clients | Reinstall required |
| Store apps | Re-register |
| Edge | Profile reconnect |
| OneDrive | Re-link |
| Event Logs | Profile warnings |

---

## Validation Checklist

- `%USERPROFILE%` points to `C:\Users\RTeston`
- New files save correctly
- Reboot succeeds
- Windows Update works
- No references to `C:\Users\Z179535` in active workflows

---

## Final Notes

This procedure:
- Preserves a required folder name
- Minimizes Windows 11 fallout
- Accepts limited technical debt in exchange for continuity

This is **controlled damage**, not a clean rebuild.

---

**End of Runbook**
