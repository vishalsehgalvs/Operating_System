# Operating System Security

> **OS security** is the set of mechanisms the kernel uses to ensure only authorized users and programs can access resources — layered defenses include **authentication** (prove who you are), **access control** (limit what you can do), **isolation** (prevent programs from interfering with each other), **encryption** (protect data at rest and in transit), and **audit logging** (detect and investigate threats).

---

## Table of Contents

1. [Why OS Security Matters](#1-why-os-security-matters)
2. [Authentication Mechanisms](#2-authentication-mechanisms)
3. [Access Control Mechanisms](#3-access-control-mechanisms)
4. [Isolation and Protection](#4-isolation-and-protection)
5. [Encryption Mechanisms](#5-encryption-mechanisms)
6. [Security Policies and Monitoring](#6-security-policies-and-monitoring)
7. [Common Security Threats](#7-common-security-threats)
8. [Defense-in-Depth Summary](#8-defense-in-depth-summary)
9. [Security Best Practices](#9-security-best-practices)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Why OS Security Matters

The OS controls **all** resources — CPU, memory, storage, network, devices. Without proper security, any program could:
- Read other users' private files
- Steal passwords or session tokens
- Crash the system or install malware
- Exfiltrate data over the network

**Analogy:** The OS is a gatekeeper of a secure building — it checks IDs at the door (authentication), issues different keycards for different rooms (access control), puts important files in safes (encryption), and keeps security cameras running (audit logging).

```
  Security model layers (defense-in-depth):
  
  Layer 1: Authentication   → "Are you who you claim to be?"
  Layer 2: Access Control   → "Are you allowed to do this?"
  Layer 3: Isolation        → "Can you interfere with other processes/users?"
  Layer 4: Encryption       → "Can you read this data if you bypass the above?"
  Layer 5: Audit Logging    → "Can we detect and investigate what happened?"
  
  If one layer is bypassed, the next layer still protects.
```

Modern threats: viruses, ransomware, privilege escalation exploits, phishing (credential theft), physical theft, insider attacks, zero-day exploits.

---

## 2. Authentication Mechanisms

**Authentication** = verifying identity before granting access. The OS must know *who* you are before deciding *what* you can do.

### Password-Based Authentication

The OS **never stores passwords in plain text** — it stores a cryptographic hash:

```c
// Simplified login flow
char* input_password = read_password_from_user();
char* hashed_input   = hash_function(input_password + stored_salt);

if (strcmp(hashed_input, stored_hash) == 0) {
    grant_access();          // hashes match → authenticated
} else {
    deny_access();           // hashes differ → wrong password
    log_failed_attempt();    // for audit and lockout detection
}

// Even if /etc/shadow is stolen, attacker gets hashes — not plain passwords
// Cracking requires trying billions of candidates (dictionary + brute force)
// Salt prevents rainbow table attacks (precomputed hash lookup tables)
```

**Linux** stores hashed passwords in `/etc/shadow` (readable only by root). **Windows** stores them in the SAM database (encrypted with SYSKEY).

### Multi-Factor Authentication (MFA)

MFA requires two or more of:
- **Something you know:** password, PIN
- **Something you have:** phone (TOTP code), hardware key (YubiKey)
- **Something you are:** fingerprint, face, iris scan (biometrics)

```
  Password only:
  Attacker steals password → immediate access

  Password + TOTP (6-digit code from phone app):
  Attacker steals password → still needs your phone to get the current 6-digit code
  → Much harder to compromise remotely
```

### Biometric Authentication

OS captures biometric data during enrollment (fingerprint, face), stores a **mathematical template** (not the raw image). During authentication, it compares live scan to template.

Windows Hello, macOS Touch ID, Linux FIDO2 support all implement this at the OS level.

---

## 3. Access Control Mechanisms

**Access control** determines what authenticated users and processes are allowed to do. Three main models:

### Discretionary Access Control (DAC)

The **resource owner** decides who can access it. Most common in personal OSes (Linux, Windows, macOS).

```bash
# Linux file permissions (DAC)
-rw-r--r--  1 john  users  2048  report.txt

# Format: [type][owner][group][others]
# -        = regular file
# rw-      = owner (john): read, write, no execute
# r--      = group (users): read only
# r--      = others (everyone else): read only

# Change permissions:
chmod 750 script.sh   # owner: rwx, group: r-x, others: ---
chown alice:devteam file.txt  # change owner and group
```

**Problem with DAC:** It's flexible but relies on users making correct decisions. A user can accidentally grant world-readable permissions to a sensitive file.

### Mandatory Access Control (MAC)

The **OS enforces system-wide policies** that even file owners cannot override. Based on security labels and clearance levels.

```
  MAC example (SELinux):
  
  Process label:  web_server_t (web server process)
  File label:     user_home_t  (user's home directory)
  
  Policy rule:    web_server_t CANNOT READ user_home_t
  
  Even if the file permissions say -rw-rw-rw- (world readable),
  SELinux BLOCKS the web server from reading it.
  File owner CANNOT override this — only admin can change MAC policy.
```

Used in: military/government systems, Android (SELinux), RHEL/Fedora (SELinux), Ubuntu (AppArmor).

### Role-Based Access Control (RBAC)

Permissions assigned to **roles**, users assigned to roles. Simplifies management in organizations.

```
  Roles:
  admin       → can install software, modify system, read all files
  developer   → can read/write source code, run builds
  viewer      → read-only access to reports
  
  Users:
  Alice → role: developer
  Bob   → role: viewer
  Carol → role: admin
  
  To give Bob write access: change his role to developer — applies immediately
  vs DAC: would need to update permissions on each individual file
```

---

## 4. Isolation and Protection

### User Mode / Kernel Mode Separation

CPU runs in two privilege levels:

```
  Kernel Mode (Ring 0):
  - Full hardware access (read/write any memory, any I/O port)
  - Only the OS kernel runs here
  - User programs cannot enter without making a system call
  
  User Mode (Ring 3):
  - Restricted — can only access process's own memory
  - Cannot directly access hardware
  - Must use system calls to request OS services
  
  What happens if user program tries a privileged instruction?
  → CPU triggers a General Protection Fault
  → OS terminates the process
  → Other processes and the kernel are unaffected
```

This is **the fundamental isolation mechanism** — even buggy or malicious user programs cannot crash the kernel or read other processes' memory.

### Process Isolation (Virtual Memory)

```
  Process A:  virtual address 0x1000 → mapped to physical frame #42
  Process B:  virtual address 0x1000 → mapped to physical frame #87
  
  Process A reads address 0x1000 → gets its own data (frame #42)
  Process A CANNOT access frame #87 — it's not in A's page table
  
  MMU (Memory Management Unit) enforces this in hardware.
  Any attempt to access unmapped memory → segmentation fault → process killed.
```

Even if Process A is compromised, it cannot read Process B's memory.

### Sandboxing

A **sandbox** is a restricted execution environment with strictly limited access to system resources.

```
  Web browser tab (sandboxed):
  
  Tab 1 (evil website):
  ├── Can render HTML/CSS/JS
  ├── Can access its own cookies
  ├── Can make HTTP requests (restricted by CORS/CSP)
  ✗ Cannot read files from your filesystem
  ✗ Cannot access Tab 2's memory or cookies
  ✗ Cannot modify OS settings
  ✗ Cannot make arbitrary system calls
  
  If Tab 1 exploits a JS engine bug:
  → Attacker is still inside the sandbox
  → Cannot escape to host OS (without a second "sandbox escape" exploit)
```

**OS-level sandboxing mechanisms:**
- **seccomp (Linux):** whitelist of allowed system calls — all others are blocked
- **AppArmor/SELinux:** MAC policy restricts what files/processes a sandboxed app can reach
- **Pledge/Unveil (OpenBSD):** explicitly declare what syscalls/paths the process needs

---

## 5. Encryption Mechanisms

### Full-Disk Encryption (FDE)

```
  Without FDE:
  Laptop stolen → thief removes hard drive → reads all files directly
  → File permissions are irrelevant if you bypass the running OS
  
  With FDE (BitLocker/FileVault/LUKS):
  Laptop stolen → hard drive contains only encrypted ciphertext
  → Without encryption key (derived from password at boot), data is unreadable
  → Thief gets: random-looking bytes. Useless.
  
  Process:
  Boot → user enters passphrase → OS derives encryption key
  → On-the-fly decryption as files are accessed
  → Completely transparent after boot — no performance impact for user
```

| Tool | Platform | Algorithm |
|------|----------|-----------|
| **BitLocker** | Windows | AES-128/256 (XTS mode) |
| **FileVault 2** | macOS | AES-XTS 128-bit |
| **LUKS** | Linux | AES-256 (configurable) |

### Secure Boot

```
  Without Secure Boot:
  Attacker installs a "bootkit" on your hard drive
  → Bootkit loads BEFORE Windows/Linux
  → Traditional antivirus runs AFTER bootkit → can't detect it
  → Bootkit has full control of system from the start
  
  With Secure Boot:
  UEFI firmware checks digital signature of bootloader
  If signature doesn't match trusted key database → REFUSE TO BOOT
  Bootkit has no valid signature → blocked before it can run
```

Secure Boot uses a chain of trust: UEFI firmware → bootloader signature → kernel signature → secure.

### Encrypted Memory / Secure Enclaves

Modern CPUs (Intel SGX, AMD SEV) support running code in **encrypted memory regions** called enclaves — the hypervisor and OS cannot read the enclave's memory, even if they're compromised.

---

## 6. Security Policies and Monitoring

### Audit Logging

The OS records security-relevant events for detection and forensics:

```
  Example audit log entries (/var/log/auth.log on Linux):
  
  [2026-06-26 09:15:01] LOGIN_SUCCESS  user=alice   from=192.168.1.20
  [2026-06-26 09:21:45] LOGIN_FAILED   user=admin   from=45.33.123.8  ← unusual source!
  [2026-06-26 09:21:47] LOGIN_FAILED   user=admin   from=45.33.123.8
  [2026-06-26 09:21:50] LOGIN_FAILED   user=admin   from=45.33.123.8
  [2026-06-26 09:21:50] ACCOUNT_LOCKED user=admin   reason=too_many_failures
  [2026-06-26 09:22:01] SUDO_USED      user=alice   command=/usr/bin/apt install
  [2026-06-26 09:45:22] FILE_ACCESS    user=alice   file=/etc/passwd  mode=read
```

**What to monitor:** login attempts/failures, sudo/admin escalations, file access on sensitive paths, network connections, process creation, cron job changes.

### Intrusion Detection System (IDS)

The OS and security tools monitor for behavioral anomalies:

```
  Rule-based IDS:
  "Alert if any process reads more than 1000 files in 10 seconds"
  → Detects ransomware encrypting your files
  
  Anomaly-based IDS:
  Baseline: web server process normally makes HTTP requests, reads /var/www/
  Anomaly: web server process suddenly reads /etc/shadow, /home/alice/.ssh/
  → Possible web shell / compromise → alert or block
```

**Windows Defender, ClamAV, OSSEC, auditd (Linux)** all implement IDS at the OS level.

### Principle of Least Privilege

```
  WRONG: Run everything as root / Administrator
  If a process gets compromised → attacker has root → game over
  
  RIGHT: Give each process/user only the permissions it actually needs
  
  Web server: reads /var/www/ → give read access to /var/www/ only
  Backup job: reads files, writes to backup dir → no execute, no network
  
  Even if web server is hacked → attacker can only access /var/www/
  Cannot read /home/user/, cannot install software, cannot make kernel calls
```

---

## 7. Common Security Threats

### Malware and Viruses

```
  OS defenses against malware:
  ✓ Code signing — only run programs with valid digital signature
  ✓ Sandboxing — limit what malware can access even if it runs
  ✓ Real-time scanning — check files/processes against known malware signatures
  ✓ Behavior monitoring — detect ransomware by mass file modification pattern
  ✓ Mandatory Access Control — even malware can't access what policy forbids
  ✓ Automatic updates — patch vulnerabilities malware exploits
```

### Privilege Escalation

Attacker gains higher privileges than authorized:

```
  Types:
  
  Vertical privilege escalation:
  Standard user → exploits kernel bug → gains root/SYSTEM
  
  Horizontal privilege escalation:
  User alice → exploits misconfigured permission → reads user bob's files
  
  Defenses:
  ✓ Keep OS patched (closes known exploits)
  ✓ Run processes as minimal-privilege users (limits damage radius)
  ✓ Kernel hardening (KASLR, SMEP, SMAP — makes exploits much harder)
  ✓ Linux capabilities — grant specific root powers without full root
```

**KASLR** (Kernel Address Space Layout Randomization): randomizes where kernel code/data is loaded in memory each boot — makes return-oriented-programming (ROP) exploits much harder since attacker doesn't know addresses.

### Data Breaches

```
  Attack vectors:
  - Weak/stolen password → authentication bypass
  - Misconfigured permissions → DAC failure
  - SQL injection → app-level access to database
  - Unpatched vulnerability → remote code execution
  
  OS-level defenses:
  ✓ Full-disk encryption → stolen hardware useless
  ✓ Access control → breached app only accesses its own data
  ✓ Audit logging → detect breach and understand scope
  ✓ Network firewall → block unexpected outbound connections (data exfiltration)
```

---

## 8. Defense-in-Depth Summary

| Layer | Mechanism | What It Prevents |
|-------|-----------|-----------------|
| **Authentication** | Passwords, MFA, biometrics | Unauthorized login |
| **Access Control** | DAC (file permissions), MAC (SELinux), RBAC | Unauthorized resource access |
| **Isolation** | User/kernel mode, virtual memory, sandboxing | Privilege escalation, process cross-contamination |
| **Encryption** | FDE, Secure Boot, HTTPS, encrypted memory | Data theft (at rest and in transit), bootkits |
| **Audit Logging** | System logs, IDS | Threat detection, forensics, compliance |
| **Least Privilege** | Minimal permissions per process/user | Limits blast radius of any compromise |

---

## 9. Security Best Practices

| Practice | Reason |
|----------|--------|
| **Keep OS updated** | Patches fix known exploits that malware uses |
| **Use strong, unique passwords + MFA** | Prevents credential-based attacks |
| **Use standard user for daily tasks** | Admin rights only when needed |
| **Enable full-disk encryption** | Protects data if device is stolen |
| **Enable Secure Boot** | Blocks bootkit malware |
| **Don't run as root unnecessarily** | Limits damage from compromised process |
| **Regularly review file permissions** | Catches accidental world-readable sensitive files |
| **Monitor audit logs** | Early detection of suspicious activity |
| **Use a firewall** | Block unexpected inbound/outbound connections |

---

## 10. Key Takeaways

- **OS security is multi-layered:** authentication → access control → isolation → encryption → monitoring; each layer independently limits damage if others are bypassed
- **Authentication** verifies identity: passwords (never stored plain, always hashed + salted), MFA (second factor = phone/key), biometrics
- **DAC** (discretionary): owner controls access (Linux `rwxrwxrwx` permissions, Windows ACLs); flexible but relies on users making correct decisions
- **MAC** (mandatory): OS enforces policy regardless of owner settings (SELinux, AppArmor); used in high-security environments
- **RBAC**: permissions assigned to roles, users assigned to roles; simplifies management in organizations
- **User mode / kernel mode** separation is the fundamental hardware isolation: user programs can't access hardware directly; kernel validates every system call
- **Virtual memory** gives each process its own isolated address space — processes can't read each other's memory (MMU enforces in hardware)
- **Sandboxing** restricts what a process can see/do: browser tabs, mobile apps, `seccomp` syscall filtering, AppArmor profiles
- **Full-disk encryption** (BitLocker/FileVault/LUKS) makes stolen hardware useless without the passphrase; uses on-the-fly AES decryption
- **Secure Boot** verifies bootloader signatures before loading — prevents bootkits that install below the OS
- **Audit logging** records security events; IDS detects anomalous behavior (brute-force logins, ransomware patterns, unexpected data access)
- **Principle of least privilege:** give users and processes only what they need — limits blast radius of any compromise
- **KASLR + SMEP/SMAP** are kernel hardening features that make exploitation much harder even after a vulnerability is found
- Encryption ≠ access control: access control works while OS is running; encryption protects data when OS is bypassed (stolen disk, offline attack)
