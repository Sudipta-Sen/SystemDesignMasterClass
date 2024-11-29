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


To efficiently manage the removal of expired entries in the key-value store, we need to implement a cron job that periodically removes entries based on their TTL 

## Approach for TTL Cleanup:
The initial cleanup query might look like this:
```sql
DELETE FROM store
WHERE NOW() - created_at > TTL;
```

However, this approach is inefficient because the database has to:

1. **Evaluate TTL for each row:** The query needs to check every row, compute whether it has expired, and then delete the expired rows.

2. **Performance Impact:** The execution time of this query depends on:
    - The number of rows to be deleted.
    - The total number of rows in the database, as the query has to scan all rows to evaluate the condition.

## Optimized Approach for TTL Cleanup:

To improve performance, we can avoid calculating expiration on the fly by **storing the exact expiration time** (`expired_at`) directly in the database when the row is created. This way, the cleanup query only needs to compare the current time to the precomputed expiration time, significantly reducing computation.

### During Row Creation:

Instead of calculating TTL at query time, calculate the expiration when the row is inserted:
```sql
INSERT INTO store (key, value, expired_at)
VALUES ('key1', 'value1', NOW() + INTERVAL TTL SECOND);
```

### Cron Job Cleanup Query:
Now the cron job simply deletes rows where the expiration time has passed:
```sql
DELETE FROM store
WHERE expired_at < NOW();
```
### Benefits of This Approach:

1. **Avoids On-the-Fly Computation:**   
    - Since the expiration time is precomputed and stored as `expired_at`, the database only needs to compare this value to the current time (`NOW()`), making the query much faster. It reduces significant time when database grows in size.

2. **Index Creation on expired_at:** 

    - By creating an index on expired_at, the database can quickly locate rows that have expired
    
    - Indexes are typically stored in cache, so the database can efficiently access only the rows that need to be deleted, rather than loading all rows into memory.

    - This approach avoids both **a full table scan** and **a full row scan**, improving performance by directly targeting the rows that need to be removed.

    - The index allows for quick identification of expired rows, minimizing the time spent searching and also reducing the overhead of loading unnecessary data into memory.

## Modified Database Schema:
Now we will change our DB schema to - 
- Key (`VARCHAR(128)`)
- Value (`BLOB`, `TEXT`, etc.)
- Expired_at (timestamp of when the key will expire)

## Efficiently Delete Expired keys:

When data expires (e.g., 1 million rows), deleting all the expired rows at once is time-consuming and can cause performance bottlenecks. If we try to delete all 1 million rows in a single query, it will take a lot of time, and the database will be heavily occupied with this task, potentially slowing down other operations. So, we need a better solution for handling large-scale deletions.

### Attempted Solution: Soft Delete

One possible solution is to implement a soft delete. In this approach:
- A new column (`is_deleted`) is added to the table.
- Instead of deleting rows, the cron job would mark the `is_deleted` column as `True` for 1 million rows.
- However, this approach still has the same problem: `updating 1 million rows` is still very time-consuming, as the database will need to stop and handle all these updates.
- Additionally, soft-deleted data will still accumulate in the database, and eventually, we will need to hard delete these rows, which does not solve the original problem.

Thus, soft delete doesn’t effectively address the problem of large-scale deletion.

### Optimized Solution: Deleting in Batches

A more efficient solution is to limit the number of rows deleted in each batch. Instead of trying to delete all expired rows at once, we can delete, for example, 1000 rows at a time.

#### Batch Deletion Process:

- The cron job will run periodically and delete a maximum of 1000 rows each time.
- The SQL query will look like this:

```sql
DELETE FROM store
WHERE expired_at < NOW()
LIMIT 1000;
```

- **LIMIT:** The `LIMIT` ensures that only a manageable number of rows are deleted in one go, allowing the database to rebalance its structure (e.g., B+ tree) only once for each batch of deletions. This reduces the overall load on the database and prevents long-running queries from causing delays.

### Handling Expired Keys in GET Request:

Since the batch deletion approach leaves some expired keys in the database until the cron job deletes them, we need to handle cases where users might query expired keys. To address this:
1. When a user fires a GET request for a key:
```
- If the key does not exist in the database, return “Key Not Found”.
- If the key exists, check the expired_at timestamp
    - If the current time is before the expired_at, return the value.
    - If the current time is after the expired_at:
        - Return “Key Not Found” to the user. -- "May introduce lazy deletion here"
```

Here we can think of **lazy deletion** i.e When a user requests a key (`GET`), the system checks the `expired_at` value. If the key has expired, it deletes the key and “Key Not Found” is returned. However, this approach comes with its own drawbacks.

### Problem with Lazy Deletion
- Lazy deletion only deletes one row at a time, and in the worst-case scenario, each delete operation could trigger a `B+ tree rebalancing`. Frequent rebalancing can make each operation more time-consuming, significantly affecting performance.

### Best Practice: Combining Batch Deletion and API-Level Expiry Checks
The optimal solution is to **delete rows in batches** (as described earlier) while also implementing **API-level checks** for expiration when handling `GET` requests. This ensures that:
- The database remains clean over time, as expired rows are removed in regular batches.
- Users don’t retrieve expired data, as the API performs the necessary checks and deletes any expired keys in batches.

