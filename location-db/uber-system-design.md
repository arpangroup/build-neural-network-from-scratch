# Mini Uber Backend in Spring Boot 🚖

A complete backend architecture for an Uber-like ride matching platform using **Spring Boot**, **Redis GEO**, and scalable system design concepts.

---

# 📌 Overview

In this project, we build the **core backend system** behind applications like:

* Uber
* Lyft
* Ola
* Rapido

The system supports:

* 🚗 Driver registration
* 📍 Real-time driver location tracking
* 👤 Ride requests
* 🧠 Smart driver matching
* 🛣️ Route calculation
* 💰 Surge pricing
* ⏱️ ETA estimation

---

# 🧱 1. System Design Architecture

```text
Rider App
   ↓
Ride Controller
   ↓
Matching Service
   ↓
Redis GEO (live drivers)
   ↓
PostGIS (persistent storage)
   ↓
Routing Engine (A*)
   ↓
Driver Assigned
```

---

# 🧠 High-Level Flow

1. Rider requests a ride
2. Backend searches nearby drivers using Redis GEO
3. Matching engine ranks drivers
4. Best driver gets assigned
5. ETA and pricing are calculated
6. Ride starts

---

# 🛠️ 2. Create Spring Boot Project

## Required Dependencies

Add these dependencies while creating the project:

* Spring Web
* Spring Data Redis
* Spring Data JPA
* PostgreSQL Driver
* Lombok

---

# 📦 3. Core Models

---

# 👤 Driver Model

Represents drivers available on the platform.

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Driver {

    private Long id;
    private String name;
    private double lat;
    private double lon;
    private boolean available;
    private double rating;
}
```

## Explanation

| Field     | Purpose                |
| --------- | ---------------------- |
| id        | Unique driver ID       |
| name      | Driver name            |
| lat/lon   | Current GPS location   |
| available | Whether driver is free |
| rating    | Driver quality score   |

---

# 🚗 Ride Request Model

Represents a rider's trip request.

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class RideRequest {

    private Long riderId;
    private double sourceLat;
    private double sourceLon;
    private double destLat;
    private double destLon;
}
```

## Explanation

| Field               | Purpose              |
| ------------------- | -------------------- |
| riderId             | User requesting ride |
| sourceLat/sourceLon | Pickup location      |
| destLat/destLon     | Destination location |

---

# ⚡ 4. Redis GEO Setup

Redis GEO is used for:

* Real-time driver tracking
* Fast nearby driver search
* Geo-spatial indexing

---

# Redis Configuration

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, String> redisTemplate(
            RedisConnectionFactory factory) {

        RedisTemplate<String, String> template =
                new RedisTemplate<>();

        template.setConnectionFactory(factory);

        return template;
    }
}
```

## Why Redis GEO?

Traditional databases are slow for live geo-search.

Redis GEO provides:

* O(log N) spatial queries
* Radius search
* Real-time updates
* In-memory speed

Perfect for Uber-like systems.

---

# 📍 5. Driver Location Service

This service updates and searches live driver locations.

---

# 🚗 Update Driver Location

```java
@Service
public class DriverLocationService {

    private final String KEY = "drivers_geo";

    @Autowired
    private StringRedisTemplate redisTemplate;

    // Add/update location
    public void updateLocation(Long driverId,
                               double lon,
                               double lat) {

        redisTemplate.opsForGeo()
                .add(KEY,
                        new Point(lon, lat),
                        driverId.toString());
    }
}
```

## Explanation

Redis stores:

```text
drivers_geo
   ├── Driver 1 → (78.48, 17.38)
   ├── Driver 2 → (78.50, 17.40)
```

Every location update overwrites the previous value.

This simulates live GPS tracking.

---

# 🔍 Search Nearby Drivers

```java
public List<String> findNearby(double lon,
                               double lat,
                               double radiusKm) {

    Circle circle = new Circle(
            new Point(lon, lat),
            new Distance(radiusKm,
                    Metrics.KILOMETERS));

    GeoResults<RedisGeoCommands.GeoLocation<String>> results =
            redisTemplate.opsForGeo()
                    .radius(KEY, circle);

    List<String> driverIds = new ArrayList<>();

    for (GeoResult<RedisGeoCommands.GeoLocation<String>> res : results) {
        driverIds.add(res.getContent().getName());
    }

    return driverIds;
}
```

## What Happens Here?

1. Rider location becomes center point
2. Redis searches all drivers within radius
3. Nearby driver IDs are returned

This operation is extremely fast.

---

# 🚗 6. Driver Service

Stores and manages drivers.

```java
@Service
public class DriverService {

