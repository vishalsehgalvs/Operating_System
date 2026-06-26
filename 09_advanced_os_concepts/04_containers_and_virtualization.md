# Containers and OS-Level Virtualization

> **Containers** are lightweight isolated environments that package an application with all its dependencies and run directly on the host OS kernel — unlike VMs they don't need a full guest OS, making them start in seconds, use megabytes instead of gigabytes, and run hundreds per machine; **OS-level virtualization** (namespaces + cgroups) is the kernel feature that makes this isolation possible.

---

## Table of Contents

1. [What is OS-Level Virtualization?](#1-what-is-os-level-virtualization)
2. [What Are Containers?](#2-what-are-containers)
3. [Containers vs Virtual Machines](#3-containers-vs-virtual-machines)
4. [How Containers Achieve Isolation](#4-how-containers-achieve-isolation)
5. [Container Lifecycle](#5-container-lifecycle)
6. [Benefits of Containers](#6-benefits-of-containers)
7. [Limitations and Challenges](#7-limitations-and-challenges)
8. [Common Use Cases](#8-common-use-cases)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is OS-Level Virtualization?

**OS-level virtualization** is a kernel feature that allows multiple isolated **user-space instances** (containers) to run simultaneously on one OS, all sharing the same kernel.

```
  Hardware Virtualization (VMs):                OS-Level Virtualization (Containers):
  ┌────────────────────────┐                  ┌────────────────────────┐
  │ VM1  │ VM2  │  VM3     │                  │ C1 │ C2 │ C3 │ C4 │ C5│
  │ OS1  │ OS2  │  OS3     │                  ├────────────────────────┤
  ├────────────────────────┤                  │     Host OS Kernel      │  ← shared!
  │      Hypervisor        │                  ├────────────────────────┤
  ├────────────────────────┤                  │    Physical Hardware    │
  │   Physical Hardware    │                  └────────────────────────┘
  └────────────────────────┘

  VMs: 3 separate kernels → heavy          Containers: 1 shared kernel → light
  Each VM: 1–10 GB RAM                     Each container: 10–200 MB RAM
  Startup: minutes                          Startup: milliseconds to seconds
```

**Analogy:** Multiple apartments in one building. They all share the same foundation and utility systems (water pipes, electrical grid = OS kernel), but each apartment is private and self-contained — occupants can't see or interfere with each other.

### Key Characteristics

- All containers share the **same kernel**
- Each container has its own **file system, process tree, network stack, and user space**
- Applications run **natively** — no emulation, no hardware translation → near-native performance
- Containers start and stop in **milliseconds to seconds**
- A single server can run **hundreds of containers** that would require dozens of VMs

---

## 2. What Are Containers?

A **container** is a standalone executable package that includes everything needed to run an application:

- Application code
- Runtime (Python/Node/JVM/etc.)
- System libraries and tools
- Configuration and environment variables

**Analogy:** A lunch box that contains everything you need for your meal. You don't rely on the cafeteria — everything's packed inside. Take it anywhere, open it, and it works the same.

```
  Traditional deployment ("works on my machine" problem):
  Dev's machine:     Python 3.9, lib v2.1, OS Ubuntu 20.04
  Test server:       Python 3.7, lib v1.9, OS CentOS 7       ← different!
  Production:        Python 3.11, lib v3.0, OS Debian 11     ← different again!
  → "It worked on my machine" → hours of debugging env differences

  Container deployment (solved):
  Container image:   Python 3.9 + lib v2.1 + Ubuntu 20.04 base layer (packaged together)
  → Run on Dev's machine → same
  → Run on Test server   → same
  → Run in Production    → same
  → "If it runs in the container, it runs everywhere"
```

The most popular container technology is **Docker**, but the underlying kernel features (namespaces + cgroups) are built into Linux itself.

---

## 3. Containers vs Virtual Machines

| Feature               | Containers                           | Virtual Machines               |
| --------------------- | ------------------------------------ | ------------------------------ |
| **Size**              | Megabytes (MB)                       | Gigabytes (GB)                 |
| **Startup time**      | Milliseconds to seconds              | Minutes                        |
| **OS kernel**         | Shared with host                     | Separate kernel per VM         |
| **Isolation level**   | Process-level (namespace isolation)  | Hardware-level                 |
| **Performance**       | Near-native (no emulation)           | Slight overhead (5–10%)        |
| **Resource usage**    | Very low overhead                    | High overhead (full OS per VM) |
| **Portability**       | Very high (small image, fast deploy) | Moderate (large image, slow)   |
| **Security boundary** | Weaker (shared kernel)               | Stronger (separate kernels)    |
| **OS compatibility**  | Must match host kernel OS            | Can run any guest OS           |

```
  Resource comparison for running 5 identical web apps:

  VMs:                                    Containers:
  [App + full Ubuntu OS] × 5             [App + minimal libs] × 5
  = 5 × 2 GB = 10 GB RAM                = 5 × 100 MB = 500 MB RAM
  = 5 × 2 min startup                   = 5 × 2 sec startup

  Containers fit 20x more apps on same hardware!
```

### When to Use Each

| Scenario                                       | Containers | VMs |
| ---------------------------------------------- | ---------- | --- |
| Microservices, cloud-native apps               | ✅         |     |
| Need different OS (run Windows on Linux host)  |            | ✅  |
| Strong security isolation required             |            | ✅  |
| Dev/test environments                          | ✅         |     |
| Running untrusted/hostile code                 |            | ✅  |
| Legacy monolithic applications                 |            | ✅  |
| Rapid scale-out (add 100 instances in seconds) | ✅         |     |

Many production systems use **both**: containers running inside VMs → security of VMs + density of containers.

---

## 4. How Containers Achieve Isolation

Containers use two core Linux kernel features: **namespaces** (what the container can see) and **cgroups** (what the container can use).

### Linux Namespaces

Namespaces partition kernel resources so each container has its own isolated view:

| Namespace   | What It Isolates              | Effect                                                                            |
| ----------- | ----------------------------- | --------------------------------------------------------------------------------- |
| **PID**     | Process IDs                   | Container sees its own process tree starting from PID 1; can't see host processes |
| **Network** | Network stack                 | Container has its own IP address, ports, routing table                            |
| **Mount**   | File system mounts            | Container has its own filesystem hierarchy                                        |
| **UTS**     | Hostname                      | Container can have its own hostname                                               |
| **IPC**     | Shared memory, message queues | Processes in different containers can't communicate via IPC                       |
| **User**    | User and group IDs            | Container can have its own root user (mapped to non-root on host)                 |

```
  Without namespaces:
  Host sees: PID 1 (init), PID 2 (kthreadd), PID 100 (app), PID 101 (container_app)
  Container_app can see ALL processes, including host's PID 100

  With PID namespace:
  Container's view: PID 1 (container_app)  ← thinks it's the only process!
  Host's view: PID 1 (init), ..., PID 101 (container_app mapped to container's PID 1)
```

**Analogy:** Each container gets its own pair of glasses that shows only what it's allowed to see.

### Control Groups (cgroups)

**cgroups** enforce resource limits and accounting — they prevent one container from hogging all the CPU, RAM, or disk I/O.

```
  Example: cgroup limits for a web server container

  CPU:     max 2 cores (out of 16 on host)
  Memory:  max 512 MB RAM (OOM killer triggered if exceeded)
  Disk I/O: max 100 MB/s reads
  Network: max 100 Mbit/s bandwidth

  Without cgroups:
  Container A goes crazy → uses 15 of 16 CPU cores → all other containers starve

  With cgroups:
  Container A hits its 2-core limit → forced to slow down → others unaffected
```

cgroups also **account** for resource usage per container — essential for cloud billing (charge per CPU/memory consumed).

```
  Complete container isolation:

  Namespace:   Container thinks it has its own system (PID 1, own hostname, own IPs)
  cgroups:     Container can't consume more than its allocated share of resources
  UnionFS:     Container has its own layered filesystem (copy-on-write)

  Together: each container is a completely self-contained, isolated process group
```

---

## 5. Container Lifecycle

### Container Images

A **container image** is a read-only template consisting of **layers**:

```
  Container Image Layers (Docker example):

  Layer 4: YOUR APP CODE           ← your changes (smallest layer, changes most often)
  Layer 3: pip install requirements ← your Python dependencies
  Layer 2: Python 3.9 runtime
  Layer 1: Ubuntu 22.04 base        ← shared by many images (cached once)

  If you change only Layer 4 (your code):
  → Only Layer 4 is rebuilt and re-uploaded (1–10 MB)
  → Layers 1–3 are cached on host and registry
  → Fast builds, small incremental uploads
```

Images are stored in **registries** (Docker Hub, GitHub Container Registry, private registries). You pull a specific image tag to deploy a specific version.

### Running, Stopping, Removing

```
  Image (read-only template)
       ↓ docker run
  Container (image + writable layer)
       ↓ make changes, write files
  Container running with state
       ↓ docker stop
  Container stopped (writable layer preserved)
       ↓ docker start
  Container resumed from where it left off
       ↓ docker rm
  Container deleted (writable layer GONE — data lost unless in a volume!)

  Important: containers are EPHEMERAL by design.
  Store persistent data in VOLUMES (outside the container).
```

### Scaling

```
  One container: handles 100 req/sec, CPU at 80%
  Traffic spikes to 500 req/sec →

  Run 5 containers from same image:
  Container 1: handles 100 req/sec
  Container 2: handles 100 req/sec
  Container 3: handles 100 req/sec
  Container 4: handles 100 req/sec
  Container 5: handles 100 req/sec

  Load balancer distributes requests across all 5.
  Traffic drops → stop 4 containers → back to 1.

  This horizontal scaling is instant with containers (seconds vs minutes for VMs).
```

---

## 6. Benefits of Containers

| Benefit             | Explanation                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| **Portability**     | "Build once, run anywhere" — same container runs on dev laptop, staging, production, any cloud |
| **Efficiency**      | 10–100× more containers per host vs VMs; share kernel, start in seconds                        |
| **Consistency**     | Eliminates "works on my machine" — image captures exact runtime environment                    |
| **Fast deployment** | New version = new image → deploy in seconds → roll back in seconds                             |
| **Isolation**       | App A can't affect App B; each has its own dependencies, no version conflicts                  |
| **Microservices**   | Each service in its own container → deploy/scale/update independently                          |
| **CI/CD**           | Build → test → deploy pipeline uses same container throughout                                  |

---

## 7. Limitations and Challenges

### Security: Weaker Isolation than VMs

```
  VMs:
  Attacker in VM1 exploits a vulnerability
  → Can only reach hypervisor (very hardened)
  → Cannot reach VM2 or Host OS

  Containers:
  Attacker in Container1 exploits a kernel vulnerability
  → Reaches host kernel (shared!)
  → Could potentially affect all other containers and host

  This is why containers are NOT recommended for running hostile/untrusted code.
  For that, use VMs (or containers inside VMs).
```

**Mitigations:** Seccomp (restrict syscalls), AppArmor/SELinux (MAC policies), non-root users in containers, read-only filesystems, regular base image updates.

### OS Compatibility Constraint

```
  Linux containers → must run on Linux host kernel
  Windows containers → must run on Windows host kernel

  Cannot run Windows container on Linux directly.
  Cannot run a Linux container with a different kernel version than host.

  Exception: Docker Desktop on Windows/Mac uses a lightweight Linux VM underneath
  → Linux containers work on Windows/Mac via this thin VM layer
```

### Persistent Storage Complexity

```
  Container is ephemeral: stop → rm → data gone!

  Solution: Docker Volumes
  docker run -v /host/data:/container/data myapp

  Files in /container/data are actually stored at /host/data
  → Container can be deleted and recreated → data persists on host

  But managing stateful apps (databases) in containers adds operational complexity.
```

---

## 8. Common Use Cases

| Use Case                     | How Containers Help                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------- |
| **Microservices**            | Each service (auth, cart, catalog, payments) in its own container → independent deploy/scale |
| **Development environments** | New team member: `docker-compose up` → complete environment in 30 seconds                    |
| **CI/CD pipelines**          | Each build runs in a fresh container → reproducible, isolated build environment              |
| **Cloud deployment**         | AWS ECS, GKE, AKS run your containers at scale on managed Kubernetes clusters                |
| **Multiple app versions**    | Run MySQL 5.7 and MySQL 8.0 side by side in containers — no conflicts                        |
| **Rapid horizontal scaling** | Traffic spike → spin up 50 more containers in seconds → scale back down when done            |

### Containers + Kubernetes

For running many containers across many machines, **Kubernetes** (K8s) orchestrates:

- Which containers run on which machines
- Restarting failed containers
- Rolling updates (no downtime deploys)
- Auto-scaling based on CPU/memory usage
- Load balancing across container instances

---

## 9. Code Examples

> Working code that demonstrates container isolation concepts — PID/network/mount namespaces and cgroup resource enforcement — in practice.

### C++ — Simple Version
Simulate container namespaces: each container has its own PID namespace (PIDs start at 1 inside), its own network IP, and its own filesystem root.

```cpp
#include <iostream>
#include <string>
#include <vector>

// Container Namespace Simulation
// Namespaces isolate resources — each container gets its own private view of:
//   PID namespace : process IDs start at 1 inside the container
//   Network namespace : own IP address
//   Mount namespace  : own root filesystem path
// The host sees ALL processes with their real (host) PIDs.

struct Process {
    int  containerPid;   // PID as seen inside the container (starts at 1)
    int  hostPid;        // Actual PID on the host kernel
    std::string name;
};

struct Container {
    std::string id;
    std::string networkIP;
    std::string rootFS;
    std::vector<Process> procs;
    int nextContainerPid = 1;
    int hostPidBase;         // Host PIDs = hostPidBase + containerPid

    Container(const std::string& cid, const std::string& ip,
              const std::string& fs, int hostBase)
        : id(cid), networkIP(ip), rootFS(fs), hostPidBase(hostBase) {}

    void spawn(const std::string& name) {
        int cpid = nextContainerPid++;
        int hpid = hostPidBase + cpid;
        procs.push_back({cpid, hpid, name});
        std::cout << "  [" << id << "] Spawned '" << name
                  << "' — container PID: " << cpid
                  << " | host PID: " << hpid << "\n";
    }

    void showIsolation() const {
        std::cout << "\nContainer: " << id << "\n";
        std::cout << "  Network namespace: IP=" << networkIP << "\n";
        std::cout << "  Mount namespace:   rootfs=" << rootFS << "\n";
        std::cout << "  PID namespace (container sees):\n";
        for (auto& p : procs) {
            std::cout << "    PID " << p.containerPid << " -> " << p.name << "\n";
        }
    }
};

class ContainerRuntime {
    std::vector<Container> containers;
    int pidCounter = 1000;   // Host PID allocation counter

public:
    Container& create(const std::string& id, const std::string& ip,
                      const std::string& image) {
        std::string rootFS = "/var/lib/containers/" + id + "/rootfs";
        containers.push_back({id, ip, rootFS, pidCounter});
        pidCounter += 100;   // Reserve 100 PID slots per container
        std::cout << "[RUNTIME] Created " << id
                  << " (image=" << image << ", ip=" << ip << ")\n";
        return containers.back();
    }

    void showHostView() const {
        std::cout << "\n=== Host PID Namespace (all processes visible) ===\n";
        for (auto& c : containers) {
            for (auto& p : c.procs) {
                std::cout << "  Host PID " << p.hostPid
                          << " -> container:" << c.id
                          << " -> '" << p.name << "'\n";
            }
        }
    }
};

int main() {
    ContainerRuntime runtime;

    auto& c1 = runtime.create("app-1", "172.17.0.2", "nginx");
    c1.spawn("nginx: master");
    c1.spawn("nginx: worker");

    auto& c2 = runtime.create("app-2", "172.17.0.3", "redis");
    c2.spawn("redis-server");

    auto& c3 = runtime.create("app-3", "172.17.0.4", "postgres");
    c3.spawn("postgres: master");
    c3.spawn("postgres: reader");

    // Each container sees only its own PIDs — isolation enforced by kernel namespaces
    c1.showIsolation();
    c2.showIsolation();
    c3.showIsolation();

    // Host sees everything with real PIDs
    runtime.showHostView();

    return 0;
}
```

### C++ — Medium / LeetCode Style
Simulate cgroups enforcing CPU quotas (throttling) and memory limits (OOM kill) per container.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <stdexcept>

// cgroups (control groups) let the kernel enforce resource limits per container.
// CPU throttling : container requests more CPU than quota -> capped at quota.
// OOM kill       : container exceeds memory limit -> process killed by kernel.

struct Cgroup {
    std::string name;
    int cpuQuotaPct;       // Max CPU % allowed
    int memoryLimitMB;     // Max memory in MB
    int cpuUsagePct  = 0;
    int memoryUsedMB = 0;
    bool throttled  = false;
    bool oomKilled  = false;
};

class CgroupManager {
    std::vector<Cgroup> cgroups;

    Cgroup& find(const std::string& name) {
        for (auto& cg : cgroups) if (cg.name == name) return cg;
        throw std::runtime_error("cgroup not found: " + name);
    }

public:
    void addContainer(const std::string& name, int cpuQuota, int memLimit) {
        cgroups.push_back({name, cpuQuota, memLimit});
        std::cout << "[CGROUP] " << name
                  << " — CPU quota: " << cpuQuota
                  << "%, Memory limit: " << memLimit << " MB\n";
    }

    // Container requests a CPU usage level; cgroup enforces the quota
    void setCpuUsage(const std::string& name, int requestedPct) {
        auto& cg = find(name);
        if (requestedPct > cg.cpuQuotaPct) {
            cg.throttled    = true;
            cg.cpuUsagePct  = cg.cpuQuotaPct;   // Cap at quota
            std::cout << "[THROTTLE] " << name << " requested " << requestedPct
                      << "% -> capped at " << cg.cpuQuotaPct << "%\n";
        } else {
            cg.throttled   = false;
            cg.cpuUsagePct = requestedPct;
            std::cout << "[CPU OK]   " << name << " using " << requestedPct
                      << "% (limit=" << cg.cpuQuotaPct << "%)\n";
        }
    }

    // Container allocates additional memory; cgroup enforces the limit
    void allocateMemory(const std::string& name, int mb) {
        auto& cg = find(name);
        if (cg.oomKilled) {
            std::cout << "[DEAD]     " << name << " already OOM-killed\n";
            return;
        }
        cg.memoryUsedMB += mb;
        if (cg.memoryUsedMB > cg.memoryLimitMB) {
            cg.oomKilled = true;
            std::cout << "[OOM KILL] " << name << " — "
                      << cg.memoryUsedMB << " MB > " << cg.memoryLimitMB
                      << " MB limit -> container killed!\n";
        } else {
            std::cout << "[MEM OK]   " << name << " using "
                      << cg.memoryUsedMB << "/" << cg.memoryLimitMB << " MB\n";
        }
    }

    void printStatus() const {
        std::cout << "\n=== cgroup Status ===\n";
        for (auto& cg : cgroups) {
            std::cout << "  " << cg.name
                      << " | CPU:" << cg.cpuUsagePct << "/" << cg.cpuQuotaPct << "%"
                      << (cg.throttled ? " [THROTTLED]" : "")
                      << " | MEM:" << cg.memoryUsedMB << "/" << cg.memoryLimitMB << "MB"
                      << (cg.oomKilled ? " [OOM-KILLED]" : "") << "\n";
        }
    }
};

int main() {
    CgroupManager mgr;
    mgr.addContainer("web-app",  25, 512);    // 25% CPU, 512 MB RAM
    mgr.addContainer("database", 50, 2048);   // 50% CPU, 2 GB RAM
    mgr.addContainer("worker",   10, 256);    // 10% CPU, 256 MB RAM

    std::cout << "\n--- Simulating workloads ---\n";
    mgr.setCpuUsage("web-app",  20);    // OK
    mgr.allocateMemory("web-app",  300);  // OK

    mgr.setCpuUsage("database", 45);    // OK
    mgr.allocateMemory("database", 1500); // OK

    mgr.setCpuUsage("worker",   80);    // THROTTLED (exceeds 10% quota)

    mgr.allocateMemory("web-app",  300);  // OOM KILL (300+300=600 > 512 MB)

    mgr.printStatus();
    return 0;
}
```

### Python — Simple Version
Simulate container namespaces: isolated PID trees, network IPs, and filesystem roots — while the host sees all real PIDs.

```python
# Container namespace simulation
# Each container has its own isolated view of:
#   PID namespace  : PIDs start at 1 inside the container
#   Network namespace : own IP address
#   Mount namespace   : own root filesystem path
# The host OS sees ALL processes with their actual (host) PIDs.

class Container:
    def __init__(self, container_id, network_ip, image, host_pid_start):
        self.container_id   = container_id
        self.network_ip     = network_ip       # Network namespace: own IP
        self.rootfs         = f'/var/lib/containers/{container_id}/rootfs'
        self.host_pid_start = host_pid_start
        self._processes     = []               # [(container_pid, host_pid, name)]
        self._next_pid      = 1               # PIDs inside the container start at 1

    def spawn(self, name):
        """Spawn a process inside this container's PID namespace."""
        container_pid = self._next_pid
        host_pid      = self.host_pid_start + container_pid
        self._processes.append((container_pid, host_pid, name))
        self._next_pid += 1
        print(f"  [{self.container_id}] Spawned {name!r} "
              f"— container PID: {container_pid}, host PID: {host_pid}")

    def show_isolation(self):
        """Show what this container's private namespaces look like."""
        print(f"\nContainer: {self.container_id}")
        print(f"  Network namespace: IP={self.network_ip}")
        print(f"  Mount namespace:   rootfs={self.rootfs}")
        print(f"  PID namespace (container's view):")
        for cpid, _, name in self._processes:
            print(f"    PID {cpid} -> {name!r}")   # Container only sees its own PIDs

    @property
    def processes(self):
        return self._processes


class ContainerRuntime:
    def __init__(self):
        self.containers   = []
        self._pid_counter = 1000   # Host PID range allocator

    def create(self, cid, ip, image):
        c = Container(cid, ip, image, self._pid_counter)
        self._pid_counter += 100   # Reserve 100 PID slots per container
        self.containers.append(c)
        print(f"[RUNTIME] Created {cid!r} from image {image!r}, ip={ip}")
        return c

    def show_host_view(self):
        """Host sees every process from every container with real PIDs."""
        print("\n=== Host PID Namespace (full view) ===")
        for c in self.containers:
            for cpid, hpid, name in c.processes:
                print(f"  Host PID {hpid} -> container:{c.container_id} -> {name!r}")


# Demo
runtime = ContainerRuntime()

c1 = runtime.create('app-1', '172.17.0.2', 'nginx')
c1.spawn('nginx: master')
c1.spawn('nginx: worker')

c2 = runtime.create('app-2', '172.17.0.3', 'redis')
c2.spawn('redis-server')

c3 = runtime.create('app-3', '172.17.0.4', 'postgres')
c3.spawn('postgres: master')
c3.spawn('postgres: reader')

# Each container sees only its own processes
c1.show_isolation()
c2.show_isolation()
c3.show_isolation()

# Host can see all processes across all containers
runtime.show_host_view()
```

### Python — Medium Level
Simulate cgroups: CPU throttling when a container exceeds its quota, and OOM kill when memory limit is exceeded.

```python
# cgroups simulation: enforce CPU and memory limits per container.
# CPU throttling : container requests more CPU than quota -> capped.
# OOM kill       : container allocates more memory than limit -> killed.

class Cgroup:
    def __init__(self, name, cpu_quota_pct, memory_limit_mb):
        self.name              = name
        self.cpu_quota_pct     = cpu_quota_pct     # Max CPU % allowed
        self.memory_limit_mb   = memory_limit_mb   # Max memory in MB
        self.cpu_usage_pct     = 0
        self.memory_used_mb    = 0
        self.throttled         = False
        self.oom_killed        = False

    def set_cpu_usage(self, requested_pct):
        """Container requests CPU; cgroup caps it at the quota."""
        if self.oom_killed:
            print(f"  [{self.name}] Already killed — ignoring")
            return
        if requested_pct > self.cpu_quota_pct:
            self.throttled    = True
            self.cpu_usage_pct = self.cpu_quota_pct   # Cap at quota
            print(f"  [THROTTLE] {self.name}: requested {requested_pct}% "
                  f"-> capped at {self.cpu_quota_pct}%")
        else:
            self.throttled    = False
            self.cpu_usage_pct = requested_pct
            print(f"  [CPU OK]   {self.name}: {requested_pct}% "
                  f"(limit={self.cpu_quota_pct}%)")

    def allocate_memory(self, mb):
        """Container allocates memory; cgroup kills it if limit exceeded."""
        if self.oom_killed:
            print(f"  [{self.name}] Already killed — ignoring")
            return
        self.memory_used_mb += mb
        if self.memory_used_mb > self.memory_limit_mb:
            self.oom_killed = True
            print(f"  [OOM KILL] {self.name}: {self.memory_used_mb}MB "
                  f"> {self.memory_limit_mb}MB -> KILLED")
        else:
            print(f"  [MEM OK]   {self.name}: {self.memory_used_mb}/"
                  f"{self.memory_limit_mb}MB")

    def status(self):
        flags = (' [THROTTLED]' if self.throttled else '') + \
                (' [OOM-KILLED]' if self.oom_killed else '')
        print(f"  {self.name:<12} CPU:{self.cpu_usage_pct:>3}/{self.cpu_quota_pct}%  "
              f"MEM:{self.memory_used_mb:>5}/{self.memory_limit_mb}MB{flags}")


class CgroupManager:
    def __init__(self):
        self.cgroups = {}

    def add(self, name, cpu_quota_pct, memory_limit_mb) -> Cgroup:
        cg = Cgroup(name, cpu_quota_pct, memory_limit_mb)
        self.cgroups[name] = cg
        print(f"[CGROUP] {name}: CPU<={cpu_quota_pct}%, RAM<={memory_limit_mb}MB")
        return cg

    def get(self, name) -> Cgroup:
        return self.cgroups[name]

    def print_status(self):
        print("\n=== cgroup Status ===")
        for cg in self.cgroups.values():
            cg.status()


# Demo
mgr = CgroupManager()
mgr.add('web-app',  cpu_quota_pct=25, memory_limit_mb=512)
mgr.add('database', cpu_quota_pct=50, memory_limit_mb=2048)
mgr.add('worker',   cpu_quota_pct=10, memory_limit_mb=256)

print("\n--- Normal workloads ---")
mgr.get('web-app').set_cpu_usage(20)
mgr.get('web-app').allocate_memory(300)
mgr.get('database').set_cpu_usage(45)
mgr.get('database').allocate_memory(1500)

print("\n--- worker CPU spike -> throttled ---")
mgr.get('worker').set_cpu_usage(80)   # Exceeds 10% quota

print("\n--- web-app memory spike -> OOM kill ---")
mgr.get('web-app').allocate_memory(300)   # 300+300=600 > 512 MB

mgr.print_status()
```

---

## 10. Key Takeaways

- **OS-level virtualization** uses the host kernel's namespace and cgroup features to create isolated environments — no separate OS needed per container
- A **container** packages an app + all its dependencies + runtime into a portable unit; runs identically everywhere
- **Namespaces** give each container its own isolated view: PID tree, network stack, filesystem, hostname, user IDs
- **cgroups** enforce resource limits (CPU, memory, I/O) per container — prevent one container from starving others; also measure usage for billing
- **Containers vs VMs:** containers are lighter (MB vs GB), faster (seconds vs minutes), share kernel (weaker isolation); VMs are heavier, slower, but fully isolated (separate kernel)
- **Container image** = read-only layered template; **running container** = image + thin writable layer; **volume** = persistent data outside container
- **Ephemeral by design:** containers can be destroyed and recreated; store any data you care about in volumes
- **Use containers for:** microservices, cloud-native apps, CI/CD pipelines, dev environments, horizontal scaling
- **Don't use containers for:** running untrusted/hostile code (use VMs); running a different OS (Windows container on Linux)
- **Security hardening:** run as non-root, use Seccomp/AppArmor to restrict syscalls, keep base images updated
- **Docker** is the most popular container tool; **Kubernetes** orchestrates containers at scale across clusters of machines
- Windows/macOS Docker Desktop runs a lightweight Linux VM behind the scenes to host Linux containers
