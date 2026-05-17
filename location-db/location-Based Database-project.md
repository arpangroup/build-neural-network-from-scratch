# Project: Location-Based Search Service


📁 Package Structure
```
com.geo.search
 ├── controller
 ├── service
 ├── model
 ├── index
 ├── util
```

OVERALL ARCHITECTURE
```
Mobile App (User / Driver)
        ↓
API Gateway (Spring Boot)
        ↓
────────────────────────────────
|  Ride Service (Core Logic)   |
────────────────────────────────
   ↓              ↓
PostgreSQL      Redis GEO
(PostGIS)       (real-time drivers)
   ↓              ↓
Routing Engine   Surge Engine
(Dijkstra/A*)    Demand calculation
```


---

## STEP 1 — Core Model
```java
package com.geo.search.model;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class Location {
    private long id;
    private String name;
    private double latitude;
    private double longitude;
}
```

## STEP 2 — Distance Utility (Haversine)
```java
package com.geo.search.util;

public class GeoDistanceUtil {

    private static final double EARTH_RADIUS = 6371;

    public static double distance(double lat1, double lon1,
                                  double lat2, double lon2) {

        double dLat = Math.toRadians(lat2 - lat1);
        double dLon = Math.toRadians(lon2 - lon1);

        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
                + Math.cos(Math.toRadians(lat1))
                * Math.cos(Math.toRadians(lat2))
                * Math.sin(dLon / 2)
                * Math.sin(dLon / 2);

        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        return EARTH_RADIUS * c;
    }
}
```

## STEP 2 — Bounding Box Filter
```java
package com.geo.search.util;

public class BoundingBoxUtil {

    public static double[] getBox(double lat, double lon, double radiusKm) {

        double deltaLat = radiusKm / 111.0;
        double deltaLon = radiusKm / (111.0 * Math.cos(Math.toRadians(lat)));

        return new double[]{
                lat - deltaLat, // minLat
                lat + deltaLat, // maxLat
                lon - deltaLon, // minLon
                lon + deltaLon  // maxLon
        };
    }
}
```

---

## STEP 4 — Grid Index (FIRST REAL OPTIMIZATION)
**Idea:** Divide world into buckets.

### Grid Key Generator
```java
package com.geo.search.index;

import com.geo.search.model.Location;
import java.util.*;

public class GridIndex {

    private final double CELL_SIZE = 0.01; // ~1km
    private final Map<String, List<Location>> grid = new HashMap<>();

    private String getKey(double lat, double lon) {
        int x = (int) (lat / CELL_SIZE);
        int y = (int) (lon / CELL_SIZE);
        return x + ":" + y;
    }

    public void add(Location loc) {
        String key = getKey(loc.getLatitude(), loc.getLongitude());

        grid.computeIfAbsent(key, k -> new ArrayList<>())
                .add(loc);
    }

    public List<Location> getNearby(double lat, double lon) {

        int x = (int) (lat / CELL_SIZE);
        int y = (int) (lon / CELL_SIZE);

        List<Location> result = new ArrayList<>();

        // check 9 surrounding cells
        for (int i = -1; i <= 1; i++) {
            for (int j = -1; j <= 1; j++) {

                String key = (x + i) + ":" + (y + j);

                if (grid.containsKey(key)) {
                    result.addAll(grid.get(key));
                }
            }
        }

        return result;
    }
}
````

---

## STEP 6 — Service Layer

```java
package com.geo.search.service;

import com.geo.search.index.GridIndex;
import com.geo.search.model.Location;
import com.geo.search.util.BoundingBoxUtil;
import com.geo.search.util.GeoDistanceUtil;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
public class LocationService {

    private final GridIndex gridIndex = new GridIndex();

    public void addLocation(Location loc) {
        gridIndex.add(loc);
    }

