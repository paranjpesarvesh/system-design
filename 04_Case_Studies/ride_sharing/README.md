# Ride-Sharing Platform (Uber/Lyft)

## Problem Statement

Design a ride-sharing platform like Uber or Lyft that connects passengers with drivers, handles real-time location tracking, pricing, and payment processing.

## Requirements

### Functional Requirements
- User registration and authentication
- Driver registration and verification
- Real-time location tracking
- Ride booking and matching
- Dynamic pricing
- Payment processing
- Rating and review system
- Trip history and receipts

### Non-Functional Requirements
- **Scale**: 10 million daily rides, 1 million concurrent users
- **Latency**: < 3 seconds for ride matching
- **Availability**: 99.99% uptime
- **Real-time**: Location updates every 5 seconds
- **Consistency**: Strong consistency for payments, eventual for locations

## Scale Estimation

### Traffic Estimates
```
Daily Active Users: 10 million
Daily Rides: 10 million
Peak Concurrent Rides: 1 million
Peak Concurrent Users: 5 million
Driver Locations per Second: 5 million
API Requests per Second: 50,000
```

### Storage Estimates
```
User Data: 10M × 2KB = 20GB
Driver Data: 5M × 3KB = 15GB
Trip Data: 10M × 1KB = 10GB/day = 3.6TB/year
Location Data: 5M × 1KB × 12/min × 60min × 24h = 86GB/day = 31TB/year
Payment Data: 10M × 500B = 5GB/day = 1.8TB/year
```

## High-Level Design

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
└─────────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────┐
│                 API Gateway                             │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   User      │  │   Driver    │  │   Trip      │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│  User DB       │  Driver DB     │  Trip DB              │
│  (PostgreSQL)  │  (PostgreSQL)  │  (PostgreSQL)         │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Real-time Services                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Location  │  │   Matching  │  │   Pricing   │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Data Stores                                │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Redis     │  │   Kafka     │  │   S3        │      │
│  │   (Cache)   │  │   (Events)  │  │   (Storage) │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
```

## API Design

### User APIs
```http
POST /api/v1/users/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "password": "securepassword"
}

POST /api/v1/users/login
{
  "email": "john@example.com",
  "password": "securepassword"
}

POST /api/v1/rides/request
{
  "pickup_location": {"lat": 37.7749, "lng": -122.4194},
  "dropoff_location": {"lat": 37.8044, "lng": -122.2711},
  "ride_type": "standard",
  "payment_method": "credit_card"
}

GET /api/v1/rides/{ride_id}
Response:
{
  "ride_id": "ride_123",
  "driver": {
    "name": "Jane Smith",
    "rating": 4.8,
    "vehicle": "Toyota Camry"
  },
  "status": "driver_assigned",
  "eta": 5,
  "price": 25.50,
  "estimated_time": 15
}
```

### Driver APIs
```http
POST /api/v1/drivers/register
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "license_number": "D1234567",
  "vehicle_info": {
    "make": "Toyota",
    "model": "Camry",
    "year": 2020,
    "color": "Blue"
  }
}

POST /api/v1/drivers/location
{
  "driver_id": "driver_456",
  "location": {"lat": 37.7749, "lng": -122.4194},
  "heading": 90,
  "speed": 0
}

GET /api/v1/drivers/earnings?start_date=2024-01-01&end_date=2024-01-31
Response:
{
  "total_earnings": 1250.75,
  "total_rides": 85,
  "average_per_ride": 14.72,
  "daily_breakdown": [...]
}
```

## Database Design

### Users Table
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_phone (phone)
);
```

### Drivers Table
```sql
CREATE TABLE drivers (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    license_number VARCHAR(50) NOT NULL,
    vehicle_make VARCHAR(100),
    vehicle_model VARCHAR(100),
    vehicle_year INTEGER,
    vehicle_color VARCHAR(50),
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_rides INTEGER DEFAULT 0,
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);
```

### Rides Table
```sql
CREATE TABLE rides (
    id BIGINT PRIMARY KEY,
    rider_id BIGINT REFERENCES users(id),
    driver_id BIGINT REFERENCES drivers(id),
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lng DECIMAL(11,8) NOT NULL,
    dropoff_lat DECIMAL(10,8) NOT NULL,
    dropoff_lng DECIMAL(11,8) NOT NULL,
    status ENUM('requested', 'driver_assigned', 'in_progress', 'completed', 'cancelled') DEFAULT 'requested',
    ride_type ENUM('standard', 'premium', 'pool') DEFAULT 'standard',
    price DECIMAL(10,2),
    distance_km DECIMAL(8,2),
    duration_minutes INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    INDEX idx_rider_id (rider_id),
    INDEX idx_driver_id (driver_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

### Driver Locations Table
```sql
CREATE TABLE driver_locations (
    id BIGINT PRIMARY KEY,
    driver_id BIGINT REFERENCES drivers(id),
    lat DECIMAL(10,8) NOT NULL,
    lng DECIMAL(11,8) NOT NULL,
    heading INTEGER,
    speed DECIMAL(5,2),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_driver_timestamp (driver_id, timestamp),
    INDEX idx_location (lat, lng)
);
```

## Real-Time Location Tracking

### Geospatial Indexing
```python
import math
from typing import List, Tuple
from dataclasses import dataclass

