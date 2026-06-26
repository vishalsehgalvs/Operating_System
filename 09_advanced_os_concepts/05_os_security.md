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

**Authentication** = verifying identity before granting access. The OS must know _who_ you are before deciding _what_ you can do.

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

| Tool            | Platform | Algorithm              |
| --------------- | -------- | ---------------------- |
| **BitLocker**   | Windows  | AES-128/256 (XTS mode) |
| **FileVault 2** | macOS    | AES-XTS 128-bit        |
| **LUKS**        | Linux    | AES-256 (configurable) |

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

| Layer               | Mechanism                                    | What It Prevents                                  |
| ------------------- | -------------------------------------------- | ------------------------------------------------- |
| **Authentication**  | Passwords, MFA, biometrics                   | Unauthorized login                                |
| **Access Control**  | DAC (file permissions), MAC (SELinux), RBAC  | Unauthorized resource access                      |
| **Isolation**       | User/kernel mode, virtual memory, sandboxing | Privilege escalation, process cross-contamination |
| **Encryption**      | FDE, Secure Boot, HTTPS, encrypted memory    | Data theft (at rest and in transit), bootkits     |
| **Audit Logging**   | System logs, IDS                             | Threat detection, forensics, compliance           |
| **Least Privilege** | Minimal permissions per process/user         | Limits blast radius of any compromise             |

---

## 9. Security Best Practices

| Practice                               | Reason                                            |
| -------------------------------------- | ------------------------------------------------- |
| **Keep OS updated**                    | Patches fix known exploits that malware uses      |
| **Use strong, unique passwords + MFA** | Prevents credential-based attacks                 |
| **Use standard user for daily tasks**  | Admin rights only when needed                     |
| **Enable full-disk encryption**        | Protects data if device is stolen                 |
| **Enable Secure Boot**                 | Blocks bootkit malware                            |
| **Don't run as root unnecessarily**    | Limits damage from compromised process            |
| **Regularly review file permissions**  | Catches accidental world-readable sensitive files |
| **Monitor audit logs**                 | Early detection of suspicious activity            |
| **Use a firewall**                     | Block unexpected inbound/outbound connections     |

---

## 10. Code Examples

> Working code that demonstrates OS security concepts — Unix DAC file permissions and RBAC with audit logging — in practice.

### C++ — Simple Version

Implement Unix-style Discretionary Access Control: users, groups, file permission bits (rwxrwxrwx), and an access check function.

