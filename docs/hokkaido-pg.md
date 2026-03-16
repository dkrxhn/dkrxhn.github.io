# Hokkaido -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Hokkaido (Proving Grounds)
**Author:** dkrxhn
**Date:** 2024-10-31

---

## TL;DR

### SMB spider found creds, BloodHound mapped a path through multiple users via password reuse and LAPS-style enumeration, eventually reaching a user with SeBackupPrivilege to grab the admin flag.
---
## Target info

- Host: `192.168.246.40`
---
## Enumeration

![Nmap results](images/hokkaido-pg_1.png)

![Nmap continued](images/hokkaido-pg_2.png)

![Nmap continued](images/hokkaido-pg_3.png)

![Web enumeration](images/hokkaido-pg_4.png)

![Web enumeration continued](images/hokkaido-pg_5.png)

![More enumeration](images/hokkaido-pg_6.png)

![Enumeration continued](images/hokkaido-pg_7.png)

![Enumeration continued](images/hokkaido-pg_8.png)

![Enumeration continued](images/hokkaido-pg_9.png)

![Enumeration continued](images/hokkaido-pg_10.png)

![Enumeration continued](images/hokkaido-pg_11.png)

![Enumeration continued](images/hokkaido-pg_12.png)

![Enumeration continued](images/hokkaido-pg_13.png)

![Enumeration continued](images/hokkaido-pg_14.png)

![Enumeration continued](images/hokkaido-pg_15.png)

![Enumeration continued](images/hokkaido-pg_16.png)

![Enumeration continued](images/hokkaido-pg_17.png)

![Enumeration continued](images/hokkaido-pg_18.png)

![Enumeration continued](images/hokkaido-pg_19.png)

![Enumeration continued](images/hokkaido-pg_20.png)

---
## SMB spider

```bash
nxc smb 192.168.246.40 -u 'info' -p 'info' -M spider_plus && cat /tmp/nxc_hosted/nxc_spider_plus/192.168.246.40.json
```

![Spider results](images/hokkaido-pg_21.png)

![Spider results continued](images/hokkaido-pg_22.png)

---
## Credential chain

![Found creds](images/hokkaido-pg_23.png)

Found: `Start123!`

info user:

![info user](images/hokkaido-pg_24.png)

Discovery:

![Discovery](images/hokkaido-pg_25.png)

![Additional enumeration](images/hokkaido-pg_26.png)

![Hash found](images/hokkaido-pg_27.png)

Hash was **not** crackable.

![More creds](images/hokkaido-pg_28.png)

Found: `hrapp-service:Untimed$Runny`

![hrapp-service access](images/hokkaido-pg_29.png)

![Further enumeration](images/hokkaido-pg_30.png)

![More enumeration](images/hokkaido-pg_31.png)

Found: `Hazel.Green:haze1988`

---
## BloodHound

BloodHound showed no outbound object control for Hazel.Green initially. Ran the collector again with `hrapp-service` creds instead of `info:info` -- now it shows a path:

![BloodHound path](images/hokkaido-pg_32.png)

![BloodHound continued](images/hokkaido-pg_33.png)

![BloodHound continued](images/hokkaido-pg_34.png)

---
## Privilege escalation -- SeBackupPrivilege

Used molly.smith creds to open cmd prompt as admin:

![Admin prompt](images/hokkaido-pg_35.png)

Even though SeBackupPrivilege shows as "disabled" it still works. Could **not** navigate to Administrator user directory directly.

![Robocopy flag](images/hokkaido-pg_36.png)

Copied to kali:

![Transfer to kali](images/hokkaido-pg_37.png)

![Flag](images/hokkaido-pg_38.png)

---
## Lessons & takeaways

- SMB spider_plus is great for quickly finding interesting files across shares
- BloodHound results change depending on which creds you collect with -- always re-run with new users
- SeBackupPrivilege works even when shown as "disabled" -- use it to copy protected files
- Credential chaining through multiple users is common in AD environments
