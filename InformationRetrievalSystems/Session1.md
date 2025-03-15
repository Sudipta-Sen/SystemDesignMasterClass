# Designing Recent Searches

## Overview:

When users interact with a search bar on an app or website, they often see a list of their recent searches, improving user experience by allowing them to easily access prior queries. In this design, we'll focus on showing the last 10 searches made by a user. To do this efficiently, we need to think about performance, scalability, and storage, and also ensure the system meets the needs of business analytics.

Arpit, while building a similar feature for Unacademy, shared some valuable insights:

1. 50% of users tap the search bar within the first 5 seconds of app login.
2. 30% of searches happen through recent searches.

With this in mind, the goal is to provide a fast and seamless user experience, while also keeping the system optimized for high traffic. Below is the thought process for designing recent searches.

## Current Search Flow

The search functionality was powered by Elasticsearch. Users had an endpoint /search, and upon making a request, the system passed parameters to the Search Microservice. This microservice would:

1. Construct an Elasticsearch query.
2. Fire the query to the Elasticsearch cluster.
3. Retrieve and return the response.

## Designing Recent Searches
To introduce recent searches, we will create another endpoint, /search/recent. When a user hits this endpoint, the search service will fetch the last 10 searches they made and return the result.

There are two options for fetching the recent searches:

### Option 1: On-the-Fly Computation
- Each time a request for recent searches comes in, we compute the last 10 searches directly from the database (Elasticsearch).

- Pros:
    - Accurate results with up-to-date searches.
    - No need for additional storage.

- Cons:
    - Slower response times since it queries the database each time.
    - Increased CPU load as each request requires computation.
    - High request volume could lead to DB bottlenecks due to frequent queries.

### Option 2: Precomputed Data
- Recent searches are computed ahead of time and stored. When a request comes in, the precomputed data is served directly.
- Pros:
    - Faster response as no computation is needed during the request.
    - Reduces CPU load.
- Cons:
    - Requires additional storage to hold precomputed data.
    - Data may be slightly stale (but for recent searches, this is usually acceptable).

This presents a classic **compute vs. storage** tradeoff. In scenarios where real-time accuracy is critical (e.g., financial transactions), we’d opt for **on-the-fly** computation. But in this case, **precomputed** data is sufficient, since:
- Users are unlikely to notice small inaccuracies in recent search results.
- The search bar is the first interaction point for most users (50% within 5 seconds of login), and it should be fast.

## Storage and Access Pattern
We need a fast-access database to store and retrieve precomputed recent searches. The access pattern is simple: **Given a user ID**, return their last 10 search queries.

### **Estimating Storage:**
- Each user has 10 recent searches.
- Assume each search query is 16 characters long.
- If there are 1 million users, the storage required would be:
    - 16 * 10 * 1,000,000 = 160MB

A **single node** is sufficient for storing this data, but not for handling the query load. With 1M users querying the platform, the database needs to scale to handle the load. Therefore, we should choose a database that supports sharding and partitioning, such as Redis, DynamoDB, or MySQL.

### **Why Redis?**
Redis is optimal for storing precomputed data because:

- Fast access: Redis operates in-memory, providing low-latency data retrieval.
- Cost-efficient: In our case, Redis only needs to store 160MB, which is relatively inexpensive.

However, Redis is in-memory and does not persist data if it crashes. But since the data is not critical and can be recomputed, this is acceptable.

## Persistent Storage for Analytics

While Redis serves as a fast-access store for recent searches, we also need to persist the search data for analytics purposes. Many teams, such as those in analytics, recommendations, and ads, can use this data to:

- Understand user behavior.
- Provide personalized recommendations (e.g., Instagram reels or targeted ads).

### What to Store?
We can store either:

1. **Bounded data:** Storing a fixed number of searches (e.g., 10 recent searches).
    - This is a bounded data problem where we know the size and can use an array or similar structure.

2. **Unbounded data:** Storing all searches made by a user.
    - This requires flattening the data across multiple records. Each query can be stored as a document (e.g., in MongoDB or Elasticsearch).

For unbounded data, we can store:

- User ID
- Search query
- Timestamp
- Geo-location

We should choose a JSON-based DB like DynamoDB, Elasticsearch, or MongoDB to store this semi-structured data.

## Data Ingestion: Synchronous vs. Asynchronous Writes
We have two ways to get search data into persistent storage:

### Synchronous Writes:
1. The search service directly writes data to the database during each search.
2. **Challenges:**
    - Increases latency (search won’t return until data is written).
    - Adds pressure on the database due to frequent writes.

### Asynchronous Writes:

- The search service writes search data to Kafka, and a broker reads from Kafka and writes to the database in batches.

- Advantages:
    - Faster response times for users.
    - Reduced database load as writes can be batched.
    - Kafka has higher throughput than traditional databases.

Asynchronous writes are preferred for this use case since it reduces latency and minimizes the load on the database.

## Choosing the Persistent Database
We need a database that provides:

1. Durability
2. High data ingestion rate
3. Partitioning and sharding support
4. JSON structure storage
5. Geo-spatial search capabilities
6. Text-based search features (lemmatization, stemming)

Based on these requirements, **Elasticsearch** is the best fit. It supports text search, analytics, geo-search, and provides partitioning/sharding, making it ideal for both real-time searches and analytics.

## Optimizing for Cost

### Reducing Storage Costs:
1. **Compression:** Not effective here as individual search queries are small.
2. **Archiving:** After a certain period, older search queries can be moved to cheaper storage like Amazon S3. This fits well with the concept of hot, warm, and cold storage, where:
    - **Hot storage** (like Redis) holds frequently accessed recent searches.
    - **Warm storage** (like Elasticsearch) holds all user searches.
    - **Cold storage** (like S3) holds older, less frequently accessed data.
### Reducing Redis Costs:
Combining Redis with persistent storage could increase latency due to slower access times from disk-based databases like DynamoDB or MongoDB. Since Redis is inexpensive for our use case (160MB of data), it remains the most cost-effective solution.

## Conclusion
To build a robust recent search feature:

1. Use **precomputed data** stored in Redis for fast, frequent access.
2. Write search data asynchronously to a persistent store like Elasticsearch using Kafka to reduce DB pressure and support analytics.
3. Optimize storage by archiving older data in cold storage and keep Redis as the fast-access cache for recent queries.

By adopting this architecture, we achieve both speed and scalability, while supporting rich analytics and reducing overall infrastructure costs.