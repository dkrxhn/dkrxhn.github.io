# PC -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** PC (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-09-30

---

## TL;DR

### Enumeration of running processes revealed attack surface. Privesc via SUID on bash.
---

## Target info

- Host: PC (Proving Grounds)

---

## Enumeration

![Nmap results](images/pc-pg_1.png)

Checked running processes:

```bash
ps -auxww
```

![Processes](images/pc-pg_2.png)

![Service enum](images/pc-pg_3.png)

![Further enum](images/pc-pg_4.png)

---

## Foothold

![Dead end](images/pc-pg_5.png)

**nada** -- moved on.

![Alternate approach](images/pc-pg_6.png)

![Shell access](images/pc-pg_7.png)

---

## Privilege escalation

![SUID exploit](images/pc-pg_8.png)

![Root shell](images/pc-pg_9.png)

Set SUID on bash and executed:

```bash
/bin/bash -p
```

Only needed `u+s /bin/bash` as the payload.

---

## Lessons & takeaways

- Always check running processes with `ps -auxww` for hidden services
- SUID on `/bin/bash` is a quick path to root via `bash -p`
---