    public List<Location> findNearby(double lat, double lon, double radiusKm) {

        double[] box = BoundingBoxUtil.getBox(lat, lon, radiusKm);

        double minLat = box[0];
        double maxLat = box[1];
        double minLon = box[2];
        double maxLon = box[3];

        List<Location> candidates = gridIndex.getNearby(lat, lon);

        List<Location> result = new ArrayList<>();

        for (Location loc : candidates) {

            if (loc.getLatitude() < minLat || loc.getLatitude() > maxLat)
                continue;

            if (loc.getLongitude() < minLon || loc.getLongitude() > maxLon)
                continue;

            double dist = GeoDistanceUtil.distance(
                    lat, lon,
                    loc.getLatitude(),
                    loc.getLongitude()
            );

            if (dist <= radiusKm) {
                result.add(loc);
            }
        }

        return result;
    }
}
````

---

## STEP 7 — REST Controller

```java
package com.geo.search.controller;

import com.geo.search.model.Location;
import com.geo.search.service.LocationService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/locations")
public class LocationController {

    private final LocationService service;

    public LocationController(LocationService service) {
        this.service = service;
    }

    @PostMapping
    public String add(@RequestBody Location loc) {
        service.addLocation(loc);
        return "added";
    }

    @GetMapping("/nearby")
    public List<Location> nearby(
            @RequestParam double lat,
            @RequestParam double lon,
            @RequestParam double radius
    ) {
        return service.findNearby(lat, lon, radius);
    }
}
````

---

## STEP 8 — Test Data
Add sample data:
```json
POST /locations

{
  "id": 1,
  "name": "Biryani Hub",
  "latitude": 17.385,
  "longitude": 78.486
}
````

### Test Query

```
GET /locations/nearby?lat=17.38&lon=78.48&radius=5
```

## What you have built now
- ✔ Bounding box filtering
- ✔ Grid indexing
- ✔ Haversine accuracy
- ✔ Fast nearby search API

Your system now behaves like:
```
Swiggy / Uber “nearby restaurants” engine (basic version)
```

---

# Phase 2 upgrades
1. GeoHash indexing (scalable distributed search)
2. KD-Tree (fast nearest-neighbor engine)
3. Redis GEO caching
4. PostgreSQL + PostGIS migration
5. Routing engine (GraphHopper style)

---

# GEOHASH SYSTEM (Production-Level Indexing)

**Why GeoHash exists** <br/>
Grid index works, but has problems:
- fixed size cells
- uneven density handling
- poor distribution at scale

GeoHash solves:
- ✔ hierarchical indexing
- ✔ prefix-based search
- ✔ distributed sharding
- ✔ fast range queries

---

## STEP1: GeoHash Implementation (Java)
```java
public class GeoHash {

    private static final int PRECISION = 12;

    private static final char[] BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz".toCharArray();

    public static String encode(double lat, double lon) {

        double[] latRange = {-90.0, 90.0};
        double[] lonRange = {-180.0, 180.0};

        StringBuilder binary = new StringBuilder();

        boolean isEven = true;

        for (int i = 0; i < PRECISION * 5; i++) {

            if (isEven) {
                double mid = (lonRange[0] + lonRange[1]) / 2;

                if (lon > mid) {
                    binary.append("1");
                    lonRange[0] = mid;
                } else {
                    binary.append("0");
                    lonRange[1] = mid;
                }
            } else {
                double mid = (latRange[0] + latRange[1]) / 2;

                if (lat > mid) {
                    binary.append("1");
                    latRange[0] = mid;
                } else {
                    binary.append("0");
                    latRange[1] = mid;
                }
            }

            isEven = !isEven;
        }

        return toBase32(binary.toString());
    }

    private static String toBase32(String binary) {
        StringBuilder hash = new StringBuilder();

        for (int i = 0; i < binary.length(); i += 5) {
            String chunk = binary.substring(i, i + 5);
            int idx = Integer.parseInt(chunk, 2);
            hash.append(BASE32[idx]);
        }
        return hash.toString();
    }
}
```

