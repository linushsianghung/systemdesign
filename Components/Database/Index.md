# Database Index

## Index
Reference:
- [PostgreSQL - Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [Use The Index, Luke!](https://use-the-index-luke.com/)
---

```text
Index is designed based on Query:
- Where clause: Specify Business Requirement
	- Equality (=): Basic Business Requirement
	- Range (<> or BETWEEN): Further Business Requirement or narrow down the scope for efficiency
- Order By clause:
	- The ordering of the Result Set

- Usually Order BY is based on the column which is related to the preferred ordering of Result Set, rather than business requirment.
- Index Ordering has to be  E - S - R, because if put Range index before Sort index, the optimiser has to sort the data first before return the Result Set

B+Tree Index  => Efficient Searching
1. Grouping
2. Ordering

Multi-Columns Index => 
1. Uniqueness
2. efficiency because of Covering 
```

## B-Tree & B+Tree
Reference:
- [索引結構演化論 B+樹](https://ithelp.ithome.com.tw/articles/10221111)
- [MySQL 的索引實現](https://ithelp.ithome.com.tw/articles/10221572)

### B+Tree high-level structure
```text
           [ 20 | 50 ]
           /    |    \
      [5|10] [20|30|40] [50|60|70]
        ↓        ↓         ↓
     leaf     leaf       leaf
     
Two node types:
- Non-leaf (internal) nodes
- Leaf nodes
```

#### Non-leaf (internal) nodes
- Key values (search keys)
- Pointers to child pages
- Navigation only
- Direct search to the correct leaf page

#### Leaf nodes (where the real data lives)
- In heap-based DBs (`PostgreSQL`, `Oracle`)
```text
Leaf node:
(key → TID)
```
- In clustered DBs (`InnoDB`, `SQL Server`)
```text
Clustered index leaf: (PK → full row)

Secondary index leaf: (secondary_key → PK)
```
- Leaf nodes form a linked list
```text
[ leaf1 ] ⇄ [ leaf2 ] ⇄ [ leaf3 ]

Range scan flow
- Find first leaf via tree traversal
- Walk leaf-to-leaf sequentially
```
This is what makes below sql fast:
```sql
SELECT * FROM users WHERE id >= 100 AND id < 200;
```

### Disk & page alignment (deep but useful)
- Each node ≈ one disk page (e.g. 8KB)
- Internal nodes packed with keys
- Leaf nodes packed with data/pointers

This is why:
- Tree height stays low
- Each lookup ≈ 3–4 I/Os

### Insert & split behavior
When a node fills:
- Node splits into two
- Middle key promoted to parent
- May cascade upward
> - Random keys (UUIDs 👀) → splits **everywhere**
> - Sequential keys → splits only at **right edge**

### Conclusion 
> A B+Tree index consists of internal nodes used only for navigation and leaf nodes that store either row pointers or actual row data. Internal nodes keep the tree shallow, while leaf nodes are linked to support efficient range scans. This design minimizes disk I/O and scales well for large datasets.
---

## Clustered vs Non-Clustered Indexes
> - A clustered index defines the **physical storage order** of table rows, while a non-clustered index is a separate structure that stores **index keys** and **pointers** to the actual rows.
> - In clustered tables, non-clustered indexes point to the clustered key. In heap tables, they point to a row identifier (RID).

### Clustered Index
- The table is the clustered index
- Leaf nodes contain actual row data
- Rows are stored in sorted order by the clustered key
> A clustered index is implemented as a B+ Tree whose leaf nodes store the actual table rows, so the table itself is physically organized as the clustered index.
```text
B+ Tree
  ├── internal nodes (primary key)
  └── leaf nodes (FULL ROW DATA)
```

### Non-Clustered Index
- A separate data structure (usually `B+Tree`)
- Leaf nodes store:
  - Index key
  - Pointer to the row (`TID` or `PK`)
```text
B+ Tree
  ├── internal nodes (keys)
  └── leaf nodes (key → pointer (block, offset))
                     ↓
                   row location (TID or PK)
                   
Heap file
┌────────────┐
│ Page 1     │ → rows (tuples)
│ Page 2     │ → rows
│ Page 3     │ → rows
└────────────┘
```
---

## Issue of UUID Primary Key
> - UUID primary keys are problematic because their randomness causes frequent page splits and poor locality in clustered indexes, leading to higher I/O, larger indexes, and worse cache efficiency.
> - UUIDs solve distributed uniqueness but hurt clustered index performance.
> - Use a sequential surrogate key for clustering and a UUID for external identity.

- **Sequential integer PK**
  - Always append to the **rightmost** leaf
  - Minimal page splits
  - Excellent **cache locality**
  - Very fast inserts
- **Random UUID PK**
  - Inserts land in **random** pages
  - Pages fill up unpredictably
  - Constant page splits

### Cache locality: the hidden killer
- Sequential PK
  - Inserts touch **the same hot page**
  - Page stays in memory
  - CPU cache-friendly
- UUID PK
  - Inserts touch **many different pages**
  - Cache thrashing
  - More disk reads

### Secondary indexes suffer
> In InnoDB, secondary indexes store the primary key. So if PK = UUID, every secondary index entry includes (secondary_key + UUID) which results in that UUID primary keys increase the size of all secondary indexes.

**Consequences**:
- Index size explodes
- Fewer entries per page
- More I/O per lookup
- Worse cache efficiency

### Write amplification effect
UUID PK causes:
- Page splits
- Extra index maintenance
- More redo / undo logging => **This is called write amplification**

### Conclusion
> UUID primary keys are problematic in clustered indexes because their randomness forces inserts into random positions in the B+ Tree, causing frequent page splits, poor cache locality, and larger secondary indexes. While UUIDs are great for distributed uniqueness, sequential or time-ordered keys perform much better for clustered storage.
---

## Index Ordering
> ORDER BY can use an index because B+Tree leaf nodes are stored in sorted order. If the ORDER BY columns match the index order, the database can perform an index scan and return rows already sorted, avoiding an explicit sort. This is especially efficient when combined with filtering predicates.

- An index scan can naturally produce ordered results.
```text
Internal nodes → navigation only
Leaf nodes     → sorted + linked

Leaf nodes look like this:
[10] → [20] → [30] → [40] → [50]
```
- The best indexes support both filtering and ordering
```sql
SELECT * FROM users WHERE country = 'US' ORDER BY age;

-- With index (country, age):
-- 1. Filter by country
-- 2. Already ordered by age
-- 3. Index-only ordering
```
---

## Hash Index
> Hash indexes are very fast for equality lookups, but they don’t support range queries, ordering, or efficient multi-column access. They also suffer from poor cache locality, concurrency issues, and higher operational risk. Since B+Tree indexes are flexible and fast enough for equality queries, most production systems standardize on B+Trees and use hash-based optimizations internally instead.

- No range queries
```text
Cannot support: <, >, BETWEEN, ORDER BY, LIKE 'a%'
```

- No ordering → no locality
```text
B+Tree:
[a] → [b] → [c]

Hash index:
bucket 3 → [a]
bucket 9 → [b]
bucket 1 → [c]

Consequences:
- Random I/O
- Poor cache behavior
```

- Concurrency & locking pain
```text
Hash buckets:
- Hot buckets
- High contention
- Coarse-grained locks

B+Trees:
- Fine-grained page locks
- Better concurrent access
```

### What does Poor Cache Behavior & Locality mean?
- In databases, cache usually means all of these layers:
  - CPU cache (L1 / L2 / L3)
  - RAM buffer pool (DB page cache)
  - OS page cache
  - Disk prefetch / read-ahead
  > Good cache behavior 👉 when accessing one piece of data makes it likely that nearby data is already in cache or will be prefetched. This is called **locality of reference**.

- Spatial & Temporal Locality
  - **Spatial**: If you access address X, you’ll soon access X+1, X+2, …. And databases love spatial locality.
  - **Temporal**: If you access address X, you’ll likely access X again soon.
> Hash indexes have poor spatial locality because entries are scattered across buckets, leading to random I/O and cache misses. B+Trees have excellent spatial locality because leaf nodes are stored in sorted order and linked, so accessing one entry often brings nearby entries into cache.

#### B+Tree
```sql
SELECT * FROM users WHERE id BETWEEN 1000 AND 1100;

-- Leaf nodes are:
-- - Sorted
-- - Stored in pages
-- - Linked sequentially
```

- Cache effects
  - One page read → many useful rows
  - `OS read-ahead` kicks in
  - Buffer pool keeps hot pages
  - `CPU cache lines` are reused

#### Hash index
```sql
SELECT * FROM users WHERE id IN (1000, 1001, 1002, ...);

-- hash(1000) → bucket 3 → page 57
-- hash(1001) → bucket 11 → page 204
-- hash(1002) → bucket 1 → page 89
```
- Cache effects
  - Touches a different memory page
  - Often a different disk block
  - No predictable access pattern
  - No sequential traversal

### Hash index Cache Efficiency
- Buffer Pool (DB Cache) => **Low cache hit ratio**
  - Pages are rarely reused
  - Cache fills with one-off pages
  - Hot pages get evicted quickly
- OS Page Cache & Read-Ahead
  - `Read-ahead`: You read page N, I’ll load N+1, N+2…
  - Next page is unrelated
  - Read-ahead is useless
- CPU Cache
  - CPU loads data in cache lines
  - With B+Tree
    - Nearby keys live in nearby memory
    - One cache line → multiple comparisons
  - With Hash Index
    - Each lookup touches unrelated memory
    - Cache lines are loaded then immediately discarded
    - This causes `Cache Thrashing`, `Pipeline Stalls` & `Lower IPC (Instructions Per Cycle)`

### Disk I/O Efficiency
- B+Tree:
  - Sequential leaf scans
  - Fewer seeks
- Hash index:
  - Random I/O per lookup
  - No batching

### Conclusion
> Ordering ⇒ locality. Because data is ordered:
> - Adjacent keys → adjacent storage
> - Adjacent storage → shared cache lines
> - Shared cache lines → fewer cache misses
> 
> Hash index explicitly breaks ordering, so it breaks locality at every cache level.
> 
> Hash indexes have poor cache behavior because they destroy locality. Each lookup jumps to an unrelated bucket and page, so pages are rarely reused. This prevents CPU cache reuse, defeats OS read-ahead, lowers buffer pool hit rate, and causes random I/O. In contrast, B+Tree leaf nodes are ordered and sequential, which makes them very cache-friendly.


## LSM Tree + SSTable
- [LSM Trees: the Go-To Data Structure for Databases, Search Engines, and More](https://medium.com/@dwivedi.ankit21/lsm-trees-the-go-to-data-structure-for-databases-search-engines-and-more-c3a48fa469d2)
- [Log Structured Merge Tree Definition](https://www.scylladb.com/glossary/log-structured-merge-tree/)
- Really fast to write, i.e. just write to memory layer first (memtable) but slow to read, i.e. have to read from multiple SSTable


- Why LSM trees trade cache locality for write throughput
- Why column stores love sequential scans

## Optimistic vs. Pessimistic locking
- **Pessimistic Locking**:
  - Lock data before modifying it
  - Other transactions must wait
  ```sql
  BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  COMMIT;
  ```

- **Optimistic Locking**:
  - No locks
  - Verify data hasn’t changed before commit
  ```sql
  -- Read current version
  SELECT balance, version FROM accounts WHERE id = 1;
  
  -- Attempt update with version check
  UPDATE accounts 
  SET balance = balance - 100, version = version + 1 
  WHERE id = 1 AND version = [read_version];
  
  -- Check if update succeeded (rows affected == 1)
  ```
> Pessimistic locking prevents conflicts by locking data before access, which guarantees consistency but reduces concurrency. Optimistic locking allows concurrent access and checks for conflicts at commit time, improving throughput but requiring retries when conflicts occur. The choice depends on contention patterns.
