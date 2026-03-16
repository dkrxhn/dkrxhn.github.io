# Mentor — HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Mentor (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-01-02

---

## TL;DR

### SNMP with community string `internal` leaked an API key. Accessed a FastAPI docs endpoint, used the API to get a JWT, then triggered a backup endpoint for command injection into a Docker container. Pivoted to Postgres for a password hash, cracked it, then SSH'd as `svc`. Found SNMP config with root's password for sudo escalation.
---
## Target info

- Host: `10.129.228.102`
- Vhost: `api.mentorquotes.htb`
---
## Enumeration

![Nmap results](images/mentor-htb_1.png)

![Web page](images/mentor-htb_2.png)

![Directory fuzzing](images/mentor-htb_3.png)

SNMP walk with community string `internal`:

```bash
time snmpbulkwalk -v2c -c internal 10.129.228.102
```

![SNMP results](images/mentor-htb_4.png)

![SNMP leaked key](images/mentor-htb_5.png)

Found API key: `kj23sadkj123as0-d213`

![Vhost discovery](images/mentor-htb_6.png)

Discovered `api.mentorquotes.htb`, added to `/etc/hosts`.

![API page](images/mentor-htb_7.png)

![API endpoints](images/mentor-htb_8.png)

---
## Foothold

Found `/docs` endpoint (FastAPI Swagger):

![FastAPI docs](images/mentor-htb_9.png)

User `james@mentorquotes.htb` visible. Used `/auth/login`:

![Login attempt](images/mentor-htb_10.png)

![JWT token](images/mentor-htb_11.png)

Got JWT: `eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...`

![API with JWT](images/mentor-htb_12.png)

Used the JWT with various API requests (`/users`, `/admin/backup`) and iterated until finding a command injection point:

![Command injection](images/mentor-htb_13.png)

Used Python reverse shell from revshells.com (escaped double quotes):

![Reverse shell](images/mentor-htb_14.png)

No bash available:

![No bash](images/mentor-htb_15.png)

---
## Lateral movement

![Container environment](images/mentor-htb_16.png)

uvicorn was running with `--reload`, meaning changes to the API auto-apply.

![Postgres connection](images/mentor-htb_17.png)

Connected to Postgres with default creds `postgres:postgres`. Found password column in the database that wasn't exposed via the API:

![Database dump](images/mentor-htb_18.png)

![Password hashes](images/mentor-htb_19.png)

![Hash cracking](images/mentor-htb_20.png)

![Cracked passwords](images/mentor-htb_21.png)

![Postgres tables](images/mentor-htb_22.png)

`\list` showed `mentorquotes_db`.

Cracked hash: `53f22d0dfa10dce7e29cd31f4f953fd8` -> `123meunomeeivani`

```bash
ssh svc@mentorquotes.htb
```

![SSH as svc](images/mentor-htb_23.png)

---
## Privesc

Checked SNMP configuration:

```bash
cat snmpd.conf | grep -v "^#" | grep .
```

![snmpd.conf](images/mentor-htb_24.png)

Found: `SuperSecurePassword123__`

![Root via sudo](images/mentor-htb_25.png)

Used the SNMP config password with `sudo su` for root.

---
## Lessons & takeaways

- SNMP community strings beyond `public`/`private` (like `internal`) can leak sensitive data -- always brute-force community strings
- FastAPI `/docs` endpoint exposes the full API spec including hidden admin routes
- Docker containers often have default Postgres credentials and can be pivoted through
- SNMP configuration files (`snmpd.conf`) can contain reused passwords
---
