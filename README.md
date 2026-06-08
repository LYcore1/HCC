# HCC - High Capacity Colorful

**A semantic deduplication and adaptive compression layer for Linux storage.**

---

## Problem

Current filesystems treat data as flat, untyped byte streams. Two files that
belong to the same logical context (e.g., a payment module) are stored
independently. Identical blocks across related files are stored multiple times.
Compression is applied uniformly regardless of data type.

This leaves capacity on the table.

---

## Approach

HCC groups files into **colored branches** based on semantic similarity rather
than file extension or directory location. A branch is a set of files that
tend to appear together, change together, or share structural patterns.

Once grouped, HCC applies:

- **Intra-branch deduplication** — identical blocks within a branch are stored once
- **Adaptive compression** — each branch gets the best algorithm for its data type
- **Colored prefetch** — accessing one file in a branch preloads related files

---

### Example
Payment Branch (🔴):
├── payment.py
├── checkout.js
├── invoice.html
└── logo.png

Auth Branch (🟡):
├── auth.py
├── login.js
├── users.sql
└── avatar.png


A `.png` and a `.py` share a color not because of their extension,
but because they serve the same context.

---

### Inspiration

The tagging behavior is loosely inspired by the Golgi apparatus — a cellular
organelle that labels proteins for routing without modifying their structure.
HCC tags files without moving them. The color table is a lightweight index
kept in memory.

---

## Expected Gains

Gains are workload-dependent. On projects with high internal redundancy
(multiple similar codebases, container images, build artifacts), effective
capacity can increase by 2-5x through deduplication alone. Compression adds
a further 1.5-3x depending on data type.

No magic. Just structural redundancy exploitation with better grouping.

---

## Technical Challenges (Open Questions)

1. **Semantic grouping is hard.** How do we determine that `payment.py` and
   `checkout.js` belong together without reading and understanding both files?
   Initial approach: structural fingerprinting (imports, includes, directory
   co-occurrence) rather than full content analysis.

2. **Grouping overhead must be sub-linear.** If grouping costs more than the
   savings it produces, it's useless. Target: O(n) classification with
   constant factors small enough to run at write time.

3. **False positives in dedup are catastrophic.** A hash collision in the
   dedup table means data loss. This is a solved problem in ZFS and Btrfs.
   HCC must match their safety guarantees.

4. **Cold start.** A new file has no history. The system needs a reasonable
   default before learning patterns.

---

## Design Decisions

| Choice | Rationale |
|---|---|
| **Zig** | Comptime evaluation, no hidden allocations, direct pointer control, built-in C interop |
| **FUSE first, kernel module later** | Faster iteration, safer crashes during development |
| **Color table in memory, not on disk** | Lookup latency matters. Table is small (~6-10 MB for typical systems) |
| **Per-branch compression strategy** | Text branches get zstd, image branches get no re-compression, binary branches get lz4 |

---

## Status

**Pre-alpha. No code yet.** Learning Zig and systems programming.
Work begins in earnest after university entrance exams.

---

## Roadmap

1. **HCC-Storage** — colored FUSE filesystem with dedup and adaptive compression
2. **HCC-Memory** — same logic applied to RAM pages (zram competitor)
3. **HCC-CPU** — color-aware task scheduling
4. **HCC-Full** — integrated optimization layer

---

## Name

HCC = High Capacity Colorful. Yes, the acronym collides with a medical term.
If the project succeeds, search engines will adapt.

---

*Questions, criticism, and contributions welcome. This is a research project
in its earliest stage.*









