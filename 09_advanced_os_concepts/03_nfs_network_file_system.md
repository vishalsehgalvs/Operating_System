# NFS: Network File System Explained

> **NFS (Network File System)** is a protocol that lets a computer access files on a remote server over a network as if they were local — the OS intercepts file operations and transparently forwards them to the NFS server via RPC, so users and apps work with remote files using exactly the same commands as local files.

---

## Table of Contents

1. [What is NFS?](#1-what-is-nfs)
2. [NFS Architecture](#2-nfs-architecture)
3. [How NFS Works](#3-how-nfs-works)
4. [NFS Versions](#4-nfs-versions)
5. [Advantages of NFS](#5-advantages-of-nfs)
6. [Disadvantages and Limitations](#6-disadvantages-and-limitations)
7. [NFS vs Other File Sharing Protocols](#7-nfs-vs-other-file-sharing-protocols)
8. [Configuring NFS](#8-configuring-nfs)
9. [Security Considerations](#9-security-considerations)
10. [Performance Tuning](#10-performance-tuning)
11. [Troubleshooting](#11-troubleshooting)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What is NFS?

**NFS (Network File System)** is a distributed file system protocol developed by Sun Microsystems in 1984. It allows one machine (server) to share a directory over a network, and other machines (clients) to mount that directory and access its files as if they were local.

**Analogy:** A shared family photo album stored in the living room. Instead of making copies for each family member, you keep one album in a central location and everyone can view it whenever they want — no copying needed.

```
  Without NFS:
  Dev1 has file → manually copies to Dev2, Dev3, Dev4
  Dev1 updates file → must re-copy to all → stale copies everywhere

  With NFS:
  Server: exports /home/projects/
  Dev1, Dev2, Dev3, Dev4: all mount /home/projects/ → all see the SAME files
  Dev1 saves a change → everyone sees it immediately
```

### Key Characteristics

| Characteristic          | Detail                                                        |
| ----------------------- | ------------------------------------------------------------- |
| **Client-server model** | Server exports directories; clients mount them                |
| **Transparent access**  | Remote files look and behave like local files to apps         |
| **Uses RPC**            | Remote Procedure Calls carry file operations over the network |
| **Platform**            | Native to Unix/Linux; also supported on Windows               |
| **Current standard**    | NFSv4 (stateful, Kerberos auth, firewall-friendly)            |

---

## 2. NFS Architecture

```
  NFS Server                    Network                  NFS Client
  ──────────────                ────────                 ──────────────
  /home/shared/ (real disk)     TCP/IP                   /mnt/shared/ (mount point)
        ↑                          ↑                            ↓
  NFS Server daemon       ←────────────────────→       NFS Client in OS
  (nfsd, rpcbind)              NFS over RPC             (intercepts file calls)

  User on client:
  open("/mnt/shared/report.txt")
  → OS sees this is NFS mount
  → NFS client sends OPEN RPC to server
  → Server opens /home/shared/report.txt on its local disk
  → Returns file handle to client
  → Client reads through NFS READ RPC calls
  → User gets the data — transparently
```

### Server Side: Exporting Directories

The server **exports** specific directories it wants to share. Only those directories are visible to NFS clients — the rest of the server's filesystem remains private.

### Client Side: Mounting

The client **mounts** an exported directory at a local mount point. After mounting:

- `ls /mnt/shared` lists files from the server's disk
- `nano /mnt/shared/config.txt` edits the file directly on the server
- Multiple clients can mount the same export simultaneously

---

## 3. How NFS Works

```
  Step-by-step: client reads a file from NFS mount

  1. App calls: open("/mnt/shared/data.csv", O_RDONLY)
  2. OS identifies /mnt/shared is an NFS mount
  3. NFS client sends: LOOKUP RPC → "give me the file handle for data.csv"
  4. Server responds: file handle (opaque ID representing the file)
  5. NFS client sends: READ RPC → "read 4096 bytes from handle X at offset 0"
  6. Server reads from its local disk, returns data
  7. NFS client stores data in local cache
  8. App receives data — just like a local read
```

### Caching Mechanism

NFS clients cache file data and metadata locally to reduce network trips:

- **Attribute cache:** caches file size, modification time, permissions
- **Data cache:** caches file contents for repeated reads

**Consistency challenge:** If Client A caches a file and Client B modifies it on the server, Client A's cache is stale. NFS uses **cache timeouts** and **close-to-open consistency** to manage this — but it's not instant.

---

## 4. NFS Versions

| Feature                | NFSv2              | NFSv3       | NFSv4                  |
| ---------------------- | ------------------ | ----------- | ---------------------- |
| **File size limit**    | 2 GB               | No limit    | No limit               |
| **Transport protocol** | UDP only           | TCP and UDP | TCP only               |
| **Stateful**           | No                 | No          | Yes                    |
| **Security**           | Basic (AUTH_SYS)   | Basic       | Strong (Kerberos)      |
| **Firewall friendly**  | No (dynamic ports) | No          | Yes (single port 2049) |
| **Performance**        | Basic              | Improved    | Optimized              |

### NFSv4 Key Improvements

- **Single port 2049 (TCP)** — firewall-friendly; previous versions used dynamically assigned RPC ports
- **Stateful protocol** — server tracks client state; enables file locking, better crash recovery
- **Strong authentication** via Kerberos — prevents IP spoofing and unauthorized access
- **Compound operations** — multiple NFS operations in one RPC call → reduced round trips
- **Delegation** — server can grant clients permission to cache data, reducing server load

For new deployments, **always use NFSv4**.

---

## 5. Advantages of NFS

| Advantage                       | Explanation                                                                 |
| ------------------------------- | --------------------------------------------------------------------------- |
| **Transparent access**          | Apps use standard file calls — no special NFS API needed                    |
| **Centralized data**            | One authoritative copy on the server; simplifies backups                    |
| **Reduced storage costs**       | No need for large disks on every workstation; concentrate storage on server |
| **Real-time sharing**           | Multiple clients see changes immediately (subject to cache timeout)         |
| **Cross-platform**              | Linux, Unix, macOS clients can all access the same NFS exports              |
| **Easy home directory sharing** | Log in at any workstation → your files are there (via NFS-mounted /home)    |

---

## 6. Disadvantages and Limitations

### Network Dependency

```
  NFS server goes down → ALL clients lose access to ALL mounted filesystems
  Network congestion → dramatically slower file access (latency adds up)

  For comparison:
  Local disk read:     ~0.1ms
  LAN NFS read:        ~1–5ms (10–50x slower)
  WAN NFS read:        ~50–200ms (500–2000x slower)
```

### Security Concerns (NFSv2/v3)

- **No strong authentication** — trusts client's claimed user ID (`AUTH_SYS`)
- Easy to spoof: attacker claims to be uid=0 (root) → gains full access
- Data travels in plaintext by default
- NFSv4 with Kerberos solves this, but requires proper infrastructure setup

### Consistency Challenges

- Caching means clients may see stale data
- File locking is advisory, not mandatory — applications must cooperate
- Simultaneous writes from multiple clients can corrupt files if apps don't use locks

---

## 7. NFS vs Other File Sharing Protocols

| Aspect                | NFS                            | SMB/CIFS                | AFS                              |
| --------------------- | ------------------------------ | ----------------------- | -------------------------------- |
| **Primary platform**  | Unix/Linux                     | Windows                 | Cross-platform                   |
| **Performance (LAN)** | Fast                           | Good                    | Moderate                         |
| **WAN performance**   | Poor                           | Moderate                | Excellent (aggressive caching)   |
| **Caching**           | Client-side                    | Client-side             | Advanced distributed caching     |
| **Security**          | Moderate (v4 = strong)         | Good (Active Directory) | Strong (Kerberos)                |
| **Scalability**       | Good                           | Moderate                | Excellent                        |
| **Complexity**        | Low–moderate                   | Low                     | High                             |
| **Best for**          | Linux/Unix shops, data centers | Windows environments    | Geographically distributed teams |

**Choosing:**

- Unix/Linux environment → **NFS** (native, seamless integration)
- Windows-centric → **SMB/CIFS** (Windows shares, Active Directory)
- Distributed offices across the globe → **AFS** (better WAN caching)

---

## 8. Configuring NFS

### Server: `/etc/exports`

```bash
# /etc/exports — defines what to share and who can access it

# Share /home/shared with entire 192.168.1.0/24 subnet — read/write
/home/shared   192.168.1.0/24(rw,sync,no_subtree_check)

# Share /var/data with specific client — read-only
/var/data      192.168.1.100(ro,sync,no_subtree_check)

# Key options:
# rw        = read-write access
# ro        = read-only access
# sync      = write data to disk before acknowledging (safer, slightly slower)
# async     = acknowledge before writing to disk (faster, less safe)
# no_subtree_check = don't verify subtree, improves reliability
# no_root_squash   = allow root on client to act as root on server (DANGEROUS)
# root_squash      = (default) remap root on client to nobody on server (safer)
```

After editing:

```bash
sudo exportfs -ra    # reload exports without restart
sudo systemctl restart nfs-server
```

### Client: Mounting

```bash
# Manual mount: mount NFS share from 192.168.1.50 at local /mnt/shared
sudo mount -t nfs 192.168.1.50:/home/shared /mnt/shared

# Verify it's mounted
df -h | grep shared
mount | grep nfs

# Permanent mount (survives reboots) — add to /etc/fstab:
192.168.1.50:/home/shared  /mnt/shared  nfs  defaults,_netdev  0  0

# _netdev tells systemd to wait for network before mounting
```

---

## 9. Security Considerations

### Authentication Methods

| Method                          | Security Level | Details                                                      |
| ------------------------------- | -------------- | ------------------------------------------------------------ |
| **AUTH_SYS** (NFSv2/v3 default) | Weak           | Trusts client's uid/gid — easily spoofed                     |
| **AUTH_NULL**                   | None           | No authentication — only for truly trusted networks          |
| **Kerberos** (NFSv4)            | Strong         | Cryptographic ticket-based authentication; prevents spoofing |

```
  AUTH_SYS weakness:
  Attacker on network claims: "I am uid=0, hostname=trusted-client"
  NFSv3 server: believes it → attacker gets root access to all files

  Kerberos:
  Client must present valid encrypted ticket from KDC (Key Distribution Center)
  No ticket → no access. Tickets expire → limited damage from stolen tickets.
```

### Encryption

- NFSv2/v3: data travels in plaintext — anyone with network access can read files
- NFSv4 with Kerberos **privacy mode**: encrypts all data in transit
- Enable: `sec=krb5p` in mount options (highest security, some CPU overhead)

### Firewall Configuration

```bash
# NFSv4 only needs one port:
sudo firewall-cmd --add-service=nfs --permanent
# Opens TCP 2049

# NFSv3 also needs:
sudo firewall-cmd --add-service=rpc-bind --permanent   # port 111
sudo firewall-cmd --add-service=mountd --permanent      # dynamic port
sudo firewall-cmd --reload
```

---

## 10. Performance Tuning

```bash
# Tune transfer block size (experiment with 8192 to 65536)
sudo mount -t nfs -o rsize=65536,wsize=65536 server:/share /mnt/share

# Key mount options for performance:
# rsize=N     = read block size in bytes
# wsize=N     = write block size in bytes
# hard        = keep retrying on server failure (prevents data loss)
# soft        = fail after timeout (prevents hangs, risks data loss)
# async       = don't wait for write acknowledgment (fast but risky)
# ac          = attribute caching enabled (default, improves performance)
# noac        = disable attribute caching (immediate consistency, slower)
# actimeo=N   = cache attribute for N seconds (default: 30-60s)
```

| Option Trade-off  | Fast but Risky    | Safe but Slower      |
| ----------------- | ----------------- | -------------------- |
| Write safety      | `async`           | `sync`               |
| On server failure | `soft`            | `hard`               |
| Consistency       | `ac` (caching on) | `noac` (caching off) |

---

## 11. Troubleshooting

```bash
# Check what server is exporting
showmount -e 192.168.1.50

# Mount failures — systematic checklist:
ping 192.168.1.50              # 1. Basic connectivity?
rpcinfo -p 192.168.1.50        # 2. NFS services running on server?
sudo systemctl status nfs-server  # 3. Server daemon up?
cat /etc/exports               # 4. Directory exported correctly?
sudo exportfs -v               # 5. What is actually exported?

# Common errors:
# "No route to host"      → network/firewall problem
# "Permission denied"     → check /etc/exports, uid mapping, root_squash
# "Stale file handle"     → server FS was remounted/rebooted; unmount and remount on client
# "mount.nfs: access denied" → check exports restrict client by IP/hostname
```

**Stale file handle** — most common NFS error:

```bash
# Server rebooted or FS was unmounted while clients had files open
sudo umount -f -l /mnt/shared   # force lazy unmount
sudo mount -t nfs server:/share /mnt/shared   # remount
```

---

## 12. Code Examples

> Working code that demonstrates NFS concepts — client-server RPC file access and client-side caching with consistency revalidation — in practice.

### C++ — Simple Version

Simulate NFS client-server: server exports a directory via RPC-style calls; client mounts it and performs transparent file operations.

```cpp
#include <iostream>
#include <string>
#include <map>
#include <vector>

// NFS Server — stores files, handles RPC-style calls.
// Real NFS uses XDR-encoded RPC over UDP/TCP; here we simulate the logic.

class NFSServer {
    std::map<std::string, std::string> files;  // {filename -> content}
    std::string exportPath;

public:
    NFSServer(const std::string& path) : exportPath(path) {
        std::cout << "[SERVER] NFS server started, exporting: " << path << "\n";
    }

    // RPC: read file content
    std::string rpc_read(const std::string& filename, std::string& err) {
        auto it = files.find(filename);
        if (it == files.end()) { err = "ENOENT"; return ""; }
        std::cout << "[SERVER] rpc_read(" << filename << ")\n";
        return it->second;
    }

    // RPC: write file content
    void rpc_write(const std::string& filename, const std::string& content) {
        files[filename] = content;
        std::cout << "[SERVER] rpc_write(" << filename << ", "
                  << content.size() << " bytes)\n";
    }

    // RPC: list directory entries
    std::vector<std::string> rpc_readdir() {
        std::cout << "[SERVER] rpc_readdir()\n";
        std::vector<std::string> names;
        for (auto& [name, _] : files) names.push_back(name);
        return names;
    }

    // RPC: create empty file
    void rpc_create(const std::string& filename) {
        files[filename] = "";
        std::cout << "[SERVER] rpc_create(" << filename << ")\n";
    }

    // RPC: delete file
    void rpc_delete(const std::string& filename) {
        files.erase(filename);
        std::cout << "[SERVER] rpc_delete(" << filename << ")\n";
    }
};

// NFS Client — mounts a remote directory; file ops are forwarded to the server.
class NFSClient {
    NFSServer* server;
    std::string mountPoint;

public:
    NFSClient(NFSServer* srv, const std::string& mp)
        : server(srv), mountPoint(mp) {
        std::cout << "[CLIENT] Mounted remote directory at " << mp << "\n\n";
    }

    std::string read(const std::string& path) {
        std::string err;
        std::string content = server->rpc_read(path, err);
        if (!err.empty()) {
            std::cout << "[CLIENT] read(" << path << ") -> Error: " << err << "\n";
            return "";
        }
        std::cout << "[CLIENT] read(" << mountPoint << "/" << path
                  << ") -> \"" << content << "\"\n";
        return content;
    }

    void write(const std::string& path, const std::string& content) {
        server->rpc_write(path, content);
        std::cout << "[CLIENT] write(" << mountPoint << "/" << path << ") done\n";
    }

    void listDir() {
        auto files = server->rpc_readdir();
        std::cout << "[CLIENT] ls " << mountPoint << "/ -> [";
        for (size_t i = 0; i < files.size(); i++) {
            std::cout << files[i] << (i + 1 < files.size() ? ", " : "");
        }
        std::cout << "]\n";
    }

    void create(const std::string& path) {
        server->rpc_create(path);
        std::cout << "[CLIENT] create(" << path << ") done\n";
    }

    void remove(const std::string& path) {
        server->rpc_delete(path);
        std::cout << "[CLIENT] delete(" << path << ") done\n";
    }
};

int main() {
    NFSServer server("/srv/shared");
    NFSClient client1(&server, "/mnt/shared");
    NFSClient client2(&server, "/mnt/shared");

    std::cout << "=== Client 1 creates and writes files ===\n";
    client1.create("readme.txt");
    client1.write("readme.txt", "Hello from client1!");
    client1.create("data.csv");
    client1.write("data.csv", "id,value\n1,100\n2,200\n");

    std::cout << "\n=== Client 2 sees the same shared directory ===\n";
    client2.listDir();
    client2.read("readme.txt");

    std::cout << "\n=== Client 1 deletes a file; Client 2 observes ===\n";
    client1.remove("data.csv");
    client2.listDir();
    client2.read("data.csv");   // ENOENT — file is gone

    return 0;
}
```

### C++ — Medium / LeetCode Style

NFS client-side caching with attribute revalidation: client caches file data and version; before using cache it checks the server's current version for consistency.

```cpp
#include <iostream>
#include <string>
#include <map>
#include <chrono>

using Clock     = std::chrono::steady_clock;
using TimePoint = Clock::time_point;

// NFSv3-style client caching with close-to-open consistency.
// Client caches file content + version number.
// Before using cached data: check if server's version matches (getattr call).
// On write: update server AND invalidate local cache immediately.

struct FileEntry { std::string content; int version = 0; };

class NFSServer2 {
    std::map<std::string, FileEntry> files;
public:
    NFSServer2() {
        files["config.txt"] = {"host=10.0.0.1\nport=8080\n", 1};
    }

    // Cheap metadata RPC — returns version only (no data transfer)
    int getVersion(const std::string& name) {
        if (!files.count(name)) return -1;
        std::cout << "[SRV] getattr(" << name << ") -> version=" << files[name].version << "\n";
        return files[name].version;
    }

    // Full read RPC — returns content + version
    std::pair<std::string,int> read(const std::string& name) {
        if (!files.count(name)) return {"", -1};
        auto& f = files[name];
        std::cout << "[SRV] read(" << name << ") v" << f.version
                  << " (" << f.content.size() << " bytes)\n";
        return {f.content, f.version};
    }

    void write(const std::string& name, const std::string& content) {
        files[name].content = content;
        files[name].version++;
        std::cout << "[SRV] write(" << name << ") -> new version=" << files[name].version << "\n";
    }
};

struct CachedEntry {
    std::string content;
    int         cachedVersion;
    TimePoint   cacheTime;
    int         ttlSeconds = 5;   // Attribute cache timeout
};

class CachingNFSClient {
    NFSServer2* server;
    std::map<std::string, CachedEntry> cache;

    bool isCacheValid(const std::string& name) {
        auto& e   = cache[name];
        int ageSec = (int)std::chrono::duration_cast<std::chrono::seconds>(
                         Clock::now() - e.cacheTime).count();
        if (ageSec >= e.ttlSeconds) {
            std::cout << "[CLI] Cache expired for '" << name
                      << "' (age=" << ageSec << "s)\n";
            return false;
        }
        // Close-to-open consistency: verify version hasn't changed on server
        int serverVer = server->getVersion(name);
        if (serverVer != e.cachedVersion) {
            std::cout << "[CLI] Cache stale: local v" << e.cachedVersion
                      << " server v" << serverVer << "\n";
            return false;
        }
        std::cout << "[CLI] Cache valid for '" << name << "' (v" << e.cachedVersion << ")\n";
        return true;
    }

public:
    CachingNFSClient(NFSServer2* srv) : server(srv) {}

    std::string read(const std::string& name) {
        if (cache.count(name) && isCacheValid(name)) {
            std::cout << "[CLI] Cache HIT — returning cached content\n";
            return cache[name].content;
        }
        auto [content, version] = server->read(name);
        cache[name] = {content, version, Clock::now()};
        return content;
    }

    void write(const std::string& name, const std::string& content) {
        server->write(name, content);
        cache.erase(name);   // Invalidate cache on write
        std::cout << "[CLI] Cache invalidated for '" << name << "'\n";
    }
};

int main() {
    NFSServer2 server;
    CachingNFSClient client(&server);

    std::cout << "=== Read 1: cache miss — fetches from server ===\n";
    std::cout << client.read("config.txt") << "\n";

    std::cout << "=== Read 2: cache hit — no server RPC ===\n";
    std::cout << client.read("config.txt") << "\n";

    std::cout << "=== Another client writes to the server ===\n";
    server.write("config.txt", "host=10.0.0.2\nport=9090\n");

    std::cout << "=== Read 3: stale cache detected — refetches from server ===\n";
    std::cout << client.read("config.txt") << "\n";

    return 0;
}
```

### Python — Simple Version

NFS client-server simulation: server exposes RPC-style file operations; client mounts the remote directory and accesses files transparently.

```python
# NFS client-server simulation using RPC-style calls.
# Server exports a directory; clients mount it and perform file operations
# as if the files were local — all I/O is transparently forwarded via "RPC".

class NFSServer:
    """Simulates an NFS server that exports a directory."""

    def __init__(self, export_path):
        self.export_path = export_path
        self._files = {}           # {filename: content}
        print(f"[SERVER] NFS server started — exporting {export_path}\n")

    def rpc_read(self, filename):
        if filename not in self._files:
            raise FileNotFoundError(f"ENOENT: {filename!r}")
        print(f"[SERVER] rpc_read({filename!r})")
        return self._files[filename]

    def rpc_write(self, filename, content):
        self._files[filename] = content
        print(f"[SERVER] rpc_write({filename!r}, {len(content)} bytes)")

    def rpc_readdir(self):
        print("[SERVER] rpc_readdir()")
        return list(self._files.keys())

    def rpc_create(self, filename):
        self._files[filename] = ''
        print(f"[SERVER] rpc_create({filename!r})")

    def rpc_delete(self, filename):
        self._files.pop(filename, None)
        print(f"[SERVER] rpc_delete({filename!r})")


class NFSClient:
    """Simulates an NFS client that mounts a remote directory."""

    def __init__(self, server: NFSServer, mount_point):
        self.server = server
        self.mount_point = mount_point
        print(f"[CLIENT] Mounted remote dir at {mount_point}\n")

    def read(self, path):
        try:
            content = self.server.rpc_read(path)
            print(f"[CLIENT] read({self.mount_point}/{path}) -> {content!r}")
            return content
        except FileNotFoundError as e:
            print(f"[CLIENT] read({path!r}) ERROR: {e}")
            return None

    def write(self, path, content):
        self.server.rpc_write(path, content)
        print(f"[CLIENT] write({self.mount_point}/{path}) done")

    def listdir(self):
        files = self.server.rpc_readdir()
        print(f"[CLIENT] ls {self.mount_point}/ -> {files}")
        return files

    def create(self, path):
        self.server.rpc_create(path)
        print(f"[CLIENT] create({path!r}) done")

    def remove(self, path):
        self.server.rpc_delete(path)
        print(f"[CLIENT] delete({path!r}) done")


# Demo: two clients sharing the same NFS server
server  = NFSServer('/srv/shared')
client1 = NFSClient(server, '/mnt/shared')
client2 = NFSClient(server, '/mnt/shared')

print("=== Client 1 creates and writes files ===")
client1.create('readme.txt')
client1.write('readme.txt', 'Hello from client1!')
client1.create('data.csv')
client1.write('data.csv', 'id,value\n1,100\n2,200\n')

print("\n=== Client 2 sees the same shared directory ===")
client2.listdir()
client2.read('readme.txt')

print("\n=== Client 2 deletes a file; client 1 observes ===")
client2.remove('data.csv')
client1.listdir()
client1.read('data.csv')   # Error — file is gone
```

### Python — Medium Level

NFS client-side caching with attribute TTL and close-to-open consistency: client revalidates cached data against server version before every use.

```python
import time

# NFS client-side caching with close-to-open consistency.
# Client caches file content + version locally.
# Before using cached data: check if cache TTL has expired OR server version changed.
# On write: immediately invalidate local cache and update the server.

class NFSServer2:
    def __init__(self):
        self._files = {
            'config.txt': {'content': 'host=10.0.0.1\nport=8080\n', 'version': 1}
        }

    def getattr(self, name):
        """Cheap metadata RPC — returns file version only (no data)."""
        entry = self._files.get(name)
        if not entry:
            return None
        print(f"[SRV] getattr({name!r}) -> version={entry['version']}")
        return entry['version']

    def read(self, name):
        """Full read RPC — returns (content, version)."""
        entry = self._files.get(name)
        if not entry:
            raise FileNotFoundError(f"ENOENT: {name!r}")
        print(f"[SRV] read({name!r}) -> v{entry['version']}, "
              f"{len(entry['content'])} bytes")
        return entry['content'], entry['version']

    def write(self, name, content):
        if name not in self._files:
            self._files[name] = {'content': '', 'version': 0}
        self._files[name]['content'] = content
        self._files[name]['version'] += 1
        print(f"[SRV] write({name!r}) -> new version={self._files[name]['version']}")


class CachingNFSClient:
    AC_TIMEOUT = 3   # Attribute cache timeout in seconds

    def __init__(self, server: NFSServer2):
        self.server = server
        self._cache = {}   # {name: {content, version, cached_at}}

    def _is_cache_valid(self, name):
        entry = self._cache.get(name)
        if not entry:
            return False
        age = time.time() - entry['cached_at']
        if age >= self.AC_TIMEOUT:
            print(f"[CLI] Cache expired for {name!r} (age={age:.1f}s)")
            return False
        # Close-to-open consistency: verify version matches server
        server_ver = self.server.getattr(name)
        if server_ver != entry['version']:
            print(f"[CLI] Cache stale: local v{entry['version']}, "
                  f"server v{server_ver}")
            return False
        print(f"[CLI] Cache valid for {name!r} (v{entry['version']})")
        return True

    def read(self, name):
        if self._is_cache_valid(name):
            print("[CLI] Cache HIT — returning cached content")
            return self._cache[name]['content']
        # Cache miss or stale — fetch from server
        content, version = self.server.read(name)
        self._cache[name] = {
            'content':   content,
            'version':   version,
            'cached_at': time.time()
        }
        return content

    def write(self, name, content):
        self.server.write(name, content)
        self._cache.pop(name, None)   # Invalidate cache immediately
        print(f"[CLI] Cache invalidated for {name!r}")


# Demo
server = NFSServer2()
client = CachingNFSClient(server)

print("=== Read 1: cache miss — fetches from server ===")
print(client.read('config.txt'))

print("\n=== Read 2: cache hit — no server RPC ===")
print(client.read('config.txt'))

print("\n=== Another client writes to the server ===")
server.write('config.txt', 'host=10.0.0.2\nport=9090\n')

print("\n=== Read 3: stale cache detected — refetches from server ===")
print(client.read('config.txt'))
```

---

## 13. Key Takeaways

- **NFS** allows computers to access files on a remote server over a network as if they were local — uses RPC to transparently forward file operations
- **Client-server model:** server _exports_ directories; client _mounts_ them at a local mount point
- **NFS uses caching** (attribute cache + data cache) to reduce network round-trips; trades consistency for performance
- **Three versions:** NFSv2 (obsolete), NFSv3 (still common), NFSv4 (current standard — use this)
- **NFSv4 advantages:** single port 2049 (firewall-friendly), stateful, Kerberos auth, compound operations
- **NFSv2/v3 security weakness:** AUTH_SYS trusts claimed user IDs — easily spoofed; only use on fully trusted private networks
- **NFSv4 + Kerberos:** cryptographic authentication; `sec=krb5p` enables encryption of data in transit
- **Server config:** `/etc/exports` defines what to share, to whom, and with what permissions
- **Client config:** `mount -t nfs server:/path /local/point` or `/etc/fstab` with `_netdev` option
- **Performance tuning:** `rsize`/`wsize` block sizes, `async` vs `sync`, `ac` vs `noac`
- **Key limitation:** depends entirely on network — server failure or congestion = clients lose access
- **NFS vs SMB:** NFS is native on Linux/Unix; SMB/CIFS is native on Windows; AFS excels over WAN
- **Common use cases:** centralized home directories, shared development workspaces, data center shared storage, VM image storage