    private final Map<Long, Driver> driverDB =
            new ConcurrentHashMap<>();

    public void registerDriver(Driver d) {
        driverDB.put(d.getId(), d);
    }

    public Driver getDriver(Long id) {
        return driverDB.get(id);
    }

    public List<Driver> getDrivers(List<String> ids) {

        List<Driver> list = new ArrayList<>();

        for (String id : ids) {

            Driver d = driverDB.get(Long.valueOf(id));

            if (d != null && d.isAvailable()) {
                list.add(d);
            }
        }

        return list;
    }
}
```

---

# 🧠 7. Matching Engine (Core Uber Logic)

This is the brain of the system.

---

# Matching Strategy

The system:

1. Finds nearby drivers
2. Calculates score
3. Chooses best driver

---

# Matching Service

```java
@Service
public class MatchingService {

    @Autowired
    private DriverLocationService locationService;

    @Autowired
    private DriverService driverService;

    public Driver findBestDriver(double lat, double lon) {

        List<String> nearbyIds =
                locationService.findNearby(lon, lat, 5);

        List<Driver> drivers =
                driverService.getDrivers(nearbyIds);

        Driver best = null;

        double bestScore = Double.MAX_VALUE;

        for (Driver d : drivers) {

            double dist = distance(
                    lat,
                    lon,
                    d.getLat(),
                    d.getLon());

            double score =
                    dist * 0.7 +
                    (1.0 / (d.getRating() + 0.1)) * 0.3;

            if (score < bestScore) {
                bestScore = score;
                best = d;
            }
        }

        return best;
    }

    private double distance(double lat1,
                            double lon1,
                            double lat2,
                            double lon2) {

        double R = 6371;

        double dLat =
                Math.toRadians(lat2 - lat1);

        double dLon =
                Math.toRadians(lon2 - lon1);

        double a =
                Math.sin(dLat / 2)
                        * Math.sin(dLat / 2)
                        + Math.cos(Math.toRadians(lat1))
                        * Math.cos(Math.toRadians(lat2))
                        * Math.sin(dLon / 2)
                        * Math.sin(dLon / 2);

        double c =
                2 * Math.atan2(
                        Math.sqrt(a),
                        Math.sqrt(1 - a));

        return R * c;
    }
}
```

---

# 🧮 Driver Scoring Formula

The score combines:

* Distance
* Driver rating

Lower score = better driver.

## Formula

$$
score = 0.7 \times distance + 0.3 \times \frac{1}{rating + 0.1}
$$
---

# 🌍 Haversine Distance Formula

Used to calculate earth distance between two GPS points.

$$
d = 2R \arctan\left(\sqrt{\frac{a}{1-a}}\right)
$$
---

# 🚖 8. Ride Service

Handles complete ride request flow.

```java
@Service
public class RideService {

    @Autowired
    private MatchingService matchingService;

    public String requestRide(RideRequest req) {

        Driver driver =
                matchingService.findBestDriver(
                        req.getSourceLat(),
                        req.getSourceLon()
                );

        if (driver == null) {
            return "No driver available";
        }

        driver.setAvailable(false);

        return "Driver assigned: "
                + driver.getName();
    }
}
```

---

# 🌐 9. REST API Layer

This exposes backend APIs.

---

# Driver Controller

```java
@RestController
@RequestMapping("/drivers")
public class DriverController {

    @Autowired
    private DriverService driverService;

    @Autowired
    private DriverLocationService locationService;

    @PostMapping("/register")
    public String register(@RequestBody Driver d) {

        driverService.registerDriver(d);

        return "registered";
    }

    @PostMapping("/location")
    public String updateLocation(
            @RequestParam Long id,
            @RequestParam double lat,
            @RequestParam double lon) {

        locationService.updateLocation(
                id,
                lon,
                lat);

        return "updated";
    }
}
```

---

# Ride Controller

```java
@RestController
@RequestMapping("/rides")
public class RideController {

    @Autowired
    private RideService rideService;

    @PostMapping("/request")
    public String request(
            @RequestBody RideRequest req) {

        return rideService.requestRide(req);
    }
}
```

---

# 🛣️ 10. Routing Engine (A*)

Routing determines the best path between source and destination.

---

# Graph Node

```java
class Node {

    String id;

    Map<Node, Double> neighbors =
            new HashMap<>();
}
```

---

# A* Formula

A* works using:

$$
f(n) = g(n) + h(n)
$$

Where:

| Formula | Meaning                   |
| ------- | ------------------------- |
| g(n)    | Actual travel cost        |
| h(n)    | Estimated remaining cost  |
| f(n)    | Total estimated path cost |

---

# Simplified A* Heuristic

```java
public class AStar {

