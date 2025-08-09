+++
title = 'Demystifying B-Tree Indexing in Databases'
date = 2025-08-09T11:33:17-07:00
draft = false
+++
 
When it comes to speeding up data retrieval, indexes are the backbone of modern databases. Among the many indexing techniques out there, **B-trees** are one of the most widely used—powering systems like MySQL, PostgreSQL, and Oracle. In this post, we’ll explore what B-tree indexing is, how it works, how it compares with LSM trees, and what optimizations make it even better.

---

## What is B-Tree Indexing?

At a high level, B-tree indexing organizes **key-value pairs in sorted order**, which allows for fast **lookups and range queries**. But beyond this similarity with structures like SSTables, B-trees follow a fundamentally different design philosophy.

### Page-Based Structure

B-trees break the database into **fixed-size blocks or pages**, typically 4KB in size. This design closely mirrors hardware-level storage, where disk I/O also happens in fixed-size blocks. Each page can reference other pages via disk addresses—similar to pointers in memory.

There’s a **single root page** at the top of the B-tree. To find a key, the database engine starts here and traverses down the tree by following references. Each intermediate page acts like a decision tree, mapping key ranges to child pages. Eventually, the search lands on a **leaf page**, which either stores the values directly or references where they can be found.

This multi-level structure makes B-trees **logarithmically scalable**. With a branching factor of 500 and 4KB pages, just four levels can index up to **256 TB of data**!

---

## Modifying B-Trees: Updates and Inserts

Updating a value in a B-tree is straightforward:

- Find the leaf page containing the key  
- Modify the value  
- Write the page back to disk  

Since page locations remain stable, references to them do not change. Inserting a new key is similar—if there’s space in the correct page, it’s inserted directly. If not, the page is split in two, and the parent page is updated accordingly. This keeps the tree **balanced**, maintaining its `O(log n)` depth.

---

## B-Trees vs. LSM Trees

B-trees and **Log-Structured Merge (LSM) Trees** take opposite approaches to storage:

| Feature          | B-Trees             | LSM Trees                   |
|------------------|---------------------|------------------------------|
| Write behavior   | In-place updates     | Append-only                  |
| Crash safety     | Requires WAL         | Built-in via immutable segments |
| Read performance | Fast for random lookups | Optimized for writes, reads can be costly without compaction |
| Concurrency      | Requires latches     | Easier to scale writes       |

The key difference lies in their **write patterns**. While B-trees modify data in place, LSM trees append data to new segments. This makes B-trees more sensitive to crashes—partially written pages can lead to corruption. To prevent this, B-trees use a **Write-Ahead Log (WAL)**, ensuring changes are safely logged before they're applied.

Concurrency is another area where B-trees require careful handling. Since multiple threads may access or modify the tree, B-tree implementations use **latches (lightweight locks)** to maintain consistency.

---

## Optimizations: B+ Trees and Beyond

Modern systems often use **B+ Trees**, a variant of B-trees with additional enhancements:

- Leaf pages include **pointers to siblings**, enabling fast in-order scans without backtracking.
- All values are stored in the **leaf nodes**, keeping internal nodes smaller and more efficient.

Some systems even hybridize B-trees and log-structured concepts. For example, **Fractal Trees** incorporate buffering techniques from LSM trees to reduce disk seeks while preserving the page-oriented structure of B-trees.

---

## Final Thoughts

B-tree indexing strikes a balance between fast reads and manageable writes, making it ideal for a wide range of workloads. While log-structured approaches like LSM trees have grown in popularity for write-heavy applications, B-trees remain the go-to choice for **OLTP systems** that demand **low-latency lookups** and **fine-grained updates**.

Understanding B-trees helps you make informed decisions when choosing a database engine or tuning its performance—and that’s the kind of knowledge that pays off at scale.

---
## References

[B-Tree Visualize](https://planetscale.com/blog/btrees-and-database-indexes)
