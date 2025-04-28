# Cardinality Estimation for Ad Impressions and Analytics

## Motivation
When measuring engagement (reactions) on a post is easy, tracking impressions (who viewed but didn’t react) is difficult.
Examples:
- News websites: no reaction buttons but need to measure readers.
- AdTech (like Google Ads): need to report number of views and reactions to advertisers.

We aim to count unique users (not track their identities) over a user-selected time window (flexible querying).

We aim to **count unique users** (not track their identities) over a **user-selected time window** (flexible querying).

## Initial Naïve Approach: Using Sets

- Idea: Use a Set to collect unique user IDs.
- Problem:
    - Set has no time awareness: Can't fetch data for a custom time frame.
    - Memory-heavy: Set stores full user IDs (4B per ID + tree overhead).

- Memory Calculation:
    - 100K unique users per minute.
    - 4B per ID → + tree overhead = ~80KB per minute.
    - 1 day → 24×60×80KB ≈ 110MB per ad per day.

Millions of ads → Unscalable.

## Problem Nature: Space vs Time vs Correctness
- Space optimization conflicts with time.
- Both can't be fully optimized unless we sacrifice correctness (allow tiny errors).

    → Introduce Approximate Cardinality Estimation.

## Solution: HyperLogLog
- HyperLogLog (HLL) is a probabilistic data structure.
- Based on Flajolet-Martin Algorithm.
- Operations supported: `Add`, `Union`, `Count`.
- Does NOT support: `Delete`, `Intersect`.

## Implementation Using Redis HyperLogLog

Redis commands:

- PFADD <source_key> <user_id>
- PFCOUNT <source_key>
- PFMERGE <destination_key> <source_key1> <source_key2> ...

Data Model:

- Key Format: adID_YYYYMMDDHHMM
- Example: A1_202504261000
- Store per minute granularity.

## Usage Flow:

1. Add View Event:

    ```nginx
    PFADD A1_202504261000 userA
    ```

2. Query Unique Views (timeframe):

    ```nginx
    PFMERGE temp A1_202504261000 A1_202504261001 A1_202504261002
    PFCOUNT temp
    ```
## System High Level Design

- Ingestion Path (Write Path)
    - User sees Ad → API server receives event.
    - API server → Kafka → Async ingestion.
    - Rule Engine (filter bad traffic) —> another Kafka topic.
    - Consumers update Redis HyperLogLog (PFADD).

- Benefits:
    - Async ingestion prevents Redis overload.
    - Rule Engine ensures quality.
    - Kafka ensures durability, reprocessing, scalability.

## Persistence and Durability

- Problems:
    - Redis is in-memory.
    - Need persistent backup.

- Solutions:
    - Dual Consumers:
        - One writes to Redis.
        - One writes raw data to S3/MySQL/PostgreSQL.
    
    - Why Backup First Kafka Topic (pre-Rule Engine)?
        - Ensures replayability if rules change.
        - Filter again from backup if needed.

## Replayability Architecture
- Pull old events from S3 (Backup DB).
- Push into Kafka for reprocessing.
- Apply Rule Engine again.
- Update Redis / persistent stores if needed.

##  Read Path (Analytics Query Path)
- Advertiser queries for ad stats.
- Query Redis for recent timeframe.
- If old data needed → pull from Persistent DB (DynamoDB or others) into Redis.
- Perform PFMERGE and PFCOUNT on Redis.
- Return result to client.

## Sharding and Storage Management

- Why?
    - Redis can't store infinite historical data.

- Strategies:
    - Shard Redis based on adID, time.
    - Move old HLL data to DynamoDB (or ElasticSearch).
    - Use TTL (time to live) for Redis keys.

## Storage Optimization: Aggregation

- Problem:
Querying 4 hours = 240 minute-level keys!

- Solution:
    - Periodic Aggregation:
        - Aggregate minute-level data → hour/day level.
        - Store aggregation in DynamoDB.

- How?
    - Aggregator server fetches minute HLLs from Redis, merges them, stores coarser-grained HLLs into DynamoDB.

## Compute-Storage Separation

- Further optimization:
    - Separate compute Redis cluster for merging/querying (not main Redis).
- Storage: Redis + DynamoDB
- Compute: Separate Redis Cluster
- Benefits:
    - Scale compute independently.
    - Reduce load on main Redis.