```cpp
#include <iostream>
#include <string>
#include <map>
#include <vector>

// Unix-style Discretionary Access Control (DAC)
// File permissions: 9 bits — owner(rwx) | group(rwx) | other(rwx)
// E.g., 0644 = rw-r--r-- : owner can read/write; group and others can read only.
// Access check: owner first, then group membership, then other.

struct FilePerm {
    std::string filename;
    std::string owner;
    std::string group;
    int mode;   // Unix octal bitmask (e.g., 0600, 0644, 0755)
};

struct User {
    std::string username;
    std::vector<std::string> groups;
};

class DACSystem {
    std::map<std::string, FilePerm> files;
    std::map<std::string, User>    users;

    bool testBit(int mode, int shift) const {
        return (mode >> shift) & 1;
    }

public:
    void addUser(const std::string& name, std::vector<std::string> groups) {
        users[name] = {name, groups};
    }

    void createFile(const std::string& fname, const std::string& owner,
                    const std::string& group, int mode) {
        files[fname] = {fname, owner, group, mode};
        // Print the permission string in rwxrwxrwx format
        std::string modeStr;
        for (int shift : {6, 3, 0}) {
            modeStr += testBit(mode, shift + 2) ? 'r' : '-';
            modeStr += testBit(mode, shift + 1) ? 'w' : '-';
            modeStr += testBit(mode, shift + 0) ? 'x' : '-';
        }
        std::cout << "[FS] " << fname << " owner=" << owner
                  << " group=" << group << " mode=" << modeStr << "\n";
    }

    bool checkAccess(const std::string& username, const std::string& fname, char op) {
        // op: 'r' = read (bit 2), 'w' = write (bit 1), 'x' = execute (bit 0)
        auto fit = files.find(fname);
        auto uit = users.find(username);
        if (fit == files.end() || uit == users.end()) return false;

        auto& file = fit->second;
        auto& user = uit->second;

        // Determine which permission set applies: owner > group > other
        int shiftBase;
        std::string relation;
        if (user.username == file.owner) {
            shiftBase = 6; relation = "owner";
        } else {
            bool inGroup = false;
            for (auto& g : user.groups) {
                if (g == file.group) { inGroup = true; break; }
            }
            shiftBase = inGroup ? 3 : 0;
            relation  = inGroup ? "group" : "other";
        }

        int opShift  = (op == 'r') ? 2 : (op == 'w') ? 1 : 0;
        bool allowed = testBit(file.mode, shiftBase + opShift);

        std::cout << "[DAC] " << username << " '" << op << "' " << fname
                  << " (" << relation << ") -> "
                  << (allowed ? "ALLOWED" : "DENIED") << "\n";
        return allowed;
    }
};

int main() {
    DACSystem dac;

    dac.addUser("alice", {"dev", "sudo"});
    dac.addUser("bob",   {"dev"});
    dac.addUser("carol", {"ops"});

    std::cout << "--- File creation ---\n";
    dac.createFile("secret.txt", "alice", "dev",  0600);  // rw------- (owner only)
    dac.createFile("shared.txt", "alice", "dev",  0664);  // rw-rw-r-- (group write)
    dac.createFile("script.sh",  "alice", "sudo", 0755);  // rwxr-xr-x (all execute)

    std::cout << "\n--- Access checks ---\n";
    dac.checkAccess("alice", "secret.txt", 'r');   // Owner   -> ALLOWED
    dac.checkAccess("bob",   "secret.txt", 'r');   // Other   -> DENIED
    dac.checkAccess("bob",   "shared.txt", 'w');   // Group   -> ALLOWED
    dac.checkAccess("carol", "shared.txt", 'w');   // Other   -> DENIED
    dac.checkAccess("carol", "shared.txt", 'r');   // Other r -> ALLOWED
    dac.checkAccess("carol", "script.sh",  'x');   // Other x -> ALLOWED

    return 0;
}
```

### C++ — Medium / LeetCode Style

Implement RBAC (Role-Based Access Control): roles with permissions, users assigned to roles, and a tamper-evident audit log for every access decision.

```cpp
#include <iostream>
#include <string>
#include <set>
#include <map>
#include <vector>
#include <ctime>

// Role-Based Access Control (RBAC)
// Permissions are assigned to roles, not directly to users.
// Users are assigned one or more roles.
// Access check path: user -> roles -> permissions -> ALLOW or DENY.
// Every access attempt is recorded to an audit log for security review.

std::string currentTime() {
    time_t t = time(nullptr);
    char buf[32];
    strftime(buf, sizeof(buf), "%H:%M:%S", localtime(&t));
    return buf;
}

struct AuditEntry {
    std::string time, user, action, result;
};

class RBACSystem {
    std::map<std::string, std::set<std::string>> rolePerms;   // role -> permissions
    std::map<std::string, std::set<std::string>> userRoles;   // user -> roles
    std::vector<AuditEntry> auditLog;

public:
    void defineRole(const std::string& role, std::set<std::string> perms) {
        rolePerms[role] = perms;
        std::cout << "[RBAC] Role '" << role << "' -> {";
        for (auto& p : perms) std::cout << p << " ";
        std::cout << "}\n";
    }

    void assignRole(const std::string& user, const std::string& role) {
        userRoles[user].insert(role);
        std::cout << "[RBAC] User '" << user << "' += role '" << role << "'\n";
    }

    bool checkAccess(const std::string& user, const std::string& perm,
                     const std::string& resource) {
        bool allowed = false;
        for (auto& role : userRoles[user]) {
            if (rolePerms.count(role) && rolePerms[role].count(perm)) {
                allowed = true; break;
            }
        }
        std::string action = perm + " " + resource;
        std::string result = allowed ? "ALLOW" : "DENY";
        auditLog.push_back({currentTime(), user, action, result});

        std::cout << "[CHECK] " << user << " " << perm << " " << resource
                  << " -> " << result << "\n";
        return allowed;
    }

    void showUserPerms(const std::string& user) const {
        std::set<std::string> all;
        for (auto& role : userRoles.at(user))
            for (auto& p : rolePerms.at(role)) all.insert(p);
        std::cout << "\nEffective permissions for '" << user << "': {";
        for (auto& p : all) std::cout << p << " ";
        std::cout << "}\n";
    }

    void printAuditLog() const {
        std::cout << "\n=== Audit Log ===\n";
        for (auto& e : auditLog) {
            std::cout << "  [" << e.time << "] " << e.user
                      << " | " << e.action << " | " << e.result << "\n";
        }
    }
};

int main() {
    RBACSystem rbac;

    rbac.defineRole("admin",     {"READ", "WRITE", "DELETE", "EXECUTE", "ADMIN"});
    rbac.defineRole("developer", {"READ", "WRITE", "EXECUTE"});
    rbac.defineRole("auditor",   {"READ"});
    rbac.defineRole("ops",       {"READ", "EXECUTE"});

    std::cout << "\n";
    rbac.assignRole("alice", "admin");
    rbac.assignRole("bob",   "developer");
    rbac.assignRole("carol", "auditor");
    rbac.assignRole("dave",  "ops");
    rbac.assignRole("dave",  "developer");   // Dave has two roles

    rbac.showUserPerms("dave");   // READ + WRITE + EXECUTE (union of ops + developer)

    std::cout << "\n--- Access checks ---\n";
    rbac.checkAccess("alice", "DELETE",  "/etc/shadow");
    rbac.checkAccess("bob",   "WRITE",   "/var/www/index.html");
    rbac.checkAccess("bob",   "DELETE",  "/etc/passwd");      // DENIED — no DELETE
    rbac.checkAccess("carol", "READ",    "/var/log/audit.log");
    rbac.checkAccess("carol", "WRITE",   "/etc/config");      // DENIED — auditor only
    rbac.checkAccess("dave",  "EXECUTE", "/usr/bin/deploy.sh");

    rbac.printAuditLog();
    return 0;
}
```

