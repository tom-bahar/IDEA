# IDEA — Usage Guide

This guide walks you through setting up and running the IDEA project from scratch. It assumes familiarity with the Linux command line but does not assume prior experience with Lucene, CMake, or deduplication systems.

---

## Prerequisites

- **OS**: Ubuntu 22.04 (strongly recommended; the install script targets this version)
- **Root/sudo access**: Required for installing system packages and clearing the OS cache during benchmarks
- **Disk space**: Several GB for dependencies, source build, and datasets
- **Time**: Installation takes ~20 minutes on a clean system

---

## Step 1 — Install Dependencies

The project depends on several libraries that must be compiled from source (Lucene++, BerkeleyDB, Boost) as well as standard system packages.

```bash
cd IDEA/
chmod +x install_dependencies.sh
./install_dependencies.sh
```

This script will:
- Install system packages: `cmake`, `g++-9`, `libssl-dev`, `libglib2.0-dev`, and others
- Download and compile **Boost 1.58.0** (C++ library collection)
- Download and compile **BerkeleyDB 4.8.30** (embedded key-value database used for reverse mappings)
- Download and compile **Lucene++ 3.0.7** (C++ full-text search engine)

> **Note**: The script assumes internet access to download these packages. If your machine is offline, you will need to transfer the tarballs manually.

After installation, add the Boost library path to your environment. The script will print the exact path; it looks like:

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/IDEA/boost_1_58_0/stage/lib/
```

Add this line to your `~/.bashrc` (or `~/.zshrc`) so it persists across sessions:

```bash
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/IDEA/boost_1_58_0/stage/lib/' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 2 — Compile the Project

```bash
chmod +x compile.sh
./compile.sh
```

This runs CMake to configure the build and then compiles the C/C++ source code. The resulting binary (`destor`) is copied to `/usr/local/bin/destor` so you can run it from anywhere.

To verify it compiled correctly:

```bash
destor -h
```

You should see a help/usage message.

> **What is `create_build_command.py`?** This Python script generates the exact CMake command with the correct paths to your locally compiled libraries. `compile.sh` calls it internally. You generally do not need to run it manually.

---

## Step 3 — Configure the Working Environment

All runtime behavior is controlled by `destor.config` in the project root. Before doing anything, you need to:

**3a. Choose a configuration preset**

Three presets are provided in `configs/`:

| Preset | Description |
|--------|-------------|
| `configs/default.config` | Base IDEA — no byte offsets, no ranking |
| `configs/offset.config` | IDEA with term-vector byte offsets |
| `configs/ranking.config` | IDEA with TF-IDF relevance scoring |

Copy the one you want to use as the active config:

```bash
cp configs/default.config destor.config
```

**3b. Create the working directories**

The config file references three directories that must exist before any data is written. Check the paths set in `destor.config` under `working-directory`, `index-directory`, and `reverse-directory`. Create them:

```bash
mkdir -p working_directories/backup_directory
mkdir -p working_directories/index_directory
mkdir -p working_directories/chunk_to_file_directory
```

> The exact paths may differ if you edited the config. Match what is in your `destor.config`.

---

## Step 4 — Prepare a Dataset

The paper uses two dataset families. See `dataset_details/README.md` for download instructions.

**Linux Kernel** — Multiple revisions of the Linux kernel source tree:
- `LNX-198`: 198 revisions (all minor versions from 2.0 to 5.9)
- `LNX-409`: 409 revisions (every 10th patch)
- `LNX-662`: 662 revisions (every 5th patch)

