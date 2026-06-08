# HCC - High Capacity Colorful

**A semantic deduplication and adaptive compression layer for Linux storage.**

---

## Motivation

Modern filesystems (ext4, Btrfs, ZFS, APFS) are well-engineered. They organize
data by inodes, directories, extents, and journals. They do their job reliably.

But they have a semantic gap.

They see bytes, not meaning. A filesystem does not know that `payment.py`,
`checkout.js`, and `invoice.html` belong to the same feature. It does not know
that `avatar.png` is part of the authentication module while `logo.png` is
part of the payment module. To the filesystem, these are independent streams
of bytes in unrelated directories.

This gap has consequences:

- **Missed deduplication opportunities.** Two files that share identical
  blocks are deduplicated only if the filesystem scans for identical blocks
  globally (expensive) or not at all.

- **Uniform compression.** A single compression algorithm and level is applied
  to all data, regardless of type. Text compresses well with zstd. JPEGs
  should not be re-compressed. Filesystems don't differentiate.

- **No contextual prefetch.** Opening `payment.py` does not hint the system
  to preload `checkout.js`. They are unrelated at the block layer.

HCC is an attempt to bridge this gap — not by replacing the filesystem,
but by adding a semantic layer on top.

---

## How It Works

HCC operates as a **FUSE filesystem** (userspace, for safety during
development) that sits between applications and the underlying real
filesystem (ext4, Btrfs, etc.).

### Two-Layer Grouping

**Layer 1: Structural Type**

Files are grouped by structural fingerprint, not just extension.

| Type Group | Criteria | Examples |
|---|---|---|
| Python | `import`/`def` patterns, `.py` | `payment.py`, `auth.py` |
| JavaScript | `require`/`export` patterns, `.js` | `checkout.js`, `login.js` |
| SQL | `SELECT`/`CREATE` patterns, `.sql` | `users.sql`, `orders.sql` |
| Images | Magic bytes, EXIF headers | `logo.png`, `avatar.jpg` |
| Config | Key-value structure, YAML/TOML/JSON | `config.yaml` |
| Binary | ELF headers, compiled output | `app`, `lib.so` |

This is fast. Structural fingerprinting reads the first 64-256 bytes of a
file and classifies it in microseconds.

**Layer 2: Semantic Color**

Within each type group, files are assigned a **color** based on:

- Import/dependency graph (what does this file include?)
- Directory co-occurrence (what files live together?)
- Git history (what files change together in commits?)
- Naming conventions (does the file name suggest a module?)

A `.py` file and a `.js` file can share the same color (e.g., `🔴 Payment`),
but because they are in different type groups, there is no collision.

```
Root
├── 📁 Python
│   ├── 🔴 Payment (payment.py, payment_utils.py)
│   ├── 🟡 Auth (auth.py, auth_middleware.py)
│   └── 🔵 UI (ui_renderer.py, ui_components.py)
│
├── 📁 JavaScript
│   ├── 🔴 Payment (checkout.js, billing.js)
│   ├── 🟡 Auth (login.js, session.js)
│   └── 🔵 UI (modal.js, dropdown.js)
│
├── 📁 Images
│   ├── 🔴 Payment (logo.png, credit_card_icons/)
│   └── 🟡 Auth (avatar.png, profile_photos/)
│
└── 📁 SQL
    ├── 🔴 Payment (transactions.sql, invoices.sql)
    └── 🟡 Auth (users.sql, sessions.sql)
```

### What HCC Does Per Branch

**1. Intra-branch Deduplication**

Within a branch (e.g., `Python/🔴 Payment`), files often share identical
blocks: imports, docstrings, base classes, utility functions. HCC identifies
these blocks and stores them once. Other files in the branch hold references.

Unlike ZFS dedup, which scans the entire pool, HCC dedup is scoped to a branch.
This reduces the hash table size, lowers CPU overhead, and eliminates
cross-context false positives.

**2. Adaptive Compression**

| Data Type | Strategy | Rationale |
|---|---|---|
| Text (code, config, logs) | zstd (level 3-15 based on access frequency) | High ratio, acceptable speed |
| Binaries (executables, libs) | lz4 | Speed over ratio |
| Images (PNG, JPEG, WebP) | No re-compression | Already compressed |
| Database files | zstd (low level) | Moderate ratio, low latency |

Cold branches (not accessed in 7+ days) get re-compressed at higher levels.

**3. Color-Aware Prefetch**

When a file in `Python/🔴 Payment` is opened, HCC preloads other files in the
same branch into the page cache. This reduces cold-start latency for workflows
that access related files together (e.g., running tests, building a module).

---

## Comparison with Existing Systems

