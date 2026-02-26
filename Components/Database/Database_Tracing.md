# Database Debug Strategy

## 🛠️ Standard Debugging Process
A systematic approach helps minimize downtime and risk:
1. Identify & Monitor
   - Use logs, monitoring tools (e.g., Prometheus, Grafana), and query analyzers.
   - Check error messages, system metrics, and slow query logs.
2. Reproduce (if possible)
   - Try to replicate the issue in a staging environment.
   - Helps confirm root cause without impacting production.
3. Isolate
   - Narrow down whether the issue is query-related, configuration-related, or infrastructure-related.
   - Example: Is the slowdown due to a specific query or overall system load?
4. Analyze
   - Use profiling tools (e.g., EXPLAIN in SQL) to inspect query execution plans.
   - Check transaction logs for deadlocks or conflicts.
5. Resolve
   - Apply fixes: optimize queries, add indexes, adjust configurations, or scale resources.
   - For replication issues, resync replicas or adjust replication lag thresholds.
6. Validate
   - Test the fix in staging before applying to production.
   - Monitor closely after deployment to ensure stability.
7. Document & Prevent
   - Record the incident, root cause, and resolution steps.
   - Implement preventive measures (alerts, automated failover, query optimization guidelines).

## Root Causes Tracing
- In **standard industry order** of operations we should look at **Hardware/System Metrics first** in 90% of cases, but only for about _2 to 5 minutes_.
- Hardware saturation affects everything, while a bad query usually affects specific workloads. We must know the environment state before blaming SQL.

### Step 0 — Confirm It’s Really the Database
Check:
- **App thread pool saturation**?
- **Downstream dependency**?
- **Network latency**?

### Start with _The Quick Scan_: `System Level`
- **Why**: Because it takes seconds to check, and it tells you where to look next.
- Check these in order:
  1. **CPU**: Is it pegged at 100%? (Usually caused by `unoptimized queries`).
  2. **Disk I/O (IOPS)**: Is the disk saturated? (Usually caused by `swapping` or massive table scans).
  3. **Memory**: Is the database using all available RAM? (Check if the `Buffer Cache hit ratio` is dropping).
  4. **Connections**: Does system reach the limit?