### Python — Simple Version

Unix-style DAC: compute owner/group/other permission bits, then check read/write/execute access for a given user.

```python
# Unix-style Discretionary Access Control (DAC)
# Files have owner, group, and permission bits: rwxrwxrwx
# Access check: determine if the user is the owner, a group member, or other
# -> then check the corresponding 3 bits for the requested operation.

class FilePerm:
    def __init__(self, filename, owner, group, mode):
        """
        mode is a 9-bit octal integer: 0o644, 0o755, etc.
        Bits 8-6: owner rwx | Bits 5-3: group rwx | Bits 2-0: other rwx
        """
        self.filename = filename
        self.owner    = owner
        self.group    = group
        self.mode     = mode

    def mode_str(self):
        s = ''
        for shift in [6, 3, 0]:
            s += 'r' if (self.mode >> (shift + 2)) & 1 else '-'
            s += 'w' if (self.mode >> (shift + 1)) & 1 else '-'
            s += 'x' if (self.mode >> shift)       & 1 else '-'
        return s


class DACSystem:
    def __init__(self):
        self.files = {}   # {filename: FilePerm}
        self.users = {}   # {username: set of group names}

    def add_user(self, username, groups):
        self.users[username] = set(groups)

    def create_file(self, filename, owner, group, mode):
        p = FilePerm(filename, owner, group, mode)
        self.files[filename] = p
        print(f"[FS] {filename}: owner={owner} group={group} "
              f"mode={p.mode_str()} ({oct(mode)})")

    def check_access(self, username, filename, op):
        """
        op: 'r' = read, 'w' = write, 'x' = execute
        Returns True if access is allowed, False if denied.
        """
        if filename not in self.files or username not in self.users:
            return False

        perm  = self.files[filename]
        groups = self.users[username]

        # Determine relationship: owner > group member > other
        if username == perm.owner:
            shift_base = 6
            relation   = 'owner'
        elif perm.group in groups:
            shift_base = 3
            relation   = 'group'
        else:
            shift_base = 0
            relation   = 'other'

        op_shift = {'r': 2, 'w': 1, 'x': 0}[op]
        allowed  = bool((perm.mode >> (shift_base + op_shift)) & 1)

        print(f"[DAC] {username} {op!r} {filename} ({relation}) "
              f"-> {'ALLOWED' if allowed else 'DENIED'}")
        return allowed


# Demo
dac = DACSystem()
dac.add_user('alice', ['dev', 'sudo'])
dac.add_user('bob',   ['dev'])
dac.add_user('carol', ['ops'])

print("--- File creation ---")
dac.create_file('secret.txt', 'alice', 'dev',  0o600)   # rw------- owner only
dac.create_file('shared.txt', 'alice', 'dev',  0o664)   # rw-rw-r-- group write
dac.create_file('script.sh',  'alice', 'sudo', 0o755)   # rwxr-xr-x all execute

print("\n--- Access checks ---")
dac.check_access('alice', 'secret.txt', 'r')   # Owner   -> ALLOWED
dac.check_access('bob',   'secret.txt', 'r')   # Other   -> DENIED
dac.check_access('bob',   'shared.txt', 'w')   # Group   -> ALLOWED
dac.check_access('carol', 'shared.txt', 'w')   # Other   -> DENIED
dac.check_access('carol', 'shared.txt', 'r')   # Other r -> ALLOWED
dac.check_access('carol', 'script.sh',  'x')   # Other x -> ALLOWED
```

