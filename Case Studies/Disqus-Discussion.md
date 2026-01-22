# Disqus

#### Question: I got below error when I create the comments_by_article table by using the CQL you provide? Coud you help to check that adn tell me why is that? Thanks!
That error occurs because of a specific limitation in Apache Cassandra.
```text
Query 1 ERROR at Line 1: : Cannot mix counter and non counter columns in the same table
```
##### Why is this happening?
In Cassandra, `counter` columns are special. They are not standard integers; they are **Conflict-Free Replicated Data Types (CRDTs)** designed for atomic increments and decrements without **read-before-write** locks.

Because the internal storage mechanism and replication logic for counters are fundamentally different from standard columns (like `text` or `uuid`), Cassandra enforces a strict rule: **A table that contains a counter column can ONLY contain counter columns and Primary Key columns.** You cannot store body text and upvotes counter in the same table.

##### The Solution
We must split the data into two separate tables:
1. **comments_by_article**: Stores the content and metadata (Standard columns).
2. **comment_vote_counts**: Stores the upvotes and downvotes (Counter columns).

---

#### Question: About the Replying to Comments (Nested Comments), I see that we have parent_id column for the reference. Could we talk more about this? Like what's the CQL to query all comments under the original comment?
This is a great question that touches on the core of NoSQL data modeling: **modeling for your queries**.
In a Relational Database (like PostgreSQL), you might use a Recursive CTE (Common Table Expression) to fetch a nested tree. In Cassandra, we don't have joins or recursion. Instead, we rely on the **Clustering Key** to group data efficiently on the disk.

##### The CQL Query
To get all direct replies (children) under a specific "original comment" (parent), you would use the following CQL:
```cassandraql
SELECT * FROM comments_by_article WHERE article_id = 'A1' AND parent_id = 00000000-0000-0000-0000-000000000000;
```

##### Why this works efficiently
This efficiency is dictated by your Primary Key definition: `PRIMARY KEY ((article_id), parent_id, comment_id)`
1. `article_id` **(Partition Key)**: Cassandra goes directly to the node holding this article's data.
2. `parent_id` **(1st Clustering Key)**: Within that partition, data is **physically sorted and grouped** by `parent_id`.
   - All comments with `parent_id = 0` (Root comments) are stored together.
   - All comments with `parent_id = A` (Replies to A) are stored together.
   - All comments with `parent_id = B` (Replies to B) are stored together.
Because of this grouping, the query above performs a **Slice Query**. It jumps to the start of the `parent_id` block and reads sequentially. It is extremely fast.

##### Handling the "Full Tree" (Nested Comments)
There are two common approaches to displaying the full nested conversation using this schema:
1. **Lazy Loading (Drill-down)**:
   - First, load top-level comments: `SELECT ... WHERE parent_id = 0000...`
   - When a user clicks "View 5 Replies", you execute the query above: `SELECT ... WHERE parent_id = clicked_comment_id`.
   - _Pros_: Very efficient, saves bandwidth.
   - _Cons_: Requires more round-trips if the user expands everything.
2. **Application-Side Tree Construction (Load All)**:
   - Query: `SELECT * FROM comments_by_article WHERE article_id = 'A1'`;
   - This fetches **all** comments for the article in one go.
   - Since the result is sorted by `parent_id`, the application can iterate through the list in linear time **O(N)** and reconstruct the tree structure (e.g., using a Hash Map).
   - _Pros_: Single database query.
   - _Cons_: Heavy payload for articles with thousands of comments.

---

#### Question: For DELETE API of Vote service, the parameters are sent by putting them in the Request Body rather than using paht or query parameter. Could you please explain what the consideration behind this? Thanks!
That is a very keen observation! In RESTful API design, passing a Request Body in a DELETE method is indeed a specific design choice that deviates slightly from the most common patterns (like DELETE /resource/{id}).

Here is the explanation of the considerations behind this design, followed by a recommendation for improvement.
##### Design Considerations
1. **Composite Identification (No Single ID)**: A "Vote" in this system is a relationship entity defined by the combination of a User and a Comment. It doesn't necessarily have a standalone, public vote_id (like /votes/12345).
   - The User is identified by the Authentication Token (JWT) in the header.
   - The Comment must be specified by the client. Since there isn't a single ID to append to the URL path, the designer likely chose the body to carry the composite key part (comment_id).
