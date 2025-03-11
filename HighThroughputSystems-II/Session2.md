# Designing Amazon S3

## Key Design Concept

- **Directories are virtual:** In S3, directories don’t have a physical existence. When a folder (like `abc`) is created and a file (like `123.jpg`) is placed in it, there’s no actual directory called `abc` on the disk. Instead, S3 uses the full path (`bucket/directory`) as the key, meaning it functions purely **as a key-value store**. This abstraction simplifies design, as files can be stored anywhere, without needing a directory structure on disk.

## Design Goals

1. **Strong Consistency:** When writing data, any subsequent read should reflect the latest written data. For instance, if a key `f` holds the value `'ball'` and is updated to `'ball1'`, the next read must return `'ball1'` immediately without any delay.

2. **Global Namespace:** Bucket names are globally unique across all regions, allowing seamless scalability. For example, a company expanding from India to the US will avoid name conflicts because bucket names are globally unique.

3. **Disaster Recovery:** S3 provides protection against natural disasters (like floods or earthquakes) by ensuring redundancy in both storage and compute infrastructure.

4. **Multi-Tenancy:** S3 supports multi-tenancy by ensuring the load of one customer does not affect others. For example, services like Amazon Prime and Netflix sharing the same infrastructure shouldn’t affect each other’s performance.

## S3 Architecture Overview

S3’s architecture is built around **storage stamps,** which are collections of racks designed to store data redundantly. Each storage stamp typically contains:

- 10–20 racks
- 18–20 servers per rack
- Capacity: 30–50 PB per storage stamp
- Redundant power supply and network connectivity ensure high availability and durability.

S3 utilizes **VIP (Virtual IP)** addresses for each storage stamp, allowing requests to be routed to the appropriate storage stamp without needing to configure DNS changes when replacing racks.

## Location Service & DNS

- **Location Service:** Responsible for mapping buckets to the storage stamps they reside in. It updates DNS entries when new buckets are created or moved.
- **Bucket URL Structure: Follows the format:**
`https://<bucket-name>.s3.<region>.amazonaws.com/<object-key>`.
    - DNS resolution maps the bucket to the correct storage stamp IP.

## Handling Storage Stamp Capacity
When a storage stamp fills up:

1. A new storage stamp is created using available racks.
2. Smaller buckets may be moved to the new stamp to make room for growing accounts in the old stamp.
3. AWS may recommend customers create new buckets if their existing bucket reaches a predefined capacity limit.

## Layers in S3’s Storage Stamp
S3’s storage stamps have three layers:

1. **Frontend:** Handles requests and user authentication.
2. **Partition Layer:** Responsible for determining the logical ownership of keys. It helps to group keys (using range-based hashing) to keep related objects together, improving access times.

3. **Stream Layer:** A distributed file system with replication, handling actual file storage. The stream layer coordinates data replication and ensures durability by maintaining the desired replication factor across servers.

**Replication Handling:** If a server fails, the stream layer replicates files to other servers to maintain the replication factor.

## Frontend Layer
- Handles user requests, performs authentication, and forwards requests to the partition layer.
- For large file transfers (e.g., a 1GB file), S3 uses chunked transfer encoding, which splits the file into smaller parts (like 1MB) and streams the data.

## Partition Layer
- Determines the partition (key range) responsible for storing a particular object.
- Uses **range-based hashing** to group keys lexicographically, ensuring that related objects are stored together on the same server.

- When the frontend receives a request, it identifies the partition server that owns the requested key and fetches the data from the appropriate storage node.

## Stream Layer
- The core storage layer in S3, responsible for distributing files across servers.
- When a file is requested, the stream layer’s coordinator identifies the locations where the file is stored and retrieves it.
- **Replication:** In case of server failure, the stream layer ensures data is replicated to maintain the desired replication factor.

## Storage Type & Durability
- S3 is optimized for **write-heavy** operations and primarily used for cold storage (e.g., audit logs). Therefore, S3 avoids using SSDs due to cost and instead employs **log-structured file systems**, which minimize disk seek times and optimize write performance.

- **Data Durability:** Replication ensures data durability both within a storage stamp and across storage stamps. To ensure data durability across stamps, asynchronous replication is used, which is preferred for efficiency.

## Multi-Tenancy and Rate Limiting
The partition layer helps to maintain multi-tenancy by isolating load spikes. Rate limiting can be applied at the partition level to ensure that a high load on one bucket doesn’t affect others. This design prevents the need for a centralized rate limiter, improving availability and fault tolerance.

## Handling Failures
1. **Partition Layer Failures:** The file system coordinator detects partition server failures and redistributes the affected files to maintain the replication factor.
2. **Partition Manager Failures:** A leader-worker strategy with leader election ensures the partition manager’s availability.