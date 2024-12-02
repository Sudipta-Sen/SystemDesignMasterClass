# Caching and Database Optimization

## Caching at Different Levels
1. CPU Cache (L1, L2):

- Quickest access memory, stores frequently used data for fast CPU access.
2. Browser Cache:

- Stores website resources (HTML, CSS, images) locally to speed up website load times on future visits.
3. Static Resources & Local Storage:

- Applications like Slack and Teams store data on the client-side for quicker access.
4. Database Cache & Materialized Views:

- Avoids redundant table joins by caching query results. Risks of staleness (outdated data) can be handled by scheduling cron jobs to refresh.
5. Database RAM Cache (Buffer Pools):

- Ensures frequently accessed data is stored in memory to avoid disk I/O on repeated access. Similar to Java's `Xms` parameter, this defines the initial amount of reserved memory.
6. API Server Cache:

- RAM-based caching at the API level improves response times but needs proper invalidation strategies to prevent staleness and excessive memory usage.
7. Cache at Load Balancer:

- Caches frequent requests at the load balancer level, reducing load on the backend by serving cached responses for repeated requests.
8. Cache at CDN Level:

- The Content Delivery Network (CDN) caches static and dynamic content closer to the user, reducing the time and cost of serving data from the origin server.
9. Runtime Language Cache:

- Runtime caches (e.g., Java HashMap or Python dict) help in caching objects or results closer to the application, reducing network and database calls.
10. Disk Cache (Locally Attached SSDs):

Using the local disk (SSD) for caching is a cheap and fast solution, especially compared to querying data from a remote database.

## Scaling Strategies in System Design

**Scalability:** The ability of a system to handle more load by increasing its capacity.

### Two Types of Scaling:
1. Vertical Scaling:
- Increase CPU, RAM, or disk space on existing servers.
- Pros: Simple implementation.
- Cons: Limited by hardware capacity.

2. Horizontal Scaling:

- Add more machines (servers) to distribute the load.
- Pros: Infinite scalability, fault tolerance.
- Cons: Complex management, needs distributed system design (like sharding databases).

### Scaling Databases

1. Scaling Reads:

- Use Read Replicas to offload read traffic from the master node.
- Replication lag could lead to stale data, so for critical operations, always read from the master node.
2. Scaling Writes:

- Once vertical scaling limits are reached, add read replicas. However, if the write load exceeds the master node's capacity, horizontal scaling of writes is required by sharding the database.

### Sharding Techniques:
- **Range-based Sharding:** Split data based on key ranges (e.g., A-M to Shard 1, N-Z to Shard 2).
- **Hash-based Sharding:** Use a hash function on keys to distribute data across different shards.

## Connection Pooling & Database Proxy
1. Connection Pools:
- API servers maintain two types of connection pools: one for read replicas and another for the master node. Depending on the query type (read/write), the appropriate pool is used.
2. Database Proxy (RDS Proxy, ProxySQL):
- If the database topology is elastic (i.e., dynamic scaling), API servers may not be aware of which nodes to connect to. In this case, a proxy server (like ProxySQL) manages all database connections and routes requests based on predefined rules.