@dataclass
class GeoPoint:
    lat: float
    lng: float

class GeospatialIndex:
    def __init__(self, grid_size=0.01):  # ~1km grid
        self.grid_size = grid_size
        self.grid = {}

    def get_grid_key(self, point: GeoPoint) -> Tuple[int, int]:
        """Convert lat/lng to grid coordinates"""
        lat_grid = int(point.lat / self.grid_size)
        lng_grid = int(point.lng / self.grid_size)
        return (lat_grid, lng_grid)

    def add_driver(self, driver_id: str, location: GeoPoint):
        """Add driver to spatial index"""
        grid_key = self.get_grid_key(location)

        if grid_key not in self.grid:
            self.grid[grid_key] = []

        self.grid[grid_key].append({
            'driver_id': driver_id,
            'location': location,
            'timestamp': time.time()
        })

    def find_nearby_drivers(self, location: GeoPoint, radius_km: float) -> List[str]:
        """Find drivers within radius"""
        # Calculate grid bounds
        lat_grid = int(location.lat / self.grid_size)
        lng_grid = int(location.lng / self.grid_size)
        grid_radius = int(radius_km / self.grid_size)

        nearby_drivers = []

        # Search surrounding grids
        for lat_offset in range(-grid_radius, grid_radius + 1):
            for lng_offset in range(-grid_radius, grid_radius + 1):
                grid_key = (lat_grid + lat_offset, lng_grid + lng_offset)

                if grid_key in self.grid:
                    for driver in self.grid[grid_key]:
                        distance = self._calculate_distance(
                            location, driver['location']
                        )

                        if distance <= radius_km:
                            nearby_drivers.append(driver['driver_id'])

        return list(set(nearby_drivers))  # Remove duplicates

    def _calculate_distance(self, point1: GeoPoint, point2: GeoPoint) -> float:
        """Calculate distance between two points using Haversine formula"""
        R = 6371  # Earth's radius in km

        lat1_rad = math.radians(point1.lat)
        lat2_rad = math.radians(point2.lat)
        delta_lat = math.radians(point2.lat - point1.lat)
        delta_lng = math.radians(point2.lng - point1.lng)

        a = (math.sin(delta_lat/2)**2 +
             math.cos(lat1_rad) * math.cos(lat2_rad) *
             math.sin(delta_lng/2)**2)
        c = 2 * math.atan2(
            math.sqrt(a),
            math.sqrt(1-a)
        )

        return R * c

# Usage
geo_index = GeospatialIndex()

# Add drivers to index
geo_index.add_driver("driver_1", GeoPoint(37.7749, -122.4194))
geo_index.add_driver("driver_2", GeoPoint(37.7849, -122.4094))
geo_index.add_driver("driver_3", GeoPoint(37.7649, -122.4294))

# Find nearby drivers
pickup_location = GeoPoint(37.7749, -122.4194)
nearby_drivers = geo_index.find_nearby_drivers(pickup_location, radius_km=2)
print(f"Nearby drivers: {nearby_drivers}")
```

### Location Updates with Kafka
```python
import json
from kafka import KafkaProducer
import time

class LocationProducer:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )

    def send_location_update(self, driver_id: str, location: dict):
        """Send driver location update to Kafka"""
        message = {
            'driver_id': driver_id,
            'lat': location['lat'],
            'lng': location['lng'],
            'heading': location.get('heading', 0),
            'speed': location.get('speed', 0),
            'timestamp': time.time()
        }

        self.producer.send(
            topic='driver_locations',
            key=driver_id,
            value=message
        )

        print(f"Sent location update for driver {driver_id}")

# Usage
location_producer = LocationProducer(['localhost:9092'])

# Simulate driver location updates
def simulate_driver_movement():
    driver_id = "driver_123"

    # Start location
    location = {
        'lat': 37.7749,
        'lng': -122.4194,
        'heading': 90,
        'speed': 50  # km/h
    }

    for i in range(10):
        location_producer.send_location_update(driver_id, location)

        # Update location (simulate movement)
        location['lng'] += 0.001  # Move east
        time.sleep(5)  # Update every 5 seconds

