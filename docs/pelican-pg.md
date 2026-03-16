# Pelican -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Pelican (Proving Grounds)
**Author:** dsec
**Date:** 2025-05-05

---

## TL;DR

### Found a vulnerable service with a public exploit on Exploit-DB. Used sudo gcore to dump a process memory and extract credentials. Escalated to root.
---
## Target info

- Host: Proving Grounds target
- Services discovered via nmap
---
## Enumeration

![Nmap results](images/pelican-pg_1.png)

![Web enumeration](images/pelican-pg_2.png)

![Service enumeration](images/pelican-pg_3.png)

![Further recon](images/pelican-pg_4.png)

![Interesting findings](images/pelican-pg_5.png)

---
## Initial foothold

Found a public exploit: <https://www.exploit-db.com/exploits/48654>

![Exploit details](images/pelican-pg_6.png)

![Exploit execution](images/pelican-pg_7.png)

![Shell obtained](images/pelican-pg_8.png)

---
## Privesc

Used `sudo gcore` to dump process memory:

```bash
sudo gcore 486
cat core.486
```

![Core dump](images/pelican-pg_9.png)

Found password in the dump: `ClogKingpinInning731`

![Password found](images/pelican-pg_10.png)

![Privilege escalation](images/pelican-pg_11.png)

![Root](images/pelican-pg_12.png)

Make sure to stabilize the shell before running gcore.

---
## Lessons & takeaways

- `sudo gcore` can dump process memory, which often contains plaintext credentials
- Always stabilize your shell before running commands that produce large output
- Check Exploit-DB for known vulnerabilities when you identify service versions
---
