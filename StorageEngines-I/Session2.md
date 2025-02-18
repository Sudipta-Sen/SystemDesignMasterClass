# Distributed Cache Design and Consistent Hashing

When a single cache node cannot hold all the data, we need to distribute the data across multiple nodes. This is achieved using various algorithms such as hash-based routing, range-based routing, or static routing. The goal is to determine the ownership of data and identify which node holds the data for a given key.

## Four Key Patterns in Distributed Cache Systems

1. **Proxy-Based Access (Pattern 1):** The client hands over the request to a proxy, which connects to the appropriate database, fetches the response, and forwards it back to the client. This setup is common in systems like load balancers, reverse proxies, and API gateways.

2. **Direct Access (Pattern 2):** The client knows exactly which database node to request data from and directly sends the request to the appropriate node.

3. **Shard Handoff (Pattern 3):** The client sends a request to any shard, and if that shard doesn’t hold the data, it returns the shard ID where the data is located. The client then sends the request to the correct shard. Redis uses this approach for master node identification.

4. **Distributed Hash Table (DHT) (Pattern 4):** Unlike the previous patterns, where one component knows the entire topology, DHTs operate without any single entity having a full view of the system. DHTs are not typically used for cache, but they are fundamental to the internet's operation.

## Challenges with Hash-Based Routing

Scaling the database by adding or removing nodes can result in extensive data movement due to the redistribution of keys. Consistent hashing helps reduce data movement by determining data ownership but does not eliminate the problem entirely.

## Data Migration Without Downtime

To add new cache nodes without downtime:

- **Approach 1:** Identify keys to be served by the new node and migrate them from existing nodes, but this is complex and difficult to configure.

- **Approach 2:** Dump the entire data from the existing cache to the new cache, introduce the new cache server, and handle replication lag until it becomes minimal. This requires careful coordination to prevent data inconsistencies.

## Consistent Hashing in Load Balancing

In a consistent hashing ring, each node is mapped to a position in an array, which determines where the data for a particular key resides. For example, a key `k1,0` might map to cache1, which holds the data for the key. When adding or removing nodes, consistent hashing minimizes data movement, though it can still lead to hot spots where some cache nodes handle a disproportionate load.

### Virtual Nodes

Virtual nodes solve the problem of skewed data distribution by giving each cache server multiple virtual presences in the hash ring. This helps distribute the load more evenly across the system, but adding new nodes still requires careful management to avoid complexity in operational implementations.

### Stateless vs. Stateful Systems

Consistent hashing works well for stateless systems (e.g., API servers, load balancers) where each node can equally handle requests. However, for stateful systems like caches, it becomes increasingly challenging as data consistency and synchronization between nodes need to be maintained.

### Graceful Shutdown and Abrupt Outages

1. Graceful Shutdown:

    - Take a data dump from the cache being shut down and move it to another cache.

    - Perform dual writes to both caches until the replication lag is minimal.

    - Remove the old cache from the system.

2. Abrupt Outages:
    - **Option 1:** Accept the data loss.
    
    - **Option 2:** Ensure high availability by maintaining replicas of data across multiple nodes. This approach increases storage requirements but allows reads and writes to be served even if one node goes down. Synchronizing data and handling merge conflicts when a node comes back online is crucial, which can be managed using lock-free data structures or conflict resolution techniques.