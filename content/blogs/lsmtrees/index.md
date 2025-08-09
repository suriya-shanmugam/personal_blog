+++
title = 'SSTables and LSM Trees in Modern Storage Engines'
date = 2025-08-08T21:07:47-07:00
draft = false
+++
Modern storage engines like RocksDB, LevelDB, and Apache Cassandra rely on powerful data structures to handle large volumes of data efficiently. Two such core concepts are SSTables (Sorted String Tables) and LSM Trees (Log-Structured Merge Trees). In this post, we’ll walk through what they are, how they work together, and how they support scalable storage systems.

## What Are SSTables?
An SSTable (Sorted String Table) is a file format that stores a sorted sequence of key-value pairs. The key idea is that all keys are ordered, and each key appears only once per file.

### Why Use SSTables?
Compared to unordered log segments (e.g., with hash indexes), SSTables offer several key benefits:

1. Efficient Merging:
Sorted data allows merging multiple files using a simple and fast merge sort algorithm.

2. Sparse Index for Faster Lookups:
Because keys are sorted, you don’t need a full in-memory index—only a sparse index storing selected key offsets. This reduces memory usage while allowing fast lookups.

3. Block Compression:
Group adjacent key-value pairs into compressed blocks. The sparse index can point to the start of each compressed block, improving I/O efficiency and reducing disk space usage.

### Real-World Usage
> Apache Cassandra stores its immutable SSTables on disk and relies heavily on their sorted nature for compaction and querying.  
>
> RocksDB and LevelDB use SSTables to persist sorted key-value pairs during memtable flushes.

## What Are LSM Trees?
A Log-Structured Merge-Tree (LSM Tree) is a write-optimized data structure that organizes multiple SSTables in levels and merges them gradually in the background.

### Key Properties
**High Write Throughput:**
Writes are first buffered in memory and then flushed sequentially to disk, making disk I/O highly efficient.

**Sorted Storage:**
Enables efficient range queries without needing to load the entire dataset into memory.

**Background Compaction:**
Merges SSTables, removes old/deleted entries, and keeps the number of SSTables manageable.

### Real-World Usage
> LevelDB (by Google): A lightweight embedded key-value store built on LSM Trees.
>
> RocksDB (by Meta): A high-performance fork of LevelDB optimized for SSDs, also based on LSM Trees.
>
> Apache HBase: Uses LSM Trees to manage large-scale time-series and columnar data.
>
> Apache Cassandra: Combines LSM Trees with distributed replication and tunable consistency.

## How SSTables and LSM Trees Work Together?
Here’s how a typical storage engine uses SSTables in an LSM Tree architecture:

1. Memtable Writes:
Incoming writes go into an in-memory structure called a memtable.

2. Flush to SSTables:
When the memtable exceeds a size threshold, it’s flushed to disk as a new SSTable.

3. Hierarchical Reads:
Read requests check:

    - Memtable

    - The newest SSTables

    - Older SSTables, in sequence

4. Compaction:
SSTables are periodically merged and compacted in the background to optimize space and remove outdated data.

## Optimizations in LSM Trees
1. Write-Ahead Logging (WAL)
To prevent data loss on crashes, every write is first recorded in a write-ahead log before being added to the memtable.

2. Bloom Filters
To avoid expensive lookups for non-existent keys, each SSTable maintains a Bloom filter, which tells us with high probability if a key is not in the file—saving disk I/O.

Used by: RocksDB, LevelDB, Cassandra

3. Compaction Strategies
Two widely used strategies:

- Size-Tiered Compaction (STC):
Merge SSTables of similar size together.

- Leveled Compaction (LCS):
Organize SSTables into levels where each level holds files of exponentially increasing size and tighter key range bounds.

> RocksDB uses leveled compaction by default for better read performance.

## Final Thoughts
SSTables and LSM Trees are at the heart of modern high-performance databases. Their elegant combination of write efficiency, sorted storage, and compaction make them perfect for workloads that demand high throughput and scalable reads.

Understanding these concepts is essential whether you’re tuning a database like RocksDB, building on top of Cassandra, or designing your own storage layer.
