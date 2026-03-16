# Nibbles -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Nibbles (HackTheBox)
**Author:** dkrxhn
**Date:** 2024-10-25

---

## TL;DR

### Found Nibbleblog CMS, guessed the password (`nibbles`), uploaded a PHP reverse shell via the My Image plugin, then escalated via a writable script that could be run with sudo.
---
## Target info

- Host: `10.129.35.101`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`
---
## Enumeration

Nmap:

![Nmap results](images/nibbles-htb_1.png)

Port 80:

![Port 80](images/nibbles-htb_2.png)

Found `/nibbleblog`:

![Nibbleblog](images/nibbles-htb_3.png)

Ran feroxbuster -- lots of results, switched to gobuster:

![Feroxbuster](images/nibbles-htb_4.png)

![Gobuster](images/nibbles-htb_5.png)

---
## Credential discovery

Found `/nibbleblog/content/private/users.xml`:

![users.xml](images/nibbles-htb_6.png)

Username: `admin`. Guessed the password as the box name: `nibbles`.

Logged in at `/nibbleblog/admin.php`:

![Admin login](images/nibbles-htb_7.png)

Settings page showed version:

![Version 4.0.3](images/nibbles-htb_8.png)

Nibbleblog 4.0.3.

---
## PHP shell upload

Uploaded pentestmonkey PHP shell to the My Image plugin.

Triggered at:

```
http://10.129.35.101/nibbleblog/content/private/plugins/my_image/image.php
```

![Reverse shell](images/nibbles-htb_9.png)

---
## Privilege escalation

Found `personal.zip`:

```bash
unzip personal.zip
```

![personal.zip contents](images/nibbles-htb_10.png)

![Root shell](images/nibbles-htb_11.png)

---
## Lessons & takeaways

- Guessing passwords based on the box/application name is worth trying early
- Nibbleblog My Image plugin allows arbitrary file upload -- classic attack vector
- Always check for writable scripts that run with elevated privileges
