# Booting Process: From Power-On to OS

> The **booting process** is the sequence of steps that transforms a powered-off computer into a fully running OS — from the hardware power-on, through POST, BIOS/UEFI firmware, boot device selection, bootloader, and kernel loading, to the init process and login screen.

---

## Table of Contents

1. [What is Booting?](#1-what-is-booting)
2. [Stage 1: Power-On](#2-stage-1-power-on)
3. [Stage 2: POST (Power-On Self-Test)](#3-stage-2-post-power-on-self-test)
4. [Stage 3: BIOS/UEFI Initialization](#4-stage-3-biosuefi-initialization)
5. [Stage 4: Boot Device Selection](#5-stage-4-boot-device-selection)
6. [Stage 5: MBR and Bootloader](#6-stage-5-mbr-and-bootloader)
7. [Stage 6: Kernel Loading](#7-stage-6-kernel-loading)
8. [Stage 7: Init Process and System Services](#8-stage-7-init-process-and-system-services)
9. [Stage 8: User Login](#9-stage-8-user-login)
10. [BIOS vs UEFI](#10-bios-vs-uefi)
11. [Common Boot Problems](#11-common-boot-problems)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What is Booting?

**Booting** = loading the OS into RAM and preparing the system for use, starting from zero (no OS running at all).

```
  Power button pressed → [nothing running]
                              ↓
              [series of steps → OS kernel in RAM]
                              ↓
                    [desktop visible, ready to use]
```

The name comes from "pulling oneself up by one's bootstraps" — starting from nothing, building up to a full running system.

**Cold boot vs Warm boot:**

| Type          | Description                                                           | When                                                |
| ------------- | --------------------------------------------------------------------- | --------------------------------------------------- |
| **Cold boot** | Full power-off → full power-on; ALL hardware initialized from scratch | After complete shutdown, power cut, hardware change |
| **Warm boot** | Restart without full power-off; skips some hardware init              | "Restart" option, soft reboot                       |

---

## 2. Stage 1: Power-On

**What happens:** Electrical power flows from the PSU to the motherboard, CPU, RAM, drives, and peripherals.

```
  Wall outlet → PSU (converts AC to DC voltages: 12V, 5V, 3.3V)

  PSU sends "Power Good" signal to motherboard once voltage is stable

  Motherboard releases CPU reset

  CPU wakes up in a known initial state:
  - All registers cleared/set to default values
  - Instruction pointer set to a fixed "reset vector" address
    (e.g., 0xFFFF_FFF0 on x86 — just 16 bytes before end of 4 GB address space)
```

**No software yet.** This is entirely hardware-controlled, happens in milliseconds.

---

## 3. Stage 2: POST (Power-On Self-Test)

**What happens:** CPU starts executing from the reset vector, which points to the BIOS/UEFI firmware chip on the motherboard. The firmware runs POST.

**POST checks:**

```
  ✓ CPU registers working
  ✓ RAM: presence detected, basic read/write test
  ✓ Video card: can output to display
  ✓ Keyboard/mouse: detected
  ✓ Storage devices: detected (HDDs, SSDs, optical drives)
  ✓ System clock: reading from CMOS battery
```

**POST outcomes:**

```
  POST PASSES → Proceed to hardware init

  POST FAILS → Beep codes (series of beeps from PC speaker)
               (Visual cards not working: only beeps can signal the error)

  Beep code examples (varies by BIOS vendor):
  - 1 long + 3 short: video card problem
  - 3 beeps: RAM error
  - Continuous long beeps: power issue
```

---

## 4. Stage 3: BIOS/UEFI Initialization

**What happens:** After POST passes, firmware initializes all hardware and reads stored configuration.

```
  Firmware reads CMOS/NVRAM settings (stored in small battery-backed memory):
  - Boot device order (e.g., SSD first, then USB, then network)
  - System date and time
  - CPU clock speed, RAM timings
  - Virtualization enable/disable
  - Secure Boot settings (UEFI only)

  Firmware initializes hardware drivers/interfaces:
  - Sets up USB controllers
  - Initializes storage controllers (SATA, NVMe)
  - Configures memory map (tells OS where RAM is, where devices are)
  - Builds ACPI tables (power management info for the OS)
```

**Analogy:** A conductor preparing the orchestra before the performance. Each instrument (hardware component) must be tuned and plugged in before the music (OS) can play.

---

## 5. Stage 4: Boot Device Selection

**What happens:** Firmware uses the boot order to find a device with a valid bootable partition.

```
  Boot order example: [SSD, USB, DVD, Network]

  Firmware checks SSD first:
  → reads first sector (512 bytes)
  → checks last 2 bytes: is it 0x55 0xAA? (MBR boot signature)
  → Yes → this device is bootable → proceed

  If SSD had no valid boot sector:
  → try USB
  → try DVD
  → try Network (PXE)
  → no device found → "No bootable device found" error shown
```

---

## 6. Stage 5: MBR and Bootloader

### MBR (Master Boot Record) — Legacy BIOS systems

```
  MBR = first 512 bytes of the disk:

  Offset    Size    Content
  0         446     Bootstrap code (small bootloader program)
  446       64      Partition table (4 primary partitions × 16 bytes each)
  510       2       Boot signature: 0x55 0xAA

  446 bytes is too small for a full bootloader.

  The MBR bootstrap code finds the active partition and loads
  the REAL bootloader from there (e.g., GRUB stage 1.5 or stage 2).
```

### GPT + UEFI — Modern systems

```
  UEFI skips MBR entirely.
  Instead, it reads the EFI System Partition (ESP):
  → A FAT32 partition containing .efi bootloader executables
  → e.g., /EFI/ubuntu/grubx64.efi  (Linux GRUB)
  →       /EFI/Microsoft/Boot/bootmgfw.efi  (Windows Boot Manager)

  UEFI loads and executes the .efi file directly.
```

### Common Bootloaders

| Bootloader      | Used For                           |
| --------------- | ---------------------------------- |
| **GRUB 2**      | Linux (almost universal)           |
| **BOOTMGR**     | Windows (Vista and later)          |
| **GRUB Legacy** | Older Linux systems                |
| **rEFInd**      | Multi-boot (macOS, Linux, Windows) |

**GRUB boot menu** (when multiple OS's are installed):

```
  ┌─────────────────────────────────────────────┐
  │ GRUB boot menu (5 second timeout)           │
  │                                              │
  │ * Ubuntu 24.04 LTS                          │  ← highlighted
  │   Ubuntu 24.04 LTS (recovery mode)          │
  │   Windows Boot Manager                      │
  │                                              │
  │ Use arrow keys to select, Enter to boot     │
  └─────────────────────────────────────────────┘
```

---

## 7. Stage 6: Kernel Loading

**What happens:** The bootloader locates the OS kernel on disk and loads it into RAM.

```
  GRUB loads vmlinuz (compressed Linux kernel) into RAM
  GRUB also loads initrd/initramfs (initial RAM disk — minimal filesystem for early boot)

  Bootloader hands off control to the kernel entry point

  BIOS → bootloader era: CPU is still in 16-bit real mode
  Kernel: switches CPU to 32-bit or 64-bit protected mode

  Kernel's own initialization sequence:
  1. Decompress kernel image (if compressed)
  2. Set up memory management (page tables, MMU)
  3. Initialize device drivers (built-in + from initramfs)
  4. Mount the initial RAM disk (initramfs) as temporary root filesystem
  5. Initialize process scheduler
  6. Initialize interrupt subsystem
  7. Switch to real root filesystem (e.g., ext4 on /dev/sda2)
  8. Start PID 1 (init process)
```

At this stage, the kernel runs in **kernel mode** with full hardware access.

---

## 8. Stage 7: Init Process and System Services

**What happens:** Kernel starts PID 1 — the **init process** — which is the parent of ALL user-space processes.

```
  Linux: PID 1 = systemd (modern) or SysV init (older)
  Windows: PID 4 = System (kernel), then smss.exe (Session Manager)
  macOS: PID 1 = launchd
```

**systemd startup (modern Linux):**

```
  systemd reads /etc/systemd/system/ unit files

  Starts services in parallel (dependency graph):

  networking.service ──┐
  bluetooth.service    ├──► graphical.target (display ready)
  sshd.service        ──┘
  cups.service (printing)
  cron.service (scheduled jobs)

  systemd-logind.service → manages user sessions
  gdm.service / lightdm → display manager (login screen)
```

This is why you see a progress bar or splash screen — systemd is starting services in dependency order, often in parallel for speed.

---

## 9. Stage 8: User Login

**What happens:** Display manager (GDM, LightDM, SDDM) presents a login screen. After authentication, a user session starts.

```
  Login screen → user types credentials
               → PAM (Pluggable Authentication Modules) verifies password
               → User session created
               → Desktop environment starts (GNOME, KDE, XFCE, etc.)
               → User-configured startup apps launch
               → Desktop visible and ready
```

**Total boot time:** 10–60 seconds depending on hardware (SSD much faster than HDD) and number of services.

---

## 10. BIOS vs UEFI

| Feature              | BIOS (Legacy)                  | UEFI (Modern)                                 |
| -------------------- | ------------------------------ | --------------------------------------------- |
| **Interface**        | Text-only, keyboard navigation | Graphical, mouse support                      |
| **CPU mode at init** | 16-bit real mode               | 32-bit or 64-bit protected mode               |
| **Disk size limit**  | 2 TB (MBR partition table)     | 9+ ZB (GPT partition table)                   |
| **Partition limit**  | 4 primary partitions           | 128 partitions (GPT)                          |
| **Boot speed**       | Slower                         | Faster (runs in native 64-bit mode)           |
| **Security**         | Limited                        | Secure Boot (blocks unauthorized bootloaders) |
| **Networking**       | No                             | Yes (PXE built-in support)                    |
| **Available since**  | 1975                           | 2005 (wide adoption ~2012)                    |

**Secure Boot (UEFI):** Checks that bootloaders and drivers are digitally signed by trusted keys before loading them. Prevents rootkits and malware from intercepting the boot process. Can be disabled in UEFI settings if needed (e.g., to install some Linux distributions).

---

## 11. Common Boot Problems

| Error                          | Cause                                                  | Fix                                                                                      |
| ------------------------------ | ------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| "No bootable device found"     | Boot order wrong, disk disconnected, MBR/GPT corrupted | Enter BIOS/UEFI → fix boot order; reconnect drive; use recovery media to repair MBR      |
| Kernel panic (Linux)           | Corrupted kernel or driver, bad hardware               | Boot from live USB → run fsck; restore kernel from backup                                |
| Blue Screen of Death (Windows) | Driver crash, hardware failure, system file corruption | Boot into safe mode; use SFC /scannow; check event logs                                  |
| Boot loop                      | Failed update, corrupted boot config                   | Boot recovery mode; restore previous configuration                                       |
| GRUB rescue prompt             | GRUB installed but can't find kernel                   | Use `set root`, `linux`, `initrd`, `boot` commands to manually boot; then reinstall GRUB |

---

## 11. Code Examples

> Working code that demonstrates the boot process in practice.

### C++ — Simple Version

Simulate all 8 boot stages as a chain of functions — each stage does its work and hands off to the next; if any stage fails, the boot halts.

```cpp
// Simulate the 8-stage OS boot sequence.
// Each stage is a function that prints what it does and returns true (success) or false (failure).
// Mirrors the real sequence: Power-On → POST → BIOS/UEFI → Boot Device → Bootloader → Kernel → Init → Login.

#include <iostream>
#include <string>
#include <vector>
#include <functional>

// ── Boot stage functions ──────────────────────────────────────────────────────

bool stage_power_on() {
    std::cout << "[1] Power-On\n";
    std::cout << "    Capacitors charged, voltage rails stable (+12V, +5V, +3.3V)\n";
    std::cout << "    CPU reset vector: fetching first instruction from ROM at 0xFFFFFFF0\n";
    return true;
}

bool stage_post() {
    // POST = Power-On Self Test: hardware checks before loading any software
    std::cout << "[2] POST (Power-On Self Test)\n";
    std::vector<std::pair<std::string, bool>> checks = {
        {"CPU registers",  true},
        {"RAM (8 GB)",     true},
        {"GPU",            true},
        {"Keyboard ctrl",  true},
        {"NVMe SSD",       true},
    };
    for (auto& [name, ok] : checks) {
        std::cout << "    " << name << ": " << (ok ? "OK" : "FAIL") << "\n";
        if (!ok) return false;   // POST failure → beep codes, halt
    }
    std::cout << "    All hardware OK → 1 short beep\n";
    return true;
}

bool stage_bios_uefi() {
    std::cout << "[3] BIOS/UEFI Firmware\n";
    std::cout << "    Reading CMOS: date/time, boot order = [NVMe, USB, Network]\n";
    std::cout << "    UEFI Secure Boot: verifying bootloader digital signature... VERIFIED\n";
    std::cout << "    Handing control to EFI System Partition bootloader\n";
    return true;
}

bool stage_boot_device() {
    std::cout << "[4] Boot Device Selection\n";
    std::cout << "    Scanning NVMe SSD (boot order #1)... found GPT partition table\n";
    std::cout << "    Located EFI System Partition (/dev/nvme0n1p1, FAT32)\n";
    return true;
}

bool stage_bootloader() {
    // Bootloader loads the kernel image into RAM
    std::cout << "[5] Bootloader (GRUB2)\n";
    std::cout << "    Reading /boot/grub/grub.cfg...\n";
    std::cout << "    Menu: Ubuntu 24.04 LTS (default, 5s timeout)\n";
    std::cout << "    Loading /boot/vmlinuz-6.5.0 (12 MB) into RAM...\n";
    std::cout << "    Loading /boot/initrd.img-6.5.0 (80 MB) into RAM...\n";
    std::cout << "    Jumping to kernel entry point\n";
    return true;
}

bool stage_kernel_init() {
    // Kernel sets up the entire execution environment
    std::cout << "[6] Kernel Initialization\n";
    std::vector<std::string> steps = {
        "Switching CPU to 64-bit protected mode",
        "Setting up interrupt descriptor table (IDT)",
        "Initializing memory manager (page tables, buddy allocator)",
        "Loading drivers: ahci (SATA), e1000e (NIC), xhci-hcd (USB)",
        "Mounting root filesystem /dev/nvme0n1p2 as / (ext4, read-only first)",
        "Remounting root filesystem read-write",
        "Spawning PID 1 (/sbin/init → systemd)",
    };
    for (const auto& step : steps) std::cout << "    " << step << "\n";
    return true;
}

bool stage_init_systemd() {
    // PID 1 = systemd: parent of all user-space processes
    std::cout << "[7] systemd (PID 1)\n";
    std::cout << "    Reading unit files from /etc/systemd/system/\n";
    std::vector<std::pair<std::string, std::string>> services = {
        {"systemd-journald",  "logging"},
        {"systemd-udevd",     "device manager"},
        {"NetworkManager",    "network"},
        {"sshd",              "SSH server"},
        {"cron",              "scheduled tasks"},
        {"gdm",               "display manager"},
    };
    for (auto& [name, desc] : services)
        std::cout << "    [  OK  ] Started " << name << " (" << desc << ")\n";
    std::cout << "    Reached target: graphical.target\n";
    return true;
}

bool stage_user_login() {
    std::cout << "[8] Login\n";
    std::cout << "    GDM: presenting login screen\n";
    std::cout << "    User authenticated → GNOME session started\n";
    return true;
}

int main() {
    std::cout << "=== Boot Sequence Simulation ===\n\n";

    // Chain of boot stages — if any fails, boot halts (real system shows error/beeps)
    std::vector<std::pair<std::string, std::function<bool()>>> stages = {
        {"Power-On",     stage_power_on    },
        {"POST",         stage_post        },
        {"BIOS/UEFI",    stage_bios_uefi   },
        {"Boot Device",  stage_boot_device },
        {"Bootloader",   stage_bootloader  },
        {"Kernel Init",  stage_kernel_init },
        {"systemd",      stage_init_systemd},
        {"User Login",   stage_user_login  },
    };

    for (auto& [name, fn] : stages) {
        if (!fn()) {
            std::cout << "\n*** BOOT FAILED at stage: " << name << " ***\n";
            return 1;
        }
        std::cout << "\n";
    }

    std::cout << "=== System fully booted ===\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style

Dependency-based boot system like systemd — services declare what they need; topological sort (Kahn's algorithm) computes the correct start order; circular dependencies are detected.

```cpp
// Simulate systemd-style service manager: boot services in dependency order.
// Classic graph problem: Topological Sort (Kahn's BFS algorithm).
// If services have a circular dependency (A needs B, B needs A), boot is impossible.

#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <queue>
#include <iomanip>
#include <stdexcept>

// ── Service definition ────────────────────────────────────────────────────────
struct Service {
    std::string              name;
    std::vector<std::string> depends_on;   // must start BEFORE this service
    int                      startup_ms;   // simulated startup time
};

// ── Boot manager ──────────────────────────────────────────────────────────────
class BootManager {
    std::unordered_map<std::string, Service>                    services;
    std::unordered_map<std::string, std::vector<std::string>>   dependents; // dep → [who needs dep]
    std::unordered_map<std::string, int>                        in_degree;  // unmet dependency count

public:
    void add(const Service& s) {
        services[s.name] = s;
        in_degree[s.name] = (int)s.depends_on.size();
        for (const auto& dep : s.depends_on)
            dependents[dep].push_back(s.name);
    }

    // Kahn's algorithm: BFS topological sort
    std::vector<std::string> resolve_order() {
        std::queue<std::string> ready;   // services whose dependencies are all satisfied
        for (const auto& [name, deg] : in_degree)
            if (deg == 0) ready.push(name);

        std::vector<std::string> order;
        while (!ready.empty()) {
            std::string cur = ready.front(); ready.pop();
            order.push_back(cur);

            // Mark this service as started — unblock anything waiting on it
            for (const auto& dep : dependents[cur])
                if (--in_degree[dep] == 0)
                    ready.push(dep);
        }

        if (order.size() != services.size())
            throw std::runtime_error("Circular dependency detected — system cannot boot!");

        return order;
    }

    void boot() {
        std::vector<std::string> order = resolve_order();
        std::cout << "=== Dependency-Ordered Boot Sequence ===\n\n";
        int total_ms = 0;
        for (const auto& name : order) {
            const Service& s = services.at(name);
            std::string deps;
            for (size_t i = 0; i < s.depends_on.size(); ++i) {
                deps += s.depends_on[i];
                if (i + 1 < s.depends_on.size()) deps += ", ";
            }
            std::cout << "  Starting: " << std::left << std::setw(22) << name;
            if (!deps.empty()) std::cout << "[after: " << deps << "]";
            std::cout << " ... " << s.startup_ms << "ms\n";
            total_ms += s.startup_ms;
        }
        std::cout << "\n  Total boot time: " << total_ms << "ms\n";
        std::cout << "=== All services running ===\n";
    }
};

int main() {
    BootManager mgr;

    // Register services (like .service files in /etc/systemd/system/)
    // Order of add() calls doesn't matter — dependencies determine order
    mgr.add({"kernel",         {},                                      0   });
    mgr.add({"udev",           {"kernel"},                              50  });
    mgr.add({"dbus",           {"kernel"},                              30  });
    mgr.add({"NetworkManager", {"udev", "dbus"},                        200 });
    mgr.add({"sshd",           {"NetworkManager"},                      80  });
    mgr.add({"cron",           {"dbus"},                                20  });
    mgr.add({"postgresql",     {"NetworkManager"},                      300 });
    mgr.add({"nginx",          {"NetworkManager", "postgresql"},        150 });
    mgr.add({"app-server",     {"nginx", "postgresql", "sshd"},         500 });

    mgr.boot();

    // Demonstrate cycle detection
    std::cout << "\n--- Cycle Detection Demo ---\n";
    BootManager bad;
    bad.add({"A", {"B"}, 0});
    bad.add({"B", {"C"}, 0});
    bad.add({"C", {"A"}, 0});  // C→A→B→C: circular!
    try {
        bad.boot();
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n";
    }
    return 0;
}
```

### Python — Simple Version

Simulate the boot sequence as a list of stage functions — each stage prints its actions; a stage can raise `RuntimeError` to simulate a boot failure.

```python
# Simulate the OS boot sequence — 7 stages from power-on to user login.
# Each stage is a function that prints what it does.
# A failed stage raises RuntimeError, which stops the boot (like a real boot failure).

def stage_power_on():
    print("[1] Power-On")
    print("    Voltage rails stable (+12V, +5V, +3.3V)")
    print("    CPU fetches first instruction from ROM at 0xFFFFFFF0")

def stage_post():
    print("[2] POST (Power-On Self Test)")
    components = [
        ("CPU registers",  True),
        ("RAM (8 GB)",     True),
        ("GPU",            True),
        ("Keyboard",       True),
        ("NVMe SSD",       True),
    ]
    for component, ok in components:
        status = "OK" if ok else "FAIL"
        print(f"    {component:<20} ... {status}")
        if not ok:
            raise RuntimeError(f"POST failed: {component} — system halted (3 beep codes)")
    print("    All hardware OK → 1 short beep")

def stage_bios_uefi():
    print("[3] BIOS/UEFI Firmware")
    print("    Reading CMOS: boot order = [NVMe SSD, USB Drive, Network]")
    print("    UEFI Secure Boot: checking bootloader signature ... VERIFIED")
    print("    Handing control to EFI System Partition")

def stage_bootloader():
    print("[4] Bootloader (GRUB2)")
    print("    Reading /boot/grub/grub.cfg")
    print("    Menu: [Ubuntu 24.04] [Windows 11]  (auto-select in 5s)")
    print("    Loading kernel /boot/vmlinuz-6.5.0  (12 MB)")
    print("    Loading initrd /boot/initrd.img-6.5.0  (80 MB)")
    print("    Jumping to kernel entry point 0x1000000")

def stage_kernel_init():
    print("[5] Kernel Initialization")
    steps = [
        "CPU → 64-bit protected mode",
        "Setting up IDT (Interrupt Descriptor Table)",
        "Initializing page tables (virtual memory)",
        "Loading drivers: ahci, e1000e, xhci-hcd",
        "Mounting root filesystem /dev/nvme0n1p2 as / (ext4)",
        "Spawning PID 1 → /sbin/init (systemd)",
    ]
    for step in steps:
        print(f"    {step}")

def stage_init_systemd():
    print("[6] systemd (PID 1)")
    print("    Reading unit files from /etc/systemd/system/")
    services = [
        ("systemd-journald", "logging"),
        ("systemd-udevd",    "device manager"),
        ("NetworkManager",   "networking"),
        ("sshd",             "SSH server"),
        ("cron",             "scheduled tasks"),
        ("gdm",              "display manager"),
    ]
    for name, desc in services:
        print(f"    [  OK  ] Started {name} ({desc})")
    print("    Reached target: graphical.target")

def stage_user_login():
    print("[7] Login")
    print("    GDM: showing graphical login screen")
    print("    User authenticated → GNOME session started")


if __name__ == "__main__":
    print("=" * 50)
    print("  Boot Sequence Simulation")
    print("=" * 50 + "\n")

    stages = [
        stage_power_on,
        stage_post,
        stage_bios_uefi,
        stage_bootloader,
        stage_kernel_init,
        stage_init_systemd,
        stage_user_login,
    ]

    for stage_fn in stages:
        try:
            stage_fn()
        except RuntimeError as e:
            print(f"\n*** BOOT FAILED: {e} ***")
            break
        print()   # blank line between stages
    else:
        print("=" * 50)
        print("  System fully booted!")
        print("=" * 50)
```

### Python — Medium Level

Dependency-based service manager using topological sort (Kahn's algorithm) — services declare dependencies, the manager computes the correct start order and detects circular dependencies.

```python
# Simulate systemd: boot services in dependency order using topological sort.
# Kahn's algorithm (BFS): start services with no pending deps, mark them done,
# unblock their dependents, repeat. If a cycle exists, not all services will start.

from collections import defaultdict, deque
from dataclasses import dataclass, field

@dataclass
class Service:
    name:       str
    depends_on: list[str] = field(default_factory=list)  # must start before this
    startup_ms: int        = 50                           # simulated start time

class BootManager:
    def __init__(self):
        self.services: dict[str, Service] = {}

    def add(self, svc: Service) -> "BootManager":
        self.services[svc.name] = svc
        return self   # allows chaining: mgr.add(...).add(...)

    def _topological_sort(self) -> list[str]:
        """Kahn's BFS topological sort. Raises RuntimeError on cycle."""
        # Build reverse graph: dep → [services that depend on dep]
        dependents  = defaultdict(list)
        in_degree   = {name: 0 for name in self.services}

        for svc in self.services.values():
            in_degree[svc.name] = len(svc.depends_on)
            for dep in svc.depends_on:
                dependents[dep].append(svc.name)

        # Seed queue with services that have no dependencies
        queue  = deque(name for name, deg in in_degree.items() if deg == 0)
        order  = []

        while queue:
            cur = queue.popleft()
            order.append(cur)
            # This service is now "started" — satisfy one dep for each dependent
            for dependent in dependents[cur]:
                in_degree[dependent] -= 1
                if in_degree[dependent] == 0:
                    queue.append(dependent)  # all its deps are now satisfied

        if len(order) != len(self.services):
            unsatisfied = set(self.services) - set(order)
            raise RuntimeError(f"Circular dependency! Cannot boot: {unsatisfied}")

        return order

    def boot(self):
        order    = self._topological_sort()
        total_ms = 0
        print("=== Dependency-Ordered Boot Sequence ===\n")
        for name in order:
            svc      = self.services[name]
            deps_str = f"  [after: {', '.join(svc.depends_on)}]" if svc.depends_on else ""
            print(f"  Starting {name:<22}{deps_str:<45} ... {svc.startup_ms}ms")
            total_ms += svc.startup_ms
        print(f"\n  Total boot time: {total_ms}ms")
        print("=== All services running ===")


if __name__ == "__main__":
    mgr = BootManager()

    # Order of .add() calls doesn't matter — dependencies determine order
    mgr.add(Service("kernel",         [],                                        0  ))
    mgr.add(Service("udev",           ["kernel"],                                50 ))
    mgr.add(Service("dbus",           ["kernel"],                                30 ))
    mgr.add(Service("NetworkManager", ["udev", "dbus"],                          200))
    mgr.add(Service("sshd",           ["NetworkManager"],                        80 ))
    mgr.add(Service("cron",           ["dbus"],                                  20 ))
    mgr.add(Service("postgresql",     ["NetworkManager"],                        300))
    mgr.add(Service("nginx",          ["NetworkManager", "postgresql"],          150))
    mgr.add(Service("app-server",     ["nginx", "postgresql", "sshd"],           500))

    mgr.boot()

    # Demonstrate cycle detection
    print("\n--- Cycle Detection Demo ---")
    bad = BootManager()
    bad.add(Service("A", ["B"]))
    bad.add(Service("B", ["C"]))
    bad.add(Service("C", ["A"]))   # A→B→C→A: circular!
    try:
        bad.boot()
    except RuntimeError as e:
        print(f"  Error: {e}")
```

---

## 12. Key Takeaways

- **Booting** = loading OS from zero to fully running state; the name means "pulling oneself up by bootstraps"
- **Cold boot** initializes all hardware from scratch; **warm boot** (restart) skips some hardware init
- **8 stages:** Power-On → POST → BIOS/UEFI init → Boot device selection → MBR/Bootloader → Kernel loading → Init process → User login
- **POST** tests all hardware on power-on; failure → beep codes (or visual error if display works)
- **BIOS/UEFI** is firmware on the motherboard; reads CMOS config (boot order, date/time, hardware settings)
- **MBR** (legacy): first 512 bytes of disk, contains 446-byte bootstrap code + partition table
- **UEFI** (modern): reads EFI System Partition directly, supports GPT, Secure Boot, faster boot, graphical interface
- **Bootloader** (GRUB on Linux, BOOTMGR on Windows): tiny program that finds kernel and loads it into RAM; enables multi-OS selection
- **Kernel init:** switches CPU mode, sets up memory management, initializes drivers, mounts filesystem, starts PID 1
- **PID 1 = init** (systemd on modern Linux, launchd on macOS): parent of all user processes; starts all services
- **UEFI Secure Boot** prevents unauthorized bootloaders from loading — important security feature against boot-time malware
