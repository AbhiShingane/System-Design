# How Indexing Makes Databases Faster

##  How Data Is Stored in a Relational Database

In relational databases, data is organized into **tables**, and each table stores data as individual **records** (rows). Consider a `users` table with the following structure:

| Column    | Size  |
|-----------|-------|
| `id`      | 4 B   |
| `name`    | 20 B  |
| `address` | 60 B  |
| `phone`   | 8 B   |
| `age`     | 8 B   |
| **Total** | **100 B per record** |

---

## Disk Block Allocation

Databases read data from disk in fixed-size chunks called **blocks**. Assume:

- **Block size** = 600 B
- **Total records** = 100

**Records per block:**
```
600 B ÷ 100 B = 6 records/block
```

**Total blocks needed:**
```
⌈100 ÷ 6⌉ = ⌈16.67⌉ = 17 blocks
```

So the entire `users` table spans **17 disk blocks**.

---

## Query Without an Index

Now consider running this query:

```sql
SELECT * FROM users WHERE age = 23;
```

Without an index, the database performs a **full table scan** — it reads every block one by one, checks each record against the condition, and collects the matching rows.

| Metric               | Value                  |
|----------------------|------------------------|
| Blocks to read       | 17                     |
| Time per block read  | ~1 sec                 |
| **Total time**       | **~17 sec** (worst case ~100 sec if scattered) |

> Every query on an un-indexed table costs a full scan, regardless of how many rows actually match.

---

## 🔍 What Is an Index?

An **index** is a small, separate lookup table that maps a column's values to the physical row locations in the main table. It is typically built on a specific column (e.g., the primary key or any frequently queried column).

### Structure of an Index Table

| Index Value (`age`) | Row ID  |
|---------------------|---------|
| 19                  | Row #5  |
| 23                  | Row #1  |
| 31                  | Row #2  |
| ...                 | ...     |

Each index entry holds only **2 fields**: the indexed value and the corresponding row ID.

- **Entry size** = 8 B (4 B for index value + 4 B for row ID)
- **Total entries** = 100

---

## Query With an Index

**Index table block allocation:**

```
Entries per block  = 600 B ÷ 8 B = 75 entries/block
Total blocks       = ⌈100 ÷ 75⌉ = ⌈1.33⌉ = 2 blocks
```

Now re-run the same query:

```sql
SELECT * FROM users WHERE age = 23;
```

**Execution steps:**

1. Scan the **index table** (2 blocks) → find `age = 23` → get `Row #1`
2. Fetch **only** the matching block from the main table (~2 blocks)

| Step               | Blocks Read |
|--------------------|-------------|
| Index scan         | 2           |
| Main table fetch   | ~2          |
| **Total**          | **~4 blocks** |

**Total time: ~4 seconds** — compared to ~100 seconds without an index.

---

## 📊 Performance Comparison

```
Without Index:  ████████████████████████████████  ~100 sec
With Index:     ████  ~4 sec
```

| Scenario       | Blocks Read | Time     |
|----------------|-------------|----------|
| No index       | 17+         | ~100 sec |
| With index     | ~4          | ~4 sec   |
| **Improvement**|             | **~25×** |

---

## Key Takeaways

- An index is a **compact reference table** — much smaller than the original data.
- Instead of scanning every record, the database jumps directly to matching rows.
- Indexes dramatically reduce **I/O operations**, which are the primary bottleneck in database performance.
- Indexes are most beneficial on columns used frequently in `WHERE`, `JOIN`, or `ORDER BY` clauses.

> **Trade-off:** Indexes speed up reads but add overhead to writes (`INSERT`, `UPDATE`, `DELETE`), since the index must also be updated. Use them thoughtfully.

---

##  Real-World Example

Suppose you have an e-commerce database with **10 million orders** and you often query:

```sql
SELECT * FROM orders WHERE customer_id = 4521;
```

- **Without index:** The DB scans all 10M rows — potentially several seconds per query.
- **With index on `customer_id`:** The DB looks up the index, finds relevant row IDs instantly, and fetches only those records — typically **milliseconds**.
