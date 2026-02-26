# Database

## Reference
- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
  - [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
  - [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
  - [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [UUIDs are Bad for Performance in MySQL - Is Postgres better? Let us Discuss](https://www.youtube.com/watch?v=Y5mWz4vK10A)
- [MySQL UUIDs – Bad For Performance](https://www.percona.com/blog/uuids-are-popular-but-bad-for-performance-lets-discuss/)
- [Choosing a Database for Systems Design](https://www.youtube.com/watch?v=6GebEqt6Ynk)

## Transaction Isolation
Reference:
- [PostgreSQL - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Database Concurrency Phenomena & ISOLATION Label: Read Phenomena and Serialization Anomaly](https://dev.to/jad_core/database-concurrency-phenomena-isolation-label-read-phenomena-and-serialization-anomaly-3485)
- [PostgreSQL Isolation Levels and Locking Summary](https://dev.to/markadel/postgresql-isolation-levels-and-locking-summary-9ac)
- [Understanding Lost Updates in PostgreSQL](https://umartahir93.medium.com/understanding-lost-updates-in-postgresql-cf76c7570bfa)
- [Transaction Isolation Levels Are A Confusing Mess](https://pavan-kalyan.dev/posts/transaction-isolation-levels-are-a-confusing-mess)
- [Deeply understand Isolation levels and Read phenomena in MySQL & PostgreSQL](https://dev.to/techschoolguru/understand-isolation-levels-read-phenomena-in-mysql-postgres-c2e)

### Read Phenomena
- **Dirty Reads**: A transaction reads data **that has been modified by another transaction but not yet committed**. If the modifying transaction rolls back, the read data becomes invalid.
  ![Dirty Reads](../pics/Dirty_Read.png)

- **Non-repeatable Reads**: A transaction reads the same data twice and **gets different results** because another transaction modified and committed the data between the reads.
  ![Non-repeatable Reads](../pics/Non-repeatable_Read.png)

- **Phantom Reads**: A transaction re-executes a query to retrieve a set of rows and **finds different rows (new or missing)** because another transaction inserted or deleted rows and committed.
  ![Phantom Reads](../pics/Phantom_Read.png)

- **Serialization Anomaly**: The outcome of a group of concurrent transactions is inconsistent with all possible sequential orderings of those transactions.
  - **Lost Update**: It occurs when two transactions read the same data and then both try to update it, but the second transaction to commit overwrites the changes made by the first, without acknowledging them.

### Isolation Level
Isolation is one of the four property of a database transaction, where at its highest level, a perfect isolation ensures that all concurrent transactions will not affect each other.

- **Read Uncommitted**: The least strict isolation level, allowing transactions to read uncommitted changes made by other transactions.
- **Read Committed**: Ensures that transactions only read data that has been committed, preventing dirty reads but allowing other concurrency issues.
- **Repeatable Read**: Ensures that data read by a transaction remains consistent throughout the transaction, preventing non-repeatable reads but not phantom reads.
- **Serializable**: The strictest isolation level, ensuring that transactions execute as if they were run sequentially, preventing all concurrency anomalies.

### SQL Isolation Levels vs. Read Phenomena
| Isolation Level  | Dirty Read              | Non-Repeatable Read | Phantom Read            | Serialization Anomaly |
|:-----------------|:------------------------|:--------------------|:------------------------|:----------------------|
| Read Uncommitted | Possible, but not in PG | Possible            | Possible                | Possible              |
| Read Committed   | Prevented               | Possible            | Possible                | Possible              |
| Repeatable Read  | Prevented               | Prevented           | Possible, but not in PG | Possible              |
| Serializable     | Prevented               | Prevented           | Prevented               | Prevented             |

### PostgreSQL Implementation
- In PostgreSQL, you can request any of the four standard transaction isolation levels, but internally **only three distinct isolation levels are implemented**, i.e., PostgreSQL's Read Uncommitted mode behaves like Read Committed. This is because it is the only sensible way to map the standard isolation levels to PostgreSQL's `multiversion concurrency control` architecture.
- The table also shows that PostgreSQL's Repeatable Read implementation does not allow phantom reads. This is acceptable under the SQL standard because the standard **specifies which anomalies must not occur at certain isolation levels**; higher guarantees are acceptable.
---

## Explain Analyze
- [PostgreSQL - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Postgres Explain Explained - How Databases Prepare Optimal Query Plans to Execute SQL](https://www.youtube.com/watch?v=P7EUFtjeAmI)
The `EXPLAIN ANALYZE` command in PostgreSQL executes the query and provides actual runtime statistics for each operation (node) in the execution plan. The actual time is presented as a range in _milliseconds_, and the output includes summary statistics for the entire query execution.

### Understanding the actual time Output
For each node in the query plan, the output includes a line that looks like: (**actual time=**`<time1>`..`<time2>` **rows**=`<R>` **loops**=`<L>`).
- `time1 (Actual Startup Time)`: The actual time (in _milliseconds_) from the **start of the statement's execution** until this specific plan node **returned its first row**.
- `time2 (Actual Total Time)`: The actual total elapsed time (in _milliseconds_) from the **start of the statement's execution** until this plan node finished **returning all rows**.
- `rows`: The total number of rows actually returned by this node per loop.
- `loops`: The number of times this plan node was executed. This is common in operations like nested loop joins, where an inner loop is executed once for each row from the outer loop.

### Key Points for Interpretation
- `Units`: All actual time values are in _milliseconds_ of real time.
- `Total Time per Node`: To get the total time actually spent in a single-threaded node that runs multiple times (e.g., the inner side of a nested loop join), you must **multiply the time2 value by the loops value**. For parallel queries, the times shown are _averages_ per worker process, and multiplication is not necessary to find the wall-clock time.
- `Execution vs. Planning Time`: The output also provides separate summary times:
  - **Planning Time**: how long the planner took to devise the plan
  - **Execution Time**: the total time the query took to execute
- `Identifying Bottlenecks`: Comparing the estimated rows (from the cost section) with the actual rows is crucial. Large discrepancies can indicate outdated table statistics, which might lead the query planner to choose an inefficient plan. Focus on `nodes with high actual time` as these are the potential performance bottlenecks.
- `Client vs. Server Time`: The Execution Time in `EXPLAIN ANALYZE` measures the time spent on the server side within the PostgreSQL process. It does not include network latency or the time it takes for the client application to fetch and display the results, which can sometimes lead to different timings compared to a client's own execution timer (e.g., psql's `\timing`).
---

## SQL vs. No-SQL Database
> SQL databases are **relational, schema-based** systems that use structured tables and SQL, and they prioritize **ACID transactions** and **strong consistency**.<br><br>
NoSQL databases are non-relational systems designed for **flexible schemas**, **horizontal scalability**, and **high availability**, often trading strict consistency for performance and scalability.

- **Relational Database Management System (*RDBMS*)**:
  - Relational databases represent and store data in **normalised** tables and rows that one can perform join operations using SQL across different database tables.
  - Choose SQL database when:
    - Correctness (**ACID**) is the most important factor of the application
    - There are a lot of relations between entities
  > SQL databases are best when data **integrity** and **complex relationships** matter.
- **Non-Relational Databases (*NoSQL*)**:
  - These databases are grouped into four categories: `Key-Value`, `Document`, `Column`, and, `Graph` stores.
  - Choose NoSQL database when:
    - Your application requires **super-low latency**. 
    - Your data are unstructured, or you do not have any relational data. 
    - You only need to **serialize** and **deserialize data** (JSON, XML, YAML, etc.).
    - You need to store a **massive amount of data**.
  > NoSQL databases are optimized for **scalability** and **availability** in distributed environments. Choosing NoSql database when the data model is evolving, the system needs massive horizontal scalability, or ultra-low latency is more important than strict consistency.

| Aspect             | SQL (Relational DB)          | NoSQL (Non-Relational DB)          |
|:-------------------|:-----------------------------|:-----------------------------------|
| Data model         | Tables (rows & columns)      | Key-value, Document, Column, Graph |
| Schema             | Fixed, Predefined            | Flexible / Schema-Less             |
| Query language     | SQL                          | DB-Specific APIs / Query Languages |
| Transactions       | Strong ACID support          | Often BASE / Eventual Consistency  |
| Consistency model  | Strong Consistency           | Eventual / Tunable consistency     |
| Use cases          | Financial, ERP, Core Systems | Big Data, Caching, Real-Time Apps  |


### NoSQL Choices
#### MongoDB
- **Document** Data Model
- **B-Tree** index & **Transaction** supported
> Rarely make sense to use in System Design Interview because nothing is `special` about this.
> But it's good for requirement of SQL like guarantees on data with more flexibilities.

#### Apache Cassandra
- **Wide Column** Data Model
- **Peer to Peer** architecture: can write to any node so good for write
- **LSM Tree** index: Really fast write
> Great for applications with write high volume and consistency is not the first priority.
> e.g. Chat Application

- Implementation
  - [DS201.17 Write Path | Foundations of Apache Cassandra](https://www.youtube.com/watch?v=mDd4I-isodE)
  - [DS201.18 Read Path | Foundations of Apache Cassandra](https://www.youtube.com/watch?v=x6g0sUi-5tw)
  - [DS201.19 Compaction | Foundations of Apache Cassandra](https://www.youtube.com/watch?v=69sHSF0iUqg)

#### Apache HBase
- **Wide Column** Data Model
- **Single Leader** replication
- **LSM Tree** index: Really fast write
- Column Oriented Storage
> Great for applications that needs fast column read
---

## Database Internals
### Hash Tables Chaining vs.Probing (Open-Addressed)
- [Hash table](https://en.wikipedia.org/wiki/Hash_table)
- [Why do we use linear probing in hash tables when there is separate chaining linked with lists?](https://stackoverflow.com/questions/23821764/why-do-we-use-linear-probing-in-hash-tables-when-there-is-separate-chaining-link)

> Chaining and open-addressing (a simple implementation of which is based on linear-probing) are used in Hashtables to resolve collisions. A collision happens whenever the hash function for two different keys points to the same location to store the value. 
> In order to store both values, with different keys that would have been stored in the same location, chaining and open-addressing take different approaches: while chaining resolves the conflict by created a linked list of values with the same hash; 
> open-addressing tries to attempt to find a different location to store the values with the same hash. The main difference that arises is in the speed of retrieving the value being hashed under different conditions. 
> 
> Let's start with [chaining](https://en.wikipedia.org/wiki/Hash_table#/media/File:Hash_table_5_0_1_1_1_1_1_LL.svg) as collision resolution.
> A chained hash table indexes into an array of pointers to the heads of linked lists. Each linked list cell has the key for which it was allocated and the value which was inserted for that key.
> When you want to look up a particular element from its key, the key's hash is used to work out which linked list to follow, and then that particular list is traversed to find the element that you're after.
> If more than one key in the hash table has the same hash, then you'll have linked lists with more than one element. Notice that after calculating the hash function, you need to get the first element from the list to get the value required.
> Therefore, you access the pointer to the head of the list and then the value: 2 operations. The downside of chained hashing is having to follow pointers in order to search linked lists.
> The upside is that chained hash tables only get linearly slower as the load factor (the ratio of elements in the hash table to the length of the bucket array) increases, even if it rises above 1.
>
> On the other hand, with [open-addressing](https://en.wikipedia.org/wiki/Hash_table#/media/File:Hash_table_5_0_1_1_1_1_0_SP.svg), a hash table indexes into an array of pointers to pairs of (key, value).
> You use the key's hash value to work out which slot in the array to look at first. If more than one key in the hash table has the same hash, then you use some scheme to decide on another slot to look in instead.
> For example, linear probing is where you look at the next slot after the one chosen, and then the next slot after that, and so on until you either find a slot that matches the key you're looking for, or you hit an empty slot (in which case the key must not be there).
> Open-addressing is usually faster than chained hashing when the load factor is low because you don't have to follow pointers between list nodes.
> However, when your HashTable starts to get full, and you have a high load factor, due to collisions happening more often, probing will require you to check more Hashtable locations before you find the actual value that you want.
> Also, you can never have more elements in the hash table than there are entries in the bucket array
> 
> At about a load factor of 0.8, chaining starts to become more efficient due to multiple collisions: you would have to probe a lot of empty cells in order to find the actual value you want with probing, 
> while with chaining you have a list of values that have the same hash key. To deal with the fact that all hash tables at least get slower (and in some cases actually break completely) when their load factor approaches 1, 
> practical hash table implementations make the bucket array larger (by allocating a new bucket array, and copying elements from the old one into the new one, then freeing the old one) when the load factor gets above a certain value (typically about 0.7).
> This is just a quick overview, as the actual data, the distribution of the keys, the hash function used and the precise implementation of collision resolution will make a difference in your actual speed.

### Concurrency Control Mechanisms
- [Chapter 13. Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
#### Predicate Lock
- [Predicate Locking](https://www.tutorialspoint.com/predicate-locking)
> A predicate lock in a database system is a lock that reserves not just a single row from being changed by other transactions, 
> but all rows that satisfy a particular condition (or predicate). These locks are usually placed with a SELECT FOR UPDATE statement.

#### Materialising Conflicts


