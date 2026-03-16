# Cockpit -- Proving Grounds (write-up)

**Difficulty:** Easy / Beginner
**Box:** Cockpit (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-05-12

---

## TL;DR

### Cockpit web interface on port 9090. Logged in with PG-provided creds. Straightforward escalation from there.
---
## Target info

- Host: Proving Grounds target
- Services discovered via nmap, Cockpit on port 9090
---
## Enumeration

![Nmap results](images/cockpit-pg_1.png)

Found Cockpit web interface on port 9090. Logged in with credentials provided by PG:

![Cockpit login](images/cockpit-pg_2.png)

---
## Exploitation

![Cockpit dashboard](images/cockpit-pg_3.png)

![Shell access](images/cockpit-pg_4.png)

![Root](images/cockpit-pg_5.png)

---
## Lessons & takeaways

- Cockpit provides a built-in terminal -- if you have valid creds, you have a shell
- Always check for web management interfaces on non-standard ports
---
