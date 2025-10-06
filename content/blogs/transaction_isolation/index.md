+++
title = 'Isolation levels in Databases'
date = 2025-10-05T18:06:55-07:00
draft = false
+++
Modern applications rely heavily on databases to store and manage critical data. But with multiple users, concurrent operations, and potential system failures, how can we ensure that data remains **accurate, consistent, and reliable**?

This is where **transactions** and the **ACID properties** come in. A transaction allows a set of database operations — reads, writes, updates, and deletes — to be executed as a single logical unit. Either all operations succeed (**commit**) or none take effect (**rollback**), providing a safety net against partial updates, crashes, and concurrency issues.

By enforcing **Atomicity, Consistency, Isolation, and Durability**, databases guarantee predictable behavior even in complex, concurrent environments. At the same time, developers must understand the trade-offs between **strict data guarantees** and **system performance**, especially when scaling or replicating data.

In this blog, we’ll explore:

* How transactions work
* The meaning and examples of **ACID properties**
* Isolation levels and their impact on concurrent transactions
* Practical guidance for choosing the right level of consistency for your applications

By the end, you’ll have a clear understanding of how databases maintain integrity, how to avoid anomalies, and how to select the best strategy for your application’s needs.

---
# What Are Transactions?

A **transaction** is a sequence of one or more read and write operations grouped into a single unit of work.
It either **commits** (successfully applies all changes) or **aborts/rolls back** (undoes all changes as if nothing happened).

### Example

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;
```

If the system crashes after the first update but before the second, the transaction is rolled back — ensuring that no account loses money.

Transactions aren’t laws of nature — they’re a design choice made to ensure data integrity and ease of development. However, as modern NoSQL systems emerged (e.g., MongoDB, Cassandra), many relaxed or dropped these guarantees to achieve scalability, replication, and partitioning by default. In many cases, transactions were either absent or weakened (offering only per-document atomicity).

---

# Single-Object Transactions

Even when modifying a single data object, atomicity and isolation remain critical.

Imagine writing a 20 KB JSON document to a database:

* If the network disconnects after sending 10 KB, will the database store a broken JSON fragment?
* If the power fails midway while overwriting the old value, could the old and new data be spliced together?
* If another client reads the document during the write, will it see a partially written version?

In databases like PostgreSQL or MySQL (InnoDB), the answers are:

* No partial data is ever stored or visible.
* The operation is atomic — you see either the old or the new value, never a mix.

Thus, every single write behaves as an all-or-nothing operation, even for a single object.

---

# Multi-Object Transactions

When an application modifies multiple rows, tables, or documents, it uses multi-object transactions.

Everything between a `BEGIN TRANSACTION` and `COMMIT` statement is part of a single atomic operation:

```sql
BEGIN TRANSACTION;

INSERT INTO orders VALUES (101, 'MacBook Pro', 2499);
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'MBP';

COMMIT;
```

If any statement in between fails — for example, if the inventory update violates a constraint — the transaction rolls back, and the new order is not saved.

In SQL systems, a multi-object transaction is scoped to the client’s TCP connection with the database. All statements issued through that connection between `BEGIN TRANSACTION` and `COMMIT` are treated as part of one logical transaction.

---

By using transactions, applications gain:

| Property        | Description                                                          |
| --------------- | -------------------------------------------------------------------- |
| **Atomicity**   | Ensures that operations are all-or-nothing.                          |
| **Consistency** | Guarantees that data remains valid before and after the transaction. |
| **Isolation**   | Prevents concurrent transactions from interfering with one another.  |
| **Durability**  | Ensures committed data survives failures.                            |

These principles — known as **ACID properties** — form the foundation of reliable transactional systems.

---

# ACID Properties in Transactions

Transactions in databases are governed by four key properties — **Atomicity, Consistency, Isolation, and Durability**, collectively known as **ACID**.
These properties ensure that database operations remain reliable even in the face of system crashes, concurrent users, or unexpected errors.

---

## A — Atomicity (Abortability)

**Definition:**
Atomicity ensures that a transaction is **all or nothing** — either every operation in the transaction succeeds, or none of them do.
If an error occurs midway, the database automatically **aborts** the transaction and **rolls back** any partial changes.

This property eliminates the problem of **partial failure**, where some changes are saved but others are not.

### Example

Consider a fund transfer between two accounts:

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;
```

If the second update fails due to a network issue or constraint error, the first update (debit from account 1) is rolled back.

### Result

