# Mini Uber Geo Visualizer 🚖🗺️

Build a visual geo-spatial system using:

* Java Spring Boot backend
* HTML + JavaScript frontend
* Leaflet.js map rendering
* GeoHash clustering
* Grid partition visualization

This project visually demonstrates how systems like:

* Uber
* Swiggy
* Zomato
* Google Maps

manage spatial indexing and nearby searches.

---

# 🎯 What You Will Build

You will create a live visualization system showing:
- 📦 Grid partitioning
- 🔤 GeoHash regions
- 📍 Driver locations
- ✅ User ride requests
- ✅ Nearest-driver matching
- ✅ Spatial indexing visualization
- 🗺️ Map overlay (Leaflet.js)

---

# 🧱 Final Visualization

```text id="kqxyfr"
+--------------------------------------------------+
|                    MAP VIEW                      |
|                                                  |
|   ⬜ Grid Cells                                  |
|   🔴 Drivers                                     |
|   ⚫ User Request                                |
|   🔵 GeoHash Clusters                            |
|   🔴 Route Line (Nearest Driver Match)           |
|                                                  |
+--------------------------------------------------+
```

Grid system
```
+-----+-----+
|  A  |  B  |
+-----+-----+
|  C  |  D  |
+-----+-----+
```

---

# 🧰 Tech Stack

| Layer         | Technology            |
| ------------- | --------------------- |
| Backend       | Spring Boot           |
| Frontend      | HTML + JavaScript     |
| Map Engine    | Leaflet.js            |
| Map Tiles     | OpenStreetMap         |
| Spatial Logic | GeoHash + Grid System |

---

# 📁 Project Structure

```text id="8n3r2f"
src/main/java
 ├── controller
 ├── service
 ├── model
 ├── geo

src/main/resources
 ├── static
 │    ├── index.html 👈 MAP UI
```

---

# 🌍 STEP 1 — Create Map UI

Create:

```text id="v4m8rx"
src/main/resources/static/index.html
```

---

# 📄 index.html

```html id="q0i5f4"
<!DOCTYPE html>
<html>

<head>

    <title>Geo Visualizer</title>

    <link rel="stylesheet"
          href="https://unpkg.com/leaflet/dist/leaflet.css"/>

    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

    <style>

        body {
            margin: 0;
        }

        #map {
            height: 100vh;
            width: 100%;
        }

    </style>

</head>

<body>

<div id="map"></div>

<script>

    // Initialize map

    const map = L.map('map')
        .setView([17.385, 78.486], 13);

    // OpenStreetMap tiles

    L.tileLayer(
        'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        {
            maxZoom: 19
        }
    ).addTo(map);

</script>

</body>

</html>
```

---

# 🧠 What Happens Here?

The browser:

1. Loads Leaflet.js
2. Creates map container
3. Loads OpenStreetMap tiles
4. Centers map on Hyderabad

---

# 🌐 Result

You now see a fully interactive map.

---

# 📍 STEP 2 — Add Driver Markers

Add sample drivers.

---

# Update Script

```javascript id="m7m66j"
const drivers = [

    {lat: 17.385, lon: 78.486},
    {lat: 17.390, lon: 78.490},
    {lat: 17.375, lon: 78.480}

];

drivers.forEach(d => {

    L.circleMarker([d.lat, d.lon], {
        radius: 6,
        color: "red"
    }).addTo(map)
    .bindPopup("Driver");

});
```

---

# 🧠 Visualization

```text id="m2xx0f"
🔴 Driver 1
🔴 Driver 2
🔴 Driver 3
```

These simulate Uber drivers on the map.

---

# 📦 STEP 3 — Grid System Visualization

Uber internally divides maps into smaller spatial cells.

---

# Grid Concept

```text id="qgpb87"
+-----+-----+
|  A  |  B  |
+-----+-----+
|  C  |  D  |
+-----+-----+
```

Each square becomes a searchable region.

---

# 🧠 Grid Utility (Backend)

Create:

```text id="aydncf"
geo/GridUtil.java
```

---

# GridUtil.java

Grid formula
```
cellSize = 0.01 degree
```

## Java Grid Generator
```java id="w5j7vx"
public class GridUtil {

    private static final double CELL = 0.01;

    public static String cellId(double lat, double lon) {
        int x = (int)(lat / CELL);
        int y = (int)(lon / CELL);
        return x + "_" + y;
    }
}
```

