# Sweep -- Vulnlab (write-up)

**Difficulty:** Medium
**Box:** Sweep (Vulnlab)
**Author:** dsec
**Date:** 2025-08-28

---

## TL;DR

### Lansweeper on port 81 with default creds. Abused scanning credentials feature with fakessh to capture SSH creds for a high-value service account. Privesc through Lansweeper admin.
---
## Target info

- Domain: Vulnlab chain
---
## Enumeration

![Nmap results](images/sweep-vl_1.png)

![Nmap continued](images/sweep-vl_2.png)

![Web enum](images/sweep-vl_3.png)

![More enum](images/sweep-vl_4.png)

![Shares](images/sweep-vl_5.png)

![Enum continued](images/sweep-vl_6.png)

![Enum continued](images/sweep-vl_7.png)

![Enum continued](images/sweep-vl_8.png)

---
## Initial access

Logged into Lansweeper on port 81 with `intern:intern`:

![Lansweeper login](images/sweep-vl_9.png)

The scanning credentials section showed `svc_inventory_lnx` creds are used via SSH.

Used [fakessh](https://github.com/fffaraz/fakessh) to capture credentials. Added my box as a scanning target, mapped the SSH credential to that IP range, then ran a quick scan.

BloodHound showed the service account is a high-value target:

![BloodHound](images/sweep-vl_10.png)

Added IP to scanning targets, mapped credential to IP range, triggered quick scan:

![Captured creds](images/sweep-vl_11.png)

Captured: `svc_inventory_lnx:'0|5m-U6?/uAX'`

![Shell](images/sweep-vl_12.png)

---
## Privilege escalation

![Privesc enum](images/sweep-vl_13.png)

**SeMachineAccountPriv didn't work.**

Privesc was through Lansweeper admin after logging in with the new creds.

---
## Lessons & takeaways

- Lansweeper stores scanning credentials that can be intercepted with a fake service (fakessh)
- Default/weak creds on management interfaces open the door to credential harvesting
- BloodHound helps identify which service accounts are worth targeting
---
