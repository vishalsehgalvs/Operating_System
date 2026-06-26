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

## 7. Code Examples

> Working code that demonstrates backup strategies and snapshot versioning in practice.

### C++ — Simple Version

Simulate full, incremental, and differential backup strategies on an in-memory file system.

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <string>
using namespace std;

// Simulated file system: filename -> version number (incremented on each change)
unordered_map<string, int> fileSystem = {
    {"report.docx", 1}, {"data.csv", 1}, {"config.ini", 1}, {"readme.txt", 1}
};

// Full backup: copy ALL files regardless of modification state
unordered_map<string,int> fullBackup() {
    cout << "[Full Backup] All files:\n";
    for (auto& [f,v] : fileSystem) cout << "  " << f << " (v" << v << ")\n";
    return fileSystem;
}

// Incremental backup: only files changed SINCE last backup (any type)
unordered_map<string,int> incrementalBackup(
        const unordered_map<string,int>& lastBackup) {
    cout << "[Incremental] Changed since last backup:\n";
    unordered_map<string,int> backup;
    for (auto& [f,v] : fileSystem) {
        auto it = lastBackup.find(f);
        if (it == lastBackup.end() || it->second != v) {
            cout << "  " << f << " (v" << v << ")\n";
            backup[f] = v;
        }
    }
    if (backup.empty()) cout << "  (nothing changed)\n";
    return backup;
}

// Differential backup: files changed SINCE the last FULL backup
unordered_map<string,int> differentialBackup(
        const unordered_map<string,int>& lastFull) {
    cout << "[Differential] Changed since last FULL backup:\n";
    unordered_map<string,int> backup;
    for (auto& [f,v] : fileSystem) {
        auto it = lastFull.find(f);
        if (it == lastFull.end() || it->second != v) {
            cout << "  " << f << " (v" << v << ")\n";
            backup[f] = v;
        }
    }
    if (backup.empty()) cout << "  (nothing changed)\n";
    return backup;
}

void modify(const vector<string>& files) {
    for (auto& f : files) {
        fileSystem[f]++;
        cout << "Modified: " << f << " -> v" << fileSystem[f] << "\n";
    }
}