2. **API Consistency**: The POST (Create Vote) and DELETE (Remove Vote) operations are symmetrical.
   - POST: { "comment_id": "C1", "vote_type": "up" }
   - DELETE: { "comment_id": "C1" } Using the body makes the request structure look consistent with the creation logic, treating the "vote criteria" as a data object.
3. **Extensibility**: Using a body allows for future extensibility without changing the URL structure. For example, if you wanted to support **Batch Deletes** (removing votes from multiple comments at once), you could easily change the body to accept an array `[ "C1", "C2" ]` without breaking the URL endpoint.

##### Recommendation: The "Best Practice" Approach
While the design above is logical, as an experienced engineer, I would recommend **avoiding Request Bodies in DELETE** requests.
- Why?
  - **RFC Compliance**: The HTTP/1.1 spec (RFC 7231) states that a payload in a `DELETE` request has "no defined semantics."
  - **Infrastructure Risk**: Many intermediaries—such as `Load Balancers`, `Proxies`, and `Web Application Firewalls (WAFs)`—may automatically **strip the body** from `DELETE` requests because they don't expect it. This causes the backend to receive an empty request, leading to errors.

##### Suggested Solution: 
Use **Query Parameters** instead. It is standard, safe, and explicitly supported by all infrastructure. `DELETE /v1/votes?comment_id=C1`

---

#### Question: It seems a bit redundant to me that we use 1 more table to store only the vote count only. What the problem will be if I only use integer type as counter and store it in the comments_by_article? And could you please tell me more about read-before-write locks? Thanks
This is a classic distributed system design question! It touches on the fundamental difference between how `Relational Databases (SQL)` and `Distributed NoSQL Databases (Cassandra)` handle concurrency.

Here is the explanation of why we need a separate table and what "Read-Before-Write" implies.
##### Why can't we just use an Integer in comments_by_article?
If you store upvotes as a standard Integer (or BigInt) column in the main table, you run into the **Lost Update** problem due to race conditions.

**The Scenario: Two users vote at the exact same time.**

Imagine the current vote count is 10.
1. User A wants to upvote. Their app reads the current count: 10.
2. User B wants to upvote. Their app reads the current count: 10.
3. User A calculates `10 + 1 = 11` and writes 11 to the database.
4. User B calculates `10 + 1 = 11` and writes 11 to the database.
**Result**: The count is 11, but it should be 12. One vote was lost.

**Why Cassandra Counter types are different**
- Cassandra `counters` are special. When you update a counter, you do not read the value first. You simply send a "delta" (change) command: `UPDATE ... SET upvotes = upvotes + 1` ...
- Cassandra applies this `+1` operation internally. Even if 100 requests come in at once, Cassandra merges them successfully (eventually) to reach the correct count.

**The Technical Limitation**
Because `counter` columns require completely different internal handling (they use Conflict-Free Replicated Data Types, or CRDTs) compared to standard text/integer columns (which use Last-Write-Wins), Cassandra enforces a strict rule:
>A table that contains a counter column can ONLY contain counter columns and Primary Key columns.

This is why we are forced to create the separate `comment_vote_counts` table.

##### What is a "Read-Before-Write" Lock?
To solve the "Lost Update" problem with standard Integers (without using special Counters), you would typically need a Lock.

**The "Read-Modify-Write" Cycle**

In a standard database, updating a value usually involves three steps:
1. **Read**: Fetch the current value from the DB to the application memory.
2. **Modify**: Calculate the new value (e.g., add 1).
3. **Write**: Save the new value back to the DB.

**The Lock**

To prevent User A and User B from reading the same 10 at the same time, the database must Lock the row.
1. User A says "I want to update this row." -> DB Locks the row.
2. User A reads 10.
3. User B tries to read -> BLOCKED (Waits).
4. User A writes 11. -> DB Unlocks.
5. User B unblocks, reads 11.
6. User B writes 12.

**Why this is bad for Disqus (High Scale)**
- **Latency**: User B has to wait for User A. If 1,000 people vote on a viral comment, they form a queue. The 1,000th person waits a long time.
- **Throughput**: Your database spends all its CPU managing locks instead of writing data.
- **Distributed Systems**: In a distributed database like Cassandra, locks are extremely expensive (requires coordination across multiple servers, often using Paxos/LWT).

**Conclusion**: Using a separate table with `counter` types allows us to achieve **non-blocking**, **high-throughput writes**. We accept the complexity of an extra table to gain massive performance and accuracy.

