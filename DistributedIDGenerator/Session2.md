
## Central ID Service with Batching: Amazon's Approach

In the context of distributed systems, we often require globally unique IDs. To meet this demand, an **ID generation service** pre-generates a pool of unique IDs. When an API server requests an ID, the service simply hands over a portion from the pre-generated batch.

But why do we need these IDs?

Take the **Order** or **Payment** service as examples. These services typically store records in **sharded databases**. Each shard must assign unique identifiers to the records. Without globally unique IDs, there would be ambiguity. 

To resolve this, API servers need to ensure that the IDs they assign to new orders or payments are unique across all shards. By using a central ID generation service, the servers can safely distribute unique IDs across their respective databases, even in distributed or sharded setups.

### Why API Servers Need ID Batches:
- API servers (like those for Payment or Order services) use the IDs to uniquely identify records across all database shards.

- Instead of making a network call to the ID generation service for every request (which would introduce network delays), the API servers request batches of IDs (e.g., 500 at a time) when they boot up or when their current batch is nearly exhausted.

### ID Distribution:
- The ID generation service hands out monotonically increasing IDs. For example, it may allocate ranges like [0, 500] or [1000, 1500] to a server.

- These ranges are stored in a SQL table (with columns for the service name and allocated range) and the current counter, ensuring that distributed services receive unique, non-overlapping ranges.

### Handling Unused IDs:

When a server is assigned a range of IDs and it crashes or shuts down, upon reboot, it will request new IDs. This can result in previously allocated IDs remaining unused.

To mitigate this, one option is to keep the ID batch size small, so that losing a few IDs due to server failures isn't a significant issue. However, a smaller batch size may lead to the server exhausting its allocated IDs more quickly. In such cases, the API server will need to repeatedly request new IDs from the ID generation service, which can introduce network delays.

The optimal solution is to balance the batch size to minimize both the risk of unused IDs and the network delays caused by frequent requests.

### Challenges with Monotonicity:
- While the IDs are monotonically increasing within each batch, monotonicity is not guaranteed across all API servers. For example, if server1 is assigned [1, 500] and server2 [501, 1000], alternating requests between the servers could result in an interleaving of IDs (e.g., request1 gets 1, request2 gets 501, request3 gets 2, and so on).

- If monotonicity is required, API servers would need to fetch one ID at a time directly from the ID generation service, but this would introduce significant overhead.

    ![](Pictures/15.png)

## Why Not Use UUID for ID Generation?
While UUID (Universally Unique Identifier) provides a convenient method for generating globally unique IDs, it introduces several performance issues when used as row IDs or unique identifier for each row in databases like MySQL.

### Key Issues with Using UUID:
1. Memory Overhead:

    For optimal database performance, leveraging indexes efficiently is critical as they are stored in memory. Indexes allow databases to quickly compute the final result set and then access the additional required details from disk. The more index rows that can fit in a single memory page, the faster the query execution.

    However, longer IDs, such as UUIDs (128-bit, 16 bytes), consume significantly more memory compared to integer IDs (32-bit, 4 bytes). This increased memory footprint means fewer index rows can fit into a single memory page, which can slow down query execution.

    For example, if you use UUIDs as row IDs along with an indexed column (e.g., age), the memory required to store a single row in the index would be 20 bytes (16 bytes for the UUID + 4 bytes for the indexed column). If the row ID were an integer, the index size would be 8 bytes (4 bytes for the integer ID + 4 bytes for the age).

    This is why shorter, monotonically increasing IDs (like integers) are generally preferred, as they help maximize the number of rows that can fit in memory, boosting overall database performance. 

2. B+ Tree Rebalancing:

    - UUIDs are not monotonically increasing, meaning they are randomly distributed. In a **B+ Tree index** (used by MySQL), inserting a new UUID might require inserting the ID in the middle of the tree, leading to **frequent rebalancing** of the B+ tree. This rebalancing introduces overhead and can significantly slow down the database, especially in high-write-throughput applications.

---------------------------

