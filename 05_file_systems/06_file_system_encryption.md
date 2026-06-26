# File System Encryption

> File system encryption transforms your stored data from readable plaintext into unreadable ciphertext — even if someone steals your hard drive, they see nothing without the decryption key; modern OSes provide this transparently through BitLocker (Windows), FileVault (macOS), and LUKS (Linux) with near-zero performance cost on modern hardware.

---

## Table of Contents

1. [What Is File System Encryption?](#1-what-is-file-system-encryption)
2. [Why Encryption Matters](#2-why-encryption-matters)
3. [File-Level vs Disk-Level Encryption](#3-file-level-vs-disk-level-encryption)
4. [Common Encryption Algorithms](#4-common-encryption-algorithms)
5. [How Encryption Works in Practice](#5-how-encryption-works-in-practice)
6. [Key Management](#6-key-management)
7. [Encryption in Popular OSes](#7-encryption-in-popular-oses)
8. [Performance Considerations](#8-performance-considerations)
9. [Limitations](#9-limitations)
10. [Best Practices](#10-best-practices)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is File System Encryption?

**File system encryption** encodes files/directories so that unauthorized users see only meaningless bytes — not the actual content.

**Diary lock analogy:**

```
  Unencrypted file:  Anyone who picks up your diary can read it word for word.
  Encrypted file:    Your diary is written in a secret code.
                     Only you (with the key) can decode it.
                     Thief picks it up → sees random gibberish.
```

**Key terms:**

| Term               | Meaning                                         |
| ------------------ | ----------------------------------------------- |
| **Plaintext**      | Original readable data: `"Hello World"`         |
| **Ciphertext**     | Encrypted unreadable data: `"8fj3k9s2m1x7q4w0"` |
| **Encryption key** | The secret used to encrypt/decrypt              |
| **Algorithm**      | The mathematical method (e.g., AES)             |

The **encryption key** is everything — losing it means losing the data permanently. There are no backdoors in properly implemented encryption.

---

## 2. Why Encryption Matters

```
  Scenario 1: Laptop stolen from a coffee shop
  No encryption → thief attaches drive to another PC → reads everything
  With encryption → thief sees: "k2m9x7p4s3w1z8q5..." → useless without key

  Scenario 2: Hard drive discarded
  No encryption → data recovery tools can extract sensitive files
  With encryption → wiping the key makes all data permanently unreadable

  Scenario 3: Regulatory compliance (HIPAA, GDPR, PCI-DSS)
  Many regulations REQUIRE encryption of sensitive data at rest
  Encryption = one checkbox in compliance audits
```

---

## 3. File-Level vs Disk-Level Encryption

### File-Level Encryption (File-Based Encryption — FBE)

- Encrypts **individual files or directories** with their own keys
- Flexible: choose which files to encrypt
- Weakness: **metadata (filenames, sizes, directory structure) may remain visible**

```
  /documents/
    tax_return_2024.pdf  ← encrypted
    public_photo.jpg     ← not encrypted (your choice)
    salary.xlsx          ← encrypted

  Attacker can still see: "there's a file called tax_return_2024.pdf"
  But cannot read its content.
```

### Disk-Level Encryption (Full Disk Encryption — FDE)

- Encrypts the **entire disk or partition** — OS, apps, all files, all metadata
- Unlock once at boot → everything decrypts transparently
- Weakness: once unlocked, everything decrypts automatically — no per-file choice

```
  Boot sequence with FDE:
  1. BIOS loads tiny pre-boot code
  2. You enter decryption password / TPM chip provides key
  3. Entire disk decrypts in real-time
  4. OS boots normally; you use it exactly as without encryption

  Attacker who steals the drive sees: gibberish from sector 0 to the last byte
```

### Comparison

| Aspect           | File-Level             | Disk-Level                |
| ---------------- | ---------------------- | ------------------------- |
| What's encrypted | Chosen files/folders   | Everything on the disk    |
| Metadata visible | Yes (filenames, sizes) | No (all hidden)           |
| User effort      | Manual selection       | Automatic once enabled    |
| Flexibility      | High                   | Low (all or nothing)      |
| Boot protection  | No                     | Yes (OS itself encrypted) |
| Key management   | Per-file complexity    | Single master key         |

---

## 4. Common Encryption Algorithms

### AES (Advanced Encryption Standard)

**The universal standard** for file system encryption — used by BitLocker, FileVault, LUKS, and virtually every modern encryption tool.

```
  AES is a symmetric block cipher:
  - Encrypts data in 128-bit (16-byte) blocks
  - Key sizes: 128-bit, 192-bit, or 256-bit
  - AES-256 = 2²⁵⁶ possible keys ≈ more keys than atoms in the universe

  Hardware acceleration:
  Modern CPUs have AES-NI (AES New Instructions) — dedicated circuits
  for AES operations. On these CPUs, encryption overhead < 5%.
```

| Algorithm | Key Size | Status                          | Used In                     |
| --------- | -------- | ------------------------------- | --------------------------- |
| AES-128   | 128-bit  | Secure                          | BitLocker default           |
| AES-256   | 256-bit  | Very secure                     | Government, LUKS, FileVault |
| XTS-AES   | 256-bit  | SSD/disk optimized              | APFS, FileVault, VeraCrypt  |
| ChaCha20  | 256-bit  | Efficient on no-AES-NI hardware | Mobile, embedded            |

**XTS mode** (XEX-based Tweaked CodeBook with ciphertext Stealing) is the standard mode for disk encryption — it ensures that identical plaintext blocks at different disk positions produce different ciphertext, preventing pattern analysis.

---

## 5. How Encryption Works in Practice

```
  WRITE path (with full disk encryption):

  Application: write("Confidential Report")
       │
       ▼
  File System Layer
       │
       ▼
  Encryption Layer:
    plaintext = "Confidential Report"
    key       = [derived from user password + TPM]
    ciphertext = AES-256-XTS(key, plaintext, sector_number)
    result    = "x7k9m2p4s8w1q5z3a2b4c6d8..."
       │
       ▼
  Physical Disk: stores "x7k9m2p4s8w1q5z3a2b4c6d8..."

  ─────────────────────────────────────────────

  READ path:

  Physical Disk: returns "x7k9m2p4s8w1q5z3a2b4c6d8..."
       │
       ▼
  Encryption Layer:
    ciphertext = "x7k9m2p4s8w1q5z3a2b4c6d8..."
    plaintext  = AES-256-XTS-decrypt(key, ciphertext, sector_number)
    result     = "Confidential Report"
       │
       ▼
  Application receives: "Confidential Report"
```

This is **transparent encryption** — applications don't know it's happening.

---

## 6. Key Management

The key is the most critical component. If someone gets your key, they can decrypt everything regardless of algorithm strength.

### Password-Derived Keys

```
  Process:
  User password → Key Derivation Function (PBKDF2 / Argon2) → Encryption key

  KDF runs 100,000+ iterations intentionally → brute-forcing is slow
  With Argon2: ~1 second per guess on modern hardware
  → Attacker needs years/centuries to try all common passwords

  Critical: choose a STRONG passphrase (12+ characters)
```

### Hardware Keys (TPM)

```
  TPM (Trusted Platform Module) = a secure chip on your motherboard

  Benefits:
  - Stores the encryption key inside the chip, never exposed to software
  - Can auto-unlock if system is in a "trusted state" (same hardware, secure boot)
  - Even malware cannot extract the key from TPM

  Risk:
  - If motherboard fails → key is gone (need recovery key backup!)

  BitLocker with TPM: no password needed at boot → seamless but less portable
  BitLocker with TPM + PIN: TPM provides hardware binding + you provide the PIN
```

### Key Escrow / Recovery Keys

```
  ALWAYS BACK UP YOUR RECOVERY KEY!

  BitLocker: generates 48-digit recovery key → save to Microsoft account, USB, printout
  FileVault: generates recovery key → store it safely
  LUKS: can have multiple passwords → keep one in a safe offline location

  If you forget your password AND lose the recovery key:
  → Data is GONE permanently. No court order, no vendor, no "password reset" helps.
```

---

## 7. Encryption in Popular OSes

### BitLocker (Windows)

- **Available in**: Windows Pro, Enterprise, Education (not Home)
- **Algorithm**: AES-128 or AES-256 in XTS mode
- **Activation**: Control Panel → BitLocker → Turn On
- **Key storage**: TPM (auto-unlock) or password/PIN at boot
- **Recovery**: 48-digit recovery key (save to Microsoft account or print)
- **Scope**: Full disk encryption — encrypts C: and any data drives

### FileVault (macOS)

- **Available in**: All modern macOS (System Settings → Privacy & Security → FileVault)
- **Algorithm**: XTS-AES-128 with 256-bit key
- **Key storage**: Password (iCloud or local recovery key)
- **T2/Apple Silicon**: Hardware-accelerated; storage is encrypted by default — FileVault just protects the login
- **Scope**: Full startup disk

### LUKS (Linux Unified Key Setup)

- **Available in**: All major Linux distributions
- **Algorithm**: AES (default AES-256 in XTS mode), also supports Serpent, Twofish
- **Activation**: During installation "Encrypt disk" option, or `cryptsetup luksFormat`
- **Supports**: Multiple passwords/key files for same volume (up to 8 "key slots")
- **Scope**: Full partition or specific volumes

```bash
# LUKS basics
cryptsetup luksFormat /dev/sdb1            # Set up encryption on a partition
cryptsetup open /dev/sdb1 my_volume        # Decrypt and create /dev/mapper/my_volume
mount /dev/mapper/my_volume /mnt/data      # Mount decrypted volume
cryptsetup close my_volume                 # Re-encrypt / close when done
```

### EFS (Windows Encrypting File System)

- File-level encryption for individual NTFS files
- Transparent for the logged-in user; others see gibberish
- Key tied to user account certificate — backup certificates!
- Less used than BitLocker; BitLocker is preferred for full-disk protection

---

## 8. Performance Considerations

```
  Hardware WITH AES-NI (all CPUs since ~2010):
  Encryption overhead: < 5%  — essentially unnoticeable
  Sequential read/write: near-identical speed with/without encryption

  Hardware WITHOUT AES-NI (old CPUs, some embedded):
  Encryption overhead: 10–30% slower disk operations

  SSD vs HDD:
  SSDs: Random I/O already fast → encryption overhead negligible
  HDDs: Seek time dominates → encryption adds marginal extra time

  Verdict for modern hardware: Enable encryption — performance impact is negligible,
  but the protection is enormous.
```

---

## 9. Limitations

### Encryption Protects Data at Rest — Not in Use

```
  Disk unlocked + system running → data is decrypted in RAM as accessed

  Threats encryption does NOT stop:
  ✗ Malware on a running system (it reads RAM)
  ✗ Attacker with physical access to a running/sleeping system
  ✗ Cold boot attacks (quickly dump RAM before it clears)
  ✗ Screen reading / shoulder surfing

  Threats encryption DOES stop:
  ✓ Stolen laptop when powered off
  ✓ Discarded drive with sensitive data
  ✓ Physical access when system is off/hibernated
  ✓ Law enforcement seizing a powered-off device without the key
```

### Encryption ≠ Backup

```
  Ransomware scenario:
  1. Malware runs while disk is unlocked
  2. Malware encrypts your files with ITS OWN key
  3. Your disk encryption is bypassed because you're logged in

  Encryption protects FROM thieves; backups protect FROM ransomware.
  You need BOTH.
```

---

## 10. Best Practices

1. **Use full disk encryption** on all laptops and mobile devices — theft risk is high
2. **Back up your recovery key** in at least 2 places (printed, separate USB, password manager, cloud account)
3. **Choose a strong passphrase** — at least 12 characters; passphrases are easier to remember and harder to crack
4. **Enable encryption BEFORE storing sensitive data** — some tools can encrypt in-place but it's riskier
5. **Shut down (not sleep/lock)** for maximum protection — sleep keeps RAM powered with keys potentially accessible
6. **Combine with backups** — encrypted + backed up = protected from both theft and ransomware
7. **Don't forget server drives** — BitLocker/LUKS work on servers too

---

## 10. Code Examples

> Working code that demonstrates file encryption concepts in practice.

### C++ — Simple Version
XOR cipher on file data — same operation encrypts and decrypts (symmetric).

```cpp
#include <iostream>
#include <string>
using namespace std;

// XOR cipher: encrypt(plaintext, key) = ciphertext
//             encrypt(ciphertext, key) = plaintext  (same operation!)
// Toy example only — do NOT use in production

string xorCipher(const string& data, uint8_t key) {
    string result = data;
    for (char& c : result)
        c ^= key;  // XOR every byte with the single-byte key
    return result;
}

// Multi-byte key: key bytes wrap around (repeating-key XOR)
string xorMultiKey(const string& data, const string& key) {
    string result = data;
    for (int i = 0; i < (int)data.size(); i++)
        result[i] = data[i] ^ (uint8_t)key[i % key.size()];
    return result;
}

void demo(const string& plaintext, uint8_t singleKey, const string& multiKey) {
    cout << "=== Single-byte XOR (key=" << (int)singleKey << ") ===\n";
    string cipher    = xorCipher(plaintext, singleKey);
    string decrypted = xorCipher(cipher,    singleKey);  // same op
    cout << "Plaintext:  \"" << plaintext  << "\"\n";
    cout << "Ciphertext: [raw bytes, length=" << cipher.size() << "]\n";
    cout << "Decrypted:  \"" << decrypted  << "\"\n";
    cout << "Match: "        << (plaintext == decrypted ? "YES" : "NO") << "\n\n";

    cout << "=== Multi-byte XOR (key=\"" << multiKey << "\") ===\n";
    string c2 = xorMultiKey(plaintext, multiKey);
    string d2 = xorMultiKey(c2,        multiKey);
    cout << "Decrypted: \"" << d2 << "\"  Match: " << (plaintext == d2 ? "YES" : "NO") << "\n";
}

int main() {
    demo("Hello, File System!", 0x42, "SECRET");
    return 0;
}
```

### C++ — Medium / LeetCode Style
Simplified block cipher: demonstrate key scheduling, SubBytes, and AddRoundKey concepts (AES-inspired, NOT real crypto).

```cpp
#include <iostream>
#include <vector>
#include <array>
#include <string>
using namespace std;

// Block cipher simulation (educational only — NOT real AES)
// Concepts shown: 16-byte blocks, SubBytes, AddRoundKey, multiple rounds

const int BLOCK = 16;

// Toy S-box: swap high/low nibbles of each byte
uint8_t sbox(uint8_t b)     { return ((b & 0x0F) << 4) | ((b & 0xF0) >> 4); }
uint8_t sbox_inv(uint8_t b) { return sbox(b); }  // self-inverse

void xorBlock(vector<uint8_t>& blk, const array<uint8_t,BLOCK>& key) {
    for (int i = 0; i < BLOCK; i++) blk[i] ^= key[i];
}

vector<uint8_t> encryptBlock(vector<uint8_t> blk,
                              const array<uint8_t,BLOCK>& key, int rounds=3) {
    for (int r = 0; r < rounds; r++) {
        for (auto& b : blk) b = sbox(b);  // SubBytes
        xorBlock(blk, key);               // AddRoundKey
    }
    return blk;
}

vector<uint8_t> decryptBlock(vector<uint8_t> blk,
                              const array<uint8_t,BLOCK>& key, int rounds=3) {
    for (int r = 0; r < rounds; r++) {
        xorBlock(blk, key);                    // undo AddRoundKey
        for (auto& b : blk) b = sbox_inv(b);  // undo SubBytes
    }
    return blk;
}

// Encrypt a string in 16-byte zero-padded blocks
vector<vector<uint8_t>> encryptFile(const string& data,
                                    const array<uint8_t,BLOCK>& key) {
    vector<vector<uint8_t>> cipher;
    for (int i = 0; i < (int)data.size(); i += BLOCK) {
        vector<uint8_t> blk(BLOCK, 0);  // zero-pad last block
        for (int j = 0; j < BLOCK && i+j < (int)data.size(); j++)
            blk[j] = (uint8_t)data[i+j];
        cipher.push_back(encryptBlock(blk, key));
    }
    return cipher;
}

string decryptFile(const vector<vector<uint8_t>>& cipher,
                   const array<uint8_t,BLOCK>& key) {
    string result;
    for (auto& blk : cipher)
        for (auto b : decryptBlock(blk, key))
            if (b != 0) result += (char)b;  // strip padding zeros
    return result;
}

int main() {
    array<uint8_t,BLOCK> key = {0x2B,0x7E,0x15,0x16,0x28,0xAE,0xD2,0xA6,
                                 0xAB,0xF7,0x15,0x88,0x09,0xCF,0x4F,0x3C};
    string plaintext = "Top Secret Data!";
    auto cipher    = encryptFile(plaintext, key);
    auto recovered = decryptFile(cipher,    key);
    cout << "Plaintext:  \"" << plaintext  << "\"\n";
    cout << "Blocks:      " << cipher.size() << " x 16 bytes\n";
    cout << "Decrypted:  \"" << recovered  << "\"\n";
    cout << "Match: " << (plaintext == recovered ? "YES" : "NO") << "\n";
    return 0;
}
```

### Python — Simple Version
XOR cipher encrypt/decrypt cycle — demonstrates that encryption and decryption are the same operation.

```python
# Simulate file encryption with XOR cipher (toy example — NOT for production)

def xor_cipher(data: bytes, key: bytes) -> bytes:
    """XOR each byte of data with corresponding key byte (key repeats)."""
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def simulate_file_encryption(plaintext: str, key: str):
    plain_bytes = plaintext.encode()
    key_bytes   = key.encode()
    print(f"Plaintext:  '{plaintext}'")
    print(f"Key:        '{key}'")

    # Encrypt (simulate writing to disk as ciphertext)
    cipher = xor_cipher(plain_bytes, key_bytes)
    print(f"Ciphertext: {list(cipher[:8])}...")

    # Decrypt (simulate reading back and decrypting)
    decrypted = xor_cipher(cipher, key_bytes)  # same XOR operation
    print(f"Decrypted:  '{decrypted.decode()}'")
    print(f"Match: {'YES' if decrypted == plain_bytes else 'NO'}")

# --- Demo 1: basic ---
simulate_file_encryption("Hello, Secret File!", "KEY42")

# --- Demo 2: different keys -> different ciphertext ---
print("\nSame message, different keys:")
msg = b"password123"
for k in [b"A", b"Z", b"MULTIKEY"]:
    enc = xor_cipher(msg, k)
    dec = xor_cipher(enc, k)
    print(f"  key={k!r:12} cipher={enc.hex()[:14]}... match={dec == msg}")
```

### Python — Medium Level
Block cipher simulation with PKCS#7 padding, toy key schedule, SubBytes, and AddRoundKey — AES concepts made visible.

```python
# Block cipher demonstration (AES-INSPIRED concept, NOT real crypto)
# Shows: PKCS#7 padding, key scheduling, SubBytes, AddRoundKey, multi-round encryption

BLOCK_SIZE = 16

def pad(data: bytes) -> bytes:
    """PKCS#7: pad to multiple of BLOCK_SIZE."""
    n = BLOCK_SIZE - (len(data) % BLOCK_SIZE)
    return data + bytes([n] * n)

def unpad(data: bytes) -> bytes:
    return data[:-data[-1]]

def toy_sbox(b: int) -> int:
    """Swap high/low nibbles (toy substitution, not real AES S-box)."""
    return ((b & 0x0F) << 4) | ((b & 0xF0) >> 4)

def xor_bytes(a: bytes, b: bytes) -> bytes:
    return bytes(x ^ y for x, y in zip(a, b))

def generate_round_keys(key: bytes, rounds: int) -> list:
    """
    Toy key schedule: derive each round key by rotating and XOR-ing
    with a round constant. Real AES uses a much more complex schedule.
    """
    keys = [key]
    for r in range(1, rounds):
        prev    = keys[-1]
        rotated = prev[1:] + prev[:1]              # rotate left 1 byte
        rcon    = bytes([r] + [0] * (BLOCK_SIZE - 1))  # round constant
        keys.append(xor_bytes(rotated, rcon))
    return keys

def encrypt_block(block: bytes, round_keys: list) -> bytes:
    state = block
    for rk in round_keys:
        state = bytes(toy_sbox(b) for b in state)  # SubBytes
        state = xor_bytes(state, rk)               # AddRoundKey
    return state

def decrypt_block(block: bytes, round_keys: list) -> bytes:
    state = block
    for rk in reversed(round_keys):
        state = xor_bytes(state, rk)                    # undo AddRoundKey
        state = bytes(toy_sbox(b) for b in state)       # undo SubBytes (self-inverse)
    return state

def encrypt_file(plaintext: str, key: bytes, rounds: int = 3) -> bytes:
    data       = pad(plaintext.encode())
    round_keys = generate_round_keys(key, rounds)
    return b"".join(
        encrypt_block(data[i:i+BLOCK_SIZE], round_keys)
        for i in range(0, len(data), BLOCK_SIZE)
    )

def decrypt_file(ciphertext: bytes, key: bytes, rounds: int = 3) -> str:
    round_keys = generate_round_keys(key, rounds)
    plain = b"".join(
        decrypt_block(ciphertext[i:i+BLOCK_SIZE], round_keys)
        for i in range(0, len(ciphertext), BLOCK_SIZE)
    )
    return unpad(plain).decode()

# --- Demo ---
key = b"MySecretKey12345"   # 16-byte key
msg = "Top Secret File Contents!"
print(f"Original:  '{msg}'")
cipher    = encrypt_file(msg, key)
recovered = decrypt_file(cipher, key)
print(f"Encrypted:  {cipher.hex()}")
print(f"Decrypted: '{recovered}'")
print(f"Match: {'YES' if msg == recovered else 'NO'}")
```

---

## 11. Key Takeaways

- **File system encryption** converts data from plaintext to ciphertext — the encryption key is the only way to decode it
- **File-level encryption** selects specific files; **full disk encryption (FDE)** encrypts everything including OS and metadata — FDE is simpler and more complete
- **AES-256** in **XTS mode** is the standard algorithm for disk encryption — extremely secure and hardware-accelerated on modern CPUs
- **TPM chip** stores keys securely in hardware — auto-unlocks on trusted systems; prevents key extraction by software/malware
- **BitLocker** (Windows Pro), **FileVault** (macOS), **LUKS** (Linux) — all use AES-256, all are transparent to the user
- Performance overhead with AES-NI hardware acceleration: **< 5%** — negligible on modern machines
- **Recovery key is critical** — lose it + forget password = permanent data loss. No backdoors exist
- **Limitation**: protects data at rest only — once the system is running and disk is unlocked, malware can read files normally
- **Encryption ≠ backup**: protects from physical theft, not from ransomware or accidental deletion → need both
- **Always back up recovery keys** in a separate secure location; never on the encrypted drive itself