| Step       | Action                      | Outcome                                         |
| ---------- | --------------------------- | ----------------------------------------------- |
| Step 1     | Withdraw 500 from Account 1 | Success                                         |
| Step 2     | Deposit 500 to Account 2    | **Fails**                                       |
| **Result** | Transaction rolled back     | Account 1 balance restored, Account 2 unchanged |

The database guarantees that **no money disappears or appears out of thin air** — maintaining atomicity.

---

## C — Consistency

**Definition:**
Consistency ensures that a transaction moves the database from **one valid state to another**.
It enforces **integrity constraints** such as foreign keys, unique indexes, or business rules.

While the database enforces physical consistency (like constraints and data types), the application logic often enforces logical consistency (like valid business rules).

### Example

Suppose there’s a constraint that the total sum of balances across all accounts must remain constant:

```sql
ALTER TABLE accounts ADD CONSTRAINT positive_balance CHECK (balance >= 0);
```

Now, if a transaction tries to withdraw more money than the account holds:

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 2000 WHERE id = 1;  -- Account has only 1000
COMMIT;
```

The `CHECK` constraint fails, and the transaction is **aborted**.

### Result

| Transaction Step      | Action              | Status                  |
| --------------------- | ------------------- | ----------------------- |
| Debit exceeds balance | Violates constraint | Transaction rolled back |
| **Final state**       | Database unchanged  | **Consistent**          |

Even if the programmer forgets to validate input, the database won’t commit inconsistent data.

---

## I — Isolation

**Definition:**
Isolation ensures that **concurrent transactions do not interfere** with one another.
Each transaction behaves **as if it were the only one** executing on the system, even when many run at the same time.

Without isolation, concurrent operations can cause problems such as:

* **Dirty reads** – Reading uncommitted data from another transaction.
* **Non-repeatable reads** – Seeing different results for the same query in one transaction.
* **Phantom reads** – Rows appearing or disappearing during a transaction due to concurrent inserts/deletes.

### Example

Two users simultaneously try to update the same account:

```sql
-- Transaction A
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Transaction B
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
```

If isolation is weak (e.g., READ UNCOMMITTED), both might read the same initial balance, resulting in incorrect final values.

With proper isolation (e.g., REPEATABLE READ or SERIALIZABLE), the database serializes these updates — one completes before the other starts.

### Result

| Isolation Level  | Behavior                                | Possible Issue         |
| ---------------- | --------------------------------------- | ---------------------- |
| READ UNCOMMITTED | Transactions can see uncommitted data   | Dirty read             |
| READ COMMITTED   | Reads only committed data               | Non-repeatable reads   |
| REPEATABLE READ  | Prevents dirty and non-repeatable reads | Phantom reads possible |
| SERIALIZABLE     | Transactions fully isolated             | Slowest but safest     |

We’ll explore these isolation levels in depth in the next section.

---

## D — Durability

**Definition:**
Durability ensures that **once a transaction is committed**, its results are **permanently saved**, even if the system crashes immediately afterward.

Databases achieve durability by writing data to **non-volatile storage** (like disk logs or SSDs) before confirming the commit to the client.

Durability ensures that once a transaction is committed, its effects are permanently recorded on stable storage or safely replicated across nodes. Databases achieve this using techniques like write-ahead logging, checkpointing, and replication. Even after crashes or power failures, committed data is recoverable and never lost.

### Example

```sql
BEGIN TRANSACTION;

UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'MBP';
COMMIT;
```

Once the commit succeeds, the update is stored in the **write-ahead log (WAL)**.
If the server crashes right after, the database recovers using the log and replays the committed transaction during restart.

### Result

| Step                | Event               | Outcome               |
| ------------------- | ------------------- | --------------------- |
| Commit acknowledged | WAL written to disk | Safe                  |
| Server crash        | Database restarts   | Transaction recovered |
| Final state         | Data persists       | **Durable**           |

---

# Isolation Level: READ COMMITTED

## Explanation

The **READ COMMITTED** isolation level guarantees two key properties:

1. **No Dirty Reads:** A transaction never reads data that another transaction has written but not yet committed.
2. **No Dirty Writes:** A transaction only overwrites data that has been committed, never uncommitted updates.

This is the **default isolation level** in many databases such as PostgreSQL and Oracle.
Each statement within a transaction sees only data that was committed *before* that statement began.

---

## Implementation

Databases typically implement READ COMMITTED using **row-level locks** and **versioning**:

* For each object (row) being written, the database stores both:

  * The **old committed value** (visible to other transactions).
  * The **new uncommitted value** (visible only to the transaction that holds the write lock).

* When a transaction commits, its new values replace the old committed ones.

* Other transactions reading the same data will always see the **most recently committed version**, ensuring that no uncommitted data is exposed.

---

## Example

Let’s simulate two concurrent transactions operating on the same account balance.

```sql
-- Transaction T1
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- Reads balance = 1000
-- (Some delay before next step)

