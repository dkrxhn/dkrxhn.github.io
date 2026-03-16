# Apex -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Apex (Proving Grounds)
**Author:** dkrxhn
**Date:** 2024-06-24

---

## TL;DR

### Exploited OpenEMR via an authenticated file read vulnerability to extract MySQL credentials from sqlconf.php, cracked the admin password hash from the database, and escalated via password reuse to root.

---

## Enumeration

![Nmap results](images/apex-pg_1.png)

![Web enum](images/apex-pg_2.png)

Found OpenEMR:

![OpenEMR Features PDF](images/apex-pg_3.png)

Searched GitHub for password handling:

```
repo:openemr/openemr "$pass"
```

![GitHub search](images/apex-pg_4.png)

```bash
feroxbuster -u http://192.168.145.145/openemr -w /usr/share/wordlists/dirb/common.txt -n -x php
```

![Feroxbuster](images/apex-pg_5.png)

![Directory listing](images/apex-pg_6.png)

![Admin panel](images/apex-pg_7.png)

- Requires auth.

![Auth required](images/apex-pg_8.png)

---

## Exploitation

![Exploit search](images/apex-pg_9.png)

![Exploit details](images/apex-pg_10.png)

![Running exploit](images/apex-pg_11.png)

- Unedited script works but **fails** to get `sqlconf`.

![sqlconf fail](images/apex-pg_12.png)

![PHPSESSID needed](images/apex-pg_13.png)

- Need PHPSESSID.

![Session cookie](images/apex-pg_14.png)

- `otgra9kr3eo0gcteikasu3ji00`

Script still fails:

![Script fail](images/apex-pg_15.png)

Had to edit the clipboard and read functions in the exploit:

![Original functions](images/apex-pg_16.png)

Changed to:

![Modified functions](images/apex-pg_17.png)

- Changed `data` variable in `paste_clipboard` function to `"path=Documents"` because this is the folder we have access to within filemanager.
- Changed `url_path` variable in `read_file` function to `%s/filemanager/Documents/%s`.
- The path for OpenEMR is under `/var/www` instead of `/var/www/html`.
- The copied `sqlconf.php` can only be viewed via the SMB share since PHP files get processed server-side.

![SMB share view](images/apex-pg_18.png)

![sqlconf.php contents](images/apex-pg_19.png)

![MySQL creds](images/apex-pg_20.png)

- `openemr:C78maEQUIEuQ`

![SSL error](images/apex-pg_21.png)

- SSL error, had to use `--ssl=0`.

```bash
mysql --ssl=0 -u openemr -p -h 192.168.145.145
```

![MySQL access](images/apex-pg_22.png)

![Database dump](images/apex-pg_23.png)

![Hash found](images/apex-pg_24.png)

- Cracked without salt.

![Cracked password](images/apex-pg_25.png)

---

## Privilege escalation

![Root via password reuse](images/apex-pg_26.png)

- Password reuse to root.

---

## Lessons & takeaways

- OpenEMR exploits often need tweaking -- the file paths and session handling vary between installations
- When PHP files are copied via exploits, view them through SMB or other non-PHP-processing means
- Always try `--ssl=0` if MySQL connections fail with SSL errors
- Password reuse between application databases and system accounts is common
