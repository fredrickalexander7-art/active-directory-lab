# Incident Writeup — Sales User Account Locked Out

## Situation
A Sales-department user, `jdoe`, was subject to a department-specific Fine-Grained Password Policy (`Sales-Lockout-Policy`) applied to the `Sales Users` security group, configured with an aggressive lockout threshold: the account locks after 2 failed login attempts, for a 30-minute window. After two incorrect password attempts from the domain-joined client `CLIENT01`, the account locked out. Attempting to log in afterward returned: *"The referenced account is currently locked out and may not be logged on to."*

## Task
Diagnose the root cause of the lockout using log evidence (not just assumption), and restore the user's access — without weakening the lockout policy for the rest of the Sales department.

## Action

1. **Initial investigation attempt:** Opened Event Viewer's Security log looking for Event ID 4740 (Account Lockout). Filtered for it — zero results, despite the account being visibly locked. This didn't match expectations and triggered a deeper investigation.

2. **Audit policy troubleshooting:** Suspected the relevant audit subcategory wasn't enabled. Checked and edited the **Default Domain Controllers Policy** under Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration. Initially enabled the wrong subcategory (Account Management → "Audit User Account Management"); still zero results after `gpupdate /force`. Corrected to the actual right location — **Logon/Logoff → "Audit Account Lockout"** — enabled Success auditing, ran `gpupdate /force` again, and confirmed the effective setting directly with:
   ```
   auditpol /get /subcategory:"Account Lockout"
   ```
   which confirmed `Success` was now active.

3. **Still no 4740 events after re-triggering the lockout.** Broadened the search instead of continuing to assume the specific Event ID: filtered Event Viewer for Event ID 4625 (general Logon Failure) over the last hour instead. This returned 17 matching events, confirming failed logon attempts against `jdoe` from `CLIENT01.corp.local` were being logged, with Status `0xC000006D` / Sub Status `0xC000006A` (wrong password) — useful, but not yet the specific lockout event.

4. **Root cause of the missing 4740 events, finally identified:** the Event Viewer being checked was on the wrong machine. Domain account-lockout decisions and their audit events are recorded on the domain controller (the authoritative source for account state), not on the client machine being logged into. Switching to check `DC01`'s Security log (instead of the client's) and re-filtering for Event ID 4740 immediately returned the real lockout events.

5. **Confirmed root cause:** Event ID 4740, logged on `DC01.corp.local`, Task Category "User Account Management," showing:
   - Account That Was Locked Out: `CORP\jdoe`
   - Caller Computer Name: `CLIENT01`
   - Timestamp matching the exact moment of the second failed attempt

## Root cause
The `jdoe` account exceeded the 2-attempt lockout threshold defined by the `Sales-Lockout-Policy` Fine-Grained Password Policy after two incorrect password attempts from `CLIENT01`. This was expected, intentional policy behavior — not a system malfunction. The investigation delay was caused by (a) an audit-policy subcategory that wasn't enabled by default, and (b) initially checking the event log on the wrong machine.

## Fix
Unlocked the account in Active Directory Users and Computers (Sales OU → `jdoe` → Account tab → "Unlock account"). Confirmed the fix by logging in as `jdoe` from `CLIENT01` afterward — login succeeded.

## Prevention / recommendations
- Document the exact audit-policy subcategories required for common security events (account lockout, password changes, etc.) as a quick-reference for future incidents, rather than relying on memory or trial-and-error under time pressure.
- When investigating domain-account events, always check the domain controller's Security log first — not the client's — since that's the authoritative source for domain-level account state changes.
- Consider whether a 2-attempt lockout threshold is appropriate for production use; it's intentionally aggressive here for lab/demonstration purposes, but a real deployment would likely use a higher threshold (5-10 attempts) to reduce accidental lockouts from simple typos.

## Screenshots
- `../Screenshots/` — lockout failure screen, Event Viewer 4740 detail (Account That Was Locked Out / Caller Computer Name), Event Viewer 4625 detail (Status/Sub Status codes), successful login after fix
