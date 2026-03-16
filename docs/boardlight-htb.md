# BoardLight -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** BoardLight (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-07-22

---

## TL;DR

### Subdomain enumeration found a CRM app. Hardcoded creds in PHP config led to SSH access. Privesc via Enlightenment desktop environment SUID exploit (CVE-2022-37706).
---

## Target info

- Host: see nmap results
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/boardlight-htb_1.png)

![Web page](images/boardlight-htb_2.png)

Added discovered hostname to `/etc/hosts`:

![Subdomain found](images/boardlight-htb_3.png)

Added subdomain to `/etc/hosts`:

![CRM app](images/boardlight-htb_4.png)

![CRM login](images/boardlight-htb_5.png)

![Enumeration](images/boardlight-htb_6.png)

![Port 33060](images/boardlight-htb_7.png)

## Exploitation

Found hardcoded credentials in PHP config:

![Hardcoded creds](images/boardlight-htb_8.png)

- `serverfun2$2023!!` -- works for user `larissa`

![CRM access](images/boardlight-htb_9.png)

Also works for SSH:

![SSH as larissa](images/boardlight-htb_10.png)

## Privilege escalation

![User enumeration](images/boardlight-htb_11.png)

- Desktop, Downloads, Pictures suggest GUI desktop environment installed

![Enlightenment found](images/boardlight-htb_12.png)

Found Enlightenment desktop manager -- CVE-2022-37706 LPE exploit:

<https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit>

![Exploit](images/boardlight-htb_13.png)

![Root shell](images/boardlight-htb_14.png)

---

## Lessons & takeaways

- Search for hardcoded creds in `/var/www/html`: `grep -rEi 'password\s*=|db_pass|api_key|auth_token' /var/www/html --include="*.php"`
- Desktop directories (Desktop, Downloads, Pictures) hint at a GUI environment -- look for related exploits
- Always try credential reuse across services (web creds -> SSH)
---