---

#### Question: Could you please explain more about the 2 different types of Pagination, namely Cursor-based and Offset-based? And what are the scenarios good for each of them?
This is a fundamental concept in API and database design, especially when dealing with large datasets like the **1 billion comments** in your Disqus system.

Here is a breakdown of the two pagination strategies, how they work under the hood, and when to use which.
##### Offset-based Pagination
This is the traditional method, often used because it is simple to implement and intuitive for users (e.g., "Go to Page 5").

**How it works**: The client requests a specific `page number` and `page size`. The database skips (`OFFSET`) a certain number of rows and returns the next set (`LIMIT`).
```sql
-- SQL Example: To get Page 3 (items 21-30):
SELECT * FROM comments ORDER BY created_at DESC OFFSET 20 LIMIT 10;
```

**Pros**:
- **Random Access**: Users can jump directly to any page (e.g., "Go to Page 50").
- **Stateless**: The server doesn't need to know about the previous request.
**Cons**:
- **Performance (The "Offset Problem")**: The database cannot simply jump to row 1,000,000. It must read and count 1,000,000 rows from the disk, discard them, and then return the next 10. As the offset grows, the query becomes linearly slower `(O(N))`.
- **Data Inconsistency (Page Drift)**: If a new comment is added to the top while a user is viewing Page 1, the last item on Page 1 might get pushed to Page 2. When the user clicks "Next", they will see that same item again (duplicate). Conversely, if an item is deleted, they might miss a record.
**Best For**:
- **Small Datasets**: Where the performance hit of scanning rows is negligible.
- **Admin Dashboards**: Where users need to jump to specific pages and data is relatively static.
- **Relational Databases**: Which support `OFFSET` natively (though performance still degrades).

##### Cursor-based Pagination (Keyset Pagination)
This method uses a unique pointer (the `cursor`) to the last item the user saw. The API asks for "10 items after this specific ID".

**How it works**: Instead of saying "skip 20 rows", the client says "give me rows where the ID is smaller than the last ID I saw". (Note: This assumes `comment_id` is sortable, like a Timestamp or TimeUUID)
```sql
-- SQL Example: To get the next page after comment with ID C_20
SELECT * FROM comments WHERE comment_id < 'C_20' ORDER BY comment_id DESC LIMIT 10;
```

