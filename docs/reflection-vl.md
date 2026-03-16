# Reflection -- Vulnlab (write-up)

**Difficulty:** Hard
**Box:** Reflection (Vulnlab)
**Author:** dsec
**Date:** 2025-10-12

---

## TL;DR

### Multi-machine chain. MSSQL creds from SMB, NTLM relay via dirtree to get more creds, BloodHound showed GenericAll over machines -> LAPS password -> DPAPI loot -> RBCD to final workstation.
---
## Target info

- Hosts: DC01 (`10.10.245.101`), MS01 (`10.10.245.102`), WS01
- Domain: `reflection.vl`
---
## Enumeration

![Nmap DC01](images/reflection-vl_1.png)

DC01:

![DC01 services](images/reflection-vl_2.png)

![DC01 enum](images/reflection-vl_3.png)

![SMB signing](images/reflection-vl_4.png)

SMB signing not required.

MS01:

![MS01 nmap](images/reflection-vl_5.png)

![MS01 enum](images/reflection-vl_6.png)

![MS01 shares](images/reflection-vl_7.png)

![MSSQL](images/reflection-vl_8.png)

![MSSQL enum](images/reflection-vl_9.png)

---
## Initial access

Found MSSQL creds:

![MSSQL creds](images/reflection-vl_10.png)

`web_staging:Washroom510`

![Access](images/reflection-vl_11.png)

![Enum](images/reflection-vl_12.png)

![More enum](images/reflection-vl_13.png)

---
## NTLM relay

Set up Responder and ntlmrelayx, then used dirtree command on MSSQL to trigger authentication:

![Responder setup](images/reflection-vl_14.png)

![Relay](images/reflection-vl_15.png)

![dirtree](images/reflection-vl_16.png)

ntlmrelayx opened an interactive SMB client shell, connected via nc:

Found more creds: `web_prod:Tribesman201`

![web_prod](images/reflection-vl_17.png)

![MSSQL access](images/reflection-vl_18.png)

Dumped from database:

- `abbie.smith:CMe1x+nlRaaWEw`
- `dorothy.rose:hC_fny3OK9glSJ`

---
## Lateral movement

BloodHound showed abbie.smith has GenericAll over MS01, and GPO LAPS is in use:

![BloodHound GenericAll](images/reflection-vl_19.png)

![LAPS GPO](images/reflection-vl_20.png)

Read LAPS password:

![LAPS password](images/reflection-vl_21.png)

Got: `H447.++h6g5}xi`

![Admin access](images/reflection-vl_22.png)

![Enum MS01](images/reflection-vl_23.png)

Found: `svc_web_staging:DivinelyPacifism98`

Used DPAPI with local admin on MS01:

```bash
netexec smb 10.10.245.246 -u administrator -p 'H447.++h6g5}xi' --dpapi --local-auth
```

![DPAPI loot](images/reflection-vl_24.png)

Found: `Georgia.Price:DBl+5MPkpJg5id`

---
## Privilege escalation

![Georgia.Price](images/reflection-vl_25.png)

Georgia.Price has GenericAll over WS01. Since we control `svc_web_staging` (which has an SPN), used RBCD:

```bash
rbcd.py -delegate-from 'svc_web_staging' -delegate-to 'WS01$' -action 'write' 'reflection.vl/Georgia.Price:DBl+5MPkpJg5id'
```

![RBCD setup](images/reflection-vl_26.png)

```bash
getST.py -spn 'cifs/WS01.reflection.vl' -impersonate 'dom_rgarner' 'reflection.vl/svc_web_staging:DivinelyPacifism98'
```

![Silver ticket](images/reflection-vl_27.png)

![WS01 access](images/reflection-vl_28.png)

Found: `Rhys.Garner:knh1gJ8Xmeq+uP`

![Root flag](images/reflection-vl_29.png)

---
## Lessons & takeaways

- NTLM relay via MSSQL dirtree is powerful when SMB signing is disabled
- DPAPI looting with `netexec --dpapi --local-auth` extracts Credential Manager secrets without touching LSASS
- RBCD attack requires: GenericAll over target machine + control of an account with an SPN
- Chain: MSSQL creds -> relay -> LAPS -> DPAPI -> RBCD is a clean multi-hop path
---