simulate_driver_movement()
```

## Ride Matching Algorithm

### Matching Service
```python
import math
from typing import List, Optional
from dataclasses import dataclass
from enum import Enum

class RideStatus(Enum):
    REQUESTED = "requested"
    DRIVER_ASSIGNED = "driver_assigned"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

@dataclass
class Driver:
    id: str
    location: tuple
    rating: float
    is_available: bool
    vehicle_type: str

@dataclass
class RideRequest:
    id: str
    rider_id: str
    pickup_location: tuple
    dropoff_location: tuple
    ride_type: str
    max_wait_time: int = 300  # 5 minutes

class RideMatchingService:
    def __init__(self, geo_index, driver_service):
        self.geo_index = geo_index
        self.driver_service = driver_service

    def find_best_driver(self, ride_request: RideRequest) -> Optional[Driver]:
        """Find best driver for ride request"""
        # Find nearby drivers
        nearby_driver_ids = self.geo_index.find_nearby_drivers(
            GeoPoint(*ride_request.pickup_location),
            radius_km=3
        )

        if not nearby_driver_ids:
            return None

        # Get driver details
        nearby_drivers = []
        for driver_id in nearby_driver_ids:
            driver = self.driver_service.get_driver(driver_id)
            if driver and driver.is_available:
                nearby_drivers.append(driver)

        if not nearby_drivers:
            return None

        # Score drivers based on multiple factors
        best_driver = self._score_drivers(nearby_drivers, ride_request)

        return best_driver

    def _score_drivers(self, drivers: List[Driver], ride_request: RideRequest) -> Driver:
        """Score drivers and return best one"""
        def calculate_score(driver: Driver) -> float:
            # Distance score (closer is better)
            distance = self._calculate_distance(
                driver.location, ride_request.pickup_location
            )
            distance_score = max(0, 100 - distance * 10)

            # Rating score (higher rating is better)
            rating_score = driver.rating * 20

            # Vehicle type score
            vehicle_score = self._get_vehicle_score(driver.vehicle_type, ride_request.ride_type)

            # Availability score
            availability_score = 100 if driver.is_available else 0

            total_score = distance_score + rating_score + vehicle_score + availability_score
            return total_score

        # Sort drivers by score (highest first)
        drivers.sort(key=calculate_score, reverse=True)
        return drivers[0]

    def _calculate_distance(self, point1: tuple, point2: tuple) -> float:
        """Calculate distance between two points"""
        lat1, lng1 = point1
        lat2, lng2 = point2

        # Simplified distance calculation
        return math.sqrt((lat2 - lat1)**2 + (lng2 - lng1)**2)

    def _get_vehicle_score(self, driver_vehicle: str, ride_type: str) -> float:
        """Get vehicle compatibility score"""
            compatibility = {
                'standard': {'standard': 100, 'premium': 50, 'pool': 80},
                'premium': {'standard': 60, 'premium': 100, 'pool': 70},
                'pool': {'standard': 80, 'premium': 70, 'pool': 100}
            }

            return compatibility.get(ride_type, {}).get(driver_vehicle, 50)

# Usage
class DriverService:
    def __init__(self):
        self.drivers = {
            'driver_1': Driver('driver_1', (37.7749, -122.4194), 4.8, True, 'standard'),
            'driver_2': Driver('driver_2', (37.7849, -122.4094), 4.5, True, 'premium'),
            'driver_3': Driver('driver_3', (37.7649, -122.4294), 4.9, False, 'standard'),
        }

    def get_driver(self, driver_id: str) -> Optional[Driver]:
        return self.drivers.get(driver_id)

# Create services
geo_index = GeospatialIndex()
driver_service = DriverService()
matching_service = RideMatchingService(geo_index, driver_service)

# Test ride matching
ride_request = RideRequest(
    id='ride_123',
    rider_id='user_456',
    pickup_location=(37.7749, -122.4194),
    dropoff_location=(37.8044, -122.2711),
    ride_type='standard'
)

best_driver = matching_service.find_best_driver(ride_request)
if best_driver:
    print(f"Best driver found: {best_driver.id} (Rating: {best_driver.rating})")
else:
    print("No available drivers found")
```

## Dynamic Pricing

### Surge Pricing Algorithm
```python
import time
from typing import Dict
from dataclasses import dataclass

@dataclass
class PricingFactors:
    base_price: float
    demand_multiplier: float
    supply_multiplier: float
    time_multiplier: float
    weather_multiplier: float

