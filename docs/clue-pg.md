# Clue -- Proving Grounds (write-up)

**Difficulty:** Hard
**Box:** Clue (Proving Grounds)
**Author:** dsec
**Date:** 2024-06-10

---

## TL;DR

### Exploited a Grafana vulnerability to extract credentials and read config files, pivoted through FreeSWITCH to get a shell as cassie, then abused cassandra-web running as root to read /etc/shadow and escalate to root.

---

## Enumeration

![Nmap results](images/clue-pg_1.png)

![Services](images/clue-pg_2.png)

![Web enum](images/clue-pg_3.png)

![More enum](images/clue-pg_4.png)

![Grafana](images/clue-pg_5.png)

![Exploit search](images/clue-pg_6.png)

- Could **not** get SSH or anything else interesting.

Found a comment on the exploit:

![Exploit comment](images/clue-pg_7.png)

---

## Exploitation

![Grafana exploit](images/clue-pg_8.png)

- `cassie:SecondBiteTheApple330`
- ALWAYS READ THE POC!
- SSH **failed** with these creds.

Used the exploit to read SSH config:

```bash
python 49362.py -p 3000 192.168.157.240 /etc/ssh/sshd_config
```

![sshd_config](images/clue-pg_9.png)

- Only `root` and `anthony` can SSH.

---

## FreeSWITCH pivot

Found FreeSWITCH event socket password:

![FreeSWITCH docs](images/clue-pg_10.png)

![Google result](images/clue-pg_11.png)

![Config read](images/clue-pg_12.png)

- Password: `StrongClueConEight021`

![FreeSWITCH access](images/clue-pg_13.png)

![Shell as cassie](images/clue-pg_14.png)

![Enumeration](images/clue-pg_15.png)

![More enum](images/clue-pg_16.png)

![cassandra-web](images/clue-pg_17.png)

![Running services](images/clue-pg_18.png)

---

## Privilege escalation

`cassandra-web` is running as root, so I can read `/etc/shadow` via path traversal:

```bash
curl --path-as-is localhost:4444/../../../../../../../../etc/shadow
```

![/etc/shadow](images/clue-pg_19.png)

![Hash cracking](images/clue-pg_20.png)

![Root access](images/clue-pg_21.png)

![Root flag](images/clue-pg_22.png)

---

## Lessons & takeaways

- Always read the full POC code -- creds were embedded in the exploit output
- When SSH is restricted by `AllowUsers`, check which users are actually permitted
- Internal services (like FreeSWITCH, cassandra-web) running as root are prime privesc targets
- Path traversal on internal web services can leak sensitive files like `/etc/shadow`
