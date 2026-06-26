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
