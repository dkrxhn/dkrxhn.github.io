# Quackerjack -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Quackerjack (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-05-01

---

## TL;DR

### Exploited a service on port 48208 (had to fix SSL errors in the exploit code). Got admin creds, escalated from there.
---
## Target info

- Host: Proving Grounds target
- Services discovered via nmap
---
## Enumeration

![Nmap results](images/quackerjack-pg_1.png)

![Web enumeration](images/quackerjack-pg_2.png)

![Service enumeration](images/quackerjack-pg_3.png)

---
## Initial foothold

Found exploit for port 48208. Had to fix an SSL error in the exploit code -- newer Python versions require SSL verification by default.

![Exploit fix](images/quackerjack-pg_4.png)

![Exploit execution](images/quackerjack-pg_5.png)

Got creds: `admin:abgrtyu`

---
## Privesc

![Post-exploitation](images/quackerjack-pg_6.png)

![Root](images/quackerjack-pg_7.png)

---
## Lessons & takeaways

- Public exploits may need fixes for newer Python versions (especially SSL verification changes)
- Always check high-numbered ports -- services running on non-standard ports can be vulnerable
---
