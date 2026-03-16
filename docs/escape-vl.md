# Escape -- Vulnlab (write-up)

**Difficulty:** Easy
**Box:** Escape (Vulnlab)
**Author:** dkrxhn
**Date:** 2025-09-06

---

## TL;DR

### Kiosk-style Windows box. Bypassed app restrictions by renaming cmd.exe to msedge.exe. Found admin creds. UAC bypass via Start-Process -Verb runAs.
---
## Target info

- Host: `10.10.90.226`
---
## Initial access

Connected via RDP:

```bash
xfreerdp /v:10.10.90.226 -sec-nla
```

![Desktop](images/escape-vl_1.png)

![Restricted environment](images/escape-vl_2.png)

Found a key string: `JWqkl6IDfQxXXmiHIKIP8ca0G9XxnWQZgvtPgON2vWc=`

Renamed `cmd.exe` to `msedge.exe` and it opened -- bypassing the application whitelist.

![cmd as msedge](images/escape-vl_3.png)

Found creds: `admin:Twisting3021`

![Creds](images/escape-vl_4.png)

![Logged in](images/escape-vl_5.png)

---
## Privilege escalation

**Tried fodhelper UAC bypass -- didn't work.**

![fodhelper](images/escape-vl_6.png)

![fodhelper fail](images/escape-vl_7.png)

Used `Start-Process` with `-Verb runAs` instead:

```powershell
Start-Process powershell.exe -Verb runAs
```

![Elevated shell](images/escape-vl_8.png)

Confirmed admin is in the Administrators group:

```
net user admin
```

![net user](images/escape-vl_9.png)

---
## Lessons & takeaways

- Application whitelisting by process name is trivially bypassable by renaming executables
- When fodhelper fails, `Start-Process -Verb runAs` can still elevate if the user is in the Administrators group
---
