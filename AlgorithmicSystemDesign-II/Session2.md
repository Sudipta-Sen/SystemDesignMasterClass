#  Uber Driver-Rider Matching System Design

## Overview

The Uber app continuously collects location data from both drivers and riders via heartbeat API calls (e.g., every 5 seconds). These updates are POST requests containing geolocation data, and they are ingested by the backend system for real-time matching. The challenge is to ensure low-latency, high availability, and fault tolerance while maintaining scalability.

## Write Path: Ingesting Heartbeats

1. Database Choice

    A fast, geo-query-supporting data store is essential. Redis fits well because of its in-memory speed and geo-spatial capabilities.

2. Optimizations
    - **Batching Writes:** Avoid writing every heartbeat individually. Batch them over time windows (e.g., 5 seconds) to reduce write load. Some level of data staleness is acceptable.
    - **Geo Partitioning:** Partition data by city since riders never request drivers from other cities.
        - Use a config store (e.g., Zookeeper) to map cities to Redis instances.
        - When a location update arrives, the API server checks Zookeeper to identify the target Redis server.

3. Fault Tolerance
    - If a Redis node for a city fails, only that city is affected temporarily.
    - Since clients emit heartbeats every 5s, the system self-recovers after node restarts.
    - To avoid downtime, add read/write replicas.

## Read Path: Matching Drivers to Riders

1. Matching Service
    - Stateless, hence horizontally scalable.
    - A matching request like "Find me a ride from Marathahalli to Sarjapura" is routed to the Redis node for that city (e.g., R1 for Bangalore).
    - Redis returns nearby drivers using its geo-spatial querying capabilities.
2. Scalability Concerns
    - Some Redis nodes may receive disproportionately high load (hotspot cities).
    - Use read replicas for scaling, though with some replication lag. For our use case, a few seconds of lag is acceptable.
3. Response Filtering
    - Matching service does not send all nearby drivers to the user.
    - Instead, it filters based on:
        - Vehicle type
        - Payment preferences (e.g., cash/card)
        - Ratings
        - Driver preferences (e.g., no airport rides after 10 PM)
4. Implementation Challenge
    - Redis lacks a SQL-like `WHERE` clause.
    - Options:
        - Build composite keys (not scalable)
        - Filter in the application layer (inefficient)
        - Use **Lua scripts (EVAL)** in Redis to do in-DB filtering.

## Scaling Lua with Redis: The EVAL.RO Revolution

- **Problem:** `EVAL` doesn’t run on read replicas (write-only command).
- **Solution:** Use `EVAL.RO`, a Gojek-proposed enhancement allowing **read-only Lua scripts** on Redis replicas.
    - Commands like `GET`, `GEOSEARCH`, etc., are supported.
    - Requires SDK changes (e.g., Go Redis SDK patched by Gojek).
- **Benefit:** Redis read and write workloads can now be scaled independently.

## Ensuring High Availability
1. Parallel Query Execution
    - Break complex queries into sub-queries based on criteria (e.g., driver rating 4–4.5, 4.5–5).
    - Execute them in parallel on different replicas.
2. Handling Hotspots
    - If one Redis node becomes overloaded (e.g., BLR & Mumbai on R1), redistribute city assignments.
    - No downtime data migration needed:
        - When cities are moved (e.g., BLR → R3), the node starts empty.
        - Within 5 seconds, it will be populated by new heartbeat updates.
    - Typically, move **smaller cities first** to reduce load.

## Key System Design Principles
- High Availability > Strong Consistency
- In-memory store with geo-queries is a must
- Multi-master with read replicas for both performance and fault tolerance
- Redis with EVAL.RO makes real-time, filtered geo-matching scalable
