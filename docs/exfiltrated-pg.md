# Exfiltrated -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Exfiltrated (Proving Grounds)
**Author:** dsec
**Date:** 2025-11-16

---

## TL;DR

### Default admin:admin on Subrion CMS. Privesc via CVE-2021-22204 (exiftool) -- crafted malicious image uploaded to a cron-watched directory.
---

## Target info

- Host: `exfiltrated.offsec` (added to `/etc/hosts`)

---

## Foothold

`admin:admin` worked for login on `http://exfiltrated.offsec`:

![Admin login](images/exfiltrated-pg_1.png)

---

## Privilege escalation

Linpeas findings:

![Linpeas](images/exfiltrated-pg_2.png)

![Cron job](images/exfiltrated-pg_3.png)

![Uploads directory](images/exfiltrated-pg_4.png)

A cron job checks the `/var/www/html/subrion/uploads` directory:

![Cron detail](images/exfiltrated-pg_5.png)

CVE-2021-22204 (exiftool RCE):

- `https://github.com/UNICORDev/exploit-CVE-2021-22204`
- Asks to install a dependency
- Works by running locally to create `image.jpg`, setting up a listener, then uploading `image.jpg` to `/var/www/html/subrion/uploads`

**Note:** the exploit-db version gives errors -- the GitHub version works.

---

## Lessons & takeaways

- Default credentials on CMS platforms are always worth trying
- Cron jobs that process uploaded files are prime targets for exploitation
- CVE-2021-22204 (exiftool) can be triggered by simply placing a malicious image where it will be processed
---