#### Hardware Issues Inspection
- **High CPU Usage**: 
    > - Check CPU usage consistently > **70–80%**
    > - High `us (user time)`
    > - High `sy (system time)`
    > - **Load average > number of CPU cores**
    > 
    > Commands:
    > - `top / htop`: Use these commands to monitor overall system CPU usage
    > - `mpstat` -P ALL 1

    - _Expensive Queries_:
      - `Missing Index`: Full table scans or Sorting large result sets
      - `Wrong Order of Composite Index`: Sorting large result sets
      - `Large in-memory operations`: Hash Join / Large Sort in Memory
      - => **Adding / Fixing Index**
    - _Poor Query Plans_:
      - Outdated statistics or Bad cardinality estimates
      - Large Joins or Wrong Join Order
      - => Using **ANALYZE** to update statistics or **Rewriting Query**
    - _Too Many Queries_:
      - Context switching overhead
      - Lock contention
      - => **Reduce Connection Count**

    > To identify expensive or poor queries in PostgreSQL that are causing high CPU usage, use the [pg_stat_statements](#enable-pg_stat_statements-extension)
    > ```sql
    > -- The total_exec_time column shows the cumulative time spent executing the query, and calls indicates how frequently it runs. A query with a low mean_exec_time but a very high number of calls can also cause high CPU.
    > SELECT query, calls, total_exec_time, rows FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
    > 
    > -- Alternatively, use `pg_stat_activity` for real time monitoring
    > SELECT pid, query, usename, datname, state, now() - query_start AS duration FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;
    > SELECT pid, query, now() - xact_start AS txn_duration FROM pg_stat_activity WHERE state != 'idle' ORDER BY txn_duration DESC;
    > ```
    > 
    > High CPU usage due to excessive database connections usually shows a spike in active sessions and often coincides with high wait times (**locking**). To confirm, check if `active_connections` approaches `max_connections`, or if `active` tasks exceed the number of CPU cores.
    > - **Check Active vs. Total Connections**: `SELECT count(*), state FROM pg_stat_activity GROUP BY state;` or `SELECT * FROM pg_stat_activity WHERE state = ‘active’;`
    > - **Compare to Max Connections**: `SHOW max_connections;`

- **High I/O Wait**:
    > - wa (I/O wait) > **10–20%** 
    > - Disk utilization close to 100%
    > - High read/write latency
    > - Cloud: Disk IOPS limit reached
    >
    > Commands:
    > - `top / htop`: A high percentage in the `iowait (I/O wait)` metric indicates that processes are waiting for disk I/O to complete.
    > - `vmstat` 1: This command reports information about _processes_, _memory_, _paging_, _block I/O_, _traps_, and _CPU activity_. It also includes an `iowait` column.
    > - `iostat` -x 1: This tool provides detailed input/output statistics for devices, partitions, and network filesystems.

    - _Too many reads/writes_:
      - `Missing Index`: Full table scans
      - => **Adding Index**
      - `Checkpoint / WAL Pressure`: Heavy write load → WAL flush → disk saturated.
      - => **Tune checkpoint / WAL settings**
    - _Buffer Pool Too Small_: 
      - `Cache Miss`: Frequent disk reads
      - => [Adjusting Buffer Pool Size](#postgresql-buffer-pool-turning) 
    - _Disk Type Limitation_:
      - `HDD instead of SSD` => **Upgrade to SSD**
      - Low-tier cloud disk throttling (IOPS limit reached)
      - Network-attached storage latency

    > PostgreSQL provides statistics that help identify which database activities or queries are causing the high I/O.
    > - [pg_stat_statements](#enable-pg_stat_statements-extension) Extension: This is the best tool to identify I/O-intensive queries.
    >   ```sql
    >   SELECT query, total_time, rows, shared_blks_read, shared_blks_hit FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;
    >   ```
    > - **EXPLAIN (ANALYZE, BUFFERS)**: Use this command on problematic queries to see how much data is read from disk (`shared blks read`) versus found in cache (`shared blks hit)`). A high ratio of `read` to `hit` indicates an I/O issue, potentially fixable with indexing or more RAM.
    > - **pg_stat_activity** View: Check the `wait_event` column. If many sessions are waiting on I/O-related events, it's a strong indicator of an I/O bottleneck.
    > - **pg_stat_bgwriter** View: This helps monitor checkpoint activity. Frequent checkpoints can cause high I/O. Check your logs for `checkpoints are occurring too frequently` messages.
    > - pg_statio_* Views: These views (e.g., `pg_statio_user_tables`) show block-level I/O statistics for tables and indexes, helping you pinpoint which relations are causing the most physical disk reads.

- **High Memory Usage**:
    > Memory usage > **90%**
    > - Swap usage increasing
    > - OOM killer events
    > 
    > Commands:
    > - `vmstat` 1
    > - `free` -m

    - _Buffer Pool Too Large_:
      - If DB memory allocation > physical RAM: **swapping starts**.
      - => [Adjusting Buffer Pool Size](#postgresql-buffer-pool-turning)
    - _Large work_mem / sort_buffer_size_
      - Hash Join / Large Sort
    - _Too Many Concurrent Connections_:
      - => Reduce Max Connections
    - _Memory Leaks_ (rare but possible): Application not releasing connections

#### Enable pg_stat_statements Extension
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

#### PostgreSQL Buffer Pool Turning
> To adjust the PostgreSQL buffer pool (`shared_buffers`) due to high memory usage, reduce its value to **20%–25%** of total system RAM. Modify shared_buffers in `postgresql.conf` and restart the service. If issues persist, reduce `work_mem` or `max_connections` to free up system memory.
> - **Locate postgresql.conf**: usually in `/etc/postgresql/{version}/main/`
> - **Edit shared_buffers**:
> ```conf
> # Example for a 32GB RAM server:
> shared_buffers = 8GB  # Recommended is 25% of total RAM [2]
> ```
> Recommendations & Best Practice:
> - **Ideal Size**: For dedicated database servers, `25% ~ 40%` of total RAM for `shared_buffers` is common, but if experiencing high memory issues, stick closer to `20% ~ 25%` to leave room for the operating system and `work_mem`.
> - Excessive `shared_buffers`: Setting this too high can starve the operating system of memory, leading to swapping and severe performance degradation.
> - Alternative Solutions: If memory usage remains high, consider reducing `max_connections` or `work_mem`, as high concurrency can cause memory exhaustion even with a properly sized `shared_buffers`.

### Deep Dive into _The Culprit_: `Query Level`
- **Why**: Once the system metrics confirm there is a "fire," the queries are almost always the "matches" that started it.
- Move to Query Analysis if:
  - `CPU or Disk I/O is high`.
  - The system metrics look normal, but specific API calls are slow.

#### Slow Query Analysis Process
- _Statistical Analysis_ (also covered in [Hardware Issues Inspection](#hardware-issues-inspection)): Is this query slow because it’s expensive, or because it runs too often?
  - Check `High Latency` / `High Frequency` / `High Total Cost`
  ```sql
  -- 3s query × 1 call => might not urgent
  -- 50ms query × 10,000 calls => real bottleneck
  SELECT query, calls, total_exec_time, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
  ```
- _Wait Event Analysis_: Sometimes plan is fine but query still slow...
  - Check if waiting on `Lock` / `IO` / `LWLock`: The problem might be concurrency or resource contention. EXPLAIN won’t show this.
  ```sql
  SELECT pid, wait_event_type, wait_event, query, now() - query_start AS duration FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;
  ```
  | pid  | wait_event_type | wait_event   | Interpretation                                                                      |
  |:-----|:----------------|:-------------|:------------------------------------------------------------------------------------|
  | 1204 | NULL            | NULL         | unning: This process is actively using the CPU                                      |
  | 1205 | IO              | DataFileRead | Reading Disk: Waiting for data to be fetched from storage into memory               |
  | 1206 | Lock            | relation     | Blocked: Waiting for a table-level lock held by another process                     |
  | 1207 | LWLock          | WALWriteLock | Writing Logs: Waiting to write to the Write-Ahead Log (common during heavy inserts) |
  | 1208 | Client          | ClientRead   | Network: Postgres is waiting for the application to send the next command           |

  > - The **NULL** Value (The Good/Active State): If both columns are **NULL**, the process is currently "on the CPU." It isn't waiting for anything external.
  > - **wait_event_type: IO**: The database is waiting for the hard drive.
  >   - Common event: DataFileRead or BufferRead
  >   - The Root Cause: The Working Data Set is larger than the Buffer Cache. Postgres has to go to the disk to find rows
  > - **wait_event_type: Lock**: This is a _Software Bottleneck_.
  >   - Common event: `relation` (waiting for a table) or `tuple` (waiting for a specific row)
  >   - The Root Cause: Transaction A is updating a row, and Transaction B is trying to update the same row or change the table schema. They are stuck in a queue
  > - **wait_event_type: LWLock (Lightweight Lock)**: These are internal Postgres locks used to protect shared memory.
  >   - Common event: WALWriteLock or ProcArrayLock
  >   - The Root Cause: Usually indicates you are pushing the database to its absolute limit of throughput (e.g., too many concurrent inserts or too many connections)
- [Running] _EXPLAIN (Without ANALYZE)_:
    - Check `Estimated Cost` / `Index Usage` / `Join Strategy`
    - If the plan already looks terrible, we know the direction
- [Blocked] _Lock Analysis_: Sometimes plan is fine but query still slow...
  ```sql
  -- Method 1: A function (available since PostgreSQL 9.6) that identifies sessions that are waiting for a lock and listf the PIDs of the sessions holding those locks.
  SELECT pid, usename, query AS blocked_query, pg_blocking_pids(pid) AS blocking_pid FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
  
  -- Method 2: Contains information on all active locks, indicating which are granted (held) and which are waiting.
  SELECT pid, locktype, mode, granted FROM pg_locks WHERE NOT granted; -- Rows where granted is false represent sessions currently blocked.
  
  SELECT pid, mode, granted, pg_blocking_pids(pid) AS blocked_by FROM pg_locks WHERE relation = 'your_table_name'::regclass; -- See both who has the lock (granted = true) and who is waiting for it.
  ```
- [IO Heavy] _Buffer / Cache Level Diagnosis_:
  - `pg_stat_database` (Database-level efficiency): The pg_stat_database view provides **system-wide** statistics, including the total number of blocks read from disk (`blks_read`) and the number of blocks found in the shared buffer cache (`blks_hit`). A cache hit ratio of over 90% is generally considered good, but values near 100% indicate most data is served from memory.
    ```sql
    SELECT blks_read, blks_hit, round(blks_hit * 100.0 / (blks_hit + blks_read), 2) AS hit_ratio FROM pg_stat_database WHERE datname = current_database();
    ```
  - ***(WARNING)*** `EXPLAIN (ANALYZE, BUFFERS)` (Query-level analysis): To analyze the buffer cache usage of a specific query, adding the `BUFFERS` option to the EXPLAIN ANALYZE command.
    ```text
    Index Scan using idx_orders_customer_id on orders  (actual time=0.046..0.048 rows=1 loops=1)
    Index Cond: (customer_id = 12345)
    # This line shows that 3 blocks were found in the shared buffer cache (shared hit) and 1 block had to be read from disk (read) for this query
    Buffers: shared hit=3 read=1
    Planning Time: 0.082 ms
    Execution Time: 0.066 ms
    ```
- ***(WARNING)*** _EXPLAIN (ANALYZE, BUFFERS)_:
    - Check `Actual Time` / `Actual Rows` / `Row Misestimation` / `Disk Reads vs Cache Hits` / `Memory Spill`

### Review _The Environment_: `Configuration`
- **Why**: This is usually a _set it and forget it_ area. You only look here if the hardware is powerful and the queries look okay, but the database still feels choked.
- Check Configuration if:
  - Recently upgraded the database version.
  - Migrated to a new server but kept the old config files.
  - See **Out of Memory (OOM)** errors despite having free RAM.

### Correlation
| Symptom                  | Likely Root Cause                              |
|:-------------------------|:-----------------------------------------------|
| High CPU + low I/O       | Bad query or missing index                     |
| High CPU + high I/O      | Large scan or massive write                    |
| High CPU sys%            | Too many context switches                      |
| Low CPU but high latency | Lock contention                                |
| High I/O wait            | Disk bottleneck                                |
| High Memory + Swap       | Too many connections / Memory misconfiguration |

### 📝 Postmortem Template
After resolving an incident, use this template to document the findings and prevent recurrence:

1. **Incident Summary**
   - What happened? (e.g., "High latency on User Profile API")
   - Impact: (e.g., "500 errors for 5% of users for 10 minutes")
2. **Timeline**
   - `[Time]` Detection -> `[Time]` Investigation -> `[Time]` Fix -> `[Time]` Verification.
3. **Root Cause Analysis (5 Whys)**
   - Why was it slow? -> "Full table scan."
   - Why full table scan? -> "Index was not used."
   - Why index not used? -> "Query used `SELECT *` preventing index-only scan."
4. **Resolution**
   - Immediate fix (e.g., "Added index" or "Modified query").
5. **Action Items**
   - Preventive measures (e.g., "Add linter rule for `SELECT *`", "Add alert for slow query log").

### Issues:
- **Avoid SELECT \*, even on a single-column tables**
Try avoiding SELECT * even on single-column tables. Just keep that in mind even if you disagree. By the end of this article, I might have you contemplate.

- A story from 2012
A backend API that is stable and runs in single digit millisecond. One day users came in to a slow and sluggish user experience. We checked the commits and nothing was obvious, most changes were benign. Just in case we reverted all commits.However, the app was still slow. Looking at the diagnostics, we noticed API response time is taking from 500 ms to up to 2 seconds at times. Where it used to be single-digit millisecond.We know nothing has changed in the backend that would’ve cause the slow down, but started looking at the database queries.

SELECT * on a table that has 3 blob fields are being returned to the backend app, those blob fields has very large documents. It turned out this table had only 2 integer columns, and the API was running a SELECT * to return and use the two fields. But later, the admin added 3 blob fields that are used and populated by another application. While those blob fields were not being returned to the client, the backend API took the hit pulling the extra fields populated by other applications, causing database, network and protocol serialization overhead.

- How database reads work
In a row-store database engine, rows are stored in units called pages. Each page has a fixed header and contains multiple rows, with each row having a record header followed by its respective columns. For instance, consider the following example in PostgreSQL:

When the database fetches a page and places it in the shared buffer pool, we gain access to all rows and columns within that page. So, the question arises: if we have all the columns readily available in memory, why would SELECT * be slow and costly? Is it really as slow as people claim it to be? And if so why is it so? In this post, we will explore these questions and more.

- Kiss Index-Only Scans Goodbye
Using SELECT * means that the database optimizer cannot choose index-only scans. For example, let’s say you need the IDs of students who scored above 90, and you have an index on the grades column that includes the student ID as a non-key, this index is perfect for this query.
  
However, since you asked for all fields, the database needs to access the heap data page to get the remaining fields increasing random reads resulting in far more I/Os. In contrast, the database could have only scanned the grades index and returned the IDs if you hadn’t used SELECT *.

- Deserialization Cost
Deserialization, or decoding, is the process of converting raw bytes into data types. This involves taking a sequence of bytes (typically from a file, network communication, or another source) and converting it back into a more structured data format, such as objects or variables in a programming language.

When you perform a SELECT * query, the database needs to deserialize all columns, even those you may not need for your specific use case. This can increase the computational overhead and slow down query performance. By only selecting the necessary columns, you can reduce the deserialization cost and improve the efficiency of your queries.

- Not All Columns Are Inline
One significant issue with SELECT * queries is that not all columns are stored inline within the page. Large columns, such as text or blobs, may be stored in external tables and only retrieved when requested (Postgres TOAST tables are example). These columns are often compressed, so when you perform a SELECT * query with many text fields, geometry data, or blobs, you place an additional load on the database to fetch the values from external tables, decompress them, and return the results to the client.
  
- Network Cost
Before the query result is sent to the client, it must be serialized according to the communication protocol supported by the database. The more data needs to be serialized, the more work is required from the CPU. After the bytes are serialized, they are transmitted through TCP/IP. The more segments you need to send, the higher the cost of transmission, which ultimately affects network latency.

Returning all columns may require deserialization of large columns, such as strings or blobs, that clients may never use.

- Client Deserialization
Once the client receives the raw bytes, the client app must deserialize the data to whatever language the client uses, adding to the overall processing time. The more data is in the pipe the slower this process.
  
- Unpredictability
Using SELECT * on the client side even if you have a single field can introduce unpredictability. Think of this example, you have a table with one or two fields and your app does a SELECT * , blazing fast two integer fields.

However, later the admin decided to add an XML field, JSON, blob and other fields that are populated and used by other apps. While your code did not change at all, it will suddenly slow down because it is now picking up all the extra fields that your app didn’t need to begin with.

- Code Grep
Another advantage of explicit SELECT is you can grep the codebase for in columns that are in use columns, so in case if you want to rename or drop a column. This makes database schema DDL changes more approachable.
  
- Summary
In conclusion, a SELECT * query involves many complex processes, so it’s best to only select the fields you need to avoid unnecessary overhead. Keep in mind that if your table has few columns with simple data types, the overhead of a SELECT * query might be negligible. However, it’s generally good practice to be selective about the columns you retrieve in your queries.