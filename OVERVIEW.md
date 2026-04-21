# IDEA — Project Overview

## What Is IDEA?

**IDEA** (Inverted Deduplication-Aware Index) is a research system that enables full-text search over **deduplicated backup data**. It was published at the [22nd USENIX Conference on File and Storage Technologies (FAST '24)](https://www.usenix.org/conference/fast24), February 2024.

The core problem IDEA solves: modern backup systems use **data deduplication** to save disk space — identical chunks of data are stored only once even if they appear in many backups. This is great for storage efficiency, but it breaks traditional full-text search, which expects each file to be stored in its entirety. IDEA bridges that gap.

---

## The Pipeline

IDEA implements a **backup-and-index pipeline** that processes files through a series of sequential phases. Think of it as an assembly line — each phase transforms the data and hands it off to the next.

```
Files on disk
     │
     ▼
[ 1. Read ]  ──────────────────── Read files into memory blocks
     │
     ▼
[ 2. Chunk ] ──────────────────── Split blocks into small chunks (~8KB each)
     │
     ▼
[ 3. Hash ]  ──────────────────── Compute a fingerprint (SHA-1) for each chunk
     │
     ▼
[ 4. Dedup ] ──────────────────── Identify duplicate chunks; mark unique ones
     │
     ▼
[ 5. Rewrite ] ─────────────────── Optimize chunk ordering to reduce fragmentation
     │
     ▼
[ 6. Filter ] ──────────────────── Pack chunks into containers; build the index
     │
     ▼
[ 7. Storage ] ─────────────────── Persist containers, recipes, and indexes to disk
```

### Phase-by-Phase Explanation

**1. Read** — The pipeline starts by reading files from a directory, breaking them into large blocks (~1MB) suitable for processing. In indexing modes, this phase can also be index-aware.

**2. Chunk** — Each block is split into smaller, variable-size chunks (around 8KB on average). The chunking algorithm uses content-defined boundaries (Rabin fingerprinting) so that a small change to a file only affects a small number of chunks — not the entire file. For text data, chunk boundaries are aligned to whitespace so that words are not split across chunks, which is important for full-text search accuracy.

**3. Hash** — Every chunk gets a SHA-1 hash (its "fingerprint"), which serves as a unique identifier. If two chunks have the same fingerprint, they are identical.

**4. Dedup (Deduplication)** — The system checks each chunk's fingerprint against a global index. If the chunk already exists in storage, it is marked as a *duplicate* and will not be stored again. Only *unique* chunks proceed to storage. This is the core mechanism that saves disk space across many backup revisions.

**5. Rewrite** — Deduplication can scatter a file's chunks across many containers stored at different locations. This phase optionally reorders or rewrites chunks to reduce fragmentation and improve future read performance. Several strategies are supported (CFL, context-based, capping, HAR).

**6. Filter** — Chunks are packed into fixed-size containers (~40KB). This is also where indexing happens, depending on the mode:
- In **Naive mode**: file content is extracted and indexed directly with a full-text search engine (Lucene++).
- In **IDEA mode**: chunk content is indexed by fingerprint (physical index), and a mapping from chunks back to files is built (reverse mapping).

**7. Storage** — Containers are written to the container store on disk. Metadata about which chunks belong to which files is stored in "recipe" files.

---

## Two Indexing Strategies

The central contribution of the paper is comparing two ways to build a searchable index over deduplicated data:

### Naive Indexing

Reconstructs files from their deduplicated chunks and indexes the full file content using a standard inverted index (Lucene++). Simple and correct, but wasteful — the same deduplicated data is re-read and re-indexed for every backup revision that contains it.

```
Backup storage → Reconstruct files → Index full content
```

### IDEA Indexing (Deduplication-Aware)

Splits the index into two layers:

1. **Physical Index** — Indexes each *unique chunk* exactly once, by its fingerprint. Because duplicates are skipped, this index is proportional to the amount of *unique* data, not the total data.

2. **Logical Index (Reverse Mapping)** — Records, for each chunk fingerprint, which files contain that chunk. This layer is what connects a chunk-level search result back to the files a user cares about.

```
Search query → Chunk index (fingerprints) → Reverse mapping → File list
```

A lookup works in two steps: first find which chunks match the query, then find which files contain those chunks.

---

## Folder Structure

```
IDEA/
├── README.md                  # Original setup instructions
├── OVERVIEW.md                # This file
├── GUIDE.md                   # Step-by-step usage guide
│
├── compile.sh                 # Compile the project
├── install_dependencies.sh    # Install all required libraries
├── create_build_command.py    # Helper to generate CMake build command
├── destor.config              # Main configuration file (edit before running)
│
├── src/                       # All source code
│   ├── destor.c / destor.h    # Main entry point and CLI argument parsing
│   ├── do_index.cpp           # Index creation and lookup logic (core of IDEA)
│   ├── do_backup.c            # Orchestrates the backup pipeline
│   ├── chunk_phase.c          # Phase 2: chunking
│   ├── hash_phase.c           # Phase 3: hashing
│   ├── dedup_phase.c          # Phase 4: deduplication
│   ├── rewrite_phase.c        # Phase 5: rewriting
│   ├── filter_phase.c         # Phase 6: filter (naive mode)
│   ├── dedup_index_filter_phase.c  # Phase 6: filter (IDEA mode)
│   │
│   ├── chunking/              # Chunking algorithm implementations
│   ├── clucene-wrapper/       # C++ wrapper for the Lucene++ search library
│   ├── index/                 # Fingerprint index for deduplication lookups
│   ├── reverse_mapping/       # Chunk-to-file reverse mapping implementations
│   ├── storage/               # Container store: reading/writing containers
│   ├── recipe/                # Recipe store: file-to-chunk metadata
│   └── utils/                 # Shared data structures and helpers
│
├── configs/                   # Ready-to-use configuration presets
│   ├── default.config         # Base IDEA (no offsets, no ranking)
│   ├── offset.config          # IDEA with byte-offset positions
│   └── ranking.config         # IDEA with TF-IDF relevance scoring
│
├── scripts/                   # Shell scripts for experiments
│   ├── create_indexes.sh      # Build both naive and IDEA indexes
│   ├── figure_10.sh           # Reproduce paper Figure 10 (lookup scaling)
│   ├── figure_11.sh           # Reproduce paper Figure 11 (multi-dictionary)
│   ├── figure_16.sh           # Reproduce paper Figure 16 (single keyword)
│   ├── clean_index.sh         # Delete index files
│   ├── clear_cache.sh         # Flush the OS page cache (requires sudo)
│   └── cleanup.sh             # Full cleanup of all generated data
│
├── keywords/                  # Pre-built keyword sets for benchmarking
│   ├── linux/                 # Keywords extracted from Linux kernel dataset
│   │   └── 128/               # 128-keyword dictionaries (also: 16/, 32/, 64/)
│   │       ├── file-low.txt   # Rare keywords (appear in few files)
│   │       ├── file-med.txt   # Medium-frequency keywords
│   │       ├── file-high.txt  # Common keywords (appear in many files)
│   │       ├── chunk-low.txt  # Rare at chunk level
│   │       ├── chunk-med.txt
│   │       └── chunk-high.txt
│   └── wiki/                  # Same structure for Wikipedia dataset
│
├── dataset_details/           # Documentation for the benchmark datasets
│   ├── README.md              # How to download and construct the datasets
│   ├── LNX-198.txt            # Linux kernel: 198 revisions
│   ├── LNX-409.txt            # Linux kernel: 409 revisions
│   ├── LNX-662.txt            # Linux kernel: 662 revisions
│   ├── WIKI-4.txt             # Wikipedia: 4 monthly snapshots
│   └── ...                    # (up to WIKI-24)
│
└── libs/                      # Pre-compiled Lucene++ shared libraries
```

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **Chunk** | A small piece of a file (~8KB), defined by content rather than fixed size |
| **Fingerprint** | A SHA-1 hash that uniquely identifies a chunk |
| **Container** | A packed group of chunks (~40KB), the physical unit stored on disk |
| **Recipe** | Metadata recording which chunks (fingerprints) make up a given file |
| **Reverse mapping** | A lookup table: given a fingerprint, which files contain that chunk? |
| **Deduplication** | Storing identical chunks only once across many backups |
| **Physical index** | The chunk-level full-text index (indexed by fingerprint) |
| **Logical index** | The file-level index built by following the reverse mapping |

---

## Configuration System

Everything the system does is controlled by `destor.config` in the project root. Before running any command, you copy one of the preset configs from `configs/` over the main config file, or edit it directly.

Important settings include:

- **Working directories** — where backup data, indexes, and mappings are stored
- **Chunk algorithm** — `rabin` (content-defined) is the default; `fixed` is simpler
- **Reverse mapping type** — how the chunk-to-file mapping is stored (`in-doc`, `db`, `external`, `lucene`)
- **Offsets mode** — whether to record byte-level positions of search terms
- **TF-IDF** — whether to enable relevance ranking in search results

---

## Performance Logging

Every run appends structured results to log files:

- **`index.log`** — Records how long indexing took, how many chunks/files were processed, and how large the index is on disk.
- **`lookup.log`** — Records search latency broken down by phase (chunk lookup, reverse mapping, path resolution), number of results, and number of keywords searched.

These logs are what the paper's figures are derived from.
