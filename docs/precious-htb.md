# Precious -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** Precious (HackTheBox)
**Author:** dsec
**Date:** 2025-08-03

---

## TL;DR

### Web app converts pages to PDF -- exploited SSRF via pdfkit (Ruby). Found SSH creds in config files. Privesc via YAML deserialization in a sudo ruby script to copy SUID bash.
---

## Target info

- Host: see nmap results
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/precious-htb_1.png)

Updated `/etc/hosts`:

![Hosts file](images/precious-htb_2.png)

![Web page](images/precious-htb_3.png)

![PDF conversion](images/precious-htb_4.png)

![PDF metadata](images/precious-htb_5.png)

- Ruby-based PDF generation

## Exploitation

![SSRF exploit](images/precious-htb_6.png)

![Reverse shell](images/precious-htb_7.png)

![Shell received](images/precious-htb_8.png)

![Enumeration](images/precious-htb_9.png)

![Config files](images/precious-htb_10.png)

Found creds in config:

![Credentials](images/precious-htb_11.png)

- `henry:Q3c1AqGHtoI0aXAYFH`

![SSH as henry](images/precious-htb_12.png)

## Privilege escalation

![sudo -l](images/precious-htb_13.png)

![Ruby script](images/precious-htb_14.png)

Copied YAML deserialization PoC from: <https://gist.githubusercontent.com/staaldraad/89dffe369e1454eedd3306edc8a7e565/raw/4b85e6fe8f5708f0a7ba87dbecb6954f8f380bee/ruby_yaml_load_sploit2.yaml>

Saved as `dependencies.yml`:

![dependencies.yml](images/precious-htb_15.png)

Payload: copy bash then set SUID and SGID for root:

![Payload](images/precious-htb_16.png)

![Root shell](images/precious-htb_17.png)

---

## Lessons & takeaways

- Webpage-to-PDF conversion can be exploited via SSRF -- check PDF metadata for the tool used
- YAML deserialization in Ruby allows arbitrary command execution
- YAML deserialization in Python is also possible (per 0xdf)
---
