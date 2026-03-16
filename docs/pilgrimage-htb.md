# Pilgrimage -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Pilgrimage (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-12-07

---

## TL;DR

### Exposed .git repo revealed ImageMagick usage. CVE-2022-44268 to read files via crafted PNG. Extracted SQLite DB with credentials. Privesc via Binwalk path traversal (CVE-2022-4510) to plant SSH keys.
---

## Target info

- Host: `pilgrimage.htb` (added to `/etc/hosts`)
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/pilgrimage-htb_1.png)

Port 80 redirects to `pilgrimage.htb`. Rescanned:

```bash
sudo nmap -p 22,80 -sCV pilgrimage.htb -vvv
```

![Rescan results](images/pilgrimage-htb_2.png)

Git repository found. Dumped it:

![Git dump](images/pilgrimage-htb_3.png)

Used `git checkout .` to restore files.

![Repo contents](images/pilgrimage-htb_4.png)

Found `magick` binary. Reviewed source:

![Index.php source](images/pilgrimage-htb_5.png)

POST requests with images go to `index.php` -- saves file in `/tmp`:

![File handling](images/pilgrimage-htb_6.png)

Creates a new filename with `uniqid()`:

![Uniqid](images/pilgrimage-htb_7.png)

Runs magick to shrink image by 50% and deletes original:

![Magick convert](images/pilgrimage-htb_8.png)

If logged in, saves path to DB:

![DB save](images/pilgrimage-htb_9.png)

Copy of magick in repo is executable:

![Magick binary](images/pilgrimage-htb_10.png)

ImageMagick 7.1.0-49 beta:

![Version](images/pilgrimage-htb_11.png)

CVE-2022-44268 -- crafted PNG with `tEXt` chunk keyword "profile" makes ImageMagick read arbitrary files.

---

## Foothold

Used exploit from `https://github.com/kljunowsky/CVE-2022-44268`:

![Exploit](images/pilgrimage-htb_12.png)

Created malicious `pngout.png` and uploaded to pilgrimage.htb:

![Upload](images/pilgrimage-htb_13.png)

Found user `emily`. No SSH keys found.

![Emily user](images/pilgrimage-htb_14.png)

Found SQLite database filepath. Binary data caused the ASCII-expecting script to error, so extracted manually:

![DB extraction](images/pilgrimage-htb_15.png)

```bash
identify -verbose 66ddff730a36c.png | grep -Pv "^( |Image)" | xxd -r -p > pilgrimage.sqlite
```

![SQLite contents](images/pilgrimage-htb_16.png)

- `emily:abigchonkyboi123`

![SSH as emily](images/pilgrimage-htb_17.png)

---

## Privilege escalation

![Process enum](images/pilgrimage-htb_18.png)

![Malwarescan script](images/pilgrimage-htb_19.png)

**Dirty Pipe didn't work.**

```bash
ps aux
```

![Root processes](images/pilgrimage-htb_20.png)

Root is running `/usr/sbin/malwarescan.sh` with `inotifywait` watching for file creations in `/var/www/pilgrimage.htb/shrunk`.

![Script contents](images/pilgrimage-htb_21.png)

Uses binwalk to check files for executables:

![Binwalk usage](images/pilgrimage-htb_22.png)

Binwalk CVE-2022-4510 -- `os.path.join` doesn't resolve `../` sequences, allowing files to be placed in unintended locations (like SSH keys).

![Binwalk exploit](images/pilgrimage-htb_23.png)

Generated SSH keys and crafted malicious PNG:

```bash
ssh-keygen -t ed25519 -f ./id_ed25519
python walkingpath.py ssh sample.png ./id_ed25519.pub
```

- `https://github.com/adhikara13/CVE-2022-4510-WalkingPath`

Uploaded the exploit PNG:

```bash
scp ./binwalk_exploit.png emily@pilgrimage.htb:/var/www/pilgrimage.htb/shrunk/
```

SSH'd as root:

```bash
ssh -i id_ed25519 root@pilgrimage.htb
```

---

## Lessons & takeaways

- Exposed `.git` directories reveal source code and binary versions
- ImageMagick CVE-2022-44268 reads arbitrary files via crafted PNG metadata
- Binary file extraction from image metadata requires manual hex conversion
- Binwalk path traversal (CVE-2022-4510) can plant SSH keys when a root process runs binwalk on user-controlled files
---
