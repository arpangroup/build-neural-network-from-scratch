# Location-Based Database System

A complete guide to building and understanding a **Location-Based Database System** used in applications like:

* Ride-sharing apps
* Food delivery systems
* Maps & navigation
* Nearby search
* GIS platforms
* Real-time driver tracking

---

# Table of Contents

1. [Common Geo Queries](#common-geo-queries)
2. [How Location Search Works Internally](#how-location-search-works-internally)
3. [Core Concepts](#core-concepts)
4. [Popular Technologies](#popular-technologies)
5. [Phase 1 — Coordinate Storage](#phase-1--coordinate-storage)
6. [Phase 2 — Distance Calculation](#phase-2--distance-calculation)
7. [Phase 3 — Radius Search](#phase-3--radius-search)
8. [Phase 4 — Bounding Box Optimization](#phase-4--bounding-box-optimization)
9. [Phase 5 — Spatial Indexing](#phase-5--spatial-indexing)
10. [Phase 6 — GeoHash Indexing](#phase-6--geohash-indexing)
11. [Phase 7 — KD-Tree](#phase-7--kd-tree)
12. [Phase 8 — Persistence](#phase-8--persistence)
13. [Phase 9 — Concurrency](#phase-9--concurrency)
14. [Phase 10 — Production Features](#phase-10--production-features)
15. [Recommended Learning Path](#recommended-learning-path)

---

# Common Geo Queries

## Distance Query

Find locations within a radius.

### Examples

* Find restaurants within **2 km**
* Find shops within **5 km**

---

## Nearest Neighbor Query

Find the closest object.

### Examples

* Find nearest hospital
* Find nearest driver

---

## Polygon Query

Search inside a custom boundary.

### Examples

* Users inside delivery zone
* Find users inside city boundary
* Find houses inside polygon area

---

## Nearby Search

### Example

* Show all drivers near me

---

## Route / Path Query

Find the best path between locations.

### Example

* Shortest road path from A to B

---

# How Location Search Works Internally

Traditional database indexes like **B-Tree** are inefficient for geographical queries.

Geospatial systems use specialized spatial indexes such as:

* R-Tree
* QuadTree
* KD-Tree
* GeoHash
* S2 Geometry
* H3 Hexagonal Indexing

These systems partition Earth into searchable regions for efficient querying.

---

# Core Concepts

## 1. Coordinate System

Most geo systems use:

* **Latitude** → North / South
* **Longitude** → East / West

### Example

Hyderabad coordinates:

```text
Latitude  = 17.3850
Longitude = 78.4867
```

---

# Popular Technologies

# SQL Databases

## PostgreSQL + PostGIS

Supports:

* Distance queries
* Polygon search
* Nearest neighbor search
* GIS operations

### Example Query

```sql
SELECT *
FROM shops
WHERE ST_DWithin(
    location,
    ST_MakePoint(78.48, 17.38)::geography,
    5000
);
```

---

## MySQL Spatial

Provides spatial indexing and geo operations.

---

# NoSQL / Search Engines

## Elasticsearch

Supports fast geo-distance queries and geo aggregations.

---

## MongoDB

Supports:

* 2D indexes
* 2DSphere indexes
* Radius search
* Polygon search

---

## Google S2 Geometry

Used by:

* Google Maps
* Uber-like systems

---

## Uber H3

Earth divided into hexagonal cells.

### Advantages

* Fast nearby search
* Easy clustering
* Heatmaps
* Efficient ride matching

### Used In

* Uber
* Swiggy/Zomato-like systems
* Analytics platforms

---

## GeoHash

Encodes coordinates into compact strings.

### Example

```text
tdr1v9
```

---

# Famous Mapping Engines

* OpenStreetMap
* OSRM (Open Source Routing Machine)
* GraphHopper (Java-based routing engine)

---

# Phase 1 — Coordinate Storage

# Step 1: Create Location Model

Each location contains:

* id
* name
* latitude
* longitude

### Example Table

| id | name   | lat    | lon    |
| -- | ------ | ------ | ------ |
| 1  | Shop A | 17.385 | 78.486 |
| 2  | Shop B | 17.390 | 78.490 |

---

# Step 2: In-Memory Storage

```java
class Location {
    long id;
    String name;
    double latitude;
    double longitude;
}
```

---

# Step 3: Add Sample Data

```java
locations.add(
   new Location(1, "Shop A", 17.3850, 78.4867)
);
```

---

# Phase 2 — Distance Calculation

Now we calculate the distance between two coordinates.

---

# Step 4: Learn Haversine Formula

Earth is spherical.

Normal Euclidean distance does not work correctly for geo coordinates.

Use the Haversine Formula:

d = 2r\arcsin\left(\sqrt{\sin^2\left(\frac{\phi_2-\phi_1}{2}\right)+\cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\lambda_2-\lambda_1}{2}\right)}\right)

Where:

* ( \phi ) = latitude
* ( \lambda ) = longitude
* ( r ) = Earth radius

This calculates real Earth distance.

---

# Step 5: Implement Distance Function

```java
public static double distance(
        double lat1,
        double lon1,
        double lat2,
        double lon2) {

    double R = 6371; // Earth radius in KM

    double dLat = Math.toRadians(lat2 - lat1);
    double dLon = Math.toRadians(lon2 - lon1);

    double a =
            Math.sin(dLat / 2) * Math.sin(dLat / 2)
            + Math.cos(Math.toRadians(lat1))
            * Math.cos(Math.toRadians(lat2))
            * Math.sin(dLon / 2)
            * Math.sin(dLon / 2);

    double c = 2 * Math.atan2(
            Math.sqrt(a),
            Math.sqrt(1 - a)
    );

    return R * c;
}
```

---

# Phase 3 — Radius Search

Goal:

Find all locations within a radius.

Example:

* Find all shops within 5 km

---

# Step 6: Naive Search

```java
public List<Location> nearby(
        double lat,
        double lon,
        double radiusKm) {

    List<Location> result = new ArrayList<>();

    for (Location loc : locations) {

        double dist = distance(
                lat,
                lon,
                loc.latitude,
                loc.longitude
        );

        if (dist <= radiusKm) {
            result.add(loc);
        }
    }

    return result;
}
```

### Complexity

```text
O(n)
```

### Problem

* 10 locations → fine
* 10 million locations → terrible

So we optimize.

---

# Phase 4 — Bounding Box Optimization

Before exact distance calculations, reduce the search area.

---

# Step 7: Create Bounding Box

Approximation:

```text
1 degree latitude ≈ 111 km
```

## Bounding Box Formula

- $\Delta lat = \frac{radius}{111}$

- $\Delta lon = \frac{radius}{111 \cdot \cos(lat)}$


### Latitude Range

```java
double latRange = radiusKm / 111.0;
```

### Longitude Range

```java
double lonRange =
      radiusKm /
      (111.0 * Math.cos(Math.toRadians(lat)));
```

Now search only within:

* minLat / maxLat
* minLon / maxLon


Compute box:
```
minLat = lat - Δlat
maxLat = lat + Δlat
minLon = lon - Δlon
maxLon = lon + Δlon
```

🧪 Example
User location:
```
lat = 17.38
lon = 78.48
radius = 5 km
```
Compute:
```
Δlat ≈ 5/111 = 0.045
```

So:
```
lat range: 17.335 → 17.425
lon range: 78.44 → 78.52
```


---


# Step 8: Filter Before Distance

nstead of all restaurants:

```
if (lat in range AND lon in range)
   consider it
```

```java
if (loc.latitude >= minLat
        && loc.latitude <= maxLat
        && loc.longitude >= minLon
        && loc.longitude <= maxLon) {

    // then calculate exact distance
}
```

Huge performance improvement.
- reduces dataset by 95%+
- fast pre-filter

---

# Phase 5 — Spatial Indexing

Stop scanning everything.

This is where real geo databases begin.

---

# Option 1 — Grid Index

💡 **Idea :** Divide Earth into cells.

Each location belongs to a cell.

### Example

Store:

```text
cell_id -> locations
```

---

# Step 9: Create Cell Key

We define:
```text
cellSize = 0.01 degree (~1km)
Cell Size = 1km
```

Convert location → cell key
```
cellX = floor(lat / cellSize)
cellY = floor(lon / cellSize)
key = cellX + ":" + cellY
```

Example:

Restaurant:
```
lat = 17.381
lon = 78.485
```

Cell:
```
cellX = 1738
cellY = 7848
key = "1738:7848"
```

```java
int gridX = (int)(latitude * 100);
int gridY = (int)(longitude * 100);

String key = gridX + ":" + gridY;
```

### Example

```text
1738:7848
```

---

# Step 10: Build Index Map

```java
Map<String, List<Location>> gridIndex;
```

Now nearby search checks only neighboring cells.

Insert:
```java
gridIndex
  .computeIfAbsent(key, k -> new ArrayList<>())
  .add(location);
```

Query:
Instead of scanning everything:
1. find user cell
2. check neighbor cells only (8 surrounding)


Example query:
User cell:
```
1738:7848
```
Search:
```
1737:7847 → 1739:7849
```
Only ~9 cells

Complexity
```
O(k) instead of O(n)
```


Instead of:

```text
10 million locations
```

You search:

```text
few neighboring cells
```

Massive improvement.

---

# Phase 6 — GeoHash Indexing

Professional geo indexing.

---

# Step 11: Understand GeoHash

GeoHash converts coordinates into strings.

### Example

```text
(17.38, 78.48) → "tepgq"
(17.39, 78.49) → "tepgw"
```
Prefix = proximity

Nearby places share prefixes:

```text
tepgq
tepgw
tepgx
```

### Step 1 — Start range
```
lat: [-90, +90]
lon: [-180, +180]
```

### Step 2 — Encode bits
Repeat:
- split latitude → 0/1
- split longitude → 0/1
Interleave bits.

### Step 3 — Base32 encoding
Convert binary → base32 string.


### Example (simplified)
Location:
```
lat=17.38, lon=78.48
```

Binary path:
```
lat: 1 0 1 1 ...
lon: 0 1 1 0 ...
```

Interleave:
```
10110110... 
```

→ Base32:
```
tepgq
```


### Storage
```java
Map<String, List<Location>> geoHashIndex;
```

### Query process
User searches:
```
prefix = "tepg"
```

Search:
```
tepg*
```

---

# Why GeoHash Is Powerful

You can query using prefix matching:

```sql
LIKE 'tepg%'
```

instead of full scans.

---

# Step 12: Implement Basic GeoHash

## Algorithm

1. Split latitude range
2. Split longitude range
3. Encode bits
4. Convert to Base32

---

## Simplified Process

Initial ranges:

```text
Latitude  = -90 to +90
Longitude = -180 to +180
```

Repeatedly divide halves.

Example:

```text
Is latitude > midpoint?
1 or 0
```

Build binary sequence → Convert to Base32.

---

# Phase 7 — KD-Tree

Efficient nearest-neighbor search.

Instead of scanning all points:

We build binary space partitioning tree.

---

# Step 13: Build KD-Tree

Tree alternates dimensions:

* Level 1 → latitude split
* Level 2 → longitude split
* Level 3 → latitude split

### Example

```text
               (17.3)
              /      \
         smaller     bigger
```

### Complexity

```text
O(log n)
```

---

# KD-Tree Node

```java
class KDNode {
    Location location;
    KDNode left;
    KDNode right;
}
```

## Build Tree
Alternate splits:
| Level | Split     |
| ----- | --------- |
| 0     | latitude  |
| 1     | longitude |
| 2     | latitude  |


Example:

Locations:
```
A(17,78)
B(18,77)
C(16,79)
```

Tree:
```
        A (lat split)
       / \
      C   B
```

---

# Step 14: Recursive Insert

```java
if(depth % 2 == 0)
    compare latitude
else
    compare longitude
```

---

# Step 15: Nearest Search

## Algorithm

1. Traverse likely branch (go left/right depending on query)
2. Track nearest distance
3. Backtrack if needed

This is how nearest-driver systems work.


## Example query
User:
```
17.5, 78.4
```
Tree traversal:
- go to A
- check B, C
- update best

Complexity:
```
O(log n)
```

---

# Phase 8 — Persistence

Currently data is memory-only.

Now persist it.

---

# Option A — File Storage

Store JSON:

```json
[
  {
    "id": 1,
    "lat": 17.3,
    "lon": 78.4
  }
]
```

---

# Option B — Custom Binary Format

Store directly as bytes:

```text
[id][lat][lon]
```

Much faster.


Java example
```java
DataOutputStream out = new DataOutputStream(file);

out.writeLong(id);
out.writeDouble(lat);
out.writeDouble(lon);
```
Why?
- fast disk read
- compact storage
- cache friendly

---

# Phase 9 — Concurrency

Support multiple reads and writes.

Use:

* ReadWriteLock
* ConcurrentHashMap

---

# Phase 10 — Production Features

# 1. Polygon Search
We check if point lies inside polygon.
```
"Find users inside a city boundary"
“restaurants inside city boundary”
```



### Algorithms

* Ray casting
* Winding number


## Algorithm (Ray Casting)
```
Draw ray from point → infinity
Count intersections
Odd → inside
Even → outside
```

---

# 2. Routing (REAL MAP ENGINE)

Represent roads as graphs.

```text
node -> road -> node
```

### Model
Road network = graph
```
A → B → C → D
```
Each road has weight:
- distance
- time
- traffic

### Algorithms

* Dijkstra (shortest path)
* A*

---

# 3. Dynamic Updates

Drivers move continuously.

Need:

* Delete old cell
* Insert new cell

---

# 4. Caching

Use:

* Redis GEO
* In-memory cache

---

# Recommended Learning Path

# Level 1 (Must Learn)

* Coordinates
* Haversine
* Radius search
* Bounding box

---

# Level 2

* Grid indexing
* GeoHash
* KD-Tree

---

# Level 3

* R-Tree
* H3
* S2 Geometry

---

# Final Notes

A modern geo-search system usually combines:

* Spatial indexes
* Distance formulas
* Routing algorithms
* Distributed storage
* Caching systems

This powers applications like:

* Google Maps
* Uber
* Swiggy
* Zomato
* Delivery systems
* Fleet tracking
* GIS analytics


---

# FINAL SYSTEM DESIGN (REAL WORLD)
```
User Request
   ↓
Bounding Box Filter
   ↓
GeoHash / Grid Index
   ↓
KD Tree / Search Engine
   ↓
Haversine Accuracy Filter
   ↓
Return Results
```