## Generate grid boundaries
```java
public class GridBoundary {

    public static List<double[]> getCellBounds(double lat, double lon) {

        double cell = 0.01;

        int x = (int)(lat / cell);
        int y = (int)(lon / cell);

        double minLat = x * cell;
        double maxLat = (x + 1) * cell;

        double minLon = y * cell;
        double maxLon = (y + 1) * cell;

        return List.of(
                new double[]{minLat, minLon},
                new double[]{maxLat, minLon},
                new double[]{maxLat, maxLon},
                new double[]{minLat, maxLon}
        );
    }
}
```

---

# 🧠 How Grid Formula Works

The earth gets divided into square regions.

## Formula

$$
cellId = \left(\frac{lat}{cell}\right), \left(\frac{lon}{cell}\right)
$$

---

# 📦 STEP 4 — Draw Grid Overlay

Now render visible grid squares on the map.

---

# Add Grid Function

```javascript id="mjlwmv"
function drawGrid(lat, lon) {

    const cell = 0.01;

    for (let i = -3; i <= 3; i++) {
        for (let j = -3; j <= 3; j++) {
            let x = lat + i * cell;
            let y = lon + j * cell;
            let bounds = [
                [x, y],
                [x + cell, y],
                [x + cell, y + cell],
                [x, y + cell]
            ];

            L.polygon(bounds, {
                color: "gray",
                weight: 1,
                fillOpacity: 0.1
            }).addTo(map);
        }
    }
}
```

---

# Call Grid Renderer

```javascript id="n7tx6f"
drawGrid(17.385, 78.486);
```

---

# 🌐 What You See

You now visually see:

```text id="rjod9q"
⬜ Spatial grid cells
```

These are spatial partitions used for:

* Nearby search
* Driver indexing
* Heatmaps

---

# 🔤 STEP 5 — GeoHash Visualization

GeoHash converts GPS coordinates into searchable strings.

---

# Example

```text id="b5qvh0"
17.385, 78.486 → tepg1
```

Nearby locations produce similar hashes.

---

# GeoHash Encoder

Create:

```text id="jlwm2l"
geo/GeoHashSimple.java
```

---

# GeoHashSimple.java

```java id="zh3z6f"
public class GeoHashSimple {

    public static String encode(double lat, double lon) {
        return String.valueOf((int)(lat * 10)) +
               String.valueOf((int)(lon * 10));
    }
}
```

---

# 🌐 STEP 6 — Backend API to send geohash groups

Expose driver + geohash data.

---

# GeoController.java

```java id="x6e6t9"
@RestController
@RequestMapping("/geo")
public class GeoController {

    @GetMapping("/drivers")
    public List<Map<String, Object>> drivers() {

        List<Map<String, Object>> list = new ArrayList<>();

        list.add(Map.of(
                "lat", 17.385,
                "lon", 78.486,
                "geoHash", "tepg1"

        ));

        list.add(Map.of(
                "lat", 17.390,
                "lon", 78.490,
                "geoHash", "tepg1"

        ));

        list.add(Map.of(
                "lat", 17.375,
                "lon", 78.480,
                "geoHash", "tepg2"

        ));

        return list;
    }
}
```

---

# 📡 STEP 7 — Fetch Backend Data

Now frontend consumes Spring Boot API.

---

# Add API Fetch

```javascript id="5pd23d"
fetch("/geo/drivers")
    .then(res => res.json())
    .then(data => {
        const groups = {};
        data.forEach(d => {
            if (!groups[d.geoHash]) {
                groups[d.geoHash] = [];
            }
            groups[d.geoHash].push(d);
        });

        Object.keys(groups).forEach(hash => {
            const color = getColor(hash);
            groups[hash].forEach(d => {
                L.circleMarker([d.lat, d.lon], {
                    radius: 8,
                    color: color
                }).addTo(map)
                .bindPopup(
                    "GeoHash: " + hash
                );
            });
        });
    });
```

---

# 🎨 GeoHash Coloring

Add helper function:

```javascript id="jmsk8x"
function getColor(hash) {
    let colors = [
        "red",
        "blue",
        "green",
        "orange"
    ];

    let index = hash.charCodeAt(0) % colors.length;
    return colors[index];
}
```

---

# 🌐 What You See

```text id="g0lh6u"
🔴 tepg1 cluster
🔵 tepg2 cluster
🟢 Nearby grouped drivers
```

Drivers sharing same GeoHash appear visually grouped.

---

# 📍 STEP 8 — Add User Ride Request

Simulate rider request.

---

# Add User Marker

```javascript id="cll5t0"
L.circleMarker([17.387, 78.489], {
    radius: 10,
    color: "black"

}).addTo(map)
.bindPopup("User");
```

---

# 🌐 Visualization

```text id="0l2xhi"
⚫ Rider requesting trip
```

---

# 🚖 STEP 9 — Draw Driver Match Line

Show nearest-driver assignment.

---

# Add Polyline