-- Transaction T2
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
COMMIT;

-- Transaction T1 continues
SELECT balance FROM accounts WHERE id = 1;  -- Reads balance = 800
COMMIT;
```

**Result:**

* T1’s first read returns 1000 (old committed value).
* T2 commits and changes the balance to 800.
* When T1 reads again, it sees 800 — the new committed value.

This behavior ensures **no dirty reads**, but T1 sees different values within the same transaction — a phenomenon known as **non-repeatable read**.

---

## Unhandled Cases (Anomalies under READ COMMITTED)

Although READ COMMITTED prevents dirty reads and writes, it still allows several anomalies:

### 1. Read Skew / Non-Repeatable Read

Two concurrent transactions start at the same time, read the same value, but one commits and modifies the value before the other finishes.
When the first transaction re-reads the value, it sees a new committed version, producing inconsistent results.

**Impact:**

* Causes inaccurate results in **backups, analytics, or integrity checks** where consistent snapshots are needed.
* Example: While generating a financial report, the total balance changes midway through the read.

---

### 2. Lost Updates

Two transactions read the same committed value and both attempt to update it:

```sql
-- T1
BEGIN;
SELECT counter FROM stats WHERE id = 1; -- Reads 5
UPDATE stats SET counter = 6 WHERE id = 1;
COMMIT;

-- T2
BEGIN;
SELECT counter FROM stats WHERE id = 1; -- Reads 5
UPDATE stats SET counter = 7 WHERE id = 1;
COMMIT;
```

**Result:**
Both read the same initial value (5), but the second commit overwrites the first — resulting in **one lost update**.

---

### 3. Write Skew

A subtler anomaly where two transactions update **different objects** based on overlapping reads.

**Scenario:**
Two doctors, Alice and Bob, are on call. At least one doctor must remain on call at all times.

| Doctor | on_call |
| ------ | ------- |
| Alice  | TRUE    |
| Bob    | TRUE    |

Now both doctors attempt to go off call concurrently.

```sql
-- Transaction T1 (Alice)
BEGIN;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- Returns 2
UPDATE doctors SET on_call = FALSE WHERE name = 'Alice';
COMMIT;

-- Transaction T2 (Bob)
BEGIN;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- Returns 2
UPDATE doctors SET on_call = FALSE WHERE name = 'Bob';
COMMIT;
```

**Expected Behavior:**
At least one doctor should stay on call.

**Actual Behavior:**
Both transactions read the same committed state (2 doctors on call) and proceed independently.
After both commits, **no one is on call** — a logical inconsistency known as **write skew**.

---

| Property           | Ensures                         | Fails To Prevent                               | Use Cases                                                               |
| ------------------ | ------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------- |
| **READ COMMITTED** | No dirty reads, No dirty writes | Non-repeatable reads, Lost updates, Write skew | Most OLTP systems where perfect consistency across reads isn’t required |

---

# Isolation Level: READ COMMITTED

## Explanation

The **READ COMMITTED** isolation level guarantees two key properties:

1. **No Dirty Reads:** A transaction never reads data that another transaction has written but not yet committed.
2. **No Dirty Writes:** A transaction only overwrites data that has been committed, never uncommitted updates.

This is the **default isolation level** in many databases such as PostgreSQL and Oracle.
Each statement within a transaction sees only data that was committed *before* that statement began.

---

## Implementation

Databases typically implement READ COMMITTED using **row-level locks** and **versioning**:

* For each object (row) being written, the database stores both:

  * The **old committed value** (visible to other transactions).
  * The **new uncommitted value** (visible only to the transaction that holds the write lock).

* When a transaction commits, its new values replace the old committed ones.

* Other transactions reading the same data will always see the **most recently committed version**, ensuring that no uncommitted data is exposed.

---

## Example

Let’s simulate two concurrent transactions operating on the same account balance.

```sql
-- Transaction T1
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- Reads balance = 1000
-- (Some delay before next step)

-- Transaction T2
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
COMMIT;

