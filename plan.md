# YOIFS Implementation Plan

## What We Have

### Infrastructure (already implemented in `index.ts`)
- `Disk` interface — read/write at byte offsets, size query
- `MemoryDisk` — in-memory 1MB disk with raw storage access
- `CorruptionSimulator` — random and sequential byte corruption
- `TestUtils` — random file/data generators
- `Logger` — colored console output
- `TestHarness` — three-level automated test suite

### Solution Skeleton (`solution.ts`)
- `FileSystem` class with the correct constructor signature (`disk: Disk`)
- Three method stubs returning `{ success: false, error: 'Not implemented' }`
  - `writeFile(filename, content): Promise<FileSystemResult<void>>`
  - `readFile(filename): Promise<FileSystemResult<Buffer>>`
  - `listFiles(): Promise<FileSystemResult<string[]>>`
- Optional `checkSystemHealth()` stub
- `calculateChecksum()` helper (uses `crypto.createHash('crc32')`)

---

## What We Need to Implement

All work goes in `solution.ts`. The three levels build on each other.

---

## Implementation Plan

### Level 1 — Basic File System Operations

**Goal:** pass all Level 1 tests (write, read, list, file-not-found).

**Design: in-memory FAT (File Allocation Table) stored at a fixed header region**

```
Disk layout (1 MB):
[0 – HEADER_SIZE)    FAT region  — serialised JSON (or binary) file table
[HEADER_SIZE – end)  Data region — raw file content, appended sequentially
```

**Steps:**

1. **FAT structure** — keep a `Map<string, { offset: number, size: number }>` in memory as the source of truth.
2. **`writeFile`**
   - Reject if file already exists (or overwrite — choose one).
   - Append content to the data region (track `nextFreeOffset`).
   - Update the in-memory FAT entry.
   - Serialise FAT to the header region on disk after every write.
3. **`readFile`**
   - Look up FAT for offset + size.
   - Return `{ success: false, error: 'File not found' }` if missing.
   - Read bytes from disk at the recorded offset.
4. **`listFiles`**
   - Return `Array.from(fat.keys())`.
5. **On construction** — attempt to deserialise FAT from the header region (handles re-use of an existing disk).

---

### Level 2 — Corruption Detection

**Goal:** zero undetected corruptions; acceptable false positive rate.

**Design: CRC-32 checksum per file stored in the FAT**

1. **`writeFile`** — compute `crc32(content)` and store the hex digest alongside offset + size in the FAT.
2. **`readFile`** — after reading bytes from disk, recompute the checksum and compare. If mismatch → return `{ success: false, error: 'Corruption detected: checksum mismatch' }`.
3. **FAT integrity** — also checksum the serialised FAT itself (store a separate header checksum). If FAT is corrupted, all reads fail with a meaningful error.

> Note: `calculateChecksum()` is already provided. `crypto.createHash('crc32')` is available via Node's built-in hash algorithms.

---

### Level 3 — Fault Tolerance (Redundancy)

**Goal:** maintain >90% successful reads up to ~5–10% corruption rate; never silently return corrupt data.

**Design: triple replication (3 copies of each file)**

Each file is stored in three separate disk locations. Reads attempt all three; the first copy that passes checksum verification wins.

1. **FAT entry** changes to `{ offsets: [n1, n2, n3], size, checksum }`.
2. **`writeFile`** — write the same content to three consecutive regions in the data area.
3. **`readFile`** — try each copy in order; return data from the first passing checksum. If all three fail → return corruption error.
4. **Space planning** — 1 MB disk, test writes 100 files × avg ~250 bytes × 3 = ~75 KB used. Well within limits.

**Optional advanced improvements (after replication works):**
- **Reed-Solomon or XOR parity** — allows reconstruction of one corrupt copy from the other two, reducing space overhead vs. triple replication.
- **FAT redundancy** — store three copies of the FAT header as well.
- **`checkSystemHealth()`** — scan all files and return `{ healthy, corrupted }` counts.

---

## File to Edit

| File | Action |
|------|--------|
| `solution.ts` | Full implementation — all three levels |

No other files need modification.

---

## Quick Verification

```bash
pnpm dev
```

Expected output after a complete implementation:
- Level 1: all three sub-tests pass (write/read, listing ≥4 files, file-not-found)
- Level 2: detected corruptions > 0, undetected corruptions = 0
- Level 3: data integrity failures = 0 across all corruption rates; successful reads stay high at low rates and degrade gracefully at high rates

---

## Implementation Order

1. Level 1 — get basic read/write/list working
2. Level 2 — add checksum to FAT entries, verify on read
3. Level 3 — switch to triple-copy storage, add FAT redundancy