**Pros**:
-**High Performance**: The database uses an index to jump directly to the row with ID C_20 and reads the next 10. It is `O(1)` or `O(Limit)` regardless of how deep you are in the dataset (Page 1 vs Page 1,000,000 takes the same time).
- **Consistency**: It is robust against concurrent writes. If new comments are added to the top, they don't affect the relative position of the cursor C_20. The user continues smoothly from where they left off.
- **Infinite Scroll Friendly**: Perfect for feeds where "Next Page" just appends data.
**Cons**:
- **No Random Access**: You cannot jump to "Page 50" without traversing pages 1 through 49 to get the cursor.
- **Implementation Complexity**: You need a unique, sortable column (or a composite key). If you sort by `upvotes` (which aren't unique), you need a tie-breaker like (`upvotes, id`).
**Best For**:
- **Large Datasets**: Like your Disqus system with billions of comments.
- **Infinite Scroll**: Social media feeds (Twitter, Facebook, Instagram).
- NoSQL Databases: Systems like **Cassandra** or **DynamoDB** often _only_ support this efficiently because they are designed for sequential reads from a partition, not random jumps.

##### Summary Comparison
| Feature         | Offset-based                                        | Cursor-based                         |
|:----------------|:----------------------------------------------------|:-------------------------------------|
| Query           | OFFSET 1000 LIMIT 10                                | WHERE id > cursor LIMIT 10           |
| Performance     | Slows down as page number increases.                | Constant speed (fast).               |
| User Experience | "Go to Page 5"                                      | "Load More" / Infinite Scroll        |
| Data Stability  | Items can be skipped or duplicated if data changes. | Stable iteration.                    |
| Complexity      | Low                                                 | Medium                               |
| Disqus Fit      | Poor (Cassandra scans are expensive).               | Excellent (Uses timeuuid naturally). |

---

#### Question: I see you add WAF component in the Architecture which is not common to see it in System Design question. Could you please tell me more about this and what's the consideration behind adding this, thanks!
You are correct that in many general System Design interviews, components like WAF (Web Application Firewall) are often omitted or abstracted away into "Load Balancers" or "Security Groups" to keep the focus on scaling the backend.
However, for a Comment System (or any system heavily reliant on User-Generated Content), adding a WAF is not just a "nice-to-have"—it is a critical architectural decision.

Here is the detailed reasoning behind adding the WAF, followed by an update to your design document.
##### Why a WAF is Critical for Disqus
A WAF operates at **Layer 7 (Application Layer)** of the OSI model. Unlike a standard network firewall (which blocks ports/IPs), a WAF inspects the actual content of the HTTP traffic (headers, body, cookies).
For Disqus, we include it for three specific reasons:
1. **Protection Against Cross-Site Scripting (XSS)**: This is the #1 threat for comment systems.
   - **The Attack**: A malicious user submits a comment containing a script: `<script>stealCookies()</script>`. If the backend saves this and serves it to other users, that script executes in _their_ browsers.
   - **The WAF Role**: The WAF inspects the `POST /comment` body. It recognizes standard XSS attack patterns (like `<script>` tags or `javascript:` URIs) and blocks the request at the edge **before** it ever reaches your application servers. It acts as a shield for your backend code.
2. **Bot and Spam Mitigation**: Comment sections are magnets for spam bots trying to post links for SEO or phishing.
   - **The Attack**: A botnet sends thousands of requests per second to post "Buy Cheap Rolex" links.
   - **The WAF Role**: Modern WAFs (like Cloudflare or AWS WAF) have "Bot Control" features. They can detect non-human behavior (e.g., lack of cookies, suspicious User-Agents, or impossible request rates) and challenge them (CAPTCHA) or block them entirely. This saves your backend `Comment Service` from processing millions of junk requests.
3. **SQL/NoSQL Injection**:
   - **The Attack**: An attacker tries to manipulate your database queries via input fields.
   - **The WAF Role**: It filters out common injection patterns (e.g., `' OR 1=1 --`) from the request parameters, adding a layer of defense in depth protecting your PostgreSQL and Cassandra databases.

---

#### Question: For a more comprehensive design, which is better? Adding a standalone Load Balancer, or having WAF or API Gateway with Load Balancer functionality? If a standalone one is better, where should I put it? If latter one is better, what products are popular in this scenario? Thanks!
This is a classic system design question that touches on the "Separation of Concerns" principle.

For a system of this scale (100M DAU, 1B comments/day), adding a standalone Load Balancer (or a distinct Load Balancing layer) is the better design choice. Here is the breakdown of why, where to place it, and the popular tools involved.
1. **Why a Standalone Load Balancer is Better**:
   While it is true that many API Gateways (like Kong or AWS API Gateway) _can_ do load balancing, and many WAFs _can_ do routing, separating them provides critical benefits:
   - **Scalability of the Gateway Itself**: The API Gateway is a piece of software that performs logic (authentication, rate limiting, header transformation). It consumes CPU. Therefore, you will likely need a **cluster** of API Gateway nodes. You need a Load Balancer in front of them to distribute traffic to that cluster.
   - **TLS Termination (SSL Offloading)**: Decrypting HTTPS traffic is CPU-intensive. A dedicated Load Balancer (often hardware-accelerated or highly optimized software like `Nginx`) handles this efficiently, letting your API Gateway focus on business logic.
   - **Resilience**: If an API Gateway node crashes, the Load Balancer detects it (Health Checks) and reroutes traffic instantly.

2. **Where should you put it?**:
   **Placement**: Between the `WAF` and the `API Gateway`.
   The logical flow is:
   1. **WAF**: "Is this request safe?" (Filters out attacks).
   2. **Load Balancer**: "Which API Gateway server is free?" (Distributes traffic & Decrypts SSL).
   3. **API Gateway**: "Which microservice handles /`comments`?" (Routes to backend).
   
3. **Popular Products**:
   If you decide to implement this layered approach, here are the industry standards:
   - Load Balancers:
     - Cloud: AWS Application Load Balancer (ALB), Google Cloud Load Balancing.
     - Self-Hosted: Nginx, HAProxy.
   - API Gateways:
     - Cloud: AWS API Gateway.
     - Self-Hosted: Kong, Traefik, Spring Cloud Gateway.
   - WAF:
     - Cloud: AWS WAF (often attaches directly to the ALB), Cloudflare.

---