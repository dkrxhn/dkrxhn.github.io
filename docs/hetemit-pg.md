# Hetemit -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Hetemit (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-05-29

---

## TL;DR

### Gained initial access through open services. Privesc by editing a writable systemd service file and rebooting via sudo.
---

## Target info

- Host: `192.168.238.117`

---

## Enumeration

Nmap results:

![Nmap results](images/hetemit-pg_1.png)

![Nmap results continued](images/hetemit-pg_2.png)

![Web enumeration](images/hetemit-pg_3.png)

![More enumeration](images/hetemit-pg_4.png)

Ran enum4linux:

```bash
enum4linux -a -u "" -p '' 192.168.238.117
```

![enum4linux results](images/hetemit-pg_5.png)

## Exploitation

![Initial access](images/hetemit-pg_6.png)

![Shell](images/hetemit-pg_7.png)

![Enumeration on target](images/hetemit-pg_8.png)

![Troubleshooting](images/hetemit-pg_9.png)

![More troubleshooting](images/hetemit-pg_10.png)

## Privilege escalation

Checked sudo permissions:

```bash
sudo -l
```

![sudo -l](images/hetemit-pg_12.png)

Ran linpeas:

![linpeas](images/hetemit-pg_13.png)

Found writable systemd service:

![Writable service](images/hetemit-pg_14.png)

`/etc/systemd/system/pythonapp.service` is writable and we have `sudo -l` permissions for `/sbin/reboot`. Edited the service to include a reverse shell:

```bash
vi /etc/systemd/system/pythonapp.service
```

- Changed `ExecStart` to a reverse shell and `User` to root.

Then rebooted:

```bash
sudo /sbin/reboot
```

Caught root shell on listener.

---

## Lessons & takeaways

- Always check for writable systemd service files with linpeas
- If you can sudo reboot, writable service files are an easy privesc path
- Edit `ExecStart` and `User` fields in the service file to escalate
---
