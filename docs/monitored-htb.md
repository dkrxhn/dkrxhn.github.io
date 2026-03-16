# Monitored -- HackTheBox (write-up)

**Difficulty:** Medium
**Box:** Monitored (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-07-16

---

## TL;DR

### SNMP enumeration leaked credentials. Used Nagios XI API to authenticate and exploit SQL injection + API user creation for a shell. Privesc via sudo nagios script symlink or process hijack after restart.
---

## Target info

- Host: `10.129.181.173`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`, `161/udp (snmp)`, `443/tcp (https)`

---

## Enumeration

![Nmap TCP results](images/monitored-htb_1.png)

![Nmap continued](images/monitored-htb_2.png)

![Web page](images/monitored-htb_3.png)

![More enumeration](images/monitored-htb_4.png)

UDP scan for SNMP:

```bash
sudo nmap -sU -p161,123,1434 -sCV 10.129.181.173 -vvv --open -Pn
```

![UDP scan](images/monitored-htb_5.png)

SNMP brute:

```bash
python /opt/snmpbrute.py -t 10.129.181.173
```

![SNMP brute](images/monitored-htb_6.png)

SNMP walk:

```bash
snmpbulkwalk -v2c -c public 10.129.181.173
```

![SNMP walk](images/monitored-htb_7.png)

![SNMP creds](images/monitored-htb_8.png)

- Found creds: `svc:XjH7VCehowpR1xZB`

## Exploitation

Logged into Nagios with discovered creds:

![Login success](images/monitored-htb_9.png)

**Wrong password gives different response:**

![Wrong pw](images/monitored-htb_10.png)

Found the API authentication endpoint:

![API endpoint](images/monitored-htb_11.png)

- `/nagiosxi/api/v1/authenticate`

![API testing](images/monitored-htb_12.png)

![GET request](images/monitored-htb_13.png)

Switched to POST:

![POST request](images/monitored-htb_14.png)

Added username and password headers, removed extras:

![Headers](images/monitored-htb_15.png)

Used API token to access site: <https://support.nagios.com/forum/viewtopic.php?t=58783>

![API token access](images/monitored-htb_16.png)

![Nagios dashboard](images/monitored-htb_17.png)

- Version 5.11

![Version](images/monitored-htb_18.png)

Reference walkthrough: [HackTheBox - Monitored](https://www.youtube.com/watch?v=Ulb2rm2qbJY)

![Root](images/monitored-htb_19.png)

SQL injection was possible but tedious manually -- used sqlmap. Involved API hacking to create a user, then called a shell. From there, used a sudo permission symlink from a nagios script to get root shell. Can also restart as sudo and hijack a process.

---

## Lessons & takeaways

- Always scan UDP -- SNMP can leak credentials via snmpbulkwalk
- Nagios XI API authentication endpoint can be used to get tokens
- SQL injection + API user creation is a powerful combo on Nagios XI
---
