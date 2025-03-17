# Designing Cricbuzz's Live Commentary System

## Requirements:
1. **Live Text Commentary:** The system should display live commentary updates to users during matches.

2. **Cost Efficiency:** The architecture must be designed to handle millions of users while keeping the infrastructure cost low.
3. **Good User Experience:** Ensure that the system provides a seamless and responsive user experience.

## Initial Architecture Overview:
- Cricbuzz maintains multiple API servers that communicate with a database to store match data.
- APIs like `/match/<match_id>/commentary` are exposed to users to fetch live commentary.
- Every user request fetches the most recent commentary from the database.
- **Challenge:** Cricbuzz is one of India’s most visited websites, with enormous traffic directed toward these APIs, creating a performance bottleneck as every request accesses the database directly.

## Introduction of Redis Cache:
To reduce database load, we introduce Redis as a caching layer between the API servers and the database.

1. Caching Strategy:

    - Redis stores the match commentary with match_id as the key and a list of commentaries as the value.
    - When a request hits the API server, it first checks Redis. If data is found, it returns the commentary from Redis.

    - If Redis does not contain the data (cache miss), the API server fetches the commentary from the database, caches it in Redis, and returns it to the user.

2. Handling Redis Failures:

    - If Redis goes down and comes back up, it is treated like a cold start.
    - API servers fall back to the database to repopulate Redis with fresh data when Redis resumes.

## Data Freshness Challenge:
- **Problem:** Redis might serve stale data if new commentary is added to the database, but Redis is not updated.
- **Solution:** The order of commentaries can be maintained by sorting them based on `ball_id` or `timestamp` in Redis. The core challenge is how to ensure Redis contains the latest data after updates in the database.

## Updating Redis:
When new commentary is added or existing commentary is edited, Redis must be updated synchronously with the database.

1. Commentary UI for Commentators:

- Commentators watch live matches and add commentary through a UI that includes:
    - Ball number (dropdown or prefilled)
    - Commentary text
    - Save button
- If commentary for a specific ball already exists, the text box displays the existing commentary for editing. Otherwise, it is blank.

2. Synchronous Update:

    - When commentators save new commentary, two API calls are made:
        - One to update the database.
        - One to update Redis.
    - **Handling Failures:** If Redis is down, the system retries the Redis update in the background while proceeding with the database update.

3. Error Handling for Commentators:

    - If Redis fails, the system doesn’t show an error to the commentators but logs the error for retries.
    - In some cases, commentators can be prompted to retry the update manually, ensuring they are confident that users see the latest updates.

## Redis Recovery:

When Redis comes back online after downtime, there are two approaches for restoring data:

1. **On-Demand Caching:** When Redis is empty, the API server fetches data from the database upon receiving a new request and caches it in Redis.
2. **Proactive Initialization:** Upon Redis recovery, the latest k commentaries of ongoing matches can be loaded into Redis using a simple SQL query.

## Redis Scalability:

- **Scaling Redis:** One Redis node may not be sufficient to handle all requests. The solution is to use Redis in cluster mode:
    - Updates are sent to the master node, which propagates changes to the child nodes.
    - Reads can be scaled horizontally by distributing the load across multiple Redis servers.

## Redis vs API Server Caching:

Given the finite number of matches and balls bowled per day, commentary data could theoretically be cached directly in the API server without Redis. However, this introduces challenges in cache invalidation:

- When commentary is updated in the database, the API server cache must also be invalidated to prevent stale data from being served.
- **Conclusion:** Redis is still necessary for handling invalidation and scaling purposes.

## API Protocol: HTTP Polling vs WebSockets:

To update commentary data from Redis to the user-facing website, we need to decide between two approaches:

1. **HTTP Polling (Pull-based):** The API server polls Redis every 2-3 seconds for updates. This method is simple, reliable, and cost-effective.

2. **WebSockets (Push-based):** WebSockets provide real-time updates by pushing new data directly to the API server as soon as it’s available in Redis. However, they come with several overheads:

    - Persistent connections
    - Edge server requirements
    - Message brokers for communication
    - CPU usage and scalability challenges
    - **Conclusion:** HTTP polling is more reliable and cost-efficient for this use case. Real-time reactivity is not a critical requirement as users are generally content with slight delays.

## Using CDN for Scalability:
To handle large geographical areas and high traffic volumes, we can introduce a CDN (Content Delivery Network) to serve commentary updates.

1. Advantages of CDN:

    - CDNs are cheaper than API server + Redis infrastructure.
    - They have built-in mechanisms to handle request deduplication, request hedging, and cache invalidation.
    - This reduces the load on the API server and Redis while ensuring scalability.

2. Cache Invalidation:

    - CDN caches commentary data and invalidates it after a few seconds to ensure updates are propagated correctly.
    - **Cache Pre-fetching:** CDNs can proactively fetch fresh data from the API server before the TTL (Time-to-Live) expires, minimizing the number of requests that hit the API server.

3. CDN vs Redis: 

    - Using a CDN reduces the number of API servers and Redis instances needed, lowering infrastructure costs. TTL management can be handled either at the CDN level or via HTTP headers in API responses.

## Architecture with CDN and Redis:
1. **Commentary Flow:** The user request goes to the CDN first, which caches live commentary updates. If the CDN cache is stale, the request is sent to the API server, which fetches fresh data from Redis or the database.

2. **CDN Advantages:**
    - Handles high traffic volumes.
    - Offloads requests from the API server and Redis.
    - Proactively manages cache freshness, ensuring users always see the latest updates.

## Handling Multiple Data Types:
- Commentary may include different elements such as text commentary, messages (e.g., "Rain stopped the match"), end-of-presentation details, or player stats.
- These elements can be ordered and displayed based on timestamps to maintain proper chronology.

## Summary:
- The live commentary system for Cricbuzz must handle high traffic efficiently while ensuring users see up-to-date commentary.

- Redis caching reduces load on the database, but we must ensure cache updates occur synchronously with the database.
- Using HTTP polling ensures reliability and cost-efficiency, while introducing a CDN further enhances scalability and reduces infrastructure costs.
- The overall architecture balances performance, cost-efficiency, and user experience by combining Redis, CDN, and well-structured caching mechanisms.