class DynamicPricingService:
    def __init__(self):
        self.base_prices = {
            'standard': 2.50,  # per km
            'premium': 4.00,
            'pool': 2.00
        }

    def calculate_price(self, ride_request: RideRequest,
                    demand_level: float, supply_level: float,
                    weather_conditions: str = 'clear') -> Dict:
        """Calculate dynamic price based on multiple factors"""

        # Base price calculation
        distance_km = self._calculate_distance(
            ride_request.pickup_location,
            ride_request.dropoff_location
        )

        base_price = self.base_prices[ride_request.ride_type] * distance_km

        # Demand multiplier (surge pricing)
        demand_multiplier = self._calculate_demand_multiplier(demand_level, supply_level)

        # Time multiplier (peak hours)
        time_multiplier = self._calculate_time_multiplier()

        # Weather multiplier
        weather_multiplier = self._calculate_weather_multiplier(weather_conditions)

        # Calculate final price
        final_multiplier = (demand_multiplier *
                          time_multiplier *
                          weather_multiplier)

        final_price = base_price * final_multiplier

        return {
            'base_price': base_price,
            'final_price': final_price,
            'multipliers': {
                'demand': demand_multiplier,
                'time': time_multiplier,
                'weather': weather_multiplier
            },
            'total_multiplier': final_multiplier
        }

    def _calculate_demand_multiplier(self, demand: float, supply: float) -> float:
        """Calculate demand-based surge multiplier"""
        if supply == 0:
            return 3.0  # Maximum surge

        demand_supply_ratio = demand / supply

        if demand_supply_ratio > 2.0:
            return 2.5  # High surge
        elif demand_supply_ratio > 1.5:
            return 2.0  # Medium surge
        elif demand_supply_ratio > 1.0:
            return 1.5  # Low surge
        else:
            return 1.0  # No surge

    def _calculate_time_multiplier(self) -> float:
        """Calculate time-based multiplier"""
        current_hour = time.localtime().tm_hour

        # Peak hours: 7-9 AM and 5-7 PM
        if (7 <= current_hour <= 9) or (17 <= current_hour <= 19):
            return 1.2  # 20% increase during peak hours
        else:
            return 1.0  # Normal pricing

    def _calculate_weather_multiplier(self, weather: str) -> float:
        """Calculate weather-based multiplier"""
        weather_multipliers = {
            'clear': 1.0,
            'rain': 1.3,
            'snow': 1.5,
            'fog': 1.2,
            'storm': 2.0
        }

        return weather_multipliers.get(weather, 1.0)

# Usage
pricing_service = DynamicPricingService()

# Calculate price for different scenarios
normal_scenario = pricing_service.calculate_price(
    ride_request, demand_level=1.0, supply_level=1.0
)

surge_scenario = pricing_service.calculate_price(
    ride_request, demand_level=2.5, supply_level=1.0
)

bad_weather_scenario = pricing_service.calculate_price(
    ride_request, demand_level=1.5, supply_level=1.0, weather_conditions='rain'
)

print(f"Normal pricing: {normal_scenario}")
print(f"Surge pricing: {surge_scenario}")
print(f"Bad weather pricing: {bad_weather_scenario}")
```

## Real-World Examples

### Uber Architecture
```
Components:
- Microservices architecture
- Real-time location tracking
- Machine learning for matching
- Dynamic pricing algorithms
- Multi-region deployment

Technologies:
- Go for high-performance services
- Kafka for event streaming
- Redis for caching
- PostgreSQL for transactional data
- Elasticsearch for analytics
```

### Lyft Architecture
```
Components:
- Service-oriented architecture
- Geospatial indexing
- Real-time communication
- Predictive pricing
- Driver scheduling

Technologies:
- Python for business logic
- WebSockets for real-time updates
- Redis for session management
- MongoDB for flexible data storage
- AWS infrastructure
```

## Interview Tips

### Common Questions
1. "How would you handle real-time location tracking?"
2. "What's your approach to ride matching?"
3. "How would you implement dynamic pricing?"
4. "How would you handle payment processing?"
5. "How would you scale to millions of users?"

### Answer Framework
1. **Requirements Clarification**: Scale, latency, real-time requirements
2. **High-Level Design**: Microservices, data flow, key components
3. **Deep Dive**: Location tracking, matching algorithm, pricing
4. **Scalability**: Caching, load balancing, data partitioning
5. **Edge Cases**: No drivers available, cancellations, payment failures

### Key Points to Emphasize
- Real-time communication challenges
- Geospatial indexing importance
- Dynamic pricing algorithms
- Payment processing and security
- Driver and passenger safety features

## Practice Problems

1. **Design real-time location tracking system**
2. **Implement efficient ride matching algorithm**
3. **Create dynamic pricing engine**
4. **Design payment processing system**
5. **Handle emergency situations and safety features**

## Further Reading

- **Books**: "The Architecture of Open Source Applications" by Amy Brown
- **Papers**: "The Google Maps Architecture", "Uber Engineering Blog"
- **Concepts**: Geospatial Indexing, Real-time Systems, Dynamic Pricing Algorithms
