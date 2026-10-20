Here is the complete, high-level architectural summary of the **CPK1 Archive Format**, incorporating every production-grade refinement, validation rule, and durability guarantee we established:

---

## CPK1 Archive Format: Architectural Summary

CPK1 is a self-contained, content-aware, incremental archiving engine designed to optimize storage ratios for mixed-media collections while ensuring high-performance updates and absolute data safety.

### 1. Intelligent Algorithmic Routing

* **Size-Threshold Split:** Automatically inspects file sizes against a user-defined threshold (`size_threshold_bytes`). Files below the threshold are compressed using lightweight, low-overhead codecs (such as **Zstandard** for fast access); files meeting or exceeding the threshold are routed to heavy-duty algorithms (such as **LZMA2/7z**) for maximum ratio savings.
* **Granular Framing:** Every file payload is stored as an independent, self-contained compression frame, allowing rapid random access and extraction without needing to decompress the entire archive.

### 2. Binary Container Architecture

The archive is bundled into a single binary file structured with redundant verification boundaries:

* **56-Byte Header:** Contains magic bytes (`"CPK1"`), format version, flags, absolute manifest offset, manifest length, and a full **BLAKE3** cryptographic manifest checksum.
* **Variable Data Blocks:** Packed compressed payload extents placed sequentially starting at byte offset 56.
* **Serialized Manifest Section:** Contains the structured index (JSON or MessagePack) mapping file paths, hashes, sizes, permissions, and exact byte extents.
* **56-Byte Footer:** Mirrors the manifest offset and length, stores an independent BLAKE3 manifest checksum, and ends with trailing magic bytes (`"CPK1END!"`). Enables **trailing-edge header recovery** if the file header is clipped or zeroed out, provided the absolute byte layout remains intact.

### 3. Atomic Copy-Forward Incremental Updates

* **Zero-Rewrite for Unchanged Data:** When updating an existing archive, the engine bypasses re-compression for static files by streaming their existing compressed payloads directly from the old container into a new staged file, updating their absolute pointers in the new manifest.
* **Header Backpatching:** Because header fields (like the manifest offset and hash) cannot be known until the entire file payload and manifest are laid out, the writer writes a placeholder header, streams data, writes the manifest and footer, and then **seeks back to byte 0** to backpatch the correct header values before syncing.

### 4. Change Detection & Integrity Modes

* **`--fast` Mode:** A best-effort cache mode that relies on file size and high-resolution nanosecond `mtime` timestamps to skip hashing for unchanged files.
* **`--strict` Mode:** Computes a full **BLAKE3 hash** of every source file on every run, providing high-confidence change detection subject to BLAKE3's cryptographic collision resistance.
* **Cryptographic Integrity:** Every archive is protected against accidental corruption by full BLAKE3 digests embedded in both the header and footer.

### 5. Crash Durability & Atomic Replacement Protocol

* **Staged Durability Sequence:** Updates are written entirely to a temporary file (`archive.cpk.tmp`). Once written, the temporary file descriptor is explicitly flushed via `file.Sync()`.
* **Atomic Swap & Directory Sync:** The temp file is swapped into place using an atomic filesystem rename (`os.Rename`). To protect against power loss on POSIX filesystems, the engine immediately opens the containing **parent directory handle** and executes a `dir.Sync()` to flush metadata inodes to non-volatile storage.

### 6. Defensive Reader Validation Engine

Before extracting or serving any payload, the reader executes a mandatory sanitization pass:

* **Extent & Boundary Enforcement:** Rejects out-of-bounds extents, overlapping file data blocks, and malformed manifest sizes.
* **Integer Safety:** Protects against integer overflows during extent calculations and buffer allocations.
* **Path Sanitization:** Blocks path traversal attempts (e.g., `../../`) and absolute paths during extraction to prevent arbitrary file write vulnerabilities.
