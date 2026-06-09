# FEEDBACK.md

**Technical critiques received in June 2026. Raw, unfiltered, and demanding serious thought.**

These are archived here for transparency and future reference. No names,
no credentials — just the ideas and the problems they point to.

---

## 1. FUSE Overhead and Latency

**Critique:** Every I/O operation through FUSE forces a context switch between
kernel and userspace. On modern NVMe drives, this adds ~50µs per operation.
A command like `git status` (which `stat()`s thousands of files) would become
painfully slow — potentially 30% or more slower than native ext4.

**Why it matters:** HCC's value proposition is "faster, smarter storage."
If the layer itself slows down basic operations, adoption is dead.

**Possible alternatives:** Replace FUSE with a lightweight daemon that observes
file access via eBPF and inotify, and performs prefetch by silently reading
files into the page cache (no interception of application I/O).

---

## 2. Online Compression Timeout Risk

**Critique:** The original design proposed compressing/decompressing files
on-the-fly inside FUSE handlers. Decompressing with zstd level 15 on a 4KB
block can take several milliseconds. Linux FUSE has implicit timeouts;
applications (like VS Code) would hang waiting for the decompressed data.

**Why it matters:** This makes online compression inside FUSE impractical.
Users would blame HCC for random freezes, even if the underlying disk is fast.

**Possible alternatives:** Move compression offline. Provide `hcc freeze`
and `hcc thaw` commands that compress/decompress cold files manually or
on a schedule, without touching the hot I/O path.

---

## 3. Daemon + eBPF Architecture (Alternative to FUSE)

**Critique:** A userspace daemon watching VFS events via eBPF hooks (on
`vfs_read`, `vfs_open`) can learn which files are accessed together. It can
then silently `read()` related files and discard the data, tricking the kernel
into caching them in the page cache. This achieves prefetch without FUSE and
without modifying the filesystem.

**Advantages:**
- Zero overhead on application I/O (no context switch per operation).
- Crash-safe: if the daemon dies, the system falls back to normal ext4/Btrfs.
- Works with any filesystem.

**Disadvantages:**
- Requires an eBPF program (C) and a Zig daemon.
- The daemon consumes a small amount of RAM (~5 MB) constantly.
- Prefetch is limited to what fits in available page cache.

---

## 4. Color Assignment Algorithm (Missing Detail)

**Critique:** Saying "we'll use git history and directory structure" is not an
algorithm. A concrete multi-layer weighted graph approach was proposed:

- **Layer 1: Structural signals** (cost: metadata only). Files in same directory
  get weight 0.5. Files with common name prefixes get 0.3.
- **Layer 2: Dependency signals** (cost: cheap AST parsing via Tree-sitter).
  Import relationships get weight 0.8.
- **Layer 3: Behavioral signals** (cost: git log analysis). Files co-committed
  in >70% of commits get weight 1.0.

Then apply a community detection algorithm (e.g., Louvain) to cluster files
into colors. Files below a confidence threshold stay uncolored.

**Why it matters:** This is the core of HCC. Without a solid algorithm,
the whole project collapses.

---

## 5. Language Choice: Zig vs Rust for FUSE

**Critique:** If HCC had kept the FUSE approach, Rust would have been a safer
choice because its borrow checker prevents memory corruption bugs in the
I/O path. However, with the Daemon architecture, Zig is an excellent fit
because it provides low-level control, no hidden allocations, and comptime
evaluation — exactly what a lean system daemon needs.

**Verdict:** For the current Daemon design, Zig remains a strong choice.

---

## Status of These Critiques

**As of June 2026, the HCC architecture still uses FUSE for the initial
prototype (M1).** This document exists to record serious concerns that
will be re-evaluated once a working prototype produces real benchmarks.

If FUSE overhead proves unacceptable, the project will pivot to the
Daemon + eBPF architecture described above. Compression will definitely
remain offline for safety.

*All critiques are welcome. If you have more, open an issue or reach out.*
