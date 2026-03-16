# Stapler -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Stapler (Proving Grounds)
**Author:** dsec
**Date:** 2025-12-14

---

## TL;DR

### FTP anonymous access leaked passwd file. Extracted usernames for brute force. WordPress exploitation for shell. Kernel exploit (39772) for root.
---

## Target info

- Host: Stapler (Proving Grounds)

---

## Enumeration

![Nmap results](images/stapler-pg_1.png)

![Service details](images/stapler-pg_2.png)

FTP anonymous login:

![FTP access](images/stapler-pg_3.png)

![FTP files](images/stapler-pg_4.png)

Got the passwd file and extracted usernames:

```bash
cat passwd | awk -F: '{print $1}' > usernames.txt
```

![Usernames](images/stapler-pg_5.png)

---

## Foothold

![Web enum](images/stapler-pg_6.png)

![WordPress](images/stapler-pg_7.png)

![WP exploitation](images/stapler-pg_8.png)

![Shell access](images/stapler-pg_9.png)

---

## Privilege escalation

Found kernel exploit 39772:

![Exploit info](images/stapler-pg_10.png)

![Compilation](images/stapler-pg_11.png)

![Root](images/stapler-pg_12.png)

---

## Lessons & takeaways

- Anonymous FTP access can leak critical files like `/etc/passwd`
- Extract usernames from passwd files for targeted brute force
- Linux kernel exploits (39772) are reliable for older kernels
---
