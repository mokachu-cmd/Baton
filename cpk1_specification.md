# CPK1 Archive Format & Engine Specification
**Version:** 2.0  
**Status:** Frozen Specification  

---

## 1. Overview & Architectural Goals

The **CPK1 Archive Format** is a self-contained, content-aware, incremental file container and orchestration specification. It is engineered for local POSIX-compliant filesystems (e.g., Linux `ext4`/`XFS`, macOS `APFS`) to optimize compression ratios across heterogeneous data collections while minimizing I/O and CPU overhead during updates.

### Core Architectural Features:
* **Heuristic Algorithmic Routing:** Configurable file-size thresholds direct payloads to lightweight codecs (e.g., Zstandard) or heavy-duty codecs (e.g., LZMA2).
* **Granular Framing:** File-level independent framing enables fast random-access extraction of single entries without decompressing adjacent data blocks.
* **Incremental Copy-Forward:** Unchanged files are copied directly from the existing archive payload stream into the new container without re-compression.
* **Dual Indexing & Recovery:** Fixed 56-byte binary header and trailing 56-byte footer provide redundant pointers and full BLAKE3 verification of the manifest payload.
* **Crash Durability:** Atomic file replacement combined with parent directory metadata synchronization protects against filesystem corruption during sudden power loss.

---

## 2. Binary Layout Specification

All multi-byte integers are serialized in **Little-Endian** byte order. Offsets are **absolute byte offsets** from the start of the file ($0$-based index).

```text
+------------------------------------------------------------------+
| Header (Fixed 56 Bytes)                                          |
| - Magic Bytes (4 bytes): "CPK1"                                  |
| - Format Version (2 bytes): uint16                               |
| - Flags / Reserved (2 bytes): uint16                             |
| - Manifest Length (8 bytes): uint64                              |
| - Manifest Offset (8 bytes): uint64                              |
| - Header Manifest Checksum (32 bytes): BLAKE3 digest of manifest |
+------------------------------------------------------------------+
| Compressed Data Payload Blocks                                   |
| - Extent 0 (Bytes 56 to N_0)                                     |
| - Extent 1 (Bytes N_0 to N_1)                                    |
| - ...                                                            |
+------------------------------------------------------------------+
| Manifest Payload                                                 |
| - MessagePack / JSON serialized index metadata                   |
+------------------------------------------------------------------+
| Footer (Fixed 56 Bytes)                                          |
| - Manifest Offset (8 bytes): uint64 (redundant)                  |
| - Manifest Length (8 bytes): uint64 (redundant)                  |
| - Footer Manifest Checksum (32 bytes): BLAKE3 digest of manifest |
| - Footer Magic / Trailer (8 bytes): "CPK1END!"                   |
+------------------------------------------------------------------+
```

