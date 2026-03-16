# Silver Platter -- TryHackMe (write-up)

**Difficulty:** Easy
**Box:** Silver Platter (TryHackMe)
**Author:** dkrxhn
**Date:** 2025-08-20

---

## TL;DR

### Generated password list with cewl, logged into Silverpeas. IDOR on notification IDs leaked SSH creds. Pivoted to tyler via log grep (adm group). Tyler had sudo root.
---

## Target info

- Host: `10.10.21.22`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`, `8080/tcp (http-proxy)`

---

## Enumeration

![Nmap results](images/silver-platter-thm_1.png)

![Web page](images/silver-platter-thm_2.png)

Quick scan:

![Quick scan](images/silver-platter-thm_3.png)

![More enumeration](images/silver-platter-thm_4.png)

![Service enum](images/silver-platter-thm_5.png)

Fuzzed port 8080:

![Fuzz 8080](images/silver-platter-thm_6.png)

Guessed `/silverpeas`:

![Silverpeas login](images/silver-platter-thm_7.png)

Generated password list from the website with cewl (rockyou didn't work):

```bash
cewl 10.10.21.22 > passwords.txt
```

![cewl results](images/silver-platter-thm_8.png)

Top password worked:

![Login success](images/silver-platter-thm_9.png)

## Exploitation

Found an IDOR -- changed notification ID=5 to ID=6:

![IDOR](images/silver-platter-thm_10.png)

- `tim:cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol`

![SSH as tim](images/silver-platter-thm_11.png)

## Lateral movement

Ran LinEnum:

![LinEnum](images/silver-platter-thm_12.png)

![Groups](images/silver-platter-thm_13.png)

- `adm` group means can read logs in `/var/log`

Searched logs for tyler:

```bash
cd /var/log && grep -iR tyler
```

![Grep results](images/silver-platter-thm_14.png)

![Password found](images/silver-platter-thm_15.png)

- Password: `_Zd_zx7N823/`

## Privilege escalation

![su tyler](images/silver-platter-thm_16.png)

![Root shell](images/silver-platter-thm_17.png)

---

## Lessons & takeaways

- Use cewl to generate passwords from the target website when rockyou fails
- Watch for IDORs when reading messages/notifications -- increment the ID
- `adm` group (`id` command) allows reading `/var/log` -- grep for usernames to find creds
- Fuzz non-standard ports (8080) for hidden directories
---