### Python — Medium Level

RBAC system: roles with permission sets, multi-role users, resource access check through the user-role-permission chain, and a full audit log.

```python
from datetime import datetime

# Role-Based Access Control (RBAC)
# Access check path: user -> roles -> permissions -> ALLOW or DENY.
# Each decision is appended to an immutable audit log for security review.

class RBACSystem:
    def __init__(self):
        self._role_permissions = {}   # {role: set of permissions}
        self._user_roles       = {}   # {user: set of roles}
        self._audit_log        = []   # [{time, user, action, result}]

    def define_role(self, role, permissions):
        self._role_permissions[role] = set(permissions)
        print(f"[RBAC] Role {role!r} -> {sorted(permissions)}")

    def assign_role(self, user, role):
        self._user_roles.setdefault(user, set()).add(role)
        print(f"[RBAC] User {user!r} += role {role!r}")

    def effective_permissions(self, user):
        """Union of all permissions from every role the user holds."""
        perms = set()
        for role in self._user_roles.get(user, []):
            perms |= self._role_permissions.get(role, set())
        return perms

    def check_access(self, user, permission, resource):
        allowed = permission in self.effective_permissions(user)
        result  = 'ALLOW' if allowed else 'DENY'
        self._audit_log.append({
            'time':   datetime.now().strftime('%H:%M:%S'),
            'user':   user,
            'action': f'{permission} {resource}',
            'result': result
        })
        print(f"[CHECK] {user} {permission!r} {resource!r} -> {result}")
        return allowed

    def show_user_summary(self, user):
        roles = sorted(self._user_roles.get(user, []))
        perms = sorted(self.effective_permissions(user))
        print(f"\nUser {user!r}: roles={roles}")
        print(f"  Effective permissions: {perms}")

    def print_audit_log(self):
        print("\n=== Audit Log ===")
        for e in self._audit_log:
            print(f"  [{e['time']}] {e['user']:<8} | {e['action']:<35} | {e['result']}")


# Demo
rbac = RBACSystem()

rbac.define_role('admin',     ['READ', 'WRITE', 'DELETE', 'EXECUTE', 'ADMIN'])
rbac.define_role('developer', ['READ', 'WRITE', 'EXECUTE'])
rbac.define_role('auditor',   ['READ'])
rbac.define_role('ops',       ['READ', 'EXECUTE'])

print()
rbac.assign_role('alice', 'admin')
rbac.assign_role('bob',   'developer')
rbac.assign_role('carol', 'auditor')
rbac.assign_role('dave',  'ops')
rbac.assign_role('dave',  'developer')   # Dave holds two roles

rbac.show_user_summary('dave')   # READ + WRITE + EXECUTE (ops union developer)

print("\n--- Access checks ---")
rbac.check_access('alice', 'DELETE',  '/etc/shadow')
rbac.check_access('bob',   'WRITE',   '/var/www/index.html')
rbac.check_access('bob',   'DELETE',  '/etc/passwd')          # DENIED
rbac.check_access('carol', 'READ',    '/var/log/audit.log')
rbac.check_access('carol', 'WRITE',   '/etc/config')          # DENIED
rbac.check_access('dave',  'EXECUTE', '/usr/bin/deploy.sh')   # ops+dev -> ALLOWED

rbac.print_audit_log()
```

---

## 11. Key Takeaways

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
