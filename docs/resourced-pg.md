# Resourced -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Resourced (Proving Grounds)
**Author:** dsec
**Date:** 2024-12-17

---

## TL;DR

### Found creds for v.ventz via enumeration. BloodHound showed a path through l.livingstone who had RBCD rights, performed a DCSync attack to get the Administrator hash and used a Kerberos ticket for access.
---
## Target info

- Host: discovered via nmap
- Domain: `resourced.local`
---
## Enumeration

![Nmap results](images/resourced-pg_1.png)

![Nmap continued](images/resourced-pg_2.png)

![SMB enumeration](images/resourced-pg_3.png)

![Creds found](images/resourced-pg_4.png)

Found: `v.ventz:HotelCalifornia194!`

![Access confirmed](images/resourced-pg_5.png)

![Further enumeration](images/resourced-pg_6.png)

---
## BloodHound

```bash
bloodhound-ce-python
```

![BloodHound path](images/resourced-pg_7.png)

Need `l.livingstone` to remote in.

![l.livingstone enumeration](images/resourced-pg_8.png)

![Hash extraction](images/resourced-pg_9.png)

`l.livingstone:19a3a7550ce8c505c2d46b5e39d6f808`

![Access as l.livingstone](images/resourced-pg_10.png)

---
## DCSync attack

![DCSync setup](images/resourced-pg_11.png)

![DCSync continued](images/resourced-pg_12.png)

![DCSync continued](images/resourced-pg_13.png)

![DCSync continued](images/resourced-pg_14.png)

![DCSync continued](images/resourced-pg_15.png)

![DCSync continued](images/resourced-pg_16.png)

![Administrator hash](images/resourced-pg_17.png)

Got the Administrator hash. Saved as `ticket.kirbi.b64`:

![Ticket saved](images/resourced-pg_18.png)

Added `resourced.local` and `resourcedc.resourced.local` to `/etc/hosts` (make sure you can ping `resourcedc.resourced.local` to confirm).

![Root access](images/resourced-pg_19.png)

---
## Lessons & takeaways

- BloodHound CE Python collector works well for initial AD mapping
- RBCD attack path: compromise a user with write access to a machine account, then abuse delegation
- Always add both domain and hostname to `/etc/hosts` and verify with ping before Kerberos attacks
