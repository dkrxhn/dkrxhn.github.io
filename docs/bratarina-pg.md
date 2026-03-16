# Bratarina -- Proving Grounds (write-up)

**Difficulty:** Easy
**Box:** Bratarina (Proving Grounds)
**Author:** dsec
**Date:** 2025-09-11

---

## TL;DR

### SMTP service exploit gave initial shell. Straightforward foothold with a known vulnerability.
---

## Target info

- Host: `192.168.165.71`
- Services discovered via nmap

---

## Enumeration

![Nmap results](images/bratarina-pg_1.png)

![Service details](images/bratarina-pg_2.png)

Directory brute force:

```bash
feroxbuster -u http://192.168.165.71 -w /usr/share/wordlists/dirb/common.txt -n
```

![Feroxbuster results](images/bratarina-pg_3.png)

---

## Foothold

![Exploit research](images/bratarina-pg_4.png)

![Exploit execution](images/bratarina-pg_5.png)

![Exploit success](images/bratarina-pg_6.png)

Stabilized the shell. **Note:** the first PTY stabilization command did not work (had a space between `;` and `import pty`), but the second command from revshells worked:

![Shell stabilization](images/bratarina-pg_7.png)

![Proof](images/bratarina-pg_8.png)

---

## Lessons & takeaways

- Watch for syntax errors in shell stabilization commands -- spacing matters
- Always check SMTP services for known exploits
---
