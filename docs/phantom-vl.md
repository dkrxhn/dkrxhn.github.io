# Phantom -- Vulnlab (write-up)

**Difficulty:** Hard
**Box:** Phantom (Vulnlab)
**Author:** dkrxhn
**Date:** 2025-09-22

---

## TL;DR

### Base64-decoded creds from enumeration. Cracked VeraCrypt backup with custom wordlist. KeePass dump -> lateral movement to svc_sspr. ICT Security group has AddAllowedToAct on DC.
---
## Target info

- Domain: Vulnlab chain
---
## Enumeration

![Nmap results](images/phantom-vl_1.png)

![Nmap continued](images/phantom-vl_2.png)

![Enum](images/phantom-vl_3.png)

![Enum continued](images/phantom-vl_4.png)

---
## Initial access

Copied base64 output from enumeration, decoded in CyberChef:

![CyberChef decode](images/phantom-vl_5.png)

Found password: `Ph4nt0m@5t4rt!`

![nxc spray](images/phantom-vl_6.png)

Valid: `ibryant:Ph4nt0m@5t4rt!`

![Shell](images/phantom-vl_7.png)

![Enum](images/phantom-vl_8.png)

---
## VeraCrypt backup

Generated a custom wordlist based on the company name pattern (hint from Vulnlab wiki):

```bash
crunch 12 12 -t 'Phantom202%^' -o wordlist.txt
```

Cracked the VeraCrypt container:

```bash
hashcat -m 13721 IT_BACKUP_201123.hc ./wordlist.txt
```

![hashcat](images/phantom-vl_9.png)

Password: `Phantom2023!`

![Mounted](images/phantom-vl_10.png)

---
## Lateral movement

![KeePass](images/phantom-vl_11.png)

![KeePass dump](images/phantom-vl_12.png)

Found: `lstanley:gB6XTcqVP5MlP7Rc`

![BloodHound](images/phantom-vl_13.png)

![Enum](images/phantom-vl_14.png)

Password reuse: `svc_sspr:gB6XTcqVP5MlP7Rc`

![svc_sspr](images/phantom-vl_15.png)

---
## Privilege escalation

All 3 users are members of ICT Security, which has AddAllowedToAct on the DC:

![ICT Security](images/phantom-vl_16.png)

---
## Lessons & takeaways

- Custom wordlists with crunch based on company naming patterns are effective for VeraCrypt
- Always check for password reuse across service accounts
- AddAllowedToAct (RBCD) on a DC from a group membership = domain compromise
---
