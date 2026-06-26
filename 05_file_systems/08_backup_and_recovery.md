# Backup and Recovery in Operating Systems

> Backup creates duplicate copies of data so it can be restored when the original is lost or corrupted; recovery is the process of actually bringing that data back — together they are the safety net that protects against hardware failure, accidental deletion, ransomware, and disasters.

---

## Table of Contents

1. [What Are Backup and Recovery?](#1-what-are-backup-and-recovery)
2. [Types of Backup Strategies](#2-types-of-backup-strategies)
3. [Backup Storage Locations](#3-backup-storage-locations)
4. [Recovery Mechanisms](#4-recovery-mechanisms)
5. [File System Features for Backup/Recovery](#5-file-system-features-for-backuprecovery)
6. [Disaster Recovery Planning: RTO and RPO](#6-disaster-recovery-planning-rto-and-rpo)
7. [Best Practices](#7-best-practices)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What Are Backup and Recovery?

**Backup** = creating and storing duplicate copies of data so it can be restored after loss or corruption.  
**Recovery** = restoring data from those copies when the original is unavailable.

**Photocopy analogy:**

```
  Before storing your only copy of an important contract in a safe:
  Make a photocopy → keep one at home, one at work, one in the cloud.

  If the safe is destroyed in a fire:
  You still have the photocopies → recovery is possible.

  Without photocopies → permanent loss.
```

**What backups protect against:**

| Threat           | Example                         |
| ---------------- | ------------------------------- |
| Hardware failure | SSD dies, RAID controller fails |
| Software bugs    | File system corruption          |
| Human error      | `rm -rf` on the wrong directory |
| Security breach  | Ransomware encrypts all files   |
| Natural disaster | Fire, flood, earthquake         |

---

## 2. Types of Backup Strategies

### Full Backup

Copies **all selected data** every time. Complete snapshot.

```
  Monday backup: copies all 500 GB
  Tuesday backup: copies all 500 GB again

  Restore: use Monday's backup → done. Simple!
  Downside: slow, expensive storage, network bandwidth heavy.
```

### Incremental Backup

Copies **only data changed since the LAST backup of any type** (full or incremental).

```
  Monday: Full backup (500 GB)
  Tuesday: Incremental — only Tuesday's changes (2 GB)
  Wednesday: Incremental — only Wednesday's changes (1 GB)
  Thursday: Incremental — only Thursday's changes (3 GB)

  To restore Thursday's state:
  Monday full + Tuesday incr + Wednesday incr + Thursday incr = 4 sets needed

  Smallest/fastest backups; slowest/most complex restore.
```

### Differential Backup

Copies **all data changed since the LAST FULL backup**.

```
  Monday: Full backup (500 GB)
  Tuesday: Differential — all changes since Monday (2 GB)
  Wednesday: Differential — all changes since Monday (5 GB, grows daily)
  Thursday: Differential — all changes since Monday (9 GB)

  To restore Thursday's state:
  Monday full + Thursday differential = 2 sets needed

  Medium storage; medium restore speed. Good compromise.
```

### Comparison

| Strategy         | Backup Size    | Backup Speed | Restore Speed | Complexity |
| ---------------- | -------------- | ------------ | ------------- | ---------- |
| **Full**         | Large          | Slow         | **Fast**      | Simple     |
| **Incremental**  | **Smallest**   | **Fast**     | Slow          | Complex    |
| **Differential** | Medium (grows) | Medium       | Medium        | Moderate   |

---

## 3. Backup Storage Locations

### Local Backup

```
  Examples: external HDD, USB drive, secondary internal drive

  Pros:  Fast backup/restore; cheap; no internet needed
  Cons:  Same physical location → fire/flood/theft destroys backup too
  Best for: Quick recovery from accidental deletion or software failure
```

### Network Backup (NAS / File Server)

```
  Examples: Network-Attached Storage (NAS), company file server

  Pros:  Centralized management; multiple machines back up to one place
  Cons:  Still at same physical site → vulnerable to site-wide disasters
  Best for: Office environments with multiple computers
```

### Offsite / Cloud Backup

```
  Examples: AWS S3, Azure Blob, Backblaze, Google Drive, tape in a vault

  Pros:  Geographically separated → survives fire/flood/earthquake at main site
  Cons:  Slow backup/restore over internet; ongoing subscription costs
  Best for: Disaster recovery; compliance; anything where data loss = business failure
```

### The 3-2-1 Rule (Industry Standard)

```
  Keep at least:
  3 copies of your data  (1 original + 2 backups)
  2 different media types (e.g., HDD + cloud)
  1 copy offsite         (protects against site-wide disaster)

  Example:
  - Original: your laptop SSD
  - Local backup: external USB hard drive at home
  - Offsite backup: cloud service (AWS S3, iCloud, Backblaze)

  This setup survives: accidental deletion, hardware failure, AND house fire.
```

---

## 4. Recovery Mechanisms

### File-Level Recovery

Restores **individual files or directories** — most common, everyday use.

```
  "I accidentally deleted my presentation from 3 days ago"
  → Browse backup from 3 days ago → select presentation.pptx → restore
```

### System-Level Recovery (System Image)

Restores the **entire OS installation** including all apps and settings.

```
  Use case: ransomware attack encrypted everything,
            OS corruption, catastrophic hardware failure

  Process:
  1. Boot from recovery USB/DVD
  2. Point to full system image backup
  3. Restore overwrites entire disk with the backup image
  4. System boots as it was at backup time
```

### Bare-Metal Recovery

Restores a complete system to **new or completely wiped hardware** — no existing OS required.

```
  Use case: Original server hardware destroyed → need to restore to new server

  Process:
  1. Boot new machine from recovery media
  2. Media detects hardware, loads drivers automatically
  3. Restore complete system image from backup
  4. System runs on the new hardware

  → "Bare metal" = starting from scratch with no software installed
```

### Point-in-Time Recovery

Restores data to its state at **a specific moment in the past**.

```
  Use case: Database corrupted at 2:03 PM; everything before was fine

  With point-in-time recovery: restore to 2:00 PM → 3 minutes of data lost
  Without it: restore to last night's backup → entire day's transactions lost

  Requires: continuous backups OR transaction logs (databases) OR frequent snapshots
```

---

## 5. File System Features for Backup/Recovery

### Snapshots

A **snapshot** captures the entire file system state at an instant using copy-on-write.

```
  At 9:00 AM: OS takes snapshot of /home

  Copy-on-write:
  If you modify file A after the snapshot:
  → OS writes new data to new blocks
  → Old blocks (from 9:00 AM snapshot) are preserved untouched

  Benefits:
  - Instant creation (no data copy needed — only changes accumulate)
  - Minimal storage overhead until many files are changed
  - Users can restore any file to any previous snapshot instantly

  Examples: APFS snapshots (macOS Time Machine), Btrfs snapshots, ZFS snapshots,
            Windows Volume Shadow Copy (VSS)
```

### Journaling (Crash Recovery)

As covered earlier: journaling protects against crash-induced inconsistency — NOT against deletion or hardware failure.

### Version Control (Per-File)

Some file systems or services track file history automatically:

```
  Google Drive "Version history" → see doc as it was 30 days ago
  Dropbox "Rewind" → browse any file version
  Windows File History → automatic versioned backups of Documents, Pictures, etc.
```

---

## 6. Disaster Recovery Planning: RTO and RPO

### Recovery Time Objective (RTO)

**"How long can we be down?"** — maximum acceptable downtime after a disaster.

```
  Example RTO definitions:
  Banking core system:   RTO = 15 minutes (downtime = regulatory violations)
  Internal wiki:         RTO = 4 hours
  Annual report archive: RTO = 48 hours

  Low RTO → need real-time replication, hot standby, or rapid restore from near-instant snapshot
  High RTO → daily tape backup is sufficient
```

### Recovery Point Objective (RPO)

**"How much data can we afford to lose?"** — maximum acceptable data loss measured in time.

```
  Example RPO definitions:
  Financial transactions: RPO = 0 (zero loss) → requires real-time replication
  Email server:           RPO = 1 hour → backups every hour
  Development code repo:  RPO = 1 day → daily backup sufficient

  RPO directly determines backup frequency:
  RPO = 1 hour → backup at least every hour
  RPO = 0      → continuous replication (not traditional backup)
```

```
                          Time →
  Last backup ────────────────────────────────── Disaster
               ←───── RPO window ─────►│
                                        │←── RTO window ──► System restored

  RPO: How far back do we go?  RTO: How fast do we get back?
```

---

## 7. Best Practices

| Practice                         | Why it matters                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------ |
| **Follow the 3-2-1 rule**        | Covers single disk failure, site disaster, and quick local restore                         |
| **Automate everything**          | Manual backups get skipped; automation runs consistently                                   |
| **Test restores regularly**      | "Your backup is untested until you've restored from it" — many backups are silently broken |
| **Encrypt backup data**          | Backups contain all your sensitive data; secure them like the originals                    |
| **Document recovery procedures** | Disaster happens at 3 AM when the expert is unavailable                                    |
| **Define RTO and RPO**           | Business requirements drive technology choices                                             |
| **Verify backup integrity**      | Run checksums/hash verification after each backup                                          |

**Backup testing reality check:**

```
  Organizations that discover their backup is broken:

  A: During routine monthly restore test → Minor inconvenience, fix and redo
  B: During a real disaster, first restore attempt → Catastrophic data loss

  Always be organization A.
  Test restores quarterly at minimum. Test the FULL recovery, not just backup creation.
```

---

## 8. Key Takeaways

- **Backup** = duplicate copies of data; **Recovery** = restoring from those copies after loss
- **Full backup**: everything every time — simple restore, large storage
- **Incremental backup**: only changes since last backup — fast/small, but needs entire chain to restore
- **Differential backup**: all changes since last FULL backup — medium storage, two-set restore
- **3-2-1 rule**: 3 copies, 2 media types, 1 offsite — covers most failure scenarios
- **Recovery types**: file-level (single files), system-level (full OS), bare-metal (new hardware), point-in-time (specific moment)
- **Snapshots** = instant file system captures using copy-on-write; enable per-file version recovery
- **RTO (Recovery Time Objective)** = max acceptable downtime → drives restore speed requirements
- **RPO (Recovery Point Objective)** = max acceptable data loss in time → drives backup frequency
- **Never skip restore testing** — the backup is only as good as your ability to actually restore from it
- **Encrypt backups** — backups contain copies of all sensitive data and face the same theft risks as the originals
- **Journaling ≠ backup**: journaling protects against crashes; backups protect against hardware failure, deletion, and ransomware