int main() {
    cout << "=== Day 0: Full Backup ===\n";
    auto full = fullBackup();

    cout << "\n--- Day 1: Modify 2 files ---\n";
    modify({"report.docx", "config.ini"});

    cout << "\n=== Day 1: Incremental (since full) ===\n";
    auto inc1 = incrementalBackup(full);

    cout << "\n--- Day 2: Modify 1 more ---\n";
    modify({"data.csv"});

    cout << "\n=== Day 2: Incremental (since day 1) ===\n";
    incrementalBackup(inc1);

    cout << "\n=== Day 2: Differential (always vs. day 0 full) ===\n";
    differentialBackup(full);  // grows each day vs. fixed full baseline

    cout << "\nRestore comparison:\n";
    cout << "  Full:         restore from 1 archive\n";
    cout << "  Incremental:  restore full + day1-inc + day2-inc (chain)\n";
    cout << "  Differential: restore full + day2-diff (just 2 archives)\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style

Snapshot-based versioning system — take, diff, and restore snapshots.

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <string>
using namespace std;

struct Snapshot {
    string label;
    unordered_map<string, string> files;  // filename -> content
};

class VersionedFS {
    unordered_map<string, string> live;   // current file system
    vector<Snapshot>              snaps;  // all snapshots
public:
    void write(const string& name, const string& content) {
        live[name] = content;
        cout << "Write: " << name << " = '" << content << "'\n";
    }
    void remove(const string& name) {
        live.erase(name);
        cout << "Delete: " << name << "\n";
    }

    void snapshot(const string& label) {
        snaps.push_back({label, live});
        cout << "[Snapshot] '" << label << "' " << live.size() << " files\n";
    }

    bool restore(const string& label) {
        for (auto& s : snaps) {
            if (s.label == label) {
                live = s.files;
                cout << "[Restore] Rolled back to '" << label << "'\n";
                printFS();
                return true;
            }
        }
        cout << "Snapshot '" << label << "' not found\n";
        return false;
    }

    void diff(const string& l1, const string& l2) {
        const Snapshot *s1 = nullptr, *s2 = nullptr;
        for (auto& s : snaps) {
            if (s.label == l1) s1 = &s;
            if (s.label == l2) s2 = &s;
        }
        if (!s1 || !s2) { cout << "Snapshot not found\n"; return; }
        cout << "Diff '" << l1 << "' -> '" << l2 << "':\n";
        for (auto& [f,v] : s2->files) {
            if (!s1->files.count(f))                 cout << "  + (added)    " << f << "\n";
            else if (s1->files.at(f) != v)           cout << "  ~ (changed)  " << f << "\n";
        }
        for (auto& [f,_] : s1->files)
            if (!s2->files.count(f))                 cout << "  - (deleted)  " << f << "\n";
    }

    void printFS() {
        cout << "  Live: ";
        for (auto& [n,c] : live) cout << n << "='" << c << "' ";
        cout << "\n";
    }
};

int main() {
    VersionedFS vfs;
    vfs.write("notes.txt",  "initial notes");
    vfs.write("readme.txt", "readme v1");
    vfs.snapshot("v1.0");

    vfs.write("notes.txt",  "updated notes");
    vfs.write("data.csv",   "a,b,c");
    vfs.remove("readme.txt");
    vfs.snapshot("v2.0");

    vfs.diff("v1.0", "v2.0");
    cout << "\nRestoring to v1.0:\n";
    vfs.restore("v1.0");
    return 0;
}
```

### Python — Simple Version

Simulate full, incremental, and differential backup strategies on an in-memory file dictionary.

```python
# Simulate Full, Incremental, and Differential backup strategies

# Simulated file system: filename -> version number
file_system = {
    "report.docx": 1,
    "data.csv":    1,
    "config.ini":  1,
    "readme.txt":  1,
}

def full_backup():
    """Copy ALL files — largest backup, fastest restore."""
    backup = dict(file_system)
    print("[Full Backup] Copied:", list(backup.keys()))
    return backup

def incremental_backup(last_backup: dict):
    """Only files changed SINCE the last backup (any type)."""
    changed = {f: v for f, v in file_system.items()
               if last_backup.get(f) != v or f not in last_backup}
    print(f"[Incremental] Changed since last backup: {list(changed.keys()) or ['(none)']  }")
    return changed

def differential_backup(last_full: dict):
    """Only files changed SINCE the last FULL backup (grows each day)."""
    changed = {f: v for f, v in file_system.items()
               if last_full.get(f) != v or f not in last_full}
    print(f"[Differential] Changed since last full: {list(changed.keys()) or ['(none)']}")
    return changed

def modify(*filenames):
    for f in filenames:
        if f in file_system:
            file_system[f] += 1
            print(f"  Modified: {f} -> v{file_system[f]}")

# --- Demo ---
print("=== Day 0: Full Backup ===")
full = full_backup()

print("\n--- Day 1 ---")
modify("report.docx", "config.ini")
print("\n=== Day 1: Incremental (since full) ===")
inc1 = incremental_backup(full)

print("\n--- Day 2 ---")
modify("data.csv")
print("\n=== Day 2: Incremental (since day 1) ===")
inc2 = incremental_backup(inc1)
print("\n=== Day 2: Differential (always vs. day 0 full) ===")
diff2 = differential_backup(full)

print("\n--- Restore comparison ---")
print(f"  Full backup size:       {len(full)} files")
print(f"  Inc day1 size:          {len(inc1)} files (small, fast backup)")
print(f"  Inc day2 size:          {len(inc2)} files (small, fast backup)")
print(f"  Diff day2 size:         {len(diff2)} files (grows vs. full)")
print("  Restore from inc: full + inc1 + inc2  (chain — slow restore)")
print("  Restore from diff: full + diff2        (only 2 archives)")
```

### Python — Medium Level

Snapshot versioning system — take/diff/restore snapshots with change tracking.

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Dict, List, Optional

@dataclass
class Snapshot:
    label:   str
    created: str
    files:   Dict[str, str] = field(default_factory=dict)  # filename -> content

class VersionedFileSystem:
    def __init__(self):
        self._live:  Dict[str, str]   = {}   # current state
        self._snaps: List[Snapshot]   = []   # history

    def write(self, name: str, content: str):
        self._live[name] = content

    def delete(self, name: str):
        self._live.pop(name, None)

    def snapshot(self, label: str) -> Snapshot:
        """Capture the current state as an immutable snapshot."""
        snap = Snapshot(
            label   = label,
            created = datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            files   = dict(self._live)
        )
        self._snaps.append(snap)
        print(f"[Snapshot] '{label}' — {len(snap.files)} files @ {snap.created}")
        return snap

    def restore(self, label: str) -> bool:
        """Roll back live FS to a previous snapshot."""
        snap = self._find(label)
        if not snap:
            print(f"Snapshot '{label}' not found."); return False
        self._live = dict(snap.files)
        print(f"[Restore] Rolled back to '{label}' — {len(self._live)} files.")
        return True

    def diff(self, label1: str, label2: str):
        """Show what changed between two snapshots."""
        s1, s2 = self._find(label1), self._find(label2)
        if not s1 or not s2: print("Snapshot not found."); return
        all_files = set(s1.files) | set(s2.files)
        print(f"Diff '{label1}' -> '{label2}':")
        changes = 0
        for f in sorted(all_files):
            if   f not in s1.files:            print(f"  + (added)    {f}"); changes += 1
            elif f not in s2.files:            print(f"  - (deleted)  {f}"); changes += 1
            elif s1.files[f] != s2.files[f]:  print(f"  ~ (modified) {f}"); changes += 1
        if not changes: print("  (no changes)")

    def list_snapshots(self):
        print("Snapshots:")
        for s in self._snaps:
            print(f"  [{s.label}]  {s.created}  {len(s.files)} files")

    def show(self):
        print("Live FS:", {k: v[:20] + ("..." if len(v) > 20 else "")
                            for k, v in self._live.items()})

    def _find(self, label: str) -> Optional[Snapshot]:
        return next((s for s in self._snaps if s.label == label), None)

# --- Demo ---
vfs = VersionedFileSystem()
vfs.write("users.json",  '{"users": ["alice"]}')
vfs.write("config.yaml", "debug: false\nport: 8080")
vfs.write("notes.md",    "# Meeting notes v1")
vfs.snapshot("release-1.0")

vfs.write("users.json",  '{"users": ["alice", "bob"]}')
vfs.write("feature.py",  "def new_feature(): pass")
vfs.delete("notes.md")
vfs.snapshot("release-2.0")

print()
vfs.diff("release-1.0", "release-2.0")
print()
vfs.list_snapshots()
print()
vfs.show()
vfs.restore("release-1.0")
vfs.show()
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
