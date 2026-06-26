# Virtual Machines and Hypervisors

> A **virtual machine (VM)** is a software-based computer — it runs a full OS and apps just like a real machine but exists entirely as files on a host; a **hypervisor** (VMM) is the software layer that creates and manages VMs, intercepting hardware requests and sharing physical resources among multiple isolated guest OSes simultaneously.

---

## Table of Contents

1. [What is a Virtual Machine?](#1-what-is-a-virtual-machine)
2. [What is a Hypervisor?](#2-what-is-a-hypervisor)
3. [Type 1 vs Type 2 Hypervisors](#3-type-1-vs-type-2-hypervisors)
4. [Virtualization Techniques](#4-virtualization-techniques)
5. [Resource Management in VMs](#5-resource-management-in-vms)
6. [VM Lifecycle Management](#6-vm-lifecycle-management)
7. [Security in Virtualization](#7-security-in-virtualization)
8. [Performance Considerations](#8-performance-considerations)
9. [Practical Use Cases](#9-practical-use-cases)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What is a Virtual Machine?

A **virtual machine (VM)** is a software emulation of a complete physical computer. It runs an OS and applications just like a real machine, but it exists as files on a host system. The guest OS inside the VM has no idea it's not running on real hardware.

**Analogy:** A house within a house. The outer house provides physical structure (walls, electricity, plumbing). The inner house has its own rooms, furniture, and occupants that operate independently — unaware of the outer structure.

```
  Without virtualization:
  [Physical Server]
  └── One OS + One Application
  → 10 servers needed for 10 applications → expensive, wasteful

  With virtualization:
  [Physical Server]
  └── Hypervisor
      ├── VM1: Linux + Web Server
      ├── VM2: Windows + Database
      ├── VM3: Ubuntu + App Server
      └── VM4: CentOS + Mail Server
  → 1 server runs 4 isolated systems → efficient!
```

### Key Components of a VM

| Component              | Description                                                              |
| ---------------------- | ------------------------------------------------------------------------ |
| **Virtual CPU (vCPU)** | Allocated share of physical CPU cores                                    |
| **Virtual Memory**     | Reserved RAM from the host system                                        |
| **Virtual Disk**       | File(s) on the host that simulate a hard drive (`.vmdk`, `.vdi`, `.vhd`) |
| **Virtual NIC**        | Software-based network interface card                                    |
| **Guest OS**           | The OS running inside the VM (Windows, Linux, etc.)                      |

### Benefits of VMs

| Benefit                 | Explanation                                                |
| ----------------------- | ---------------------------------------------------------- |
| **Isolation**           | Each VM is sandboxed — crashes/malware can't affect others |
| **Resource efficiency** | Multiple VMs share one physical machine                    |
| **Portability**         | VM is just files — copy it to another host                 |
| **Safe testing**        | Break things in a VM without risk to real systems          |
| **Legacy support**      | Run old OS versions on modern hardware                     |

---

## 2. What is a Hypervisor?

A **hypervisor** (also called a Virtual Machine Monitor / VMM) is the software layer that sits between physical hardware and VMs. It:

- Creates and manages VMs
- Allocates CPU, memory, disk, and network to each VM
- Intercepts hardware requests from guest OSes and fulfills them on the physical hardware
- Enforces isolation between VMs

**Analogy:** An apartment building manager. Allocates apartments (VMs) to tenants (guest OSes), handles utilities (hardware resources), and ensures residents don't interfere with each other.

```
  How a hypervisor handles a VM memory request:

  Guest OS inside VM1 requests: "Allocate 512 MB of memory"
         ↓
  Hypervisor intercepts the request
         ↓
  Checks available physical RAM on host
         ↓
  Allocates 512 MB from physical RAM to VM1
         ↓
  Creates virtual-to-physical memory mapping
         ↓
  Tells guest OS: "Done — here's your memory"
         ↓
  Guest OS thinks it allocated real RAM (it has no idea it's virtual)
```

---

## 3. Type 1 vs Type 2 Hypervisors

### Type 1 — Bare Metal Hypervisor

Runs **directly on physical hardware** — no host OS underneath. Has direct access to all hardware resources.

```
  ┌─────────────────────────┐
  │ VM1  │ VM2  │ VM3  │ VM4│
  ├─────────────────────────┤
  │   Type 1 Hypervisor     │  ← runs directly on metal
  ├─────────────────────────┤
  │     Physical Hardware   │
  └─────────────────────────┘

  Examples: VMware ESXi, Microsoft Hyper-V, Xen, KVM (Linux)
```

**Analogy:** A building owner who owns the building and allocates apartments directly.

### Type 2 — Hosted Hypervisor

Runs **as an application on top of a host OS**. The host OS manages hardware; the hypervisor manages VMs on top.

```
  ┌──────────────────┐
  │  VM1  │  VM2    │
  ├──────────────────┤
  │ Type 2 Hypervisor│  ← an application on host OS
  ├──────────────────┤
  │  Host OS         │  ← Windows, macOS, Linux
  ├──────────────────┤
  │  Physical Hardware│
  └──────────────────┘

  Examples: Oracle VirtualBox, VMware Workstation, Parallels Desktop (Mac)
```

**Analogy:** A subletter who rents from a main tenant — extra layer between them and the building owner.

### Type 1 vs Type 2 Comparison

| Feature               | Type 1 (Bare Metal)             | Type 2 (Hosted)                     |
| --------------------- | ------------------------------- | ----------------------------------- |
| **Installed on**      | Directly on hardware            | On top of host OS                   |
| **Performance**       | Higher — direct hardware access | Lower — extra OS layer overhead     |
| **Resource overhead** | Minimal                         | Higher (host OS consumes resources) |
| **Use case**          | Enterprise data centers, cloud  | Personal use, dev/test              |
| **Setup complexity**  | More complex                    | Easy to install                     |
| **Cost**              | Often commercial/expensive      | Many free options                   |
| **Security**          | Better isolation                | Depends on host OS security         |
| **Examples**          | VMware ESXi, Hyper-V, KVM       | VirtualBox, VMware Workstation      |

---

## 4. Virtualization Techniques

### Full Virtualization

The hypervisor provides a **complete simulation of hardware**. The guest OS runs **unmodified** — it doesn't know it's in a VM.

```
  Guest OS: "I need to write to memory address 0x1000"
  Hypervisor: intercepts → binary translation → executes safely
  Guest OS: (has no idea this happened)
```

- Technique: **binary translation** — hypervisor catches privileged instructions and translates them
- Any unmodified OS (Windows, Linux) can run as a guest
- Slight overhead from binary translation

### Paravirtualization

The guest OS is **modified** to know it's running in a VM. Instead of trying to execute hardware instructions directly, it makes **hypercalls** directly to the hypervisor.

```
  Guest OS (modified): "Hypervisor, please allocate memory for me"  ← hypercall
  Hypervisor: handles it directly

  vs Full Virtualization:
  Guest OS (unmodified): tries to execute hardware instruction
  Hypervisor: intercepts, translates, executes  ← more overhead
```

- Faster (no binary translation overhead)
- Requires modifying guest OS (not always possible, e.g., proprietary Windows)
- Example: Xen paravirtualized Linux guests

### Hardware-Assisted Virtualization

Modern CPUs include **hardware extensions** specifically designed for virtualization — Intel VT-x and AMD-V. The CPU itself handles many virtualization tasks that previously required software.

```
  Intel VT-x adds:
  - VMX root mode (hypervisor runs here)
  - VMX non-root mode (guest OS runs here — cannot accidentally escape)
  - VMENTER / VMEXIT instructions for controlled transitions

  CPU handles: privilege level switching, memory isolation, trapping privileged ops
  → Hypervisor doesn't need binary translation
  → Near-native performance for most workloads
```

Today, essentially all Type 1 and Type 2 hypervisors use hardware-assisted virtualization when available.

---

## 5. Resource Management in VMs

### CPU Virtualization

```
  Physical server: 16 cores
  VMs: VM1(4 vCPUs), VM2(8 vCPUs), VM3(2 vCPUs), VM4(2 vCPUs) = 16 vCPUs

  Hypervisor scheduler:
  - Maps vCPUs to physical cores
  - Applies time-slicing when vCPU demand > physical cores
  - Respects VM priority levels and reservations

  CPU over-commitment:
  Can assign more vCPUs than physical cores exist (acceptable if VMs aren't
  all 100% busy simultaneously — similar to memory over-commit)
```

### Memory Virtualization

Two levels of address translation:

```
  Normal OS:           Virtual Address → Physical Address

  VM:  Guest Virtual → Guest Physical → Host Physical
                              ↑
                    Hypervisor manages this mapping
```

**Memory optimization techniques:**

- **Memory ballooning:** hypervisor driver inside guest OS inflates a "balloon" to reclaim unused RAM back to the host pool
- **Transparent page sharing:** hypervisor identifies identical pages across VMs (e.g., same OS kernel) and maps them to same physical page → saves RAM
- **Memory swapping:** hypervisor can swap VM memory pages to disk under pressure

### Storage Virtualization

VM's disk = a file on the host (`.vmdk` for VMware, `.vdi` for VirtualBox, `.vhd` for Hyper-V). Hypervisor translates guest I/O operations into file operations on host storage.

### Network Virtualization

```
  Physical NIC ← shared by all VMs via →  Virtual Switch (vSwitch)
                                          ├── VM1 Virtual NIC
                                          ├── VM2 Virtual NIC
                                          └── VM3 Virtual NIC

  VMs appear as separate machines on the network
  vSwitch routes packets between VMs and to the physical network
```

---

## 6. VM Lifecycle Management

### Snapshots

A **snapshot** captures the complete state of a VM at an instant — CPU registers, memory contents, disk state.

```
  Before risky operation:  TAKE SNAPSHOT
  Run experiment / install untested software
  If it goes wrong:        REVERT TO SNAPSHOT  ← entire state restored in seconds
  If it works:             DELETE SNAPSHOT     ← free up disk space
```

### Cloning

**Clone = exact copy of a VM.** Instead of installing and configuring a new OS from scratch, clone an existing VM. Used for:

- Deploying many identical web/app servers
- Creating dev/test copies of production environments

### Live Migration

Moving a running VM from one physical host to another **without shutting it down** — zero downtime.

```
  VM running on Host A
  ↓ (memory pages transferred to Host B in background)
  ↓ (VM continues running on Host A during transfer)
  ↓ (final brief pause: last memory pages + CPU state transferred)
  ↓ (VM resumes on Host B)
  Total downtime: milliseconds to a few seconds

  Use cases: hardware maintenance, load balancing, disaster recovery
```

---

## 7. Security in Virtualization

### VM Isolation

The hypervisor enforces strict isolation — in theory, a VM cannot read another VM's memory or storage. This is similar to process isolation but at a larger scale (whole OS vs single process).

### VM Escape

**VM escape** is when malicious code inside a VM exploits a hypervisor vulnerability to execute on the host or access other VMs. Extremely serious — attacker breaks out of the sandbox entirely.

- Rare but has occurred (e.g., CVE-2015-3456 "VENOM" affecting QEMU)
- Defense: keep hypervisor patched, minimal attack surface, restrict VM privileges

### Hypervisor as a Security Target

If the hypervisor is compromised, the attacker controls ALL VMs on that host. This makes hypervisor security critical — keep it minimal and updated.

---

## 8. Performance Considerations

### Overhead Sources

| Source            | What Causes Overhead                                             |
| ----------------- | ---------------------------------------------------------------- |
| CPU               | vCPU scheduling, instruction interception/translation            |
| Memory            | Extra address translation layer (guest physical → host physical) |
| I/O               | Device emulation, virtual switch packet processing               |
| Context switching | Switching between VMs (VMENTER/VMEXIT on each switch)            |

With hardware-assisted virtualization, **typical overhead is 5–10%** for most workloads. VMs perform very close to bare metal.

### Optimization Tips

```
  ✓ Use paravirtualized drivers (virtio for Linux guests) — faster than emulated hardware
  ✓ Enable hardware virtualization in BIOS (VT-x / AMD-V)
  ✓ Don't over-commit memory (causes ballooning/swapping — huge performance hit)
  ✓ Use dedicated storage I/O paths (iSCSI, NVMe pass-through) for I/O-intensive VMs
  ✓ Disable unnecessary services inside guest OS
  ✓ Match vCPU count to actual workload (more vCPUs = more scheduling overhead)
```

---

## 9. Practical Use Cases

| Use Case                    | How VMs Help                                                  |
| --------------------------- | ------------------------------------------------------------- |
| **Cloud computing (IaaS)**  | AWS EC2, Azure VMs — each "rented server" is a VM             |
| **Dev/Test environments**   | Test on Windows/Linux/macOS simultaneously on one laptop      |
| **Server consolidation**    | Replace 20 physical servers with 20 VMs on 1–2 physical hosts |
| **Disaster recovery**       | Snapshot VM; restore in minutes after failure                 |
| **Legacy software support** | Run 10-year-old OS in a VM on modern hardware                 |
| **CI/CD pipelines**         | Spin up clean VMs for each build, test, then destroy          |
| **Security sandboxing**     | Open suspicious files inside a disposable VM                  |

---

## 10. Code Examples

> Working code that demonstrates virtual machine and hypervisor concepts — VM lifecycle management and memory overcommitment — in practice.

### C++ — Simple Version

Simulate a Type 2 hypervisor managing multiple VMs: create, start, pause, snapshot, and restore.

```cpp
#include <iostream>
#include <string>
#include <vector>

// Simulated Type 2 Hypervisor
// Runs on a host OS and manages VMs.
// Each VM has virtual CPU, virtual RAM, virtual disk, and a lifecycle state.

enum class VMState { STOPPED, RUNNING, PAUSED };

struct VM {
    std::string name;
    int vcpus;           // Number of virtual CPUs
    int ramMB;           // Virtual RAM in MB
    int diskGB;          // Virtual disk in GB
    VMState state = VMState::STOPPED;
    std::string snapshotData;   // Stored snapshot (empty if none)

    VM(std::string n, int c, int r, int d)
        : name(n), vcpus(c), ramMB(r), diskGB(d) {}

    std::string stateStr() const {
        if (state == VMState::RUNNING) return "RUNNING";
        if (state == VMState::PAUSED)  return "PAUSED";
        return "STOPPED";
    }
};

class Hypervisor {
    std::vector<VM> vms;
    int physicalRAM_MB;
    int allocatedRAM = 0;

    VM* findVM(const std::string& name) {
        for (auto& vm : vms) if (vm.name == name) return &vm;
        return nullptr;
    }

public:
    Hypervisor(int ram) : physicalRAM_MB(ram) {
        std::cout << "Hypervisor started — Host RAM: " << ram << " MB\n\n";
    }

    void createVM(const std::string& name, int vcpus, int ramMB, int diskGB) {
        if (allocatedRAM + ramMB > physicalRAM_MB) {
            std::cout << "  ERROR: Not enough host RAM to create '" << name << "'\n";
            return;
        }
        vms.push_back({name, vcpus, ramMB, diskGB});
        allocatedRAM += ramMB;
        std::cout << "[CREATE] VM '" << name << "' — vCPUs:" << vcpus
                  << " RAM:" << ramMB << "MB Disk:" << diskGB << "GB\n";
    }

    void start(const std::string& name) {
        VM* vm = findVM(name);
        if (!vm || vm->state == VMState::RUNNING) return;
        vm->state = VMState::RUNNING;
        std::cout << "[START]   '" << name << "' -> RUNNING\n";
    }

    void pause(const std::string& name) {
        VM* vm = findVM(name);
        if (!vm || vm->state != VMState::RUNNING) return;
        vm->state = VMState::PAUSED;
        std::cout << "[PAUSE]   '" << name << "' -> PAUSED\n";
    }

    void snapshot(const std::string& name) {
        VM* vm = findVM(name);
        if (!vm) return;
        vm->snapshotData = "snap[state=" + vm->stateStr() +
                           ",ram=" + std::to_string(vm->ramMB) + "MB]";
        std::cout << "[SNAP]    '" << name << "' -> snapshot: " << vm->snapshotData << "\n";
    }

    void restore(const std::string& name) {
        VM* vm = findVM(name);
        if (!vm || vm->snapshotData.empty()) {
            std::cout << "  No snapshot for '" << name << "'\n";
            return;
        }
        vm->state = VMState::STOPPED;   // Simplified: restore to stopped state
        std::cout << "[RESTORE] '" << name << "' restored from: " << vm->snapshotData << "\n";
    }

    void listVMs() const {
        std::cout << "\n--- VM List (RAM: " << allocatedRAM << "/" << physicalRAM_MB << " MB) ---\n";
        for (auto& vm : vms) {
            std::cout << "  " << vm.name << " | vCPUs:" << vm.vcpus
                      << " | RAM:" << vm.ramMB << "MB | Disk:" << vm.diskGB
                      << "GB | " << vm.stateStr() << "\n";
        }
    }
};

int main() {
    Hypervisor hv(4096);   // Host has 4 GB RAM

    hv.createVM("WebServer",  2, 512,  20);
    hv.createVM("Database",   4, 2048, 100);
    hv.createVM("MailServer", 2, 512,  50);

    hv.listVMs();

    std::cout << "\n--- Operations ---\n";
    hv.start("WebServer");
    hv.start("Database");
    hv.snapshot("Database");   // Save state while running
    hv.pause("Database");
    hv.restore("Database");    // Roll back to snapshot

    hv.listVMs();
    return 0;
}
```

### C++ — Medium / LeetCode Style

Simulate memory overcommitment: hypervisor tracks more committed guest RAM than physical RAM, then uses balloon driver to reclaim from idle VMs.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

// Memory overcommitment: hypervisor promises more total RAM to VMs than
// physical RAM available. When actual usage pressure is high, the balloon
// driver inflates inside the most idle guest, forcing that guest to release
// pages back to the hypervisor for redistribution.

struct GuestVM {
    std::string name;
    int committedMB;     // Total RAM promised to this VM
    int usedMB;          // RAM the guest is actually using right now
    int balloonMB = 0;   // Pages reclaimed by balloon (returned to host)

    int effectiveMB() const { return usedMB - balloonMB; }
    int freeMB()      const { return committedMB - usedMB - balloonMB; }
};

class OvercommittedHypervisor {
    int physicalMB;
    std::vector<GuestVM> guests;

    int totalCommitted() const {
        int t = 0; for (auto& g : guests) t += g.committedMB; return t;
    }
    int totalEffective() const {
        int t = 0; for (auto& g : guests) t += g.effectiveMB(); return t;
    }

public:
    OvercommittedHypervisor(int physMB) : physicalMB(physMB) {}

    void addGuest(const std::string& name, int commitMB, int usedMB) {
        guests.push_back({name, commitMB, usedMB});
        std::cout << "[ADD] " << name << ": committed=" << commitMB
                  << "MB, using=" << usedMB << "MB\n";
    }

    // Reclaim 'reclaimMB' from the guest with the most free memory
    void activateBalloon(int reclaimMB) {
        GuestVM* target = nullptr;
        int maxFree = 0;
        for (auto& g : guests) {
            if (g.freeMB() > maxFree) { maxFree = g.freeMB(); target = &g; }
        }
        if (!target) { std::cout << "No balloon candidate\n"; return; }
        int inflate = std::min(reclaimMB, maxFree);
        target->balloonMB += inflate;
        std::cout << "\n[BALLOON] Inflated '" << target->name << "' by "
                  << inflate << "MB -> host reclaimed " << inflate << "MB\n";
    }

    void printStatus() const {
        std::cout << "\n=== Memory Status ===\n";
        std::cout << "Physical:      " << physicalMB << " MB\n";
        std::cout << "Committed:     " << totalCommitted() << " MB"
                  << (totalCommitted() > physicalMB ? "  [OVERCOMMITTED]" : "") << "\n";
        std::cout << "Effective use: " << totalEffective() << " MB"
                  << (totalEffective() > physicalMB ? "  [PRESSURE!]" : "  [OK]") << "\n\n";
        for (auto& g : guests) {
            std::cout << "  " << g.name
                      << " committed:" << g.committedMB
                      << " using:" << g.usedMB
                      << " balloon:" << g.balloonMB
                      << " net:" << g.effectiveMB() << "MB\n";
        }
    }
};

int main() {
    OvercommittedHypervisor hv(4096);   // 4 GB physical

    // Total committed = 6 GB but physical = 4 GB — overcommitted!
    hv.addGuest("VM-Web",   2048, 800);    // Committed 2 GB, using only 800 MB
    hv.addGuest("VM-DB",    2048, 1900);   // Committed 2 GB, using 1.9 GB
    hv.addGuest("VM-Cache", 2048, 600);    // Committed 2 GB, using only 600 MB

    hv.printStatus();

    std::cout << "--- Memory pressure detected: activating balloon ---\n";
    hv.activateBalloon(800);   // Reclaim 800 MB from the most idle guest

    hv.printStatus();
    return 0;
}
```

### Python — Simple Version

Type 2 hypervisor managing VM lifecycle: create, start, pause, snapshot, restore.

```python
# Simulated Type 2 Hypervisor
# Manages multiple VMs. Each VM has virtual CPU, RAM, disk, and state.
# Operations: create, start, pause, snapshot, restore.

class VM:
    def __init__(self, name, vcpus, ram_mb, disk_gb):
        self.name     = name
        self.vcpus    = vcpus
        self.ram_mb   = ram_mb
        self.disk_gb  = disk_gb
        self.state    = 'STOPPED'   # STOPPED, RUNNING, PAUSED
        self.snapshot = None        # Stored snapshot dict (None if none taken)

    def __repr__(self):
        return (f"VM({self.name!r}, vCPUs={self.vcpus}, "
                f"RAM={self.ram_mb}MB, Disk={self.disk_gb}GB, state={self.state!r})")


class Hypervisor:
    def __init__(self, host_ram_mb):
        self.host_ram_mb   = host_ram_mb
        self.allocated_ram = 0
        self.vms           = {}
        print(f"Hypervisor started — Host RAM: {host_ram_mb} MB\n")

    def _get(self, name):
        vm = self.vms.get(name)
        if not vm:
            print(f"  VM '{name}' not found")
        return vm

    def create_vm(self, name, vcpus, ram_mb, disk_gb):
        if self.allocated_ram + ram_mb > self.host_ram_mb:
            print(f"  ERROR: Not enough RAM to create '{name}'")
            return
        vm = VM(name, vcpus, ram_mb, disk_gb)
        self.vms[name] = vm
        self.allocated_ram += ram_mb
        print(f"[CREATE] {vm}")

    def start(self, name):
        vm = self._get(name)
        if vm and vm.state != 'RUNNING':
            vm.state = 'RUNNING'
            print(f"[START]   '{name}' -> RUNNING")

    def pause(self, name):
        vm = self._get(name)
        if vm and vm.state == 'RUNNING':
            vm.state = 'PAUSED'
            print(f"[PAUSE]   '{name}' -> PAUSED")

    def take_snapshot(self, name):
        vm = self._get(name)
        if vm:
            vm.snapshot = {'state': vm.state, 'ram_mb': vm.ram_mb}
            print(f"[SNAP]    '{name}' snapshot saved: {vm.snapshot}")

    def restore_snapshot(self, name):
        vm = self._get(name)
        if vm and vm.snapshot:
            vm.state = vm.snapshot['state']
            print(f"[RESTORE] '{name}' restored -> state={vm.state}")
        elif vm:
            print(f"  No snapshot for '{name}'")

    def list_vms(self):
        print(f"\n--- VMs (RAM: {self.allocated_ram}/{self.host_ram_mb} MB) ---")
        for vm in self.vms.values():
            print(f"  {vm}")


# Demo
hv = Hypervisor(host_ram_mb=4096)
hv.create_vm("WebServer",  vcpus=2, ram_mb=512,  disk_gb=20)
hv.create_vm("Database",   vcpus=4, ram_mb=2048, disk_gb=100)
hv.create_vm("MailServer", vcpus=2, ram_mb=512,  disk_gb=50)
hv.list_vms()

print("\n--- Operations ---")
hv.start("WebServer")
hv.start("Database")
hv.take_snapshot("Database")
hv.pause("Database")
hv.restore_snapshot("Database")
hv.list_vms()
```

### Python — Medium Level

Memory overcommitment simulation with balloon driver reclamation and duplicate page-sharing detection.

```python
import hashlib

# Memory overcommitment: hypervisor commits more total guest RAM than physical.
# Balloon driver: inflates inside idle guests to reclaim pages for the host.
# Page sharing: VMs running the same OS share identical pages (one physical copy).

class GuestVM:
    def __init__(self, name, commit_mb, active_mb):
        self.name       = name
        self.commit_mb  = commit_mb    # Promised to guest
        self.active_mb  = active_mb    # Guest is actually using
        self.balloon_mb = 0            # Reclaimed by balloon driver
        self.pages      = {}           # {page_id: content_hash}

    def net_cost(self):
        return self.active_mb - self.balloon_mb

    def free_mb(self):
        return self.commit_mb - self.active_mb - self.balloon_mb

    def load_pages(self, page_data):
        """Load simulated pages: {page_id: content_string}."""
        for pid, content in page_data.items():
            self.pages[pid] = hashlib.md5(content.encode()).hexdigest()


class OvercommittingHypervisor:
    def __init__(self, physical_mb):
        self.physical_mb  = physical_mb
        self.guests       = {}
        self.pages_saved  = 0

    def add_guest(self, vm: GuestVM):
        self.guests[vm.name] = vm
        print(f"[ADD] {vm.name}: committed={vm.commit_mb}MB, active={vm.active_mb}MB")

    def inflate_balloon(self, target_name, inflate_mb):
        """Force a specific guest to release memory back to the host."""
        vm = self.guests[target_name]
        actual = min(inflate_mb, vm.free_mb())
        vm.balloon_mb += actual
        print(f"[BALLOON] Inflated '{vm.name}' by {actual}MB -> host reclaimed {actual}MB")

    def detect_shared_pages(self):
        """Scan all VMs for duplicate page content -> merge to one physical copy."""
        seen = {}   # {hash: first_vm_name}
        for vm in self.guests.values():
            for pid, h in vm.pages.items():
                if h in seen:
                    self.pages_saved += 1
                    print(f"  [SHARE] hash {h[:8]}... shared: '{seen[h]}' and '{vm.name}' "
                          f"-> 1 physical copy (saved 1 page frame)")
                else:
                    seen[h] = vm.name

    def print_status(self):
        total_committed = sum(g.commit_mb for g in self.guests.values())
        total_effective = sum(g.net_cost() for g in self.guests.values())
        print(f"\n{'='*48}")
        print(f"Physical RAM:    {self.physical_mb} MB")
        print(f"Total committed: {total_committed} MB "
              f"({'OVER' if total_committed > self.physical_mb else 'OK'})")
        print(f"Total effective: {total_effective} MB "
              f"({'PRESSURE' if total_effective > self.physical_mb else 'fits'})")
        print(f"Pages saved (sharing): {self.pages_saved}")
        for g in self.guests.values():
            print(f"  {g.name:<12}: commit={g.commit_mb}MB active={g.active_mb}MB "
                  f"balloon={g.balloon_mb}MB net={g.net_cost()}MB")


# Demo
hv = OvercommittingHypervisor(physical_mb=4096)

vm1 = GuestVM("VM-Web",   commit_mb=2048, active_mb=800)
vm2 = GuestVM("VM-DB",    commit_mb=2048, active_mb=1900)
vm3 = GuestVM("VM-Cache", commit_mb=2048, active_mb=600)
for vm in [vm1, vm2, vm3]:
    hv.add_guest(vm)

# All three VMs run the same OS base — they share kernel and libc pages
vm1.load_pages({'p1': 'kernel_page', 'p2': 'libc_page', 'p3': 'web_data'})
vm2.load_pages({'p1': 'kernel_page', 'p2': 'libc_page', 'p3': 'db_data'})
vm3.load_pages({'p1': 'kernel_page', 'p2': 'cache_data'})

hv.print_status()

print("\n--- Page sharing scan ---")
hv.detect_shared_pages()

print("\n--- Memory pressure: inflate balloon in VM-Web ---")
hv.inflate_balloon("VM-Web", 600)

hv.print_status()
```

---

## 11. Key Takeaways

- A **VM** is a software emulation of a full computer — guest OS has no idea it's not on real hardware
- A **hypervisor** (VMM) sits between hardware and VMs; intercepts guest hardware requests and manages resource allocation
- **Type 1 (bare metal):** runs directly on hardware (ESXi, Hyper-V, KVM) — better performance, used in production/cloud
- **Type 2 (hosted):** runs as an app on host OS (VirtualBox, VMware Workstation) — easier to use, for personal/dev use
- **Full virtualization:** guest OS unmodified; hypervisor uses binary translation; any OS works as guest
- **Paravirtualization:** guest OS modified to make hypercalls; faster, but requires OS modification
- **Hardware-assisted (Intel VT-x / AMD-V):** CPU handles virtualization tasks natively; near-native performance; used by all modern hypervisors
- **Memory virtualization:** guest virtual → guest physical → host physical; two levels of page tables
- **Memory optimization:** ballooning (reclaim idle RAM), transparent page sharing (deduplicate identical pages)
- **Snapshots** capture full VM state for instant rollback; **cloning** copies VMs instantly; **live migration** moves running VMs with near-zero downtime
- **VM escape** = attacker breaks out of VM to reach hypervisor/host — rare but critical; keep hypervisor patched
- **Performance overhead:** ~5–10% with hardware-assisted virtualization; use paravirtualized drivers for best I/O performance
- **Cloud IaaS** (AWS, Azure, GCP) is built on virtualization — every "cloud server" you rent is a VM