### 2.1 Fixed Header (Bytes 0–55)
| Byte Offset | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x00` | 4 | `char[4]` | `magic` | String ASCII `"CPK1"` (`0x43`, `0x50`, `0x4B`, `0x31`) |
| `0x04` | 2 | `uint16` | `version` | Format version number (e.g., `2`) |
| `0x06` | 2 | `uint16` | `flags` | Bit flags reserved for future extensions (default `0`) |
| `0x08` | 8 | `uint64` | `manifest_len` | Size of the serialized manifest section in bytes |
| `0x10` | 8 | `uint64` | `manifest_offset` | Absolute file offset pointing to manifest start |
| `0x18` | 32 | `byte[32]` | `manifest_hash` | Full 256-bit BLAKE3 cryptographic digest of manifest |

### 2.2 Compressed Data Payload Region (Bytes 56 to `manifest_offset - 1`)
Packed sequence of binary extents. Data start is strictly enforced at byte offset **56**. Each file payload is wrapped as an independent stream or frame:
* **Zstandard Payload Framing:** Standard Zstandard frame containing frame content size headers (`ZSTD_c_checksumFlag = 1`).
* **LZMA2 Payload Framing:** Raw LZMA2 stream wrapped with `liblzma` header boundaries.

### 2.3 Serialized Manifest Section
Binary object stream (MessagePack or UTF-8 JSON) encoding file hierarchy, layout pointers, checksums, and attributes.

### 2.4 Fixed Footer (`file_size - 56` to `file_size - 1`)
| Offset Rel. to End | Size (Bytes) | Type | Field Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `-56` | 8 | `uint64` | `manifest_offset` | Redundant copy of manifest start offset |
| `-48` | 8 | `uint64` | `manifest_len` | Redundant copy of manifest length |
| `-40` | 32 | `byte[32]` | `manifest_hash` | Redundant full 256-bit BLAKE3 digest of manifest |
| `-8` | 8 | `char[8]` | `footer_magic` | ASCII `"CPK1END!"` (`0x43`,`0x50`,`0x4B`,`0x31`,`0x45`,`0x4E`,`0x44`,`0x21`) |

#### Recovery Policy:
Trailing-edge recovery allows reading the manifest if the header is zeroed or corrupt. Recovery requires that `file_size`, the 56-byte footer, and internal absolute offsets remain intact. If data has been prepended or truncated from the tail, absolute offsets will fail validation checks.

---

## 3. Manifest Schema & Attributes

Timestamps are stored as signed 64-bit integers representing **nanoseconds since UNIX epoch** (`1970-01-01T00:00:00Z`). File paths are strictly normalized using relative forward slashes (`/`).

```json
{
  "version": 2,
  "created_at_unix_ns": 1791730000000000000,
  "default_algorithm": "zstd",
  "size_threshold_bytes": 65536,
  "threshold_inclusive": true,
  "files": {
    "src/main.go": {
      "path": "src/main.go",
      "type": "file",
      "hash": "a1b2c3d4e5f67890abcdef12345678901234567890abcdef1234567890123456",
      "compressed_hash": "f8e7d6c5b4a39281701928374655647382910495867493021524130294857610",
      "algorithm": "zstd",
      "codec_framing": "zstd-independent-frame",
      "compression_level": 3,
      "uncompressed_size": 4200,
      "compressed_extent_size": 1500,
      "offset": 56,
      "length": 1500,
      "mod_time_unix_ns": 1791728400000000000,
      "permissions": "0644"
    },
    "assets/data.csv": {
      "path": "assets/data.csv",
      "type": "file",
      "hash": "11223344556677889900aabbccddeeff11223344556677889900aabbccddeeff",
      "compressed_hash": "99887766554433221100ffeeddccbbaa99887766554433221100ffeeddccbbaa",
      "algorithm": "lzma2",
      "codec_framing": "lzma2-stream",
      "compression_level": 6,
      "uncompressed_size": 10485760,
      "compressed_extent_size": 1200000,
      "offset": 1556,
      "length": 1200000,
      "mod_time_unix_ns": 1791700000000000000,
      "permissions": "0644"
    },
    "docs/link": {
      "path": "docs/link",
      "type": "symlink",
      "link_target": "../assets/data.csv",
      "permissions": "0777"
    }
  }
}
```

---

## 4. Execution Rules & Operating Modes

### 4.1 Scanning & Change Detection Modes
* **`--strict` Mode (Default):** Computes a full BLAKE3 cryptographic digest of source files. Provides change detection subject to BLAKE3's $2^{256}$ collision resistance.
* **`--fast` Mode:** Performs a best-effort cache check using file size and nanosecond `mtime`. If both match the prior manifest, the strict source hashing step is bypassed.

### 4.2 The Operational Triad & I/O Boundaries
1. **Unchanged Files (Read Once in Strict Mode):** Source files are hashed during the scan phase. If the hash matches the prior manifest, the file is **not read a second time and not recompressed**; its existing compressed extent is copied forward.
2. **Changed Files (Read Twice in Strict Mode):** Modified or newly added files are read once during the scan phase to compute the manifest hash, and a second time during the compression streaming phase.
3. **Per-File Observation Window:** During the second read of a changed file, the engine computes a streaming hash of the compressed input. If this diverges from the scan-phase hash, the engine halts and raises `ErrFileModifiedDuringRun`. This is a per-file verification of the scan observation and does not constitute a global filesystem snapshot.

---

## 5. Incremental Copy-Forward & Staging Workflow

An update transaction modifies the archive using an atomic copy-forward mechanism:

```text
[Existing CPK Archive] ------------(Copy Verbatim Extents)-----------\
                                                                     v
[Source File Scan] ---> [Compress New/Modified Files] ---> [New Temp Archive (*.tmp)]
                                                                     |
                                                               (file.Sync)
                                                                     |
                                                               (os.Rename)
                                                                     |
                                                               (dir.Sync)
                                                                     v
                                                          [Published CPK Archive]