    public double heuristic(double lat1,
                            double lon1,
                            double lat2,
                            double lon2) {

        return Math.sqrt(
                Math.pow(lat1 - lat2, 2)
                        +
                Math.pow(lon1 - lon2, 2)
        );
    }
}
```

---

# 💰 11. Surge Pricing

Dynamic pricing based on:

* Rider demand
* Driver supply

---

# Surge Service

```java
public class SurgeService {

    public double calculate(int demand,
                            int supply) {

        double surge =
                (double) demand
                        / Math.max(1, supply);

        return Math.max(1.0, surge);
    }
}
```

---

# Surge Formula

$$
surge = \max\left(1.0, \frac{demand}{supply}\right)
$$

---

# ⏱️ 12. ETA Calculation

Estimate driver arrival time.

---

# ETA Service

```java
public class ETAService {

    public double eta(double distanceKm) {

        double avgSpeed = 30.0;

        return (distanceKm / avgSpeed) * 60;
    }
}
```

---

# ETA Formula

$$
ETA = \frac{distance}{speed} \times 60
$$

---

# 🧪 13. Testing the Flow

---

# STEP 1 — Register Driver

```http
POST /drivers/register
```

Example JSON:

```json
{
  "id": 1,
  "name": "Ravi",
  "lat": 17.38,
  "lon": 78.48,
  "available": true,
  "rating": 4.8
}
```

---

# STEP 2 — Update Driver Location

```http
POST /drivers/location?id=1&lat=17.38&lon=78.48
```

---

# STEP 3 — Request Ride

```http
POST /rides/request
```

Example JSON:

```json
{
  "riderId": 101,
  "sourceLat": 17.385,
  "sourceLon": 78.486,
  "destLat": 17.450,
  "destLon": 78.390
}
```

---

# 🚀 What You Built

You now have:

- ✅ Uber-like backend
- ✅ Redis GEO live tracking
- ✅ Real-time matching engine
- ✅ Driver ranking system
- ✅ Ride assignment flow
- ✅ Basic routing engine
- ✅ ETA calculation
- ✅ Surge pricing logic

---

# 🧠 Real Industry Concepts Used

This architecture demonstrates:

| Concept              | Used In           |
| -------------------- | ----------------- |
| Geo-Spatial Indexing | Redis GEO         |
| Distributed Caching  | Redis             |
| Matching Algorithms  | Driver allocation |
| Graph Algorithms     | Routing           |
| Dynamic Pricing      | Surge engine      |
| Real-Time Systems    | Driver tracking   |

---

# 🏗️ Production Improvements

In real-world systems:

| Current        | Production Upgrade |
| -------------- | ------------------ |
| In-memory Map  | PostgreSQL/PostGIS |
| Simple A*      | Google Maps / OSRM |
| Single app     | Microservices      |
| Sync calls     | Kafka async events |
| Basic matching | ML-based dispatch  |

---

# 🔥 Next-Level Upgrades

You can extend this into:

## 1. Kafka Event-Driven Architecture

* Driver events
* Ride events
* Async matching
* Real-time streaming

---

## 2. PostGIS Migration

Use PostgreSQL + PostGIS for:

* Persistent geo storage
* Spatial indexing
* Advanced geo queries

---

## 3. Real Road Routing

Integrate:

* OpenStreetMap
* OSRM
* GraphHopper

---

## 4. Distributed System Design

Add:

* API Gateway
* Load balancing
* Multi-region deployment
* Circuit breakers

---

## 5. Full Microservices Architecture

Split into:

* driver-service
* ride-service
* pricing-service
* location-service
* notification-service
* payment-service

---

# 🎯 Final Summary

This project demonstrates the core backend engineering principles behind ride-sharing systems like Uber.

You implemented:

* Real-time geo tracking
* Driver discovery
* Intelligent matching
* Route estimation
* Dynamic pricing
* REST APIs

This is a strong foundation for building:

* Ride-sharing apps
* Delivery platforms
* Logistics systems
* Fleet management systems


# Better Architecture

## Improvement1. Real-Time Driver Tracking via WebSockets
Currently:
```
Driver → REST API → Update Location
```
Production systems use:
```
Driver App → WebSocket → Streaming Location Updates
```
Technologies
- Spring WebSocket
- STOMP
- Socket.IO
- Kafka Streams


## Improvement2. Kafka Event-Driven Architecture
```
Ride Requested Event
        ↓
