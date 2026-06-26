# File System Security: Authentication and Access Control

> File system security is the set of mechanisms that decide who can access what — it combines authentication (proving who you are), user accounts and groups (organizing who belongs where), file permissions (setting what each entity can do), and access control models (DAC vs MAC) to protect the CIA triad: Confidentiality, Integrity, and Availability.

---

## Table of Contents

1. [What Is File System Security?](#1-what-is-file-system-security)
2. [Authentication](#2-authentication)
3. [User Accounts and Groups](#3-user-accounts-and-groups)
4. [File Permissions and Ownership](#4-file-permissions-and-ownership)
5. [Access Control Models: DAC vs MAC](#5-access-control-models-dac-vs-mac)
6. [Password Security](#6-password-security)
7. [Session Management](#7-session-management)
8. [Advanced Security Features](#8-advanced-security-features)
9. [Multi-Factor Authentication (MFA)](#9-multi-factor-authentication-mfa)
10. [Security Best Practices](#10-security-best-practices)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is File System Security?

**File system security** = mechanisms and policies that control access to files and directories. It ensures only the right people can do the right things to the right files.

**Hotel key card analogy:**

```
  Hotel rooms = files
  Key cards = authentication credentials
  Front desk = authentication system
  Room tiers (suite vs standard) = permission levels

  You prove who you are (authentication) → get a key card (session token)
  → key card grants access to certain rooms (permissions) but not others
```

**The CIA Triad of file security:**

| Goal                | What it means                                  | Example                               |
| ------------------- | ---------------------------------------------- | ------------------------------------- |
| **Confidentiality** | Only authorized users can VIEW files           | Private docs visible only to owner    |
| **Integrity**       | Only authorized users can MODIFY files         | System files protected from tampering |
| **Availability**    | Authorized users can ALWAYS access their files | User can always reach their home dir  |

---

## 2. Authentication

**Authentication** = verifying that you are who you claim to be before granting system access.

```
  Without authentication:
  Anyone sitting at a keyboard gets full access to all files — catastrophic!
```

### Authentication Methods

| Method                | Description                         | Example                         |
| --------------------- | ----------------------------------- | ------------------------------- |
| **Password-based**    | User provides a secret string       | Username + password login       |
| **Biometric**         | Physical characteristics            | Fingerprint, face ID, iris scan |
| **Token-based**       | Physical/digital device you possess | Security key, TOTP app          |
| **Certificate-based** | Digital cryptographic certificate   | SSH key authentication          |

### Authentication Flow

```
  1. User enters: username=alice, password=MySecret123
  2. OS looks up alice's record → retrieves stored hash
  3. OS computes: Hash(salt + "MySecret123") → compares with stored hash
  4. Match: create session, assign permissions
     No match: deny access, log the failed attempt

  Key point: OS NEVER stores the actual password — only a one-way hash!
```

---

## 3. User Accounts and Groups

| Account Type             | Privileges                | Typical Use             |
| ------------------------ | ------------------------- | ----------------------- |
| **Root / Administrator** | Unrestricted              | System admin tasks      |
| **Standard User**        | Own files + approved apps | Everyday computing      |
| **Guest**                | Minimal, temporary        | Public/shared terminals |
| **Service Account**      | Specific to one service   | Background daemons      |

**Groups** let admins assign permissions to many users at once:

```
  Group "developers" (GID 700):
    Members: alice, bob
    Permissions: Read, Write, Execute on /projects

  Group "guests" (GID 900):
    Members: charlie
    Permissions: Read only on /public

  Instead of managing alice and bob individually, manage the "developers" group.
```

---

## 4. File Permissions and Ownership

Every file has: **owner** (a user), **group** (a group), and **permissions** for owner / group / others.

### Permission Types

| Permission | Symbol | For Files      | For Directories            |
| ---------- | ------ | -------------- | -------------------------- |
| Read       | `r`    | View content   | List directory             |
| Write      | `w`    | Modify content | Create/delete files in dir |
| Execute    | `x`    | Run as program | Enter/traverse directory   |

### Unix Permission Format

```
  -rwxr-xr--
  │└─┬─┘└─┬─┘└─┬─┘
  │ owner  group  others
  └── file type (- = regular, d = directory, l = symlink)

  Example: -rw-r--r--  report.txt
    Owner (alice): read + write    (rw-)
    Group (devs):  read only       (r--)
    Others:        read only       (r--)

  Numeric (octal): rw-r--r-- = 644
                   rwxr-xr-- = 754
```

---

## 5. Access Control Models: DAC vs MAC

### Discretionary Access Control (DAC)

**File owner decides who gets access** — most common model for personal/enterprise systems.

```
  Alice creates file: confidential.txt → Alice is the owner
  Alice CAN: set permissions, share with bob, revoke access at will

  # Give bob read access
  chmod o+r confidential.txt   (simple)
  setfacl -m u:bob:r confidential.txt  (ACL — per-user)
```

- Flexible; owner has full control
- Risk: a compromised owner account = attacker controls access

### Mandatory Access Control (MAC)

**System-wide security policy enforced by the OS; users cannot override it.**

```
  Security levels: Top Secret > Secret > Confidential > Unclassified

  Bob has "Secret" clearance
  File report.txt is labeled "Top Secret"

  Result: Bob CANNOT access report.txt
  Even if the file owner says "let bob read it" — MAC overrides that.
```

- Used in military/government systems (SELinux, AppArmor on Linux; Windows Mandatory Integrity Control)
- More secure but less flexible

```mermaid
graph LR
    A[User requests file access] --> B[DAC check:\nFile permissions / ACL]
    B -->|Passes| C{MAC enabled?}
    C -->|Yes| D[MAC policy check:\nSecurity labels]
    D -->|Passes| E[ACCESS GRANTED]
    D -->|Fails| F[ACCESS DENIED]
    C -->|No| E
    B -->|Fails| F
```

---

## 6. Password Security

Passwords are never stored as plaintext — only as **salted hashes**.

```
  Password storage:
  User creates: "MySecret123"
  System: random_salt = "x9k2p7"
  System stores: { salt: "x9k2p7", hash: Hash("x9k2p7" + "MySecret123") }

  Login verification:
  User enters: "MySecret123"
  System: Hash("x9k2p7" + "MySecret123") == stored_hash?
  Match → authenticate!

  Why salt? Without salt:
  - Attacker precomputes a "rainbow table" of all hashes for common passwords
  - Looks up your hash → finds password in milliseconds
  With salt:
  - Every user has a DIFFERENT hash for the same password
  - Precomputed tables become useless — attacker must recompute for each user
```

**Good password policy (modern):**

- Minimum 12+ characters (length beats complexity)
- Passphrase style: "correct-horse-battery-staple"
- Check against lists of known-compromised passwords
- Only force reset if breach is detected (not arbitrary 90-day rotation)

---

## 7. Session Management

After authentication, the OS creates a **session** so you don't have to re-authenticate for every file operation.

```
  Session lifecycle:
  1. Login → OS creates session ID (e.g., "sess_abc123xyz")
  2. Session stores: user ID, permissions, login time, last-activity time
  3. Every file operation includes implicit session validation
  4. Session ends: explicit logout OR inactivity timeout OR system shutdown

  Session token ≈ wristband at an amusement park:
  Show ticket once (login) → get wristband (session) → use it all day
```

**Security considerations:** Session timeout (auto-lock after idle), screen lock, session tokens must be non-guessable and expire.

---

## 8. Advanced Security Features

### Audit Logging

```
  Sample audit log:
  2024-01-15 09:30:15 | alice  | READ   | /home/alice/report.txt | SUCCESS
  2024-01-15 09:35:10 | charlie| DELETE | /shared/config.ini     | DENIED ← suspicious!
  2024-01-15 10:12:33 | alice  | CHMOD  | /home/alice/script.sh  | SUCCESS
```

Audit logs record: timestamp, user, action type, file path, result (success/denied). Critical for compliance, breach investigation, and detecting unauthorized access.

### File Attributes

| Attribute | Effect                                            | Use Case                             |
| --------- | ------------------------------------------------- | ------------------------------------ |
| Read-only | Prevents modification/deletion                    | Protect docs from accidental changes |
| Hidden    | Not shown in normal listings                      | OS config files                      |
| System    | Marks as critical OS file                         | Boot and kernel files                |
| Immutable | Cannot be modified, deleted, renamed even by root | Audit log protection                 |

---

## 9. Multi-Factor Authentication (MFA)

**MFA** requires proof from two or more independent factor categories:

| Factor Type            | Description                 | Examples                                   |
| ---------------------- | --------------------------- | ------------------------------------------ |
| Something you **know** | Memorized secret            | Password, PIN, security question           |
| Something you **have** | Physical/digital possession | TOTP app, SMS code, security key (YubiKey) |
| Something you **are**  | Biometric                   | Fingerprint, face, iris                    |

```
  MFA login flow:
  1. Enter username + password  ← Factor 1 (know)
  2. System sends code to phone OR asks for fingerprint  ← Factor 2 (have / are)
  3. Both factors verified → session created, access granted

  Attack scenario:
  Attacker steals Alice's password (Factor 1 compromised)
  Still needs Alice's phone or fingerprint (Factor 2) → BLOCKED
```

---

## 10. Security Best Practices

### Principle of Least Privilege

```
  Bad: alice runs daily tasks with Administrator account
  → Malware that infects alice's browser runs with full system access

  Good: alice runs daily tasks as Standard User
  → Needs to confirm (sudo / UAC) for any admin action
  → Malware is contained to user-level permissions only
```

### Regular Audits

| Task                     | Purpose                              | Frequency |
| ------------------------ | ------------------------------------ | --------- |
| Review user accounts     | Remove inactive/unnecessary accounts | Monthly   |
| Check file permissions   | Ensure nothing over-permissioned     | Quarterly |
| Analyze access logs      | Detect suspicious patterns           | Weekly    |
| Review group memberships | Remove stale group assignments       | Quarterly |

### Defense in Depth

```
  Layer 1: Strong authentication (password + MFA)
  Layer 2: File permissions (least privilege)
  Layer 3: ACLs (fine-grained per-user rules)
  Layer 4: Encryption (data at rest protected)
  Layer 5: Audit logging (detect and investigate breaches)
  Layer 6: Backup (recover from ransomware/deletion)

  No single layer is sufficient — all layers together = true security
```

---

## 11. Key Takeaways

- **File system security** enforces Confidentiality, Integrity, and Availability (CIA triad) for stored data
- **Authentication** verifies identity — passwords are stored as salted hashes, never plaintext
- **Salt** defeats rainbow table attacks — same password produces different hash for each user
- **Unix permissions** = 9 bits (rwx for owner/group/others) — simple but limited to 3 subjects
- **ACLs** extend permissions to any individual user or group — fine-grained access control
- **DAC** (Discretionary): file owner controls access — flexible, used in most personal/enterprise systems
- **MAC** (Mandatory): OS-enforced security labels that owner cannot override — used in high-security environments
- **MFA** requires multiple independent factors — compromising one factor is not enough
- **Audit logs** record all access attempts with timestamps — essential for compliance and breach detection
- **Principle of least privilege**: grant only the minimum permissions needed — limits damage from compromised accounts
- **Session tokens** represent authenticated access; always set timeouts and require re-auth on sensitive operations