## STEP 2 — GeoHash Index Store
```java
public class GeoHashIndex {
    private Map<String, List<Location>> index = new HashMap<>();
    public void add(Location loc) {
        String hash = GeoHash.encode(loc.getLatitude(), loc.getLongitude());

        index.computeIfAbsent(hash, k -> new ArrayList<>())
              .add(loc);
    }
}
````

## STEP 4 — GeoHash Search
We search:
- exact cell
- neighboring prefixes

Query flow:
```
user → geohash prefix → fetch nearby buckets
```

Example:

User:
```
lat=17.38 lon=78.48 → tepgq
```
Search:
```
tepgq*
tepgp*
tepgw*
```

### Java search
```java
public List<Location> searchNearby(String hashPrefix) {
    List<Location> result = new ArrayList<>();
    for (String key : index.keySet()) {
        if (key.startsWith(hashPrefix)) {
            result.addAll(index.get(key));
        }
    }
    return result;
}
```

### REAL USE CASE FLOW (GeoHash)
User searches:
```
“restaurants near me”
```

Step:
```
lat/lon → GeoHash → prefix lookup → candidate list
```
Then we refine using KD-tree or Haversine.


---

# KD-TREE (NEAREST NEIGHBOR ENGINE)
***Why KD-Tree?*** <br/>
GeoHash gives candidates.

But KD-tree gives:
- ✔ exact nearest point
- ✔ fast log(n) search
- ✔ dynamic spatial partitioning

## STEP 1 — Node Structure
```java
class KDNode {
    Location location;
    KDNode left;
    KDNode right;
}
```

## STEP 2 — Build KD Tree
We alternate splits:
| Depth | Axis      |
| ----- | --------- |
| 0     | latitude  |
| 1     | longitude |
| 2     | latitude  |

## Build algorithm
```java
public class KDTree {
    private KDNode root;
    public KDNode build(List<Location> points, int depth) {
        if (points.isEmpty()) return null;
        int axis = depth % 2;

        points.sort((a, b) -> axis == 0
                ? Double.compare(a.getLatitude(), b.getLatitude())
                : Double.compare(a.getLongitude(), b.getLongitude()));

        int median = points.size() / 2;

        KDNode node = new KDNode();
        node.location = points.get(median);

        node.left = build(points.subList(0, median), depth + 1);
        node.right = build(points.subList(median + 1, points.size()), depth + 1);

        return node;
    }
}
```

## STEP 3 — Nearest Neighbor Search
**Idea:** We traverse tree and prune branches. <br/>

### Algorithm:
- Go to closest branch
- Save best distance
- Check if other branch could contain closer point
- Backtrack if needed

```java
public class KDTreeSearch {
    private Location best;
    private double bestDist = Double.MAX_VALUE;
    public Location nearest(KDNode node, double lat, double lon, int depth) {
        if (node == null) return best;

        double dist = distance(lat, lon,
                node.location.getLatitude(),
                node.location.getLongitude());

        if (dist < bestDist) {
            bestDist = dist;
            best = node.location;
        }

        int axis = depth % 2;

        KDNode next;
        KDNode other;

        if (axis == 0) {
            if (lat < node.location.getLatitude()) {
                next = node.left;
                other = node.right;
            } else {
                next = node.right;
                other = node.left;
            }
        } else {
            if (lon < node.location.getLongitude()) {
                next = node.left;
                other = node.right;
            } else {
                next = node.right;
                other = node.left;
            }
        }

        nearest(next, lat, lon, depth + 1);

        // check if we must explore other side
        if (shouldCheckOther(node, lat, lon, axis)) {
            nearest(other, lat, lon, depth + 1);
        }

        return best;
    }

    private boolean shouldCheckOther(KDNode node, double lat, double lon, int axis) {
        return true; // simplified for concept
    }
}
```

KD-Tree Complexity
```
Average: O(log n)
Worst: O(n)
```

---

## REAL SYSTEM FLOW (GeoHash + KDTree together)

### Step 1 — User request
```
Find nearest restaurants
```

### Step 2 — GeoHash filter
```
reduce 10M → 5K candidates
```

### Step 3 — KD-Tree search
```
5K → top 10 nearest
```

### Step 4 — Haversine final check
```
exact ranking
```

### Final result
```
1. Biryani Hub (1.2 km)
2. Spice Villa (1.6 km)
3. Food Street (2.0 km)
```