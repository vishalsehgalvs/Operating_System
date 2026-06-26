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

| Component | Description |
|-----------|-------------|
| **Virtual CPU (vCPU)** | Allocated share of physical CPU cores |
| **Virtual Memory** | Reserved RAM from the host system |
| **Virtual Disk** | File(s) on the host that simulate a hard drive (`.vmdk`, `.vdi`, `.vhd`) |
| **Virtual NIC** | Software-based network interface card |
| **Guest OS** | The OS running inside the VM (Windows, Linux, etc.) |

### Benefits of VMs

| Benefit | Explanation |
|---------|-------------|
| **Isolation** | Each VM is sandboxed — crashes/malware can't affect others |
| **Resource efficiency** | Multiple VMs share one physical machine |
| **Portability** | VM is just files — copy it to another host |
| **Safe testing** | Break things in a VM without risk to real systems |
| **Legacy support** | Run old OS versions on modern hardware |

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

| Feature | Type 1 (Bare Metal) | Type 2 (Hosted) |
|---------|-------------------|----------------|
| **Installed on** | Directly on hardware | On top of host OS |
| **Performance** | Higher — direct hardware access | Lower — extra OS layer overhead |
| **Resource overhead** | Minimal | Higher (host OS consumes resources) |
| **Use case** | Enterprise data centers, cloud | Personal use, dev/test |
| **Setup complexity** | More complex | Easy to install |
| **Cost** | Often commercial/expensive | Many free options |
| **Security** | Better isolation | Depends on host OS security |
| **Examples** | VMware ESXi, Hyper-V, KVM | VirtualBox, VMware Workstation |

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

| Source | What Causes Overhead |
|--------|---------------------|
| CPU | vCPU scheduling, instruction interception/translation |
| Memory | Extra address translation layer (guest physical → host physical) |
| I/O | Device emulation, virtual switch packet processing |
| Context switching | Switching between VMs (VMENTER/VMEXIT on each switch) |

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

| Use Case | How VMs Help |
|----------|-------------|
| **Cloud computing (IaaS)** | AWS EC2, Azure VMs — each "rented server" is a VM |
| **Dev/Test environments** | Test on Windows/Linux/macOS simultaneously on one laptop |
| **Server consolidation** | Replace 20 physical servers with 20 VMs on 1–2 physical hosts |
| **Disaster recovery** | Snapshot VM; restore in minutes after failure |
| **Legacy software support** | Run 10-year-old OS in a VM on modern hardware |
| **CI/CD pipelines** | Spin up clean VMs for each build, test, then destroy |
| **Security sandboxing** | Open suspicious files inside a disposable VM |

---

## 10. Key Takeaways

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
