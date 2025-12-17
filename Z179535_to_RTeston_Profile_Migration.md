# Windows User Profile Migration Runbook  
**Z179535 → RTeston (Pragmatic / Least-Bad Approach)**

> ⚠️ DISCLAIMER  
> This procedure is **not Microsoft-supported**.  
> It is a **controlled profile path reassignment** intended to preserve a required home folder name (`RTeston`) when rebuilding the profile is not an option.  
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
