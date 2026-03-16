# Sunday -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Sunday (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-10-27

---

## TL;DR

### Finger enumeration found users. SSH brute force on non-standard port. Cracked shadow hash. Sudo ALL for root.
---

## Target info

- Host: Sunday (HackTheBox)
- Services discovered: `79/tcp (finger)`, `6787/tcp (ssh)`

---

## Enumeration

![Nmap results](images/sunday-htb_1.png)

Port 79 -- finger:

![Finger enum](images/sunday-htb_2.png)

Port 6787:

![Port 6787](images/sunday-htb_3.png)

Found users: `sammy` & `sunny`

![User enumeration](images/sunday-htb_4.png)

---

## Foothold

![Brute force](images/sunday-htb_5.png)

`sunny:Sunday` worked on port 6787 SSH login.

![Sunny shell](images/sunday-htb_6.png)

Found shadow file backup:

![Shadow file](images/sunday-htb_7.png)

![Shadow hashes](images/sunday-htb_8.png)

```
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::
```

Cracked sammy's hash:

![Cracked hash](images/sunday-htb_9.png)

- `sammy:cooldude!`

---

## Privilege escalation

```bash
sudo -l
```

![Sudo permissions](images/sunday-htb_10.png)

`(ALL) ALL` -- full sudo access as sammy.

Alternate route with wget:

![Wget trick](images/sunday-htb_11.png)

- `-I` (input-file) flag -- error leaks file data (in this case the flag)
- Could also use wget to overwrite troll script with a shell, but the script resets every 5 seconds

---

## Lessons & takeaways

- Finger (port 79) is a goldmine for user enumeration
- Always check non-standard SSH ports
- Shadow file backups are a common find for hash cracking
- `sudo (ALL) ALL` is instant root
---