> Disclaimer: To understand the performance impact, create two MySQL tables—  
>- one with a monotonically increasing integer ID
>- another with UUID as the ID

> Then insert 10 million rows into each. We can observe that the table with the integer ID performs faster and requires less index space than the table using UUIDs.
---------------------------

### Security Considerations:

- UUIDs offer better security because they are unpredictable, making it harder for an attacker to guess the next ID. On the other hand, sequential integer IDs are more predictable and could expose sensitive information if used improperly.

    ![](Pictures/16.png)

### MongoDB’s Approach:
- MongoDB uses a custom 12-byte unique ID composed of:
    - 4 bytes for the epoch time,
    - 5 bytes for random digits,
    - 3 bytes for a counter.

The timestamp at the start ensures IDs are sorted chronologically, improving performance in certain use cases.
![](Pictures/17.png)

## Example from Flickr’s Ticketing System:
- Flickr avoids using UUIDs for generating IDs due to their inefficiency in indexing. Since UUIDs are 128-bit values, they significantly increase the cumulative size of indexed data, which impacts the memory usage of the database.

    To optimize performance, the cumulative size of indexed data should be as small as possible, as this directly correlates with the **bare minimum RAM** needed for efficient database operations. The smaller the index, the more data can fit in memory, leading to faster query execution.

    ![](Pictures/18.png)

### Flickr's ID Generation System with MySQL Auto-Increment

Flickr employs MySQL’s auto-increment feature to generate unique IDs, which is used for its sharded Post database. This ID generation system works as follows:

- Flickr has a ticketing server dedicated to generating IDs using the auto-increment feature of MySQL, so basically the ticketing server is a MySQL server.
- When a user posts something, the API server interacts with the ticketing server to obtain a globally unique ID.
- This ID is then utilized to store the post in the sharded Post database, ensuring each post has a unique identifier across the shards.

Yes this system may face throughput limitations but it works well within Flickr’s operational requirements.

![](Pictures/19.png)

### MySQL Table Structure

In Flickr's ID generation system, the **ticketing server** relies on a two-column design, consisting of an `id` (which is the primary key) and a `stub`, which is a **single-character field**.

- The `id` column is **auto-incremented**.
- `stub` only contains unique characters.
- To trigger the auto-increment mechanism, **data must be inserted** into the row. Flickr uses the `stub` column to handle this, where a single character is inserted into the `stub` column.

However, the process of **deleting a row** and then **inserting** a new character into the stub column just to generate a new ID can be costly for the database.

![](Pictures/20.png)

### MySQL Optimizations:
- Instead of DELETE and INSERT operations to generate new IDs (which are expensive), MySQL provides two efficient options:
    1. `INSERT … ON DUPLICATE KEY UPDATE:` 
        - This tries to insert a row, and if it fails due to a unique key constraint, it updates the row.
        - This is why the `stub` column is made unique. Without the unique constraint, the `UPDATE` clause wouldn't trigger.
        - Example Query:
        ```sql
        INSERT INTO tickets(stub) VALUES('a') ON DUPLICATE KEY UPDATE id = id + 1;
        ```
    2. `REPLACE INTO:`
        - This works by trying to insert a row. If a row with the same unique key already exists, the old row is deleted and a new one is inserted.
        - However, this approach is 32 times slower than the previous one
    
    Therefore, the optimized query becomes:
    ```sql
    INSERT INTO tickets(stub) VALUES('a') ON DUPLICATE KEY UPDATE id = id + 1;
    ```

    Note: The table contains only one row, with two fields: `id`, which stores the latest used ID, and `stub`, which holds a single character `'a'` without any other value. 

    ![](Pictures/21.png)

### Single Point of Failure in Ticketing System
In the current setup -- 
1. The Client makes a request to create a post.
2. The Server sends a request to the Ticketing Server to retrieve a unique ID for the post.
3. The MySQL Server returns the unique ID.
4. The Server stores the post along with the unique ID in the Post Database.

![](Pictures/22.png)

Here MySQL server i.e. the ticketing server can become a single point of failure.

