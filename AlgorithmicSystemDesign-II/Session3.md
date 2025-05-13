# Designing the Follow/Following Service in Social Media Platforms

In a social media platform, the **Follow/Following** relationship forms the foundation of user interactions. The goal is to model this efficiently at scale while considering read-heavy access patterns.

## Problem Statement
- Users can follow other users.
- Potentially, every user can follow every other user (N * N relationship).
- Example: With **1 million users**, there can be **1 trillion possible follow relations**.
- This is a **directed graph** problem (A follows C → directed edge A → C).

But **does this need a Graph DB?**
Let's analyze.

## Graph DB vs Relational DB: Why Relational DB Works Better
- Use Case: A user (with 200K followers) tweets. The post needs to be shown to all followers.
- Fetching 200K followers at once is not feasible.
- **Graph DBs struggle with deep pagination:**
    - They are great for recommendation engines, fraud detection, but not for social media scales.
    - Pagination degrades performance significantly after a few pages.
- **Relational DBs handle pagination efficiently** because:

Hence, for **deep pagination and massive scale**, Relational DBs are preferred.

## Data Modeling in Relational DB

### Table Design
- Table: edge
- Columns: `src (source user)`, `dst (destination user)`
- Primary Key: `(src, dst)`

### Access Patterns
1. Get all followings of User A
    ```sql
    SELECT * FROM edge WHERE src = 'A';
    ```
    - Efficient (sequential scan on ordered src key).
2. Get all followers of User B
    ```sql
    SELECT * FROM edge WHERE dst = 'B';
    ```
    - **Inefficient** (requires a full table scan since data is ordered by `src`).

### Optimizing for Followers Query
- Option 1: Create secondary index on `(dst, src)` → doubles memory.
- But both "following" and "followers" queries are important.
- A balanced solution is **needed**.

### Sharding by Source (src)
- Shard DB by src:
    - Query 1 (following) stays efficient.
    - Query 2 (followers) becomes a cross-shard query.

Problem remains for followers query.
- Problem remains for followers query.

## Bidirectional Storage with "State" Column
To solve this:
- Introduce a state column:
    - F → follows
    - FB → followed by

For every relation "A follows B", store two entries:
1. `(A, B, F)`
2. `(B, A, FB)`

### Optimized Queries:
1. Get all followings of B:
    ```sql
    SELECT * FROM edge WHERE src = 'B' AND state = 'F';
    ```
2. Get all followers of B:
    ```sql
    SELECT * FROM edge WHERE src = 'B' AND state = 'FB';
    ```
    Both are **single-shard queries** and avoid cross-shard scans.

### Choosing the Right Primary Key Order

#### Primary Key Options:
- `(src, dst, state)` → inefficient for followers query.
- `(src, state, dst)` → better but still needs improvement.
- Adding priority/position field to sort data for feeds:
    - Recent followers first.
    - Prioritize frequently visited users.

Final Primary Key:
- `(src, state, pos, dst)`

Reason
- Efficient sequential scans by `src` and `state`.
- Ordered by `pos` for fast **priority-based pagination**.
- Keeps `dst` to ensure uniqueness and support efficient "does A follow B" lookups.

### Efficient Pagination

#### Inefficient Approach:
```sql
SELECT * FROM edge WHERE src = 'B' AND state = 'FB' ORDER BY pos LIMIT 10 OFFSET 10;
```
- Offset requires skipping rows → expensive for large offsets.

#### Optimized Approach (Seek Method):
```sql
SELECT * FROM edge WHERE src = 'B' AND state = 'FB' AND pos > :last_seen_pos ORDER BY pos LIMIT 10;
```
- Seek to last read position → B+ Tree lookup → sequential scan.
- Reduces query time from O(N) to O(log N).

## Summary: Twitter's FlopDB Design

- This architecture is similar to Twitter's FlopDB.
- It's a **read-heavy system**:
    - Feed construction happens frequently.
    - New followers/following events are much less frequent.

- By:
    - Using a relational DB,
    - Duplicating data with a `state` column,
    - Choosing the right primary key order,
    - Implementing seek-based pagination,
we achieve high performance at social media scale.