## Handle `DEL` Request:

When users send a **DEL** command to delete a key from the database before its expiration, the typical approach is to immediately delete the row. However, this can be inefficient:

- Hard Deletion Problem:
    - If we perform a hard delete (i.e., physically remove the row), the database may need to **rebalance the B+ tree** for just one deletion, which can be time-consuming, especially if this happens frequently for many individual rows.

### Solution: Soft Deletion without an Extra Column
To avoid unnecessary rebalancing for each deletion, we can implement soft deletion. The benefit here is that we don't need to introduce an additional column (like `is_deleted`). Instead, we can repurpose the existing `expired_at` column to mark keys as "deleted" by setting their expiration time to `-1`.

When a user fires a **DEL(K)** command:
- **DB Query:**
    ```sql
    UPDATE store
    SET expired_at = -1
    WHERE key = K;
    ```

### How This Solves the Problem:
1. **No Immediate Deletion:** The key is not deleted immediately. Instead, the `expired_at` column is updated to `-1`, indicating that the key is considered deleted. This will also not cause any B+ tree rebalancing.

2. **No Changes to Cron Job or GET Request:**
    - The cron job that periodically deletes expired keys will automatically delete the soft-deleted keys during the next cycle because the condition `expired_at < NOW()` will still hold true.

    - Similarly, the GET request logic remains the same. When the API checks `NOW() < expired_at` for a deleted key, it will return "Key Not Found," as the condition will be false.

## Handling PUT (Insert/Update) Operation:
The `PUT` operation involves either updating the value of an existing key or inserting a new key-value pair if it doesn’t exist. the standard approach involves multiple steps:

- Steps
    - First, check if the key exists (`GET(K)`).
    - If the key exists, update its value (`UPDATE(K, V)`).
    - If the key doesn't exist, insert the new key-value pair (`INSERT(K, V)`).

### Problem with This Approach:
- Each operation (GET, UPDATE/INSERT) requires two network round-trips, which can be inefficient. Let's assume:
    - Each round trip takes 2ms.
    - The overall process (GET + UPDATE/INSERT) will take 4ms.
    - Additionally, each operation must be wrapped inside a transaction, adding 4ms for starting and ending the transaction.
    - So the total `PUT` query execution time becomes 8ms, which is huge.

###  Optimized Solution: Using UPSERT (REPLACE in MySQL, UPSERT in PostgreSQL)
Instead of performing multiple steps, we can use an UPSERT statement, which combines the logic of updating the key if it exists and inserting a new key if it doesn't:
- **DB Query:**
    ```sql
    UPSERT INTO store (key, value, expired_at)
    VALUES ('K', 'V', NOW() + INTERVAL 30 MINUTE);
    ```

### How This Solves the Problem:
1. Reduced Network Round-Trips:
    - Only one round-trip to the database is needed, reducing the total time from 4ms to 2ms.

2. No Explicit Transaction Management:
    - The database automatically wraps the single UPSERT command inside a transaction, eliminating the need for the API server to manage transaction start and end signals.

## GET Operation
For the GET(K) operation, we want to retrieve the value only if the key exists and has not expired:
- DB Query:
    ```sql
    SELECT *
    FROM store
    WHERE key = K AND expired_at > NOW();
    ```
This ensures that only non-expired keys are returned.

## Optimization for DELETE Operation:
We can optimize the **DEL(K)** operation to handle cases where the key is already expired. Instead of always updating the `expired_at` to `-1`, we only perform the update if the key is not already expired:

- Optimized DB Query:
    ```sql
    UPDATE store 
    SET expired_at = -1
    WHERE key = 'K' AND expired_at > NOW();
    ```

### How This Optimizes the Process:
1. Avoid Redundant Disk I/O:
    - If the key has already expired, there is no need to update its `expired_at` value again. This reduces unnecessary disk I/O.
2. Efficient Deletion:
    - By creating an index on the `expired_at` field and assuming it is stored in RAM, we can avoid disk I/O for updates. By selectively updating only valid (non-expired) keys, we minimize disk operations, resulting in improved performance.

## Summarising all the Operations 
- PUT(K, V, TTLTime)
    ```sql
    UPSERT INTO store (key, value, expired_at)
    VALUES ('K', 'V', NOW() + INTERVAL TTLTime MINUTE);
    ```

- GET(K)
    ```sql
    SELECT *
    FROM store
    WHERE key = K AND expired_at > NOW();
    ```
- DEL(K)
    ```sql
    UPDATE store 
    SET expired_at = -1
    WHERE key = 'K' AND expired_at > NOW();
    ```
- Cron Job / Batch Deletion
    ```sql
    DELETE FROM store
    WHERE expired_at < NOW()
    LIMIT 1000;
    ```

Users will only interact with the **PUT**, **GET**, and **DEL** APIs, while the underlying SQL queries will be executed internally. The actual SQL operations are abstracted from the user, ensuring they do not need to know the structure or workings of our key-value store database.

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