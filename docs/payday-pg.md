# Payday -- Proving Grounds (write-up)

**Difficulty:** Easy
**Box:** Payday (Proving Grounds)
**Author:** dkrxhn
**Date:** 2024-08-07

---

## TL;DR

### Default admin creds on a web app led to a PHP reverse shell. Found MySQL creds and password reuse for a local user who had full sudo privileges.

---

## Enumeration

![Nmap results](images/payday-pg_1.png)

![Web enum](images/payday-pg_2.png)

![Directory enum](images/payday-pg_3.png)

![More enum](images/payday-pg_4.png)

![Service discovery](images/payday-pg_5.png)

![Web app](images/payday-pg_6.png)

![App details](images/payday-pg_7.png)

---

## Exploitation

Found `/admin.php` with default credentials:

- `admin:admin`

![Admin panel](images/payday-pg_8.png)

Used a PHP Ivan shell from revshells to get a reverse shell.

---

## Privilege escalation

Found MySQL running:

![MySQL discovery](images/payday-pg_9.png)

![MySQL enum](images/payday-pg_10.png)

```bash
mysql -h 127.0.0.1 -u root -p
```

Found SSH authorized key:

![SSH key](images/payday-pg_11.png)

Checked for SUID binaries:

![SUID](images/payday-pg_12.png)

Found interesting backup/config files:
- `/var/lib/belocs/hashfile.old`
- `/var/backups/infodir.bak`
- `/etc/dovecot/dovecot.conf.bak`
- `/var/cache/debconf/passwords.dat`
- `/etc/mysql/conf.d/old_passwords.cnf`

Password reuse: `patrick:patrick`

```bash
su patrick
sudo -l
```

![sudo -l](images/payday-pg_13.png)

- Full sudo privileges.

---

## Lessons & takeaways

- Always try default credentials on admin panels -- `admin:admin` still works more often than you'd think
- Check MySQL for stored credentials and SSH keys
- Password reuse between database users and system accounts is a common privesc path
