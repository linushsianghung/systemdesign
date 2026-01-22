# LeetCode

#### Question: For API Gateway and Load Balancer, do we really need both of them? I've heard that some Reverse Proxy, like Nginx, can have both functionalities, is that right?
That's an excellent and very practical question! You've hit on a common point of confusion when looking at modern architecture diagrams.

The short answer is: **Yes, you are right**. A powerful reverse proxy like Nginx can indeed be configured to perform the functions of both a load balancer and a basic API gateway.
However, in a large-scale microservices architecture like this one, **using two separate, specialized components is often the better design choice** due to a clear separation of concerns and better scalability.
Let's break down the distinct roles they play in our design.

##### The Role of the Load Balancer
Think of the Load Balancer (like AWS Application Load Balancer or Nginx in a simple configuration) as the main traffic cop at the front door of your entire application. Its primary responsibilities are:

1. **Distributing Traffic**: Its main job is to take all incoming user requests and distribute them across multiple instances of the next layer in the system, which in our case is the API Gateway. This prevents any single gateway instance from being overwhelmed.
2. **Health Checks**: It constantly pings the API Gateway instances to ensure they are healthy. If an instance becomes unresponsive, the load balancer automatically stops sending traffic to it, ensuring high availability.
3. **SSL Termination**: It can handle the decryption of HTTPS traffic, offloading that computational work from your application servers.

It operates at a lower level, primarily concerned with network traffic, not the content of the API calls themselves.

##### The Role of the API Gateway
The **API Gateway** (like AWS API Gateway, Kong, or Tyk) is a more intelligent, application-aware layer. It sits between the Load Balancer and your microservices. Its primary responsibilities are:

1. **Request Routing**: This is its most important job. It inspects the request path (e.g., `/api/users/...` or `/api/problems/...`) and intelligently routes the request to the correct downstream microservice (`UserService`, `ProblemService`, etc.).
2. **Authentication & Authorization**: It can verify user identity (e.g., by validating a JWT token) before a request ever reaches your core services.
3. **Rate Limiting & Throttling**: It protects your services from being overwhelmed by too many requests from a single user or IP address.
4. **Request/Response Transformation**: It can modify requests or responses on the fly, for example, to adapt to different API versions.
5. **Centralized Logging & Monitoring**: It provides a single point to log all incoming API calls.

#### Why Use Both? The "Separation of Concerns" Argument
While Nginx can do both, separating the roles gives you a cleaner, more scalable, and more maintainable system:

| Concern          | Handled by Load Balancer                                                                    | Handled by API Gateway                                                               |
|------------------|---------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| **Availability** | Ensures the API Gateway layer itself is highly available by balancing across its instances. | Ensures the backend microservices are available by routing to healthy ones.          |
| **Scalability**  | Allows you to scale the number of API Gateway instances horizontally.                       | Allows you to scale the number of individual microservice instances independently.   |
| **Complexity**   | Manages simple, network-level routing.                                                      | Manages complex, application-level routing and business logic (auth, rate limiting). |

In our design, the flow looks like this: `User Request` -> `Load Balancer` (Distributes traffic to a healthy Gateway) -> `API Gateway` (Inspects URL, authenticates, and routes to the correct Microservice) -> `Microservice`

This layered approach is a standard pattern for building robust, large-scale systems. You use the best tool for each specific job. While you could configure one super-powered Nginx instance to do everything, you would end up with a very complex configuration file that is hard to manage and debug. Using dedicated services makes the whole system easier to reason about.

---

#### Question: I have a similar question which is about the Contest and Leaderboard Services. Their main responsibilities seem quite simple. If I combine both of them into one bigger service, is there any concern of this decision?
That's a very insightful question, and it touches upon a fundamental decision in microservices architecture: **where do you draw the boundaries between services**?

You're right that on the surface, the `Contest Service` and `Leaderboard Service` seem closely related, both dealing with contest-specific data and user scores. Combining them into one larger service might seem simpler at first glance. However, there are several significant concerns with this decision, especially given the scale and real-time requirements of your system.

Let's break down the responsibilities again and then discuss the concerns of combining them:

##### Current Responsibilities:
- `Contest Service`:
  - Manages the lifecycle of contests (creation, start, end).
  - Associates problems with contests.
  - Receives notifications from the `Code Execution Service` about correct submissions for a contest.
  - Calculates the *raw score and penalty time* for a user based on their submission history for a given problem within a contest.
  - Triggers updates to the `Leaderboard Service` when a user's score changes.
  - Interacts with `UserDB` and `ProblemDB` to get contest participant and problem details.

- `Leaderboard Service`:
  - Receives score updates from the `Contest Service`.
  - Maintains the real-time ranking using an optimized data structure (like Redis Sorted Sets).
  - Serves leaderboard data to clients with low latency.
  - Its primary concern is the *presentation and fast retrieval* of ranked data.

##### Concerns with Combining Contest Service and Leaderboard Service:
1. ***Differing Scaling Requirements & Bottlenecks***:
   - **Contest Service (Write-Heavy Logic)**: This service needs to scale to handle the computation of scores for potentially thousands of submissions per second during peak contest times. Its workload is CPU-bound (score calculation) and involves database writes (updating user contest progress).
   - **Leaderboard Service (Read-Heavy Cache)**: This service needs to scale to handle the reads from 10,000 concurrent users constantly refreshing the leaderboard. Its workload is primarily I/O-bound (serving from Redis cache) and needs extremely low latency.
   - **Concern**: If combined, you'd be forced to scale both components together, even if one isn't under heavy load. For example, after a contest ends, the score calculation load drops significantly, but the leaderboard viewing load might remain high. You'd be over-provisioning resources for the contest logic or under-provisioning for the leaderboard reads, leading to inefficient resource utilization or performance bottlenecks.
2. ***Single Point of Failure***:
   - If a combined service encounters an issue (e.g., a bug in score calculation, a memory leak from heavy leaderboard queries), both the ability to process contest submissions and display the leaderboard would be affected.
   - **Concern**: Separating them provides better fault isolation. If the leaderboard cache goes down, contest submissions can still be processed and scores calculated (though not immediately visible on the leaderboard). If the contest logic has a bug, the leaderboard might still display stale data but remain functional.
3. ***Complexity and Maintainability***:
   - A combined service would have more responsibilities, more complex internal logic, and a larger codebase.
   - **Concern**: This makes it harder to develop, test, and deploy. Changes to the score calculation algorithm could inadvertently introduce bugs in the leaderboard display logic, and vice versa. Debugging becomes more challenging.
4. ***Technology Stack Focus***:
   - The `Contest Service` might primarily interact with a relational database for persistent contest state and user progress.
   - The `Leaderboard Service` heavily leverages an in-memory data store like Redis Sorted Sets for its core functionality.
   - **Concern**: While a single service can use multiple data stores, having a service focused on a specific data access pattern (e.g., Redis for fast reads) often leads to a cleaner, more optimized implementation. Combining them might dilute this focus.
5. ***Data Consistency vs. Availability/Latency Trade-offs***:
   - The `Contest Service` is responsible for ensuring the *correctness* and *consistency* of scores.
   - The `Leaderboard Service` prioritizes *availability and low latency* for displaying rankings, often accepting eventual consistency with the underlying score data for performance.
   - **Concern**: Combining these two concerns into one service can make it harder to manage these trade-offs effectively. You might end up with a system that is neither optimally consistent nor optimally fast.

##### Conclusion:
While combining them might seem simpler initially, for a system with the scale and real-time requirements of a LeetCode contest, keeping the `Contest Service` and `Leaderboard Service` separate is a **stronger design decision**.

This separation allows for:
- **Independent Scaling**: Scale each component based on its specific load profile.
- **Improved Fault Isolation**: A problem in one doesn't necessarily bring down the other.
- **Clearer Responsibilities**: Each service has a well-defined, focused purpose.
- **Easier Development and Maintenance**: Smaller, more focused services are easier to work with.
- **Optimized Technology Choices**: Each service can leverage the best tools for its specific job (e.g., Redis for leaderboard, PostgreSQL for contest state).

The slight overhead of inter-service communication (Contest Service notifying Leaderboard Service) is a small price to pay for these significant benefits in a high-performance, high-availability system.

---