-- Transaction T1 continues
SELECT balance FROM accounts WHERE id = 1;  -- Reads balance = 800
COMMIT;
```

**Result:**

* T1’s first read returns 1000 (old committed value).
* T2 commits and changes the balance to 800.
* When T1 reads again, it sees 800 — the new committed value.

This behavior ensures **no dirty reads**, but T1 sees different values within the same transaction — a phenomenon known as **non-repeatable read**.

---

## Unhandled Cases (Anomalies under READ COMMITTED)

Although READ COMMITTED prevents dirty reads and writes, it still allows several anomalies:

### 1. Read Skew / Non-Repeatable Read

Two concurrent transactions start at the same time, read the same value, but one commits and modifies the value before the other finishes.
When the first transaction re-reads the value, it sees a new committed version, producing inconsistent results.

**Impact:**

* Causes inaccurate results in **backups, analytics, or integrity checks** where consistent snapshots are needed.
* Example: While generating a financial report, the total balance changes midway through the read.

---

### 2. Lost Updates

Two transactions read the same committed value and both attempt to update it:

```sql
-- T1
BEGIN;
SELECT counter FROM stats WHERE id = 1; -- Reads 5
UPDATE stats SET counter = 6 WHERE id = 1;
COMMIT;

-- T2
BEGIN;
SELECT counter FROM stats WHERE id = 1; -- Reads 5
UPDATE stats SET counter = 7 WHERE id = 1;
COMMIT;
```

**Result:**
Both read the same initial value (5), but the second commit overwrites the first — resulting in **one lost update**.

---

### 3. Write Skew

A subtler anomaly where two transactions update **different objects** based on overlapping reads.

**Scenario:**
Two doctors, Alice and Bob, are on call. At least one doctor must remain on call at all times.

| Doctor | on_call |
| ------ | ------- |
| Alice  | TRUE    |
| Bob    | TRUE    |

Now both doctors attempt to go off call concurrently.

```sql
-- Transaction T1 (Alice)
BEGIN;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- Returns 2
UPDATE doctors SET on_call = FALSE WHERE name = 'Alice';
COMMIT;

-- Transaction T2 (Bob)
BEGIN;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE;  -- Returns 2
UPDATE doctors SET on_call = FALSE WHERE name = 'Bob';
COMMIT;
```

**Expected Behavior:**
At least one doctor should stay on call.

**Actual Behavior:**
Both transactions read the same committed state (2 doctors on call) and proceed independently.
After both commits, **no one is on call** — a logical inconsistency known as **write skew**.

---

| Property           | Ensures                         | Fails To Prevent                               | Use Cases                                                               |
| ------------------ | ------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------- |
| **READ COMMITTED** | No dirty reads, No dirty writes | Non-repeatable reads, Lost updates, Write skew | Most OLTP systems where perfect consistency across reads isn’t required |

---

# Repeatable Read Isolation Level

## Explanation

**Repeatable Read (Snapshot Isolation)** ensures that all reads within a transaction see a consistent snapshot of the database as it existed at the time the transaction started. Even if other transactions commit changes after it begins, the running transaction continues to see only its own snapshot view.

This level solves problems like **read skew** (where different reads within the same transaction return inconsistent results) by guaranteeing that data read once will not change if read again later in the same transaction.

---

## Implementation

Snapshot isolation is most commonly implemented using **Multi-Version Concurrency Control (MVCC)**.
Key implementation principles include:

* **Readers never block writers, and writers never block readers.**
* Each transaction sees data that was committed before it started, ignoring uncommitted or later transactions.
* **Write locks** are used to prevent dirty writes, while **reads** require no locks.
* The database maintains **multiple versions** of an object to serve transactions starting at different times.
* Updates are often implemented as a **delete + create** operation internally to maintain version history.

### Visibility Rules

When a transaction starts, the database determines which committed versions are visible:

1. Ignore writes from transactions that were in-progress when the current transaction started.
2. Ignore writes from **aborted** transactions.
3. Ignore writes from **transactions that started after** the current transaction.
4. All other committed writes are **visible**.

---

## Example

**Scenario:**
Two analysts run transactions on a `salary` table.

```
T1: BEGIN TRANSACTION
T1: SELECT SUM(salary) FROM employees; -- returns 200K

T2: BEGIN TRANSACTION
T2: UPDATE employees SET salary = salary + 5000 WHERE id=1;
T2: COMMIT

