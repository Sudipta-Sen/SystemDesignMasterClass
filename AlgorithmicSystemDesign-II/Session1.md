# Geo Proximity Service Design - Efficient Nearby Search at Scale

## Problem Statement:

Given a user’s location (latitude, longitude), find all nearby users within a k km radius. This is essential in services like Uber, Tinder, and Facebook. Since location (x, y) is not unique (multiple users can be in the same location), the naive KNN approach becomes expensive at scale.

## Naive Approach:

Using K-Nearest Neighbors (KNN), we compute the distance from the given point to all others, which leads to a time complexity of O(n) — not feasible at scale where n is very large.

## Divide and Conquer Strategy

Using K-Nearest Neighbors (KNN), we compute the distance from the given point to all others, which leads to a time complexity of `O(n)` — not feasible at scale where `n` is very large.

## Divide and Conquer Strategy:

Initially, we can partition users by country, state, city, and locality. However, within a region, KNN still requires O(n) checks. Precomputing all distances could help if locations were static, but they are highly dynamic in apps like Uber or Tinder, so precomputation is not viable.

## 1D Range Query Approximation:

If we could represent location by a single dimension k, then we can perform efficient range queries like:

```sql
SELECT * FROM users WHERE location BETWEEN x-5 AND x+5;
```

For efficient range queries, we can use data structures like B+ Trees that support fast lookups and range scans:

- Find `x` in O(log n)
- Scan O(K) for K qualifying users

B+ Trees are used by MySQL, MongoDB, DynamoDB, etc. Other structures like skip lists and segment trees also support 1D range queries efficiently.

## The Dimensionality Problem:

The Earth is a 2D surface (lat, lon), and range queries in 1D aren’t sufficient. Two possible strategies:

1. Convert 2D coordinates into a single dimension.
2. Do separate range queries on `x` and `y`, and take intersection.

**Why #2 doesn’t work:**
If `x` and `y` are stored independently, querying on `x` without `y` leads to logically infinite matches on `y`, breaking proximity constraints.

## Solution: Space-Filling Curves (1D Mapping)

We need a function `f(x, y) → z` that maps 2D coordinates to 1D while preserving spatial proximity:

- If two points are close in 2D, their `z` values should also be close.
- If two points are far, `z` values should also be far apart.

This leads us to **space-filling curves** like:
- Hilbert Curve
- Z-Curve
- Morton Order (Z-order)
- Reverse N Curve

These curves help preserve locality when flattening 2D to 1D.

## GeoHash and Trie-Based Storage

Assume the world is a rectangular grid. We recursively split it and represent locations as binary strings:
- Split: left = 0, right = 1
- More splits = more precision
- A user at a location could be represented by: `10011` (5 splits)

When multiple users fall into the same cell, they share a prefix. To find nearby users:
- Match exact or nearby prefixes using **prefix matching**
- This is efficiently supported using **a Trie**

In Tinder:
- If user A and B have prefix `10011`, they are close.
- If user C is in adjacent box: `10010`, it is slightly farther.
- To include C, broaden search by reducing prefix depth (e.g., `1001*`)

Trie allows efficient search and **progressive expansion:**
- Start with leaf node
- Expand to parent and siblings if not enough results

## Use Case: Google Maps

Moving or zooming around Google Maps is like flipping bits of a GeoHash.
- Top-left/right = flip specific bits
- Each region = "squad"
- Geohash encoding gives us a compact 1D representation of 2D locations

## GeoHash Prefix Match - SQL Implementation

Simple SQL for finding users near a location:
```sql
SELECT * FROM people WHERE substr(geohash, 1, 5) = 'tdrlv';
```
tdrlv is the prefix GeoHash of Bangalore.

## Monetization Possibilities
1. **Progressive Zoom Charges:** Charge users based on zoom level or traversal depth (Google Maps-like pricing).

2. **Premium Listings:** For Tinder/Maps, maintain multiple lists at each trie node:
- Premium (Gold, Silver, etc.)
- Free Users
- Show premium first regardless of distance constraints

E.g., A restaurant paying more may show up even if the map is zoomed out.