+++
title = 'Intro to Indexing'
draft = false
+++
# Why Do We Need Indexing?

## Let's Implement the Simplest Key-Value Store Using Bash and a File

These two functions implement a very basic key-value store using Bash:

```bash
# Append a key-value pair to the database file
db_set(){
	echo "$1 $2" >> database
}

# Get the most recent value for a given key
db_get(){
	grep "^$1 " database | tail -n 1 | awk '{print $2}'
}
```

> This is an example of a **log-structured** or **append-only** data file. Each time we write a new value, we simply append to the end.

---

## The Problem with Lookups

Every time you want to look up a key using `db_get`, it has to **scan the entire file** from beginning to end, searching for all occurrences of the key. It then picks the most recent one.

In algorithmic terms, the cost of a lookup is **O(n)** — that is, the time taken grows linearly with the number of records.

> For example, if the file has 1,000 entries, it may need to check all 1,000 lines to find the correct value.

---

## What’s the Solution? Indexing!

To efficiently find the value for a particular key, we need a different structure: an **index**.

> An **index** is additional metadata that acts as a signpost, helping you quickly locate the data you want — without scanning the entire dataset.

---

## How Does an Index Work?

Let’s say our data file looks like this:

```
database:
A 10
B 20
C 30
```

An index might look like:

```
A → line 1
B → line 2
C → line 3
```

So instead of scanning line-by-line, we can jump directly to the location for key `B`.

In real-world databases, indexes are much more sophisticated, but the principle remains the same.

---

## What’s the Trade-Off?

Indexes greatly improve **read performance**, but they come with costs:

-  **Slower writes** — because the index also needs to be updated every time you write data.
-  **Extra storage** — because indexes consume space.

That’s why most databases don’t index everything by default. Instead, they let **you — the developer or DBA — decide** which columns to index based on query patterns.

---

## What’s Next?

In the next part of this series, we’ll explore:

- How different types of indexes (e.g., Hash, B-Tree) work
- How real databases maintain indexes efficiently during updates and deletions

Stay tuned!
