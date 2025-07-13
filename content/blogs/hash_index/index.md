+++
title = 'Hash Indexing'
draft = false
+++

# Hash Indexing: A Foundation for Key-Value Storage

Hash indexing is a method used to index key-value data. It is one of the simplest and most fundamental indexing strategies, forming the basis for more complex structures like B-trees and LSM-trees.

## Key-Value Indexing and Hash Functions

Key-value stores are quite similar to the `dictionary` or `HashMap` data types found in most programming languages, typically implemented using hash functions.

### Simple Idea

- **Indexing**: Maintain an in-memory hash map that maps each key to a byte offset in the data file—indicating where the value is stored.
- **File Storage**: Use append-only files (as introduced in the previous blog post).

**Example:**

If the file contains:

```
0   url1 data1
25  url2 data2
50  url3 data3
```

Then the hash map looks like:

```
"url1" → 0
"url2" → 25
"url3" → 50
```

---

## How to Scale Append-Only Storage?

### Introducing: Data Compaction

To avoid endlessly growing files, we segment the logs and periodically merge them to remove duplicates and deleted (tombstoned) entries.

### What Is Data Compaction?

- Store logs in **segments**
- The last segment is the most recent
- Periodically merge segments:
  - Discard outdated or deleted keys
  - Retain only the latest values

**Example:**

```
Segment 1:
    "url1": <data1>
    "url2": <data2>
    "url3": <data3>

Segment 2:
    "url1": <TOMBSTONE>
    "url2": <new_data2>
    "url4": <data4>

After compaction:
    "url2": <new_data2>
    "url3": <data3>
    "url4": <data4>
```

The compaction process runs in a **background thread**, allowing the main thread to continue serving requests without blocking.

---

## In-Memory Segment Hash Maps

Each segment maintains its own in-memory hash map.

When looking up a key:
- First check the most recent segment’s hash map
- If not found, check older segments in order

This design keeps recent data retrieval fast while gradually cleaning up old data in the background.

---

## Why Use Append-Only File Systems?

- **Faster Writes**: Appending and merging are sequential operations, faster than random writes (especially on HDDs, and still favorable on SSDs).
- **Simpler Crash Recovery**: With immutable segments, you avoid partial overwrites or corrupted files.
- **Reduces Fragmentation**: Periodic compaction prevents file fragmentation and stale data buildup.

---

## What Are the Trade-Offs?

- **No Efficient Range Queries**:
  - Hash indexes do not preserve key order
  - You can't easily scan from `kitty00000` to `kitty99999`
- **Memory Limitations**:
  - Hash table must fit entirely in memory for good performance
  - Disk-based hash maps are difficult to scale
- **On-Disk Hash Maps Are Problematic**:
  - Random I/O makes lookups slow
  - Resizing is expensive
  - Collision handling is complex

### Example

Suppose you have 10 billion keys—too large for RAM—so you store the hash table on disk.

```
Lookup: To find "alice@example.com", you must calculate its hash and jump to the right disk offset.
```

- This causes multiple random I/O operations
- Updates and inserts are also inefficient

> Hash tables are designed for fast, random access—but only when the full structure fits in memory. On disk, they become slow and hard to maintain.

---

## Final Thoughts

Hash indexes are a powerful choice for exact key lookups in memory-efficient scenarios. However, their limitations with range queries and disk access pave the way for more advanced structures like SSTables, LSM-trees, and B-trees.

In the next post, we’ll explore those alternatives—and how they build on the simplicity of hash indexing to scale better with large datasets.