| Feature | ext4 | ZFS | Btrfs | HCC |
|---|---|---|---|---|
| Base organization | Inodes, directories | Inodes, datasets | Inodes, subvolumes | Type + color branches |
| Deduplication | None | Pool-wide (expensive) | Filesystem-wide | Branch-scoped |
| Compression | None (per-fs option) | Per-dataset | Per-filesystem | Per-branch, adaptive |
| Semantic awareness | None | None | None | Type + color |
| Prefetch | Basic (sequential) | Basic | Basic | Color-aware |
| Runs in | Kernel | Kernel | Kernel | Userspace (FUSE) |

HCC does not replace these filesystems. It mounts on top of them.

---

## When HCC Helps

- **Monorepos** with multiple languages and shared dependencies
- **Container image builds** with repeated base layers
- **Data science projects** with many similar notebooks and datasets
- **Game development** with large asset trees and engine files
- **Backup directories** with versioned snapshots of the same project

---

## When HCC Does NOT Help

- **Single-purpose servers** (database, web server with static content)
- **Media libraries** (photos, videos — already compressed, low redundancy)
- **Encrypted files** (no structural fingerprinting possible)
- **Write-heavy workloads with no redundancy** (overhead without benefit)
- **Systems where CPU is the bottleneck, not storage**

HCC is not a universal improvement. It is a tool for specific workloads.

---

## Technical Challenges

1. **Color assignment accuracy.** Import graphs and directory co-occurrence
   are signals, not ground truth. A file may import from a module without
   truly belonging to it. Initial approach: conservative coloring — if
   uncertain, leave uncolored.

2. **Re-classification cost.** When a file changes significantly, should its
   color change? Re-classifying an entire branch on every write is expensive.
   Approach: re-evaluate on read, not write. Lazy coloring.

3. **Dedup safety.** A hash collision in the dedup table means silent data
   corruption. HCC uses BLAKE3 (cryptographic) for block hashing, matching
   the safety standard of ZFS.

4. **FUSE overhead.** Userspace filesystems add latency (~10-50µs per
   operation). For most workloads this is negligible compared to SSD latency
   (~100µs). For latency-critical applications, a kernel module (Phase 2)
   is the goal.

5. **Cold start.** A fresh system has no color data. Default: group by
   directory structure and extension. The learner improves colors over time.

---

## Design Decisions

| Choice | Rationale |
|---|---|
| **Zig** | Comptime for compile-time configuration; no hidden allocations; direct pointer control; built-in C interop for kernel interfaces |
| **FUSE (Phase 1)** | Safe iteration; crashes don't panic the kernel; easier debugging |
| **BLAKE3** | Fast, cryptographic, no known collisions |
| **Two-layer grouping** | Type-first eliminates cross-type dedup errors; color is scoped |
| **Lazy coloring** | Re-evaluate on read, not write; avoids write amplification |
| **Color table in memory** | ~6-10 MB for typical systems; negligible footprint |

---

## Project Status

**Pre-alpha. No code yet.**

Currently learning Zig and systems programming. Active development begins
after university entrance exams (Summer 2026).

This repository is a public design document. Feedback, criticism, and
technical discussion are welcome.

---

## Roadmap

| Phase | Scope | Timeline |
|---|---|---|
| **HCC-Storage** | FUSE filesystem with type grouping, color tagging, branch-scoped dedup, and adaptive compression | TBD |
| **HCC-Memory** | Same semantic logic applied to RAM page management (zram alternative) | TBD |
| **HCC-CPU** | Color-aware process scheduling on heterogeneous cores | TBD |
| **HCC-Full** | Integrated optimization layer across storage, memory, and compute | Long term |

---

## Name

HCC = High Capacity Colorful.

Shares an acronym with Hepatocellular Carcinoma. If the project succeeds,
search results will adapt. If it fails, the name was at least memorable.

---

## Further Reading

- [ZFS Deduplication](https://openzfs.github.io/openzfs-docs/man/7/zfsprops.7.html#dedup)
- [Btrfs Deduplication](https://btrfs.readthedocs.io/en/latest/Deduplication.html)
- [FUSE (Filesystem in Userspace)](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)
- [BLAKE3 Hash Function](https://github.com/BLAKE3-team/BLAKE3)
- [Zig Language](https://ziglang.org/)
- [zram (Linux)](https://www.kernel.org/doc/html/latest/admin-guide/blockdev/zram.html)

---






---

## A Note on Color

The term "color" is, for now, a working metaphor. I don't yet know how color
assignment will be implemented in practice — whether through import graph
analysis, directory co-occurrence, git history, or something else entirely.

This is an open research question. Everything in its time.*This is a research project in its earliest stage. All claims are theoretical
until benchmarked. All design decisions are open to revision.*
