# Performance Issue Investigation Runbook

## Step 0 — Confirm It’s Really the Database
Check:
- **App thread pool saturation**?
- **Downstream dependency**?
- **Network latency**?
---

## LAYER 1 — Scope & Impact
Is this isolated or systemic?

###  How many queries are slow?
- One endpoint?
- Many endpoints?
- Whole system?

If everything slow it's likely `system/config/hardware`. If one query slow, it's might be `SQL issue`

### How impactful is it?
- Use the [pg_stat_statements](#enable-pg_stat_statements-extension) to check `High Latency` / `High Frequency` / `High Total Cost` to get a **High Level View**
  ```sql
  -- 3s query × 1 call => might not urgent
  -- 50ms query × 10,000 calls => real bottleneck
  SELECT query, calls, total_exec_time, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
  ```
---

## LAYER 2 — System Health Check
Take a look at **Hardware/System Metrics first** (only for about _2 to 5 minutes_) to get a System Diagnostics and it will tell us where to look next.

#### If `High CPU Usage` => **Compute Saturation**
```shell
top / htop #Use these commands to monitor overall system CPU usage
mpstat -P ALL 1
```
- CPU usage consistently > `70 ~ 80%`
- High `us (user time)`
- High `sy (system time)`
- `Load average` > **number of CPU cores**

#### If `High I/O Wait` => **Disk Bottleneck.**
```shell
top / htop    # A high percentage in the `wa (I/O wait)` metric indicates that processes are waiting for disk I/O to complete.
iostat -x 1   # This tool provides detailed input/output statistics for devices, partitions, and network filesystems.
vmstat 1      # This command reports information about _processes_, _memory_, _paging_, _block I/O_, _traps_, and _CPU activity_. It also includes an `iowait` column.
```
- wa (I/O wait) > `10 ~ 20%`
- `Disk utilization` close to 100%
- High read/write latency
- _Cloud_: `Disk IOPS` limit reached

#### If `High Memory Usage` => **Memory Pressure**
```shell
top / htop    # Use these commands to monitor overall system memory usage
free -hm
vmstat 1
```
- Memory usage > `90%`
- `Swap` usage increasing
- `OOM` killer events

#### If `Active Sessions` close to `Max Connections` => **Connection Saturation**
```sql
SELECT count(*) FROM pg_stat_activity WHERE state='active';

SHOW max_connections;
```

#### Commands Reference:
- [The htop Command | Linux Essentials Tutorial](https://www.youtube.com/watch?v=bKWZJ3_5ODc)
- [System Monitoring with mpstat on Oracle Linux](https://www.youtube.com/watch?v=tVxE2C6Gwh0)
- [Linux Memory Monitoring: The free Command Explained for Beginners](https://www.youtube.com/watch?v=hSRNyDfjkkw)
- [Top 10 ways to monitor Linux in a Terminal](https://www.youtube.com/watch?v=4isEhE2rvmA)

### 3️⃣ Immediate Mitigation Actions
#### 🔥 Safe Immediate Actions
- Kill clearly runaway query
- Kill blocking session (if confirmed safe)
- Temporarily increase work_mem for specific session
- Enable statement logging temporarily

#### 🚨 Dangerous Actions (Require Senior Approval)
- Restart database
- Change shared_buffers
- Change checkpoint parameters
- Disable autovacuum

### 4️⃣ Root Cause Isolation
#### High CPU Usage might be because:
- _Expensive Queries_:
    - `Missing or Incorrect Index`: Full table scans or Sorting large result sets
    - `Large in-memory operations`: Hash Join / Large Sort in Memory
    - => **Adding / Fixing Index**
    - Wrong (Inner & Outer) Join Order or Large Join
    - => **`ANALYZE` / Rewriting Query (Filter Early)**
    - [Large In-Memory Sort Issue](https://chatgpt.com/share/69a13733-98f0-8000-827f-7b5189d8d34f)
    - [Hash Join vs Nested Loop Join](https://chatgpt.com/share/69a13759-eff8-8000-b293-c3c0210547ce)
- _Poor Query Plans_:
    - Outdated statistics or Bad cardinality estimates
    - => Using **ANALYZE** to update statistics
- _Too Many Concurrent Connection_:
    - Context switching overhead
    - Lock contention
    - [Too Many Connections Issue](https://chatgpt.com/share/69a0fe2a-b810-8000-a9ab-17761f3fccc2)
    - => **Reduce Connection Count**

#### High I/O Wait might be because:
- _Too many reads/writes_:
    - `Missing Index`: Full table scans
    - => **Adding Index**
    - `Checkpoint / WAL Pressure`: Heavy write load → WAL flush → disk saturated.
    - => **Tune checkpoint / WAL settings**
- _Buffer Pool Too Small_:
    - `Cache Miss`: Frequent disk reads
    - [Buffer Pool Tuning Issue](https://chatgpt.com/share/69a13cb1-3acc-8000-ba1c-ba4a26ed0bc8)
    - => [Adjusting Buffer Pool Size](#postgresql-buffer-pool-turning)
- _Disk Type Limitation_:
    - `HDD instead of SSD` => **Upgrade to SSD**
    - Low-tier cloud disk throttling (IOPS limit reached)
    - Network-attached storage latency

#### High Memory Usage might be because:
- _Buffer Pool Too Large_:
    - If `shared_buffers + (work_mem × concurrency) > physical RAM`: **swapping starts**.
- _work_mem Too Large_:
    - `work_mem` is per query **& per execution node**.
    - total work_mem usage = `active_connections × nodes × work_mem`
    - => Keep global work_mem moderate & increase per-session work_mem for heavy analytics
- _Too Many Concurrent Connections_:
    - `max_connections` controls total memory explosion
    - => Use **Connection Pooling** (e.g., `PgBouncer`)
- [Buffer Pool Tuning Issue](https://chatgpt.com/share/69a13cb1-3acc-8000-ba1c-ba4a26ed0bc8)
- => [Adjusting Buffer Pool Size](#postgresql-buffer-pool-turning)
---

##  LAYER 3 — Query-Level Diagnosis
### _Statistical Analysis_:
- Is this query slow because it’s expensive, or because it runs too often?
- Use the [pg_stat_statements](#enable-pg_stat_statements-extension) to check `High Latency` / `High Frequency` / `High Total Cost`
  ```sql
  -- 3s query × 1 call => might not urgent
  -- 50ms query × 10,000 calls => real bottleneck
  SELECT query, calls, total_exec_time, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
  ```

### _Wait Event Analysis_:
Identify **Lock Wait**
```sql
SELECT pid, state, wait_event_type, wait_event, query FROM pg_stat_activity WHERE state != 'idle';
```
| pid   | state               | wait_event_type | wait_event   | query                                       | Interpretation                                                            |       
|:------|:--------------------|:----------------|:-------------|:--------------------------------------------|:--------------------------------------------------------------------------|
| 12345 | active              | NULL            | NULL         | SELECT * FROM long_running_table WHERE...   | Running: This process is actively using the CPU                           |
| 12346 | idle in transaction | Client          | ClientRead   | INSERT INTO "ingress_message" ("create...   | Network: Postgres is waiting for the application to send the next command |
| 12347 | active              | Lock            | relation     | UPDATE table_a SET column_b = 'value' ...   | Blocked: Waiting for a table-level lock held by another process           |
| 12348 | active              | IO              | DataFileRead | SELECT column_c FROM table_b WHERE id = ... | Reading Disk: Waiting for data to be fetched from storage into memory     |
| 12349 | idle in transaction | Client          | ClientRead   | SELECT 1                                    |                                                                           |

>The **NULL** Value (The Good/Active State): If both columns are **NULL**, the process is currently "on the CPU." It isn't waiting for anything external.
>- **wait_event_type: IO**: The database is waiting for the hard drive.
>  - Common event: DataFileRead or BufferRead
>  - The Root Cause: The Working Data Set is larger than the Buffer Cache. Postgres has to go to the disk to find rows
>- **wait_event_type: Lock**: This is a _Software Bottleneck_.
>  - Common event: `relation` (waiting for a table) or `tuple` (waiting for a specific row)
>  - The Root Cause: Transaction A is updating a row, and Transaction B is trying to update the same row or change the table schema. They are stuck in a queue
>- **wait_event_type: LWLock (Lightweight Lock)**: These are internal Postgres locks used to protect shared memory.
>  - Common event: WALWriteLock or ProcArrayLock
>  - The Root Cause: Usually indicates you are pushing the database to its absolute limit of throughput (e.g., too many concurrent inserts or too many connections)

### _[IF RUNNING] Analyse Query Behaviour_
- Check Estimated Cost / Index Usage / Join Strategy
  ```shell
  EXPLAIN
  ```
- Check Actual Time / Actual Rows (Row Misestimation)
- To analyze the buffer cache usage (Cache Hits vs Disk Reads / Memory Spill) of a specific query
  ```shell
  EXPLAIN (ANALYZE, BUFFERS)
  ```
  ```text
  Index Scan using idx_orders_customer_id on orders  (actual time=0.046..0.048 rows=1 loops=1)
  Index Cond: (customer_id = 12345)
  # This line shows that 3 blocks were found in the shared buffer cache (shared hit) and 1 block had to be read from disk (read) for this query
  Buffers: shared hit=3 read=1
  Planning Time: 0.082 ms
  Execution Time: 0.066 ms
  ```
  
### _[IF BLOCKING] Lock Analysis_:
- Identify the blocking session and kill that
  ```sql
  SELECT pid, pg_blocking_pids(pid) FROM pg_stat_activity WHERE state = 'active';
  ```

  | pid   | pg_blocking_pids |
  |:------|:-----------------|
  | 12347 | {12353}          |

- Kill Blocker
  ```sql
  SELECT pg_terminate_backend(12353);
  ```

### _[IF I/O Heavy] Buffer Cache Analysis_:
```sql
-- hit_ratio > 95% => Healthy System
SELECT blks_read, blks_hit, round(blks_hit * 100.0 / (blks_hit + blks_read), 2) AS hit_ratio FROM pg_stat_database WHERE datname = current_database();

SELECT sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) AS cache_hit_ratio FROM pg_stat_database;
```

### _[PRODUCTION] Auto Explain_:
It automatically logs slow plans and very useful when:
- Slow query only appears under production load
- Hard to reproduce locally
```sql
LOAD 'auto_explain';
```

## LAYER 4 — Configuration & Concurrency
### Checkpoint Pressure?
- checkpoint_timeout
- max_wal_size
- synchronous_commit

### Autovacuum & Bloat?
```sql
SELECT relname, n_dead_tup FROM pg_stat_user_tables;
```
High dead tuples? → Vacuum problem.

## PostgreSQL Buffer Pool Turning
`Buffer pool = Database Page Cache in Shared Memory`. When a query needs data:
1. Check shared_buffers
2. If not there → read from disk
3. Store it in buffer pool

### Buffer Pool Too Small
- Frequently accessed data doesn’t fit so cache eviction happens often
- Cache hit ratio drops => More disk reads & I/O wait increases

Cache hit ratio:
```sql
SELECT sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) AS hit_ratio FROM pg_stat_database;

-- OLTP workload => aim > 95–99% 
-- Analytical workload => lower is acceptable
```

### Buffer Pool Too Large
> In PostgreSQL, there are TWO layers of caching:
> - Postgres `shared_buffers`
> - `Linux page cache` (OS cache)

- OS has very little memory so OS page cache shrinks
- Other memory consumers compete then system may start **swapping**

Check:
- Is swap used?
- Does vmstat show swap in/out?

### Best Practice
1. Ensure no swapping (first priority)
2. Then increase shared_buffers gradually
3. Monitor hit ratio & I/O
4. Stop when marginal benefit diminishes

- **Ideal Size**: For dedicated database servers, `25% ~ 30%` of total RAM for `shared_buffers` is common, but if experiencing high memory issues, stick closer to `20% ~ 25%` to leave room for the operating system and `work_mem`.
- Excessive `shared_buffers`: Setting this too high can starve the operating system of memory, leading to swapping and severe performance degradation.
- Alternative Solutions: If memory usage remains high, consider reducing `max_connections` or `work_mem`, as high concurrency can cause memory exhaustion even with a properly sized `shared_buffers`.

> Why not 80% like MySQL?<br/>
> Because Postgres **_relies heavily on OS page cache_**.
> In contrast:<br/>
> MySQL InnoDB buffer pool often uses **60–75% of RAM** (because MySQL relies less on OS cache)

### Configuration
> To adjust the PostgreSQL buffer pool (`shared_buffers`) due to high memory usage, reduce its value to **20%–25%** of total system RAM. Modify shared_buffers in `postgresql.conf` and restart the service. If issues persist, reduce `work_mem` or `max_connections` to free up system memory.
> - **Locate postgresql.conf**: usually in `/etc/postgresql/{version}/main/`
> - **Edit shared_buffers**:
> ```conf
> # Example for a 32GB RAM server:
> shared_buffers = 8GB  # Recommended is 25% of total RAM [2]
> ```

## Postgres Memory Architecture
Postgres memory is split into two major categories:
- **Shared Memory (Global)**: It's allocated once and shared across all connections when Postgres starts, which includes:
  - shared_buffers
  - WAL buffers
  - shared catalog cache
  - lock tables

- **Per Connection Memory**: It's allocated per process (Postgres is `process-per-connection`), which includes:
  - work_mem
  - local memory contexts
  - temp buffers

### shared_buffers
- It's Postgres’ internal page cache which acts as first-level cache before disk and stores `Table pages` & `Index pages`
- It's `Global (shared by all sessions)` and `Fixed size` with `LRU-like eviction`.
- Typical Sizing is` 20%–30% of RAM`: When Too Small or Large => [PostgreSQL Buffer Pool Turning](#postgresql-buffer-pool-turning)

### work_mem
work_mem is Per `Operation`, `Query`, & `Connection` and used for:
- Sort operations
- Hash joins
- Hash aggregates
- DISTINCT
- Materialization
```text
For example: work_mem = 64MB

If one query does 2 hash joins & 1 sort then that single query can use: 3 × 64MB = 192MB
Now imagine there are 50 concurrent queries and that’s potentially: 50 × 192MB = 9.6GB. This is how memory explosions happen.
```

#### work_mem Too Small
In execution plan, we see below messages which mean **Operation spilled to disk**
```text
- Sort Method: external merge Disk: 20480kB or 
- Hash Batches: 4
```

#### work_mem Too Large
- Memory pressure
- Swap
- System-wide slowdown

#### Best Practice
Instead of globally increasing, uses it for Per `Session` & `Heavy Query`
```sql
SET work_mem = '128MB';
```

### maintenance_work_mem
It's used for `Heavy maintenance operations` but does NOT affect `SELECT performance` directly, like:
- **CREATE INDEX**
- **VACUUM**
- **ALTER TABLE**
- **REINDEX**

#### maintenance_work_mem Too Small
- Index build slow
- Vacuum slow
- Autovacuum slow
- Table bloat increases

#### maintenance_work_mem Too Large
- During multiple index builds
- Multiple autovacuum workers
- Memory pressure possible

### Golden Rule for PostgreSQL Production-Grade Memory Budget Formula
> Total Memory Consumption ≈ `shared_buffers` + (`active_connections` × avg_work_mem_usage) + (`autovacuum_workers` × `maintenance_work_mem`) + OS overhead

- Never tune shared_buffers alone.
  - max_connections
  - work_mem
  - parallel workers
  - autovacuum settings

- For most production systems:
```text
shared_buffers = 25% RAM
effective_cache_size = 50–75% RAM
```

#### Example
- Keep `20–25%` for OS
- shared_buffers Start with `25%`
- Calculate Worst-Case work_mem

```text
Set:
- shared_buffers = 8GB
- work_mem = 16MB
- maintenance_work_mem = 1GB
- max_connections = 200

work_mem worst case:
200 connections × 2 operations × 16MB = 6.4GB
maintenance_work_mem:
1GB × autovacuum workers (say 3) = 3GB

Total potential => 8 + 6.4 + 3 = 17.4GB
Even when we plus below, it's still safe within 32GB.
- OS
- Page cache
- Background workers
```

#### Tuning Philosophy
1. Avoid swap at all cost.
2. Keep shared_buffers moderate.
3. Keep work_mem conservative globally.
4. Increase work_mem only for analytical workloads.
5. Monitor concurrency more than memory knobs.

## Postmortem
- [ ] Root cause identified
- [ ] Why monitoring didn’t prevent it?
- [ ] Preventive action defined
- [ ] Config or query change documented
- [ ] Load test performed if needed

## Reference
### Enable pg_stat_statements Extension
> - **Locate postgresql.conf**: usually in `/etc/postgresql/{version}/main/`
> - **Modify postgresql.conf**: Add `pg_stat_statements` to the `shared_preload_libraries` parameter.
> ```conf
> shared_preload_libraries = 'pg_stat_statements'
> ```
> - **Restart PostgreSQL**
> ```shell
> sudo systemctl restart postgresql
> ```
> - **Create the extension**:
> ```sql
> CREATE EXTENSION pg_stat_statements;
> ```