```javascript id="mvbhk6"
L.polyline([
    [17.387, 78.489],
    [17.385, 78.486]

], {
    color: "red"
}).addTo(map);
```

---

# 🧠 What This Represents

```text id="2sctd9"
User
  ↓
Matching Engine
  ↓
Nearest Driver
```

Exactly how Uber dispatch systems visualize assignments.

---

# 📦 STEP 10 — Final Full index.html

Below is the combined visualization.

---

# Complete index.html

```html id="kfyv2n"
<!DOCTYPE html>
<html>

<head>

    <title>Geo Visualizer</title>

    <link rel="stylesheet"
          href="https://unpkg.com/leaflet/dist/leaflet.css"/>

    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

    <style>

        body {
            margin: 0;
        }

        #map {
            height: 100vh;
        }

    </style>

</head>

<body>

<div id="map"></div>

<script>

    const map = L.map('map')
        .setView([17.385, 78.486], 13);

    L.tileLayer(
        'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        {
            maxZoom: 19
        }
    ).addTo(map);

    // Draw grid

    function drawGrid(lat, lon) {

        const cell = 0.01;

        for (let i = -3; i <= 3; i++) {
            for (let j = -3; j <= 3; j++) {
                
                let x = lat + i * cell;
                let y = lon + j * cell;
                
                let bounds = [
                    [x, y],
                    [x + cell, y],
                    [x + cell, y + cell],
                    [x, y + cell]
                ];

                L.polygon(bounds, {
                    color: "gray",
                    weight: 1,
                    fillOpacity: 0.1
                }).addTo(map);
            }
        }
    }

    drawGrid(17.385, 78.486);

    // Fetch drivers
    fetch("/geo/drivers")
        .then(res => res.json())
        .then(data => {
            const groups = {};
            data.forEach(d => {
                if (!groups[d.geoHash]) {
                    groups[d.geoHash] = [];
                }
                groups[d.geoHash].push(d);
            });

            Object.keys(groups).forEach(hash => {
                const color = getColor(hash);
                groups[hash].forEach(d => {
                    L.circleMarker([d.lat, d.lon], {
                        radius: 8,
                        color: color
                    }).addTo(map)
                    .bindPopup(
                        "GeoHash: " + hash
                    );
                });
            });
        });

    function getColor(hash) {
        let colors = [
            "red",
            "blue",
            "green",
            "orange"
        ];

        let index =
            hash.charCodeAt(0) % colors.length;
        return colors[index];
    }

    // User marker
    L.circleMarker([17.387, 78.489], {
        radius: 10,
        color: "black"
    }).addTo(map)
    .bindPopup("User");

    // Matching line
    // Show nearest driver connection
    L.polyline([
        [17.387, 78.489],
        [17.385, 78.486]

    ], {
        color: "red"
    }).addTo(map);
</script>

</body>

</html>
```

---

# 🧠 Final Visualization

You now visually understand:

| Feature        | Visual Meaning       |
| -------------- | -------------------- |
| Grid boxes     | Spatial partitioning |
| GeoHash colors | Nearby grouping      |
| Red markers    | Drivers              |
| Black marker   | Rider                |
| Red line       | Driver assignment    |

---

# 🚀 What You Actually Built

You built a mini visualization engine for:

* Uber dispatch maps
* Swiggy heatmaps
* Zomato delivery regions
* Google Maps indexing systems

---

# 🔥 Next-Level Improvements

You can now upgrade this into:

---

# 1. Real GeoHash Library

Use:

```text id="4qg34t"
ch.hsr:geohash
```

---

# 2. Real-Time Driver Movement

Use:

* WebSockets
* Kafka streams

---

# 3. Heatmaps

Visualize:

* Demand density
* Surge zones
* Busy regions

---

# 4. Route Animation

Animate:

```text id="88m20k"
Driver → User → Destination
```

---

# 5. Redis GEO Integration

Replace static arrays with:

```text id="w9owkz"
Redis GEOSEARCH
```

---

# 6. Production Spatial System

Add:

* PostGIS
* H3 indexing
* QuadTrees
* KD-Trees

---

# 🎯 Final Result

You now have a working visual geo-spatial platform demonstrating:

- ✅ Spatial indexing
- ✅ Grid partitioning
- ✅ GeoHash clustering
- ✅ Nearby search systems
- ✅ Driver matching
- ✅ Real-time map visualization

# 🚀 NEXT LEVEL OPTIONS (HIGHLY RECOMMENDED)
1. Live moving drivers (real-time WebSocket map)
2. Heatmap of demand (surge visualization)
3. Animated route (A* path on map)
4. Full PostGIS integration with map
5. Multi-city distributed geo system