# Postfish -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Postfish (Proving Grounds)
**Author:** dsec
**Date:** 2025-07-28

---

## TL;DR

### Enumerated users/services, SSH'd in with discovered creds. Privesc via sudo 1.8.31 root exploit.
---

## Target info

- Host: see nmap results

---

## Enumeration

![Nmap results](images/postfish-pg_1.png)

![Nmap continued](images/postfish-pg_2.png)

![Enumeration](images/postfish-pg_3.png)

![More enumeration](images/postfish-pg_4.png)

## Exploitation

SSH'd in with discovered creds:

![SSH login](images/postfish-pg_5.png)

## Privilege escalation

Ran linpeas:

![linpeas output](images/postfish-pg_6.png)

Found vulnerable sudo version. Used sudo 1.8.31 root exploit:

<https://github.com/mohinparamasivam/Sudo-1.8.31-Root-Exploit>

![Root shell](images/postfish-pg_7.png)

---

## Lessons & takeaways

- Always check sudo version -- 1.8.31 has a known root exploit
- Linpeas highlights vulnerable sudo versions automatically
---
