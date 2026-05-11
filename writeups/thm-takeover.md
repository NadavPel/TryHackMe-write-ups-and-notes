# Takeover — TryHackMe Write-up

## Room Info

- **Difficulty:** Easy
- **Topic:** Subdomain Enumeration, SSL Certificate Inspection
- **OS:** Linux

---

## Objective
Find a hidden flag on a THM machine with the domain `futurevera.thm`.

---

## Step 1 — vhost Enumeration

We need to find hidden subdomains using gobuster in vhost mode.

**Mistakes along the way:**
- Used `dns` mode instead of `vhost` — didn't work because the domain doesn't exist in real DNS
- Got tons of false positives with status 421 
- Forgot the `-k` flag to skip TLS verification — added it

**Final command:**
```bash
gobuster vhost -u "https://futurevera.thm" \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -k --append-domain 
```

**Results:**
- `blog.futurevera.thm`
- `support.futurevera.thm`

---

## Step 2 — Adding to /etc/hosts

Without this, the browser doesn't know where to go — the domain doesn't exist in real DNS.

```
10.10.x.x      support.futurevera.thm
10.10.x.x      blog.futurevera.thm 
```

**Note:** Status 421 (Misdirected Request) occurs when the server receives a request but doesn't recognize the domain. The fix is adding the domain to `/etc/hosts` so the request arrives with the correct Host header.

---

## Step 3 — Exploring the Sites

| Site | What we found |
|---|---|
| `blog.futurevera.thm` | A blog about space |
| `support.futurevera.thm` | Site under maintenance |

---

## Step 4 — Rabbit Holes

The following paths were all investigated and led nowhere:

**gobuster dir** on blog and support — found `/assets`, `/css`, `/js`, `/server-status` (403) — nothing interesting.

**exiftool** on images and video from /assets — no interesting metadata found.

**robots.txt** on both sites — 404.

**vhost enumeration on blog.futurevera.thm** — returned many subdomains that looked promising:
- `myadmin.blog.futurevera.thm`
- `editblog.blog.futurevera.thm`
- `sslvpn.blog.futurevera.thm` — looked very promising because of the name, but led nowhere
- Many more...

We added several of them to `/etc/hosts` and visited them — none contained anything useful.

> **Lesson:** Not every finding leads to a breakthrough. A big part of the job is knowing when to abandon a direction and try something else.

---

## Step 5 — The Real Find

We inspected the **SSL Certificate** of `support.futurevera.thm` through the browser (click the padlock → Certificate Details).

Inside the **Subject Alternative Names (SANs)** we found a hidden subdomain:
```
secrethelpdesk934752.support.futurevera.thm
```

Added it to `/etc/hosts` and visited it — the URL bar revealed the flag:
```
flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com
```

The flag was embedded as a SAN pointing to an AWS S3 bucket.

---
## Real-World Relevance

This challenge simulates a real **information disclosure** vulnerability.

In real penetration tests, SSL certificates are public — anyone can read them.
If a developer registers an internal or sensitive domain as a SAN (e.g. internal AWS bucket, staging server, admin panel), it becomes visible to anyone who inspects the certificate.

This is a reportable finding in a pentest report because it exposes internal infrastructure that was never meant to be public.

---

## Real-World Relevance

This challenge simulates a real **information disclosure** vulnerability.

In real penetration tests, SSL certificates are public — anyone can read them.
If a developer registers an internal or sensitive domain as a SAN (e.g. internal 
AWS bucket, staging server, admin panel), it becomes visible to anyone who inspects 
the certificate.

This is a reportable finding in a pentest report because it exposes internal 
infrastructure that was never meant to be public.

**What is an S3 Bucket?** S3 (Simple Storage Service) is AWS's cloud storage 
solution — essentially a folder in the cloud used to store files like images, 
backups, logs, and static assets.

A common misconfiguration is leaving a bucket **publicly accessible**, allowing 
anyone to list or download its contents. Combined with a leaked bucket name (like 
in this challenge), an attacker could potentially access sensitive files or even 
perform a **Subdomain Takeover** if the bucket no longer exists but the SAN record 
still points to it.

---

## Concepts Learned

| Concept | Explanation |
|---|---|
| **vhost enumeration** | Finding subdomains via HTTP Host Header, not DNS lookup |
| **SAN (Subject Alternative Names)** | Field in SSL certificate listing additional domains the cert covers |
| **421 Misdirected Request** | Server doesn't recognize the domain in the request — fix: add to `/etc/hosts` |
| **`-k` / `--no-tls-validation`** | Skip TLS certificate verification in gobuster/curl |
| **S3 Bucket** | AWS cloud file storage service |
| **`/etc/hosts`** | Local DNS file on your machine — overrides real DNS |
| **Rabbit hole** | An investigation path that looks promising but leads nowhere |

---

## Key Takeaway

**Always inspect SSL certificates** — SANs can contain hidden subdomains that can't be found any other way. It's an information source most people completely overlook.

---

> **Fun fact:** This room is called "Takeover" for a reason — 
> the real attack here isn't just finding the flag, it's that 
> an attacker could register the exposed S3 bucket name and 
> take over the subdomain entirely.