To address this, we add more servers, and a gossip protocol is needed for communication, which leads us to the earlier issue of distributed monotonicity. However, monotonicity is not requirement in this case, allowing for a different approach to handle the failure.

The solution involves using two MySQL database servers, where instead of simply incrementing the counter (`counter++`), they increment the counter by 2 (`counter+=2`). One server starts from 1 and the other starts from 2. As a result, one server generates odd IDs and the other generates even IDs, ensuring that both servers generate unique IDs continuously. This approach avoids the single point of failure, but it sacrifices monotonicity, which is acceptable in this specific scenario.

![](Pictures/23.png)

## Calculate cumulative size of indexes

### PostgreSQL
PostgreSQL provides functions like pg_total_relation_size() to calculate the size of indexes on a table.

To get the size of all indexes on a specific table:
```sql
SELECT pg_size_pretty(pg_indexes_size('your_table_name'));
```

To get the cumulative size of all indexes in the entire database:

```sql
SELECT pg_size_pretty(SUM(pg_relation_size(indexrelid)))
FROM pg_index;
```

### MySQL
In MySQL, the `information_schema` table stores details about the size of indexes.

To calculate the size of all indexes in a specific table:
```sql
SELECT table_name AS `Table`, 
       ROUND(SUM(index_length) / 1024 / 1024, 2) AS `Index Size (MB)`
FROM information_schema.tables
WHERE table_schema = 'your_database_name'
GROUP BY table_name;
```
This will give the index size per table, and you can sum it up to get the total index size for the whole database.

## Snowflake ID Generation: Twitter and Instagram Use Cases

### Snowflake: Twitter
Snowflake is an ID generation algorithm designed to generate unique identifiers for Twitter objects such as tweets, users, and messages. These IDs are 64-bit integers broken into three parts:

1. **41 bits:** Epoch timestamp in milliseconds.

2. **10 bits:** Machine ID, which represents the API server's unique identifier. The challenge is ensuring that every new server that starts up has a distinct machine ID. Since 10 bits are allocated for this purpose, it allows for 1024 unique machine IDs. When a new server spins up, it must communicate with a configuration database to acquire the first available machine ID. This process is similar to the pattern of allocating limited resources, such as users competing for a limited number of seats in a system. The server requests an ID, and the configuration database assigns one that is available, ensuring no two servers have the same ID.

3. **12 bits:** A per-machine counter that can generate 4096 unique IDs per millisecond.

This setup allows each server to generate 2^12 (4096) unique IDs in 1 millisecond. Since each app server has a unique machine ID, there’s no collision across servers.

![](Pictures/24.png)

The snowflake ID generator runs as a native function in the application server, eliminating the need for network calls or external dependencies, making it highly efficient. While this approach doesn’t guarantee strict monotonicity, the IDs are roughly sortable by time since the timestamp is part of the structure.

Twitter uses **cursor-based pagination** instead of limit-offset pagination. In this method, you provide the ID of the last tweet you viewed, and the Twitter API returns the tweets that follow that ID. Similarly, DynamoDB uses a combination of a hash key and a sort key for pagination. In the API response, it provides a **"last evaluated key"** and an **"exclusive start key,"** which act as markers to retrieve the next set of results. This approach is more efficient than limit-offset pagination because it directly navigates to the next data chunk, avoiding the need to scan skipped records.

![](Pictures/25.png)

### Cursor-Based Pagination vs. Limit-Offset Pagination
**Cursor-Based Pagination** is more efficient than limit-offset pagination because it leverages the structure of the B+ tree, which stores leaf nodes ordered by the primary key. When performing cursor-based pagination (e.g., using `id > n`), the system takes logarithmic time (`logn`) to find the starting point and then fetches the next `k` records sequentially, resulting in a time complexity of `logn + k`.

In contrast, Limit-Offset Pagination requires the system to scan and skip records. For instance, if you want to retrieve rows after skipping the first 200, the system must evaluate and skip those 200 rows before fetching the needed data, which is less efficient. This additional scanning makes limit-offset pagination slower, especially with larger datasets.