T1: SELECT SUM(salary) FROM employees; -- still returns 200K (unchanged)
T1: COMMIT
```

**Result:**
Even though `T2` committed an update, `T1` continues to see the consistent snapshot of data (as of its start). This prevents *read skew* and ensures **repeatable reads**.

---

## Unhandled Cases

Despite solving several read anomalies, **Repeatable Read** does not address all concurrency conflicts — particularly those involving **writes**.

### 1. Lost Updates

Occurs when two concurrent transactions perform read-modify-write operations on the same data.
Example:

```
T1: SELECT counter FROM metrics WHERE key='foo'; -- returns 10
T2: SELECT counter FROM metrics WHERE key='foo'; -- returns 10
T1: UPDATE metrics SET counter=11 WHERE key='foo';
T2: UPDATE metrics SET counter=11 WHERE key='foo';
```

**Result:** One update overwrites the other, losing a modification.

**Solutions:**

* **Atomic write operations:**

  ```sql
  UPDATE metrics SET counter = counter + 1 WHERE key = 'foo';
  ```
* **Explicit locking** before read-modify-write:

  ```sql
  SELECT * FROM metrics WHERE key='foo' FOR UPDATE;
  ```

---

### 2. Phantoms (Write Skew)

A **phantom read** occurs when one transaction’s write changes the result set of another’s query.
For example, in a hospital scheduling system:

```
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;
T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;
-- Both see 1 doctor on call.
T1: UPDATE doctors SET on_call=false WHERE id=1;
T2: UPDATE doctors SET on_call=false WHERE id=2;
-- Both commit -> 0 doctors on call.
```

**Result:** System invariant (at least one doctor on call) is violated.

**Solution:**
Use **materializing conflicts** — introduce a lock on a concrete set of rows or use higher isolation levels like **Serializable** to detect and abort conflicting transactions.

---

### 3. Write Skew in Derived Constraints

Even though transactions don’t update the same row, they indirectly violate constraints that depend on multiple rows (e.g., scheduling, inventory allocation).

**Mitigation:**
Convert logical constraints into explicit locking or enforce them through **check constraints** and **serializable isolation**.

---

# Serializable Isolation Level

## Explanation

**Serializable isolation** is the **strongest isolation level** in database systems.
It guarantees that even though transactions run concurrently, the **end result is equivalent to executing them one after another in a serial order**.

In other words, no set of concurrent transactions can produce an effect that could not occur if those same transactions were executed sequentially.

This level eliminates all read and write anomalies — including **dirty reads**, **non-repeatable reads**, **phantoms**, and **write skew** — ensuring full consistency.

---

## Implementations

There are three primary strategies for achieving serializability in modern databases, each with different trade-offs between **performance** and **isolation strength**.

---

### 1. Literal Serial Execution

**Approach:**
Transactions are executed one at a time in a single thread, guaranteeing serializability by design.

**Feasibility factors:**

* With modern **large memory systems**, most OLTP datasets can fit entirely in RAM.
* **OLTP transactions** are typically short and touch a small subset of records.
* Therefore, running them serially eliminates concurrency issues without noticeable slowdown.

**Examples:**

* Implemented by systems like **VoltDB**, **H-Store**, **Redis**, and **Datomic**, where transactions are short and deterministic.

**Result:**
Since only one transaction executes at a time, there are no conflicts, no locks, and no rollbacks due to concurrency — achieving perfect serializability naturally.

---

### 2. Two-Phase Locking (2PL)

**Approach:**
Two-phase locking enforces **strict lock acquisition and release rules** to ensure no conflicting transactions proceed simultaneously.

#### Mechanism:

* Each database object (row, record, etc.) is associated with a **lock**:

  * **Shared lock (S):** multiple transactions can hold it for reading.
  * **Exclusive lock (X):** only one transaction can hold it for writing.

#### Locking Rules:

1. To **read**, acquire a *shared* lock.

   * Multiple transactions can share this lock.
2. To **write**, acquire an *exclusive* lock.

   * Blocks both readers and writers until released.
3. Locks are held **until the transaction completes** (commit or abort).

   * Hence, “two phases”:

     * **Growing phase:** acquiring locks.
     * **Shrinking phase:** releasing all locks at the end.

#### Example:

```
T1: READ account_balance (shared lock)
T2: tries WRITE account_balance (exclusive lock) → must wait
T1: COMMIT → releases lock
T2: proceeds with WRITE
```

**Result:**
No conflicting reads or writes can occur simultaneously — transactions are effectively serialized by lock dependencies.

#### Limitations:

* **Reduced concurrency**: readers block writers and vice versa.
* **Deadlocks** may occur if transactions wait for each other’s locks.
* **Phantom Problem:** occurs when a transaction reads a range of rows that another transaction later inserts into.

**Solution:**
Use **predicate locks** or **index-range locks** to prevent phantom writes by locking the *range condition* rather than individual rows.

---

### 3. Serializable Snapshot Isolation (SSI)

**Approach:**
A modern, **optimistic concurrency control** technique combining **snapshot isolation** (MVCC) with serializability checks.

Unlike two-phase locking, SSI allows transactions to **proceed without blocking** and detects conflicts **only at commit time**.

#### Key Concepts:

* Transactions execute on **consistent snapshots** (from MVCC).
* At commit, the system checks for **serialization anomalies** such as:

  * **Stale reads:** a transaction read outdated data that was later modified.
  * **Write-read conflicts:** one transaction writes data that another previously read.

If such conflicts are found, **one of the transactions is aborted and retried**.

#### Example:

```
T1: Reads product inventory (50)
T2: Reads product inventory (50)
T1: Decreases inventory by 10 → writes 40
T2: Decreases inventory by 10 → writes 40
-- Conflict detected at commit time → one transaction aborted
```

**Result:**
Although transactions appear to execute concurrently, the system ensures only one final serializable order of execution is accepted.

#### Advantages:

* High concurrency like snapshot isolation.
* Guarantees full serializability without long lock waits.

#### Disadvantages:

* Transactions may be **aborted and retried** frequently under high contention.

#### Implementations:

* Found in **PostgreSQL (Serializable Snapshot Isolation mode)**, **FoundationDB**, and **CockroachDB**.

---


| Approach                                  | Concurrency | Blocking                     | Common In               | Notes                                             |
| ----------------------------------------- | ----------- | ---------------------------- | ----------------------- | ------------------------------------------------- |
| **Literal Serial Execution**              | None        | None                         | VoltDB, Redis           | Simplest, works for in-memory OLTP                |
| **Two-Phase Locking (2PL)**               | Low         | High (readers/writers block) | MySQL (InnoDB), Oracle  | Classical, robust but slower                      |
| **Serializable Snapshot Isolation (SSI)** | High        | None (optimistic)            | PostgreSQL, CockroachDB | Modern, best balance between safety & concurrency |

---

## Outcome

**Serializable isolation** is the gold standard for correctness, ensuring complete transactional integrity.
However, it comes with trade-offs — reduced throughput (in 2PL) or occasional rollbacks (in SSI).
Choosing the right implementation depends on your workload:

* **Short, high-speed OLTP:** literal serial execution.
* **Complex, high-contention workloads:** 2PL.
* **Mixed read/write workloads:** SSI for balance between speed and correctness.

---

# Comparison of Isolation Levels

| Isolation Level     | Prevents Dirty Reads | Prevents Non-Repeatable Reads | Prevents Phantoms / Write Skew | Concurrency              | Typical Use Case                                                       |
| ------------------- | -------------------- | ----------------------------- | ------------------------------ | ------------------------ | ---------------------------------------------------------------------- |
| **READ COMMITTED**  | Yes                    | **No**                             | **No**                              | High                     | Most OLTP systems where perfect repeatable reads are not required      |
| **REPEATABLE READ** | Yes                    | Yes                             | **No** (Write Skew possible)        | Medium                   | Systems needing repeatable reads, e.g., analytics or reporting         |
| **SERIALIZABLE**    | Yes                    | Yes                             | Yes                              | Low (2PL) / Medium (SSI) | Critical systems needing full consistency, e.g., banking, reservations |


# Conclusion

* **ACID properties** — Atomicity, Consistency, Isolation, Durability — are the foundation of reliable database systems.
* Isolation levels define how transactions interact and what anomalies are prevented, balancing **consistency vs. concurrency**.
* Modern databases face a trade-off between **scalability, replication, and partitioning** versus full ACID guarantees.
* **Application use case matters**: choose a datastore and isolation level based on the consistency requirements and performance needs.

  * For example:

    * High-throughput analytics may tolerate READ COMMITTED or REPEATABLE READ.
    * Financial or reservation systems require SERIALIZABLE isolation.

**Key Takeaway:**
Understanding ACID properties and isolation levels allows developers and architects to make **informed decisions**, ensuring data integrity without unnecessarily compromising performance or scalability.

---
# Reference
- [Designing-data-intensive-applications - Book by Martin Kleppmann](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/)

