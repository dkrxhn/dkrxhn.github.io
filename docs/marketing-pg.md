# Marketing -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Marketing (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-01-17

---

## TL;DR

### Subdomain and directory enumeration led to a Limesurvey admin panel. RCE exploit gave foothold. Privesc involved mlocate group membership, symlink tricks, and sudo ALL.

---

## Target info

- Host: `marketing.pg`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/marketing-pg_1.png)

Added host to `/etc/hosts`. Ran feroxbuster, found `/old` directory. Source code revealed:

![Source code](images/marketing-pg_2.png)

Found subdomain `customers-survey.marketing.pg`:

![Subdomain discovery](images/marketing-pg_3.png)

- Found email: `admin@marketing.pg`

![Admin panel](images/marketing-pg_4.png)

Found `/admin` path.

---

## Foothold

![Admin login](images/marketing-pg_5.png)

![Limesurvey admin](images/marketing-pg_6.png)

![Limesurvey exploration](images/marketing-pg_7.png)

![Exploit attempt](images/marketing-pg_8.png)

![Failed attempt](images/marketing-pg_9.png)

**First exploit attempt failed.**

![Further attempts](images/marketing-pg_10.png)

Used Limesurvey RCE exploit from: `https://github.com/Y1LD1R1M-1337/Limesurvey-RCE/tree/main`

![Shell obtained](images/marketing-pg_11.png)

---

## Privilege escalation

User was in the `mlocate` group. Needed to use symlinks against entries in the `mlocate.db` file to discover a credential file in `m.sanders` home directory. Symlink comparison revealed creds for `m.sanders`.

After pivoting to `m.sanders`, had `sudo ALL` -- straight to root.

---

## Lessons & takeaways

- The `mlocate` group grants access to the locate database, which can reveal sensitive file paths
- Symlink tricks can be used to read files you normally cannot access
- Always check group memberships after landing a shell
---
