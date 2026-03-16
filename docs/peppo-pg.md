# Peppo -- Proving Grounds (write-up)

**Difficulty:** Intermediate
**Box:** Peppo (Proving Grounds)
**Author:** dsec
**Date:** 2025-09-24

---

## TL;DR

### Enumeration led to initial access. Privilege escalation via Docker.
---

## Target info

- Host: Peppo (Proving Grounds)

---

## Enumeration

![Nmap results](images/peppo-pg_1.png)

![Service details](images/peppo-pg_2.png)

---

## Foothold

![Initial access](images/peppo-pg_3.png)

---

## Privilege escalation

Escalated via Docker:

![Docker privesc](images/peppo-pg_4.png)

![Root](images/peppo-pg_5.png)

---

## Lessons & takeaways

- Docker group membership is a known privesc vector
- If a user can run Docker, they can mount the host filesystem and access everything
---
