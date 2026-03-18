# Storage
## Reference:
- [Amazon S3 (Object Stores) - In Practice | Distributed Systems Deep Dives](https://www.youtube.com/watch?v=zplIwqWBhwg)
- [Object Storage in System Design Interviews](https://www.youtube.com/watch?v=RvaMHMxHjp4)
- [Design an Amazon S3 or Object Storage](https://www.youtube.com/watch?v=chG5HV-mGa0)
- [Everything in its write place: Cloud storage abstraction with Object Store](https://dropbox.tech/infrastructure/abstracting-cloud-storage-backends-with-object-store)
- [A Guide to Data Chunking](https://www.couchbase.com/blog/data-chunking/)
---

## What is Object Storage
### Compare with Block Storage or File System:
| Type           | Organized By    | Good For                  |
|:---------------|:----------------|:--------------------------|
| Block Storage  | Blocks          | Databases                 |
| File Storage   | Directory tree  | Shared file systems       |
| Object Storage | Key (Object ID) | Massive unstructured data |

### Core Mechanism of Object Storage
Object storage does NOT invent new physical storage hardware. What it changes is `The abstraction + data organization + metadata system.`

- Object Storage ultimately runs on block devices.
```text
Object Storage
    ↓
Local filesystem or raw disks
    ↓
Block storage (SSD/HDD)
```
- Object storage systems are built around three core internal subsystems:
  1. Metadata Service
  2. Data Placement Engine
  3. Storage Nodes (blob containers)

### Storage Unit of Object Storage?
- The fundamental unit is:
  - **Object (logically)**
  - **Chunk (physically)**
- Internally, chunks are usually stored as:
  - **Files** in local filesystem (common)
  - **Raw segments** in custom storage engine

### Implementation Models
#### Model A — Filesystem-Based (Simpler Systems)
Used by:
  - MinIO (by default)
  - Many self-hosted systems
```text
User-space distributed system
+ Local filesystem
+ Disks
```

#### Model B — Custom Storage Engine (Hyperscalers)
Used by:
  - Amazon S3
  - Google Cloud Storage
  - Azure Blob

They often bypass normal filesystem and:
  - Manage disks directly
  - Use custom LSM-like storage
  - Use append-only log structures
  - Optimize for sequential write
---

## How Object Storage Works
### The Core Architecture: **Flat Address Space**
Every piece of data is bundled into an Object, which consists of:
- A **Unique Identifier (OID)**: A long string of characters used to retrieve the object.
- **Metadata**: Customizable information (e.g., "Camera Type: iPhone," "Expiration Date: 2027").
- **The Data itself**: (e.g., a photo, video, or log file).
```text
Object = { Key (Unique ID), Metadata, Binary Data }
```
### The Retrieval Mechanism: **Key-Value Store**
When **PUT**ing an object into a system (like `Amazon S3`), the system takes the **Key** (the filename/ID) and uses a **Hashing Algorithm** to determine exactly which physical disk and server should hold that data.

### What is a Presigned URL?
- A Presigned URL is a temporary, signed HTTP URL that grants limited access to an object. It contains:
  - Object path
  - Expiration time
  - Signature (HMAC)
  - Access policy (GET / PUT)
  ```text
    https://s3.amazonaws.com/bucket/avatar.png?
    X-Amz-Algorithm=AWS4-HMAC-SHA256
    &X-Amz-Credential=...
    &X-Amz-Expires=300
    &X-Amz-Signature=abc123...
  ```
- Without Presigned URL: `Client → Backend → Object Storage`
- With Presigned URL: `Client → Object Storage (Backend only signs)`
  - Lower backend bandwidth
  - Lower latency
  - Scales massively
  - Backend stateless

## The Architecture of Storage Storage System
In a typical Object Storage architecture, the workload is split between two main types of nodes:

### Metadata Server (The Brain)
The **Metadata Server** (`MDS`) is a specialized service or dedicated server that manages the namespace of the storage system. It doesn't store the actual files (the bits and bytes of your cat photos); instead, it stores the **ledger**.
What it handles:
- **Object Mapping**: It knows exactly which physical storage nodes hold the pieces of Object.
- **Custom Metadata**: It stores the user-defined tags (e.g., Author: Jane, Department: Finance).
- **Access Control**: It checks if client actually have permission to see the data.
- **Replication Logic**: It keeps track of whether an object has enough healthy replicas or erasure-coded fragments across the cluster.

When asking the system for a file, the process usually looks like this:
- **The Request**: The application sends a GET request for Object_ABC.
- **The Lookup**: The request hits the **Metadata Server** (or a `Gateway/Proxy`). The MDS looks up Object_ABC in its database.
- **The Direction**: The MDS replies with: Object_ABC is broken into 3 pieces located on Server 10, Server 15, and Server 22.
- **The Retrieval**: The application fetches the data directly from those Storage Nodes.

### The Storage Nodes (The Body)
These are _dumb_ but reliable servers packed with high-capacity hard drives. Their only job is to store the raw data chunks and serve them up when the `Metadata Server` tells them to.

## Deep Dive to Metadata Server
In a traditional centralized system, the Metadata Server (MDS) handles the mapping. But in modern hyper-scaled systems, we try to move that **hashing** responsibility away from a single brain to avoid bottlenecks.

### Architecture of Metadata Server
- **Centralized Architecture**:
  - Some systems use a **single cluster** of metadata servers. This is easy to manage but can get overwhelmed if you have billions of small files.
  > #### Lookup Flow (MDS is _Before_)
  > 
  > In architectures like **HDFS** or some enterprise storage arrays, the Metadata Server is the Gatekeeper. It sits **before** the actual data placement happens in the logic. 
  > 1. **Client** says: "I want to store file.mp4."
  > 2. **Metadata Server** checks permissions and says: "Okay, I'll call that Object_123."
  > 3. The **Placement Engine** (built into the MDS) looks at the cluster and decides: "Put it on Node A and Node B."
  > 4. **Metadata Server** writes this location into its database (the _Index_). 
  > 5. **Client** is told: "Go talk to Node A and Node B."
  > 
  > **Key Point**:
  > - In this flow, the Metadata Server **owns** the `Placement Engine`. The location is a stored fact in a database.
  > - To find the data later, you must ask the MDS. If the MDS database gets too big, the whole system slows down.

- **Distributed (Decentralized) Architecture**: 
  - Newer systems (like `Ceph`) try to get rid of a central metadata server by using a mathematical algorithm called `CRUSH`.
  - The client calculates where the data is located using math, rather than asking a central _brain_. This allows the system to scale to exabytes without the _brain_ exploding.
  - Modern Object Storage relies so heavily on the Hashing of `Placement Engine` which turns a _search_ problem into a _math_ problem.
  > #### Algorithmic Flow (MDS is _Beside_)
  > In modern systems like `Ceph` or `OpenStack Swift`, the Metadata Server and the Placement Engine are more like partners working in parallel. This is how they achieve _infinite_ scale.
  >
  > 1. **Client** says: "I want to store file.mp4.
  > 2. **Metadata Server** says: "Great, I've recorded that you own a file called file.mp4 and its ID is Object_ABC. I don't know where it's going yet, but I've noted the name.
  > 3. The **Placement Engine** (using a Hashing Algorithm / Mathematical Map like `CRUSH`) runs a math equation: `Hash(ObjectID, ClusterMap) => PhysicalLocation`, ex. Node_92.
  > 4. **Client (or Gateway)** performs that math and sends the data directly to Node 92.
  > 
  > **Key Point**: 
  > - In this flow, the location isn't _stored_ in a database; it is **calculated**.
  > - The Metadata Server handles the _human_ metadata (filenames/dates), while the Placement Engine handles the _physical_ math. They happen almost simultaneously.
  > - Because it’s just math, the client can figure out where the data is without asking a Metadata Server.

### Summary
- The **Metadata Server** is usually the first point of contact for `Authentication` and `Naming`.
- The **Placement Engine** is the _Logic_ that determines the `Physical Destination`.
> In modern cloud storage, we prefer the Algorithmic approach because if the Metadata Server had to "remember" the location of trillions of objects, the database would become so large it would eventually crash!
---

## Erasure Coding
### Core Concept of Erasure Coding
- Split data into chunks
- Generate parity chunks using math (Parity is mathematically generated redundancy, not full copies.)
- Store them across multiple nodes
- Recover missing chunks using parity

```text
# Example: 4+2 Erasure Coding
1. Suppose we have a file:  [DATA]
2. We split into 4 data chunks: D1 D2 D3 D4
3. Then generate 2 parity chunks: P1 P2
4. Total chunks stored: D1 D2 D3 D4 P1 P2

Now, even if ANY 2 chunks are lost, we can reconstruct the original data. And the storage overhead is 6 / 4 = 1.5x instead of 3x replication.
```
### Erasure Coding Is Perfect for Object Storage
- Object storage workload:
  - Write once
  - Read occasionally
  - Large files
  - Cold data

- Erasure coding is ideal because:
  - CPU cost acceptable
  - Storage savings massive
  - Throughput optimized for large objects

---

## GCS
- [Product overview of Cloud Storage](https://docs.cloud.google.com/storage/docs/introduction)
- [What is binary large object (BLOB) storage?](https://cloud.google.com/discover/what-is-binary-large-object-storage)
---

## AWS S3
- [Building and operating a pretty big storage system called S3](https://www.allthingsdistributed.com/2023/07/building-and-operating-a-pretty-big-storage-system.html)
- [FAST '23 - Building and Operating a Pretty Big Storage System](https://www.youtube.com/watch?v=sc3J4McebHE)
- [Uploading and copying objects using multipart upload in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
---

## Hierarchical Namespace
Reference: [Hierarchical namespace](https://docs.cloud.google.com/storage/docs/hns-overview)