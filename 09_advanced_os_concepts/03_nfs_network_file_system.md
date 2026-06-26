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

| Characteristic | Detail |
|---------------|--------|
| **Client-server model** | Server exports directories; clients mount them |
| **Transparent access** | Remote files look and behave like local files to apps |
| **Uses RPC** | Remote Procedure Calls carry file operations over the network |
| **Platform** | Native to Unix/Linux; also supported on Windows |
| **Current standard** | NFSv4 (stateful, Kerberos auth, firewall-friendly) |

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

| Feature | NFSv2 | NFSv3 | NFSv4 |
|---------|-------|-------|-------|
| **File size limit** | 2 GB | No limit | No limit |
| **Transport protocol** | UDP only | TCP and UDP | TCP only |
| **Stateful** | No | No | Yes |
| **Security** | Basic (AUTH_SYS) | Basic | Strong (Kerberos) |
| **Firewall friendly** | No (dynamic ports) | No | Yes (single port 2049) |
| **Performance** | Basic | Improved | Optimized |

### NFSv4 Key Improvements

- **Single port 2049 (TCP)** — firewall-friendly; previous versions used dynamically assigned RPC ports
- **Stateful protocol** — server tracks client state; enables file locking, better crash recovery
- **Strong authentication** via Kerberos — prevents IP spoofing and unauthorized access
- **Compound operations** — multiple NFS operations in one RPC call → reduced round trips
- **Delegation** — server can grant clients permission to cache data, reducing server load

For new deployments, **always use NFSv4**.

---

## 5. Advantages of NFS

| Advantage | Explanation |
|-----------|-------------|
| **Transparent access** | Apps use standard file calls — no special NFS API needed |
| **Centralized data** | One authoritative copy on the server; simplifies backups |
| **Reduced storage costs** | No need for large disks on every workstation; concentrate storage on server |
| **Real-time sharing** | Multiple clients see changes immediately (subject to cache timeout) |
| **Cross-platform** | Linux, Unix, macOS clients can all access the same NFS exports |
| **Easy home directory sharing** | Log in at any workstation → your files are there (via NFS-mounted /home) |

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

| Aspect | NFS | SMB/CIFS | AFS |
|--------|-----|---------|-----|
| **Primary platform** | Unix/Linux | Windows | Cross-platform |
| **Performance (LAN)** | Fast | Good | Moderate |
| **WAN performance** | Poor | Moderate | Excellent (aggressive caching) |
| **Caching** | Client-side | Client-side | Advanced distributed caching |
| **Security** | Moderate (v4 = strong) | Good (Active Directory) | Strong (Kerberos) |
| **Scalability** | Good | Moderate | Excellent |
| **Complexity** | Low–moderate | Low | High |
| **Best for** | Linux/Unix shops, data centers | Windows environments | Geographically distributed teams |

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

| Method | Security Level | Details |
|--------|---------------|---------|
| **AUTH_SYS** (NFSv2/v3 default) | Weak | Trusts client's uid/gid — easily spoofed |
| **AUTH_NULL** | None | No authentication — only for truly trusted networks |
| **Kerberos** (NFSv4) | Strong | Cryptographic ticket-based authentication; prevents spoofing |

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

| Option Trade-off | Fast but Risky | Safe but Slower |
|------------------|---------------|----------------|
| Write safety | `async` | `sync` |
| On server failure | `soft` | `hard` |
| Consistency | `ac` (caching on) | `noac` (caching off) |

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

## 12. Key Takeaways

- **NFS** allows computers to access files on a remote server over a network as if they were local — uses RPC to transparently forward file operations
- **Client-server model:** server *exports* directories; client *mounts* them at a local mount point
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