Despite its inefficiency, limit-offset pagination is necessary for cases where complex filters (e.g., multiple `WHERE` and `HAVING` clauses) are involved. Some systems, like Elasticsearch, offer both pagination methods. For standard queries, limit-offset pagination is available, but beyond a certain limit, Elasticsearch switches to its **Scroll API** for cursor-based pagination, which stores the result set separately and allows for efficient navigation through the results.

A good analogy is Google search: while there may be thousands of matching results, Google shows only 10 results per page, with users scrolling to subsequent pages as needed—similar to how pagination works with Elasticsearch's Scroll API.

### Snowflake: Instagram

Instagram's version of Snowflake is tailored to meet the platform’s requirements for batch processing and time-based sorting of posts. Like Twitter, they wanted efficient 64-bit IDs for indexing, but they required the ID generation to be sorted by time for displaying posts in a user’s feed.

Instagram’s 64-bit ID structure:

1. **41 bits:** Epoch timestamp in milliseconds (since Jan 1, 2011).
2. **13 bits:** Database shard ID (up to 8096 logical shards).
3. **10 bits:** Per-shard sequence number, allowing 1024 IDs per millisecond per shard.

In 1 millisecond, across 8,096 database shards, the system can generate 1,024 different IDs, totaling around 8 million IDs per millisecond. These 8,096 logical shards are not physical entities but virtual partitions, allowing scalability by logically splitting data across physical servers.

Each physical server hosts multiple logical shards, meaning that while there are fewer physical machines, each holds hundreds of logical shards. Think of creating a database in MySQL as creating one of these logical shards.

This setup makes horizontal scaling seamless since the system treats the database as already partitioned across different servers. This solves the hotspot problem, where a single physical server gets overwhelmed with traffic. For example, if two servers, VM1 and VM2, hold data, and VM2 starts becoming overloaded, the system can identify and transfer less busy shards to other servers, thereby distributing the load efficiently.

Without logical shards, moving data from one machine to another would require scanning and copying individual rows, which is time-consuming. However, with logical shards, you can transfer entire shards in bulk, which is much faster. Additionally, adding new physical servers becomes easier—simply move some logical shards to the new machine, enhancing scalability and load balancing.

![](Pictures/26.png)

Each shard contains a post table (e.g., `insta5.post`), and when a post is created, the `id` column, instead of auto-generating, calls a stored procedure (e.g., `insta5.next_id()`) to retrieve a unique ID. The databases are named `insta5`, `insta6`, etc., with each having its own stored procedure for ID generation, like `insta5.next_id()` or `insta6.next_id()`. The stored procedure calculates the timestamp, shard ID, and sequence number, ensuring the IDs are unique within the platform without relying on an external service.

![](Pictures/27.png)
![](Pictures/28.png)

This approach keeps the ID generation logic within the database itself, making it a great use case for stored procedures in production environments,  proving that they can be valuable even at scale.

----------
> Disclaimer: Arpit shows code here for snowflake in instagram`, do it by yourself. 
--------

### Snowflake: Discord

Discord's implementation of Snowflake is nearly identical to Instagram’s, except that their epoch starts at the beginning of 2015, the year Discord was founded.

![](Pictures/29.png)

#### Advantages of Snowflake Algorithm
1. **Scalability:** Snowflake can generate a massive number of unique IDs per millisecond across multiple servers, enabling distributed systems to scale effectively.

2. **Decentralized:** Snowflake runs as a native function, removing the need for centralized ID generation services or network calls, ensuring low latency.

3. **Customizable:** Companies like Instagram and Discord have customized Snowflake for their specific use cases, such as defining custom epoch start dates or adding unique bits for machine IDs and sequence numbers.

By using Snowflake, companies can avoid the inefficiencies of UUIDs, which are larger in size, harder to index, and may lead to higher disk usage and slower query times in databases. Snowflake's IDs are smaller, efficient, and can be roughly sorted by time, making them ideal for high-performance distributed systems.