Kafka
        ↓
Matching Service
        ↓
Driver Assigned Event
        ↓
Notification Service
```

## Improvement3. Proper Microservices Architecture
```
API Gateway
    ↓
Ride Service
Driver Service
Location Service
Pricing Service
Notification Service
Payment Service
Matching Service
```

## Improvement4. Smarter Driver Matching
Current logic:
```
Nearest + Rating
```

Production systems consider:
- Driver acceptance rate
- Cancellation history
- Traffic conditions
- Driver idle time
- Rider preferences
- Vehicle type
- Predicted demand

## Advanced Matching Score

$$
score=w1​d+w2​r+w3​t+w4​c+w5​i
$$

where
| Variable | Meaning           |
| -------- | ----------------- |
| d        | Distance          |
| r        | Rating            |
| t        | Traffic           |
| c        | Cancellation rate |
| i        | Idle time         |


## Improvement5. Real Routing Engine
Your current A* implementation is simplified.

Production systems use:
- OpenStreetMap
- GraphHopper
- OSRM
- Google Maps APIs

## Improvement6. Add Features
- Traffic-aware routing
- Dynamic rerouting
- Toll optimization
- Multi-stop rides

## Improvement7. Dynamic Surge Pricing Engine
Current:
```
demand / supply
```

Real surge systems consider:
- Area demand density
- Time of day
- Weather
- Events/concerts
- Historical trends
- Driver availability forecast

Advanced Surge Formula

$$
surge=base×demandFactor×weatherFactor×trafficFactor
$$

## Improvement8. Add Ride State Machine
Right now rides instantly assign.

Real rides have states.

**Ride Lifecycle**
```
REQUESTED
   ↓
MATCHED
   ↓
DRIVER_ARRIVING
   ↓
STARTED
   ↓
COMPLETED
   ↓
PAID
```
**Why Important?** Prevents invalid transitions.

Example:
```
COMPLETED → STARTED ❌
```

## Improvement9. Add Authentication & Authorization
Currently anyone can call APIs.

Roles
| Role   | Permissions     |
| ------ | --------------- |
| Rider  | Request rides   |
| Driver | Update location |
| Admin  | Manage system   |



## Improvement10. Distributed Caching
Cache:
- Driver metadata
- ETA
- Pricing
- Popular routes

Technologies
- Redis
- Hazelcast
- Caffeine

## Improvement11. Notification System
- Real systems notify users.
- Push notifications
- SMS
- Email
- In-app notifications

Example Events
```
Driver Assigned
Ride Started
Ride Completed
Payment Failed
```

## Improvement12. Payment Integration

## Improvement13. Observability & Monitoring
Production systems require visibility.
- Prometheus
- Grafana
- ELK Stack
- OpenTelemetry

| Metric        | Why                  |
| ------------- | -------------------- |
| Ride latency  | User experience      |
| Match time    | Dispatch performance |
| Redis latency | Geo search health    |
| Kafka lag     | Event bottlenecks    |


## Improvement14. Fault Tolerance & Resilience
Add
- Retries
- Circuit breakers
- Bulkheads
- Timeouts

Use
- Resilience4j
- Spring Retry

## Improvement15. Multi-Region Deployment
Architecture
- US-East Region
- Europe Region
- India Region
Benefits
- Lower latency
- Disaster recovery
- Regional scaling

## Improvement16. Machine Learning Improvements
This is where systems become “Uber-level”.

## ML Applications
| Feature         | ML Use                 |
| --------------- | ---------------------- |
| ETA prediction  | Traffic prediction     |
| Surge pricing   | Demand forecasting     |
| Driver matching | Reinforcement learning |
| Fraud detection | Anomaly detection      |


## Improvement17. Add Kubernetes Deployment
Containerize services.
- Docker
- Kubernetes
- Helm
- Istio
Benefits
- Auto scaling
- Self-healing
- Rolling deployments


## Improvement18. Add API Gateway
Central entry point.

Features
- Authentication
- Rate limiting
- Request routing
- Logging

Tools
- Spring Cloud Gateway
- Kong
- NGINX

## Improvement19. Add Rate Limiting
Prevent abuse.

Example
```
100 ride requests/minute/user
```

Tools
- Redis rate limiting
- Bucket4j
- API Gateway policies

## Improvement20. Build Real-Time Analytics
Track business metrics.

Dashboard Metrics
| Metric              | Example  |
| ------------------- | -------- |
| Active rides        | 12,300   |
| Driver online count | 4,000    |
| Avg ETA             | 3.5 mins |
| Revenue/hour        | ₹8.2L    |


