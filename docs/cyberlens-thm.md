# CyberLens -- TryHackMe (write-up)

**Difficulty:** Easy
**Box:** CyberLens (TryHackMe)
**Author:** dsec
**Date:** 2025-10-07

---

## TL;DR

### Apache Tika on port 61777 vulnerable to CVE-2018-1335. Base64-encoded PowerShell reverse shell for initial access. AlwaysInstallElevated for SYSTEM.
---
## Target info

- Host: `cyberlens.thm`
- Services discovered: `80/tcp (http)`, `61777/tcp (Apache Tika)`
---
## Enumeration

![Nmap results](images/cyberlens-thm_1.png)

![Web page](images/cyberlens-thm_2.png)

![More enum](images/cyberlens-thm_3.png)

Added `cyberlens.thm` to `/etc/hosts`.

![Jetty](images/cyberlens-thm_4.png)

Identified Jetty 8.y.z-SNAPSHOT.

Port 61777:

![Tika](images/cyberlens-thm_5.png)

![Tika version](images/cyberlens-thm_6.png)

---
## Initial access

Used [CVE-2018-1335](https://github.com/RhinoSecurityLabs/CVEs/blob/master/CVE-2018-1335/CVE-2018-1335.py) (updated for Python3). Also works with Python2.

![CVE exploit](images/cyberlens-thm_7.png)

Used PowerShell base64-encoded reverse shell from revshells.com.

---
## Privilege escalation

WinPEAS found AlwaysInstallElevated:

![AlwaysInstallElevated](images/cyberlens-thm_8.png)

![SYSTEM](images/cyberlens-thm_9.png)

---
## Lessons & takeaways

- Apache Tika instances exposed on non-standard ports are often vulnerable to CVE-2018-1335
- AlwaysInstallElevated = instant SYSTEM via malicious MSI
- Base64-encoded PowerShell payloads from revshells.com are reliable for Windows targets
---
