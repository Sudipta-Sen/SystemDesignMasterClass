# Key-Value Store Design on Relational Database

## Overview

A key-value store retrieves a value based on a given key. In this design, the underlying database is RDBMS, providing the following core functionalities:

- GET: Retrieve a value by its key.
- PUT: Insert or update a value.
- DEL: Delete a key-value pair.
- TTL (Time-to-Live): Set an expiration time for each key.

The key challenges addressed in this design include efficient handling of expired keys, batch deletions, and horizontal scalability.

## Requirements:

- **Key Operations:** Support `GET`, `PUT`, `DEL`, and `TTL`.
- **Synchronous Operations:** All operations should be performed synchronously.
- **TTL Management:** Keys will have an expiration time (TTL), which is set on a per-key basis.
- **Scalability:** The system should be horizontally scalable.
- **Storage Engine:** The underlying storage engine is an SQL-based RDBMS.

## Data Schema:
Initially, a basic schema for the key-value store could look like this:

- Key (`VARCHAR(128)`)
- Value (`BLOB`, `TEXT`, etc.)
- Created_at (timestamp to store when the key was created)
- TTL (time-to-live for each key)

However, for `performance optimization`, the schema is refined to include an `expired_at` column that stores the exact time when the key will expire. This allows faster lookups and avoids unnecessary computations at query time.

- Key (`VARCHAR(128)`)
- Value (`BLOB`, `TEXT`, etc.)
- Expired_at (timestamp of when the key will expire)

## Query for Deleting Expired Keys:

Instead of calculating expiration dynamically at query time, the expiration time is precomputed during key insertion. The cron job for cleanup looks like this:

```sql
DELETE FROM store
WHERE expired_at < NOW()
LIMIT 1000;
```

- **LIMIT:** The `LIMIT` ensures that no more than 1000 rows are deleted at a time, reducing the load on the database.
- **Indexing:** An index on `expired_at` improves query performance, as the database only needs to scan a small part of the table instead of the whole table.

## Handling Key Expiration:

1. Lazy Deletion:
    - When a user requests a key (`GET`), the system checks the `expired_at` value. If the key has expired, it is deleted and “Key Not Found” is returned.
    - This approach ensures that expired keys are removed during regular API calls without needing a separate cleanup process.

2. Immediate Deletion:
    - When a `DEL` command is fired, instead of hard deletion, the key's `expired_at` value is set to `-1`. This effectively marks the key as deleted, and the same cron job for expired keys will later remove it.

    ```sql
    UPDATE store 
    SET expired_at = -1
    WHERE key = 'K';
    ```

## Handling PUT Requests:
The `PUT` operation involves either updating the value of an existing key or inserting a new key-value pair if it doesn’t exist. To optimize performance:

- Use `UPSERT` (also known as `REPLACE` in MySQL) to handle both insertions and updates in a single query.

```sql
UPSERT INTO store (key, value, expired_at)
VALUES ('K', 'V', NOW() + INTERVAL 30 MINUTE);
```
This reduces the overhead of network calls, as there is no need for separate `GET` and `PUT` operations to check whether the key exists before updating.

## Optimization for DELETE:
A small optimization in the `DEL` operation is to check if the key has already expired before updating the `expired_at` value:

```sql
UPDATE store 
SET expired_at = -1
WHERE key = 'K' AND expired_at > NOW();
```

## Storage-Compute Separation:
In modern databases, the storage and compute layers are often separated for scalability. In this design:

- **Storage:**  The storage layer provides durable data storage, typically using an RDBMS (like MySQL).
- **Compute:** The compute layer handles query processing, including filtering, joining, and merging data.

By separating the two, the system can scale the compute nodes (key-value API servers) independently of the storage nodes, allowing for horizontal scalability.

## Scaling Read and Write Operations:

1. Scaling Reads:

    - If the system has more reads than writes, read replicas can be added to offload read requests from the master node.

    - To handle potential replication lag, users can specify whether they require consistent reads (from the master) or eventually consistent reads (from the replica).

2. Scaling Writes:
    - Write operations are more challenging to scale. Once the system reaches its limit, sharding (dividing the database into smaller chunks) is necessary.

    - Sharding strategies can include range-based partitioning (e.g., partitioning based on alphabetical order) or hash-based partitioning (using a hash function to distribute keys across shards).

## Sharding and Topology:

When the database is sharded, each master node is responsible for a specific range of data. To manage data distribution:

- A zookeeper (or similar service) is used to track the topology of the database and assign queries to the correct shard.

- The KV-API server queries the zookeeper to find the correct shard for each request.