# ZenPhoto -- Proving Grounds (write-up)

**Difficulty:** Easy
**Box:** ZenPhoto (Proving Grounds)
**Author:** dsec
**Date:** 2024-07-17

---

## TL;DR

### Exploited ZenPhoto with a known RCE exploit to get a shell, ran linpeas, and escalated to root via Dirty COW.

---

## Enumeration

![Nmap results](images/zenphoto-pg_1.png)

![Web enum](images/zenphoto-pg_2.png)

![More enum](images/zenphoto-pg_3.png)

![ZenPhoto](images/zenphoto-pg_4.png)

![Exploit search](images/zenphoto-pg_5.png)

![Exploit details](images/zenphoto-pg_6.png)

---

## Exploitation

```bash
php 18083.php 192.168.220.41 /test/
```

![RCE](images/zenphoto-pg_7.png)

Pivoted to a better shell.

---

## Privilege escalation

Uploaded and ran `linpeas.sh`:

![linpeas output 1](images/zenphoto-pg_8.png)

![linpeas output 2](images/zenphoto-pg_9.png)

![Kernel info](images/zenphoto-pg_10.png)

- Ran Dirty COW.
- Shell died and SSH **did not** work, but was able to connect back in and switch to the `firefart` user.

![Root as firefart](images/zenphoto-pg_11.png)

---

## Lessons & takeaways

- Dirty COW can crash your shell -- have a backup connection method ready
- Old applications like ZenPhoto often have public RCE exploits that work out of the box
- When a kernel exploit kills your shell, reconnect through the original exploit path
