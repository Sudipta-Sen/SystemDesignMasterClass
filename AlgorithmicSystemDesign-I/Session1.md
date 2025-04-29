# Cardinality Estimation for Ad Impressions and Analytics

## Motivation
The importance of these metrics becomes clear when we think about counting reactions on a post, which is relatively easier. However, many people view a post without reacting to it — so how do we measure that?

Examples:
- News websites: no reaction buttons but need to measure readers.
- AdTech (like Google Ads): need to report number of views and reactions to advertisers.

    ![](Pictures/1.png)

We aim to **count unique users** (not track their identities) over a **user-selected time window** (flexible querying).

## Use Case in AdTech Firms

Let’s consider an example from adtech companies, such as Google Ads. Whenever someone runs a campaign on Google Ads, we need to show them two key metrics over time:
- The number of views (impressions) their ad received, and
- The number of reactions (if applicable)

In this section, we won’t focus on how to plot these numbers as a time series graph. Instead, we’ll discuss how to calculate these metrics for a specific time range.

Given a particular timeframe, our goal is to show the count of unique users who viewed the ad — not the user details, just the total count.

![](Pictures/2.png)

From the above diagram, it is clear that when calculating the total number of unique users between time A and time B, we cannot simply split the interval and say:

`Total unique users from A to B = Total unique users from A to C + Total unique users from C to B (where A < C < B)`

This is because the same user might appear in both intervals (A to C and C to B), leading to double-counting if not handled carefully.

## Initial Idea: Using a Set for Uniqueness

The first data structure that comes to mind when we hear "unique" is a **Set**.

So, a basic idea would be:
- Every time a user visits an ad, we insert their user ID into a set.
- The size of the set would give us the count of unique users.

However, the problem is that **users can select arbitrary timeframes** to view unique counts. A single set per ad would not be sufficient because it doesn’t account for **time granularity** — we cannot filter visits based on time.

#### **How can we solve this? (Hint: Key-Value pairs)**

Let’s maintain one key per ad per minute, where:
- The key could be in the format `adId_timestamp`, e.g., `a1_202504241301`, `a1_202504241302`, etc.
- This key represents minute-level granularity — for example:
    - `a1_202504241301` → 24th April 2025, 13:01
    - `a1_202504241302` → 24th April 2025, 13:02
- The value for each key is a Set containing **user IDs who visited the ad during that specific minute**.

Now, when a user asks for the unique user count between a custom time range:
- We fetch all relevant sets corresponding to the selected minutes.
- Perform a union of these sets to get the overall unique users.
- Return the size of the resulting set.

#### **But is this scalable? Let's do some math:**
- Assume each user ID is 4 bytes long.
- Suppose each minute an ad is viewed by 10K unique users.

Then, per minute:
- Set size (just data) = 4B × 10K = 40KB
- However, a Set is typically backed by a tree or hash structure internally, with additional overhead (pointers, metadata, etc.).
- This overhead could easily make the in-memory size around 80KB per 10K users 

Per ad, per day:
- Total number of sets = 24 hours × 60 minutes = **1440 sets**.
- Total size ≈ 1440 × 80KB ≈ **110MB per ad per day**.

Now imagine millions of ads running for multiple days — the storage requirement becomes massive!

Moreover:
- Set union is computationally expensive.
- It cannot be efficiently done on disk; **sets must be loaded into memory**.
- Union requires iterating through all elements, making it both time- and memory-intensive.

**Can we optimize this further?**

## Space vs Time vs Correctness — A Trilemma

Let’s think carefully:
- We don't actually need to know who saw the ad.
- We just need to know how many unique users saw it.

However, to calculate **uniqueness**, we still need some form of **user identity**. In system design, there's a common trade-off between space and time:
- If we optimize for one, we often sacrifice the other.
- Ideally, we want to optimize both — but that's only possible if we are willing to **give up on 100% correctness**.

So, the reality is a trilemma:
- **Space vs Time vs Correctness**

If a small error in the unique user count (say, 0.01%) is acceptable, then it’s a practical compromise.

This problem is a classic example where giving up a little correctness can massively save on space and time.

#### **The idea: presence over identity**

We don’t need to store the actual user IDs — we just need to record their **presence**.

- This is the same principle behind a Bloom Filter:
    - We don’t store the word "apple" itself;
    - Instead, we pass it through a hash function and just mark its presence.

Similarly, for our ad-tech scenario, we need a structure that supports:
- Adding elements
- Merging (Union)
- Getting an approximate count (Length)

We don't need:
- Removal
- Intersection

## Introducing HyperLogLog

- HyperLogLog is a **probabilistic data structure** designed for cardinality estimation (i.e., estimating the number of distinct elements).
- It is based on the Flajolet–Martin algorithm
- It supports the exact operations we need:
    - `add` to HyperLogLog
    - `merge` two HyperLogLogs (union)
    - `count` (approximate unique count)
- However, HyperLogLog cannot perform:
    - remove
    - intersection

#### **Efficiency Comparison**

To give a sense of efficiency:
- A Set that stores user IDs might take 4MB.
- A HyperLogLog representing the same data might only take 12KB.
- Downside:
    - HyperLogLog trades off perfect accuracy for huge savings in space and computation time.

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
