# NOTES.md

**A messy, honest notebook. Ideas that aren't ready for the README. Questions I 
don't have answers to yet. Assumptions that might be wrong. This is the lab, 
not the showroom.**

---

## Table of Contents

1. [Type-First or Color-First?](#type-first-or-color-first)
2. [How Many Colors Does a Real Project Need?](#how-many-colors-does-a-real-project-need)
3. [Color Rotation and Security](#color-rotation-and-security)
4. [Manual Config vs Auto-Learning](#manual-config-vs-auto-learning)
5. [FUSE Overhead — How Bad Is It?](#fuse-overhead--how-bad-is-it)
6. [What If Color Assignment Is Wrong?](#what-if-color-assignment-is-wrong)
7. [Reboot Persistence — xattrs or Daemon?](#reboot-persistence--xattrs-or-daemon)
8. [Cross-Group Association Table](#cross-group-association-table)
9. [Compression Policy Details](#compression-policy-details)
10. [Dedup Safety — BLAKE3 and Collision Risk](#dedup-safety--blake3-and-collision-risk)
11. [Monorepo Analysis — Which Repos to Study?](#monorepo-analysis--which-repos-to-study)
12. [Kernel Module — When, Not If?](#kernel-module--when-not-if)
13. [Assumptions to Validate](#assumptions-to-validate)
14. [Rejected Ideas (and Why)](#rejected-ideas-and-why)
15. [Future — Memory and CPU Layers](#future--memory-and-cpu-layers)

---

## Type-First or Color-First?

**Status: Open. Waiting for data.**

There are two ways to structure the hierarchy:

| | Type-First | Color-First |
|---|---|---|
| Grouping | Python/ → 🔴 Payment | 🔴 Payment/ → Python, JS, SQL |
| Dedup | Safe (Python vs Python) | Risky (Python vs JS shares little) |
| Prefetch | Needs cross-group table | Natural (all Payment files together) |
| Complexity | Medium | High |

**What I need to decide:** Run the same benchmark on 3 real repos with both 
approaches. Whichever gives better prefetch hit rate without killing dedup 
efficiency wins.

**Candidate repos:** Linux kernel, Django, React.

---

## How Many Colors Does a Real Project Need?

**Status: The most important unanswered question.**

If the answer is 10, the whole design is simple. If it's 1000, I need a 
completely different architecture.

| Color Count | Implication |
|---|---|
| 10-20 | Simple array. Linear scan fine. |
| 50-100 | Hash table needed. |
| 500+ | Need hierarchical colors (sub-branches). |
| 1000+ | Rethink everything. |

**Plan:** Clone Linux, Chromium, Django, React. Manually count semantic modules. 
Don't write code. Just count. Get real numbers.

---

## Color Rotation and Security

**Status: Philosophical question. Not urgent.**

Should colors change over time?

**The problem:** Static colors could leak project structure. If someone reads 
your color table, they know `payment.py` and `checkout.js` are related. 
That's information.

**But:** Rotating colors means the learner resets. Prefetch gets worse for a 
while. Is the security benefit worth the performance hit?

**Three options on the table:**

- **Static (my default):** Simple. Fast. Colors survive reboots.
- **Rotating (paranoid):** Colors change every N hours or on reboot. Old 
  mappings logged but not kept in memory.
- **Dual (overengineered):** Two colors per file. One static for prefetch, 
  one rotating for external visibility.

**Questions I need to answer:**
- Can the color table actually be exploited?
- How long does a full recolor take on a 100 GB project?
- Does any existing filesystem do metadata rotation? (Probably not.)

**Current verdict:** Static for now. Revisit if someone reports a CVE.

---

## Manual Config vs Auto-Learning

**Status: Leaning manual. Might be wrong.**

I said "start with a `.hcc-colors.yaml` file." But will anyone use it?

**Manual config pros:**
- Predictable. No surprises.
- User knows their project better than any heuristic.
- Zero learning curve for the system.

**Manual config cons:**
- Nobody wants to write config files.
- New files get no color until the config is updated.
- Doesn't scale for large teams.

**Auto-learning pros:**
- Zero setup.
- Adapts as the project changes.
- Feels like magic.

**Auto-learning cons:**
- Wrong guesses erode trust.
- Hard to debug why a file got a color.
- Needs training data (git history, import graphs).

**Possible middle ground:** Auto-assign with a confidence score. Low confidence 
→ leave uncolored. User can override in the YAML. Best of both?

---

## FUSE Overhead — How Bad Is It?

**Status: Need benchmarks.**

FUSE adds latency. Every `stat()`, `read()`, `write()` goes through userspace.

| Operation | ext4 direct | Via FUSE | Overhead |
|---|---|---|---|
| `stat()` | ~10µs | ? | ? |
| `read()` 4KB | ~20µs | ? | ? |
| `ls -R` (1000 files) | ? | ? | ? |

**What I fear:** `git status` and `ls` become painfully slow.

**Test plan:**
1. Mount a real project via a minimal FUSE pass-through.
2. Run `git status`, `find`, `ls -R`, `grep -r`.
3. Measure wall clock time vs native ext4.
4. If overhead >30%, FUSE is a dead end. Start planning kernel module.

---

## What If Color Assignment Is Wrong?

**Status: Lazy re-evaluation. But details matter.**

**Current plan:** Don't fix it on write. Re-evaluate on read.

**But:**
- What if a file moves? Does the color follow?
- What if a file gets rewritten entirely? Same color or recolor?
- What if the user disagrees with the color? Override UI? CLI?

**Ideas:**
- `hcc recolor --path ./src` — force re-scan a directory
- `hcc color --file payment.py` — show current color and confidence
- `hcc recolor --file payment.py --color 🔴` — manual override

---

## Reboot Persistence — xattrs or Daemon?

**Status: xattrs. But daemon has benefits.**

**Option A: xattrs**
- Store color + metadata as extended attributes on the underlying filesystem.
- Survives reboot. No daemon needed.
- But: not all filesystems support xattrs well. Network mounts? Maybe not.

**Option B: Daemon + database**
- A small daemon maintains an in-memory + on-disk database of colors.
- More flexible. Can store complex relationships.
- But: another process to maintain. Another thing that can crash.

**Current choice:** xattrs for M1. Revisit if xattrs become a bottleneck.

---

## Cross-Group Association Table

**Status: Needed if Type-First wins.**

If I go Type-First, I need a table that says "Python/🔵 UI is related to 
JavaScript/🩶 UI."

**Questions:**
- How many cross-group associations exist in a real project? 10? 100?
- Can this be inferred automatically? (Files that change in the same commits?)
- Does the table need to be bidirectional?

**Size estimate:** If there are 20 colors and 5 type groups, the table is 
at most 20×20 = 400 entries. Tiny. Not a problem.

---

## Compression Policy Details

**Status: Mostly decided. Edge cases remain.**

| Data | Algorithm | Level | Rationale |
|---|---|---|---|
| Python/JS/SQL | zstd | 3 | Fast enough for hot data |
| Cold text (7+ days) | zstd | 15 | Better ratio, rarely accessed |
| Binaries (ELF, .so) | lz4 | default | Speed > ratio |
| Images | none | - | Already compressed |
| Config files | zstd | 1 | Small files, low CPU |

**Open questions:**
- What's the threshold for "cold"? 7 days? 30 days? Configurable?
- Should I recompress on access? (Cold file read → decompress, keep warm copy?)
- How to detect already-compressed files? Magic bytes?

---

## Dedup Safety — BLAKE3 and Collision Risk

**Status: Solved in theory. Must verify in practice.**

BLAKE3 is cryptographic. Collision probability is effectively zero.

But:
- I'm not deduplicating full files. I'm deduplicating blocks.
- Block size? 4KB? 64KB? Variable?
- Smaller blocks = more dedup opportunities, more hash table entries.
- Need to benchmark: 4KB vs 16KB vs 64KB vs variable (content-defined chunking).

**Security:** ZFS and Btrfs already do this safely. Copy their approach. 
No need to invent new cryptography.

---

## Monorepo Analysis — Which Repos to Study?

**Status: Planning phase. No data yet.**

Candidate repos and what I want to learn from each:

| Repo | Why | What to Measure |
|---|---|---|
| Linux kernel | Massive C codebase, clear subsystems | Color count, redundancy within subsystems |
| Chromium | Multi-language monorepo | Cross-type associations, prefetch potential |
| Django | Python web framework | Typical web project structure |
| React | JavaScript library | JS ecosystem patterns |

**Metrics to collect:**
- Number of logical modules (colors)
- Files per module
- Redundant blocks within modules
- Cross-module file associations (co-commits, co-imports)

---

## Kernel Module — When, Not If?

**Status: Distant future. Don't think about it yet.**

If FUSE overhead is too high, or if HCC proves useful enough to deserve 
kernel-level performance, a kernel module is the natural evolution.

**But:**
- Kernel modules are orders of magnitude harder to write and debug.
- A bug in a kernel module panics the machine. A bug in FUSE crashes a process.
- Not until Storage v1.0 is stable and benchmarked.

---

## Assumptions to Validate

These are things I believe but haven't proven:

1. **Files within a semantic branch share significant redundant blocks.**
   If they don't, dedup is useless.

2. **Users will tolerate manual color config.**
   If they won't, auto-learning is mandatory.

3. **FUSE overhead is acceptable for development workloads.**
   If it's not, the whole FUSE-first approach is wrong.

4. **Most projects have 10-100 logical modules.**
   If it's 1000+, the color table architecture needs a rethink.

5. **Prefetch hit rate is high enough to justify the complexity.**
   If it's <50%, color-aware prefetch isn't worth it.

---

## Rejected Ideas (and Why)

### Global Color (No Type Layer)
**Rejected:** Cross-type dedup is mostly useless. Python and JS share 
little block-level structure. Risk of false positives too high.

### Full AI/ML Color Assignment
**Rejected:** Too complex for v1. Requires training data, model, GPU. 
Maybe in 5 years. Manual + heuristics first.

### Storage + Memory + CPU All at Once
**Rejected:** I'm one person. Scope must match capacity. Storage first.

---

## Future — Memory and CPU Layers

**Status: Documented so I don't forget. Not actively planned.**

### Memory Layer (HCC-Memory)
- Apply color logic to RAM pages.
- Pages from the same branch stay in memory together, get evicted together.
- Compete with zram on compressed memory.
- "Cold" pages compressed with higher levels.

### CPU Layer (HCC-CPU)
- Color-aware task scheduling.
- Tasks in the same branch run on adjacent cores.
- P-cores for hot branches, E-cores for cold.
- Probably an eBPF-based approach, not a scheduler rewrite.

---

*Last updated: June 2026. Everything here is subject to change, deletion, 
or complete reversal as I learn more. This is thinking out loud, not a 
commitment.*