```

### 5.1 Staging and Backpatching Protocol
1. **Lock Acquisition:** Acquire an exclusive advisory lock (e.g., `flock`) on `archive.cpk.lock`.
2. **Unique Temp File Creation:** Create a securely randomized temporary file in the target archive's containing directory (`os.CreateTemp(dir, ".cpk-staging-*.tmp")`).
3. **Header Placeholder:** Write a zeroed 56-byte header placeholder to offset `0`.
4. **Data Stream Processing:**
   * **Unchanged Entries:** Copy the exact byte extent (`length`) from the old archive to the temp file at its current write pointer. Validate the copied bytes against `compressed_hash` during the copy stream. Record the *new* offset.
   * **New/Modified Entries:** Stream source bytes into the assigned compressor (Zstd if `< threshold`; LZMA2 if `>= threshold`). Compute the compressed extent hash and record new extent boundaries.
5. **Manifest & Footer Writing:** Serialize the updated manifest payload, write it immediately after the last data block, compute its BLAKE3 hash, and append the 56-byte footer.
6. **Header Backpatching:** Seek back to offset `0` and write the real 56-byte header containing the final `manifest_offset`, `manifest_len`, and BLAKE3 `manifest_hash`.
7. **Flushing:** Call `file.Sync()` on the temporary file descriptor.

### 5.2 Atomic Swap & Transaction States
* Execute `os.Rename(temp_file, target_file)`.
* Open the parent directory handle (`os.Open(filepath.Dir(target_file))`) and invoke `dir.Sync()`.

#### State Definitions:
* **Pre-Commit Failure:** If any step prior to `os.Rename` fails, remove `temp_file` and release the lock. The existing archive remains unchanged.
* **`ErrPublishedDurabilityUncertain`:** If `os.Rename` succeeds but `dir.Sync()` fails (e.g., filesystem driver limitations), report that the archive is published and visible on disk, but durability against immediate power loss is unconfirmed. Do **not** roll back or delete the published file.

---

## 6. Defensive Reader Validation Engine

A reader must perform a strict validation pass before executing any extraction or payload access:

1. **Header/Footer Consistency:** Verify magic bytes (`"CPK1"` and `"CPK1END!"`). Read the manifest offset and length from both header and footer. If they mismatch, prefer header pointers unless header checksum validation fails.
2. **Manifest Verification:** Compute the BLAKE3 digest of the serialized manifest section and verify it against `manifest_hash`.
3. **Extent Boundary Bounds:** Ensure that for every file entry:
   $$\text{offset} \ge 56 \quad \text{AND} \quad (\text{offset} + \text{length}) \le \text{manifest\_offset}$$
4. **Overlap & Range Check:** Reject any entry where payload extents overlap in the binary stream or where `offset + length` causes integer overflow.
5. **Payload Integrity:** Verify the compressed extent against `compressed_hash` before decompressing. During extraction, compute the decompressed stream hash and verify it against `hash`.
6. **Path Traversal & Symlink Safeguards:**
   * Reject absolute paths (e.g., `/etc/passwd`) or relative traversal paths containing `..` segments.
   * Clean destination paths to ensure they reside strictly within the root extraction directory.
   * Before writing a file, inspect intermediate path components. If any component is a pre-existing symlink, extraction must abort to prevent traversal writes out of tree.

---

## 7. Threat Model & Platform Constraints

### 7.1 Platform Scope
CPK1 targets local POSIX-compliant filesystems (`ext4`, `XFS`, `APFS`). Network-attached storage (NFS, SMB/CIFS) is unsupported for strict guarantees: advisory locking may fail silently, and directory metadata flushing may return unsupported errors.

### 7.2 Security Boundaries
* **In-Scope Protections:** Detects accidental bit-rot corruption, protects against unintentional file overwrites, prevents invalid uncoordinated local updates via cooperative locking, and isolates extraction within target directories under non-adversarial conditions.
* **Out-of-Scope Limitations:** Does not prevent malicious local tampering (cryptographic hashes without digital signatures do not provide authenticity), does not protect against concurrent extraction tree modifications, and does not provide system-wide snapshot isolation against local processes modifying source files during execution.