**Wikipedia** — Monthly English Wikipedia dumps:
- `WIKI-4` through `WIKI-24` (4 to 24 consecutive monthly snapshots, starting Jan 2017)
- Source: [Internet Archive](https://archive.org)

For testing purposes you can use any directory on your system — the backup command accepts any file path.

---

## Step 5 — Run a Backup

A backup reads files, chunks and deduplicates them, and writes the result to the container store. No index is built at this stage.

```bash
destor /path/to/your/dataset
```

The system will print progress as it processes files. When finished, you should see a summary of how many chunks were stored and how many were deduplicated.

> **What happens on disk?** The backup creates a container store under `working-directory` and recipe files recording which chunks compose each backed-up file. Nothing is searchable yet — that requires the next step.

To back up multiple revisions (simulating incremental backups), run the command again with a different or updated directory. The deduplication system will reuse chunks from previous runs.

---

## Step 6 — Build an Index

After backing up data, you build a search index. Two modes are available:

### Naive Index

Reconstructs each backed-up file from its chunks and indexes the content with Lucene++. This is the baseline approach.

```bash
destor -n
```

### IDEA Index (Deduplication-Aware)

Indexes each unique chunk once (physical index) and builds a reverse mapping from chunks back to files (logical index).

```bash
destor -q
```

> **Inline indexing** (building the index during backup, not after): append the dataset path to either command:
> ```bash
> destor -n /path/to/dataset   # Naive inline
> destor -q /path/to/dataset   # IDEA inline
> ```

Both commands append timing and size statistics to `index.log`.

---

## Step 7 — Run a Lookup (Search)

Once an index exists, you can search it.

### Search with keywords on the command line

```bash
# Naive index lookup
destor -m keyword1 keyword2 keyword3

# IDEA index lookup
destor -l keyword1 keyword2 keyword3
```

### Search with keywords from a file

The `keywords/` directory contains pre-built keyword sets. Each file holds one keyword per line.

```bash
# Naive lookup, 128 medium-frequency Linux keywords
destor -m -f "keywords/linux/128/file-med.txt"

# IDEA lookup, same keywords
destor -l -f "keywords/linux/128/file-med.txt"
```

Results are printed to stdout. Timing statistics are appended to `lookup.log`.

---

## Step 8 — Inspect System State

At any point you can print a summary of what is currently stored:

```bash
destor -s
```

This shows backup statistics: number of revisions, total data, unique data, deduplication ratio, and index sizes.

---

## Running the Paper's Experiments

The `scripts/` directory contains shell scripts that reproduce the figures from the FAST '24 paper.

### Full experiment workflow

```bash
# 1. Start fresh
scripts/clean_index.sh

# 2. Set base configuration
cp configs/default.config destor.config

# 3. Back up your dataset (repeat for each revision)
destor /path/to/dataset/revision_1
destor /path/to/dataset/revision_2
# ...

# 4. Build both indexes
scripts/create_indexes.sh

# 5. Run lookup experiments
scripts/figure_10.sh    # Varies keyword count (16, 32, 64, 128)
scripts/figure_11.sh    # Tests multiple keyword dictionaries
scripts/figure_16.sh    # Single keyword lookup

# 6. For offset/ranking experiments, switch config and re-index
cp configs/offset.config destor.config
scripts/clean_index.sh
scripts/create_indexes.sh
scripts/figure_11.sh    # Now produces Figure 15 data
```

> **Cache clearing**: Some scripts call `scripts/clear_cache.sh` which flushes the OS page cache using `sudo`. This ensures lookup times are measured from a cold cache (as if the data has not been recently accessed), which gives more realistic and reproducible results. You may be prompted for your password.

---

## Configuration Reference

Key options in `destor.config`:

| Option | Values | Description |
|--------|--------|-------------|
| `working-directory` | path | Where containers and recipes are stored |
| `index-directory` | path | Where the Lucene++ index is stored |
| `reverse-directory` | path | Where the reverse mapping is stored |
| `chunk-algorithm` | `rabin`, `fixed`, `ae`, `tttd` | How to split files into chunks |
| `chunk-avg-size` | bytes (e.g. `8192`) | Target average chunk size |
| `chunking-type` | `whitespace`, `normal` | Whether to align chunk boundaries to whitespace |
| `reverse-mapping` | `in-doc`, `db`, `external`, `lucene` | How to store chunk-to-file mappings |
| `indirect-path-strings` | `yes`, `no` | Store file IDs, not full paths, in the mapping |
| `offsets-mode` | `none`, `term-vectors` | Whether to record byte positions of matched terms |
| `tf-idf` | `yes`, `no` | Enable TF-IDF relevance ranking |

---

## Understanding the Log Files

### `index.log` (one line per index creation run)

| Column | Meaning |
|--------|---------|
| `index_type` | `naive` or `dedup` |
| `total_time` | Wall time for the entire indexing run (ms) |
| `process_data_time` | Time spent reading and indexing chunk/file data |
| `create_reverse_mapping_time` | Time spent building the chunk-to-file mapping (IDEA only) |
| `file_num` | Number of files indexed |
| `chunk_num` | Number of chunks processed |
| `chunk_index_size` | Size of the Lucene++ chunk index on disk |
| `complete_index_size` | Total index size including reverse mapping |

### `lookup.log` (one line per search run)

| Column | Meaning |
|--------|---------|
| `index_type` | `naive` or `dedup` |
| `keywords_num` | How many keywords were searched |
| `total_lookup_time` | Total search time (ms) |
| `index_lookup_time` | Time for the Lucene++ query |
| `reverse_lookup_time` | Time to resolve fingerprints to file paths (IDEA only) |
| `chunk_found` | Number of matching chunks |
| `results` | Number of matching files returned |

---

## Troubleshooting

**`destor: command not found`**
The binary was not installed. Check whether `compile.sh` completed without errors, and verify `/usr/local/bin/destor` exists.

**Lucene++ linking errors at runtime**
The shared libraries are missing from `LD_LIBRARY_PATH`. Re-run the export command from Step 1 or check your `.bashrc`.

**`ERROR: working directory does not exist`**
The directories in `destor.config` have not been created. Follow Step 3b.

**Lookup returns no results**
- Ensure the index was built for the same mode you are querying (`-n` for naive `-m`, `-q` for IDEA `-l`).
- Check that `destor.config` was not changed between indexing and lookup (the index is tied to a specific working directory).
- Try a keyword you know exists in the dataset.

**`clear_cache.sh` fails**
This script requires root access. Run it with `sudo scripts/clear_cache.sh` or ensure your user has the necessary permissions.
