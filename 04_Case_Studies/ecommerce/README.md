# E-commerce Platform (Amazon/Shopify)

## Problem Statement

Design a scalable e-commerce platform like Amazon or Shopify that handles product catalog management, user shopping experiences, order processing, and payment integration.

## Requirements

### Functional Requirements
- User registration and authentication
- Product catalog management
- Search and filtering capabilities
- Shopping cart and checkout
- Order management and tracking
- Payment processing
- Inventory management
- Review and rating system
- Recommendation engine

### Non-Functional Requirements
- **Scale**: 100 million daily active users, 10 million concurrent users
- **Latency**: < 200ms for search queries, < 2 seconds for checkout
- **Availability**: 99.99% uptime
- **Consistency**: Strong consistency for orders, eventual for inventory
- **Security**: PCI DSS compliance for payments

## Scale Estimation

### Traffic Estimates
```
Daily Active Users: 100 million
Daily Orders: 5 million
Peak Concurrent Users: 10 million
Product Catalog Size: 500 million products
Search Queries per Second: 100,000
API Requests per Second: 200,000
```

### Storage Estimates
```
User Data: 100M × 2KB = 200GB
Product Data: 500M × 5KB = 2.5TB
Order Data: 5M × 2KB = 10GB/day = 3.6TB/year
Review Data: 50M × 1KB = 50GB/day = 18TB/year
Image Storage: 500M × 100KB = 50TB
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
│  │   User      │  │   Product   │  │   Order     │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│  User DB        │  Product DB      │  Order DB          │
│  (PostgreSQL)   │  (PostgreSQL)    │  (PostgreSQL)      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Search & Analytics Services                │
│                                                         │
│  ┌─────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │   Search    │  │   Recommendation│  │   Analytics │  │
│  │   Service   │  │   Service       │  │   Service   │  │
│  └─────────────┘  └─────────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Data Stores                                │
│                                                         │
│  ┌─────────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │   Elasticsearch │  │   Redis     │  │   S3        │  │
│  │   (Search)      │  │   (Cache)   │  │   (Storage) │  │
│  └─────────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## API Design

### User APIs
```http
POST /api/v1/users/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepassword"
}

POST /api/v1/users/login
{
  "email": "john@example.com",
  "password": "securepassword"
}

GET /api/v1/users/profile
Response:
{
  "user_id": "user_123",
  "name": "John Doe",
  "email": "john@example.com",
  "addresses": [...],
  "payment_methods": [...]
}
```

### Product APIs
```http
GET /api/v1/products/search?q=laptop&category=electronics&page=1&limit=20
Response:
{
  "products": [
    {
      "id": "prod_123",
      "name": "Gaming Laptop",
      "price": 1299.99,
      "rating": 4.5,
      "image_url": "https://...",
      "in_stock": true
    }
  ],
  "total": 150,
  "page": 1,
  "limit": 20
}

GET /api/v1/products/{product_id}
Response:
{
  "id": "prod_123",
  "name": "Gaming Laptop",
  "description": "High-performance gaming laptop",
  "price": 1299.99,
  "category": "electronics",
  "brand": "TechBrand",
  "specifications": {...},
  "reviews": [...],
  "related_products": [...]
}
```

### Order APIs
```http
POST /api/v1/orders
{
  "user_id": "user_123",
  "items": [
    {
      "product_id": "prod_123",
      "quantity": 1,
      "price": 1299.99
    }
  ],
  "shipping_address": {...},
  "payment_method": {...}
}

GET /api/v1/orders/{order_id}
Response:
{
  "order_id": "order_456",
  "status": "confirmed",
  "items": [...],
  "total": 1299.99,
  "shipping_address": {...},
  "tracking_number": "TR123456"
}
```

## Database Design

### Users Table
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email)
);
```

### Products Table
```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category VARCHAR(100),
    brand VARCHAR(100),
    in_stock BOOLEAN DEFAULT TRUE,
    stock_quantity INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_category (category),
    INDEX idx_brand (brand),
    FULLTEXT idx_name_description (name, description)
);
```

### Orders Table
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    status ENUM('pending', 'confirmed', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(10,2) NOT NULL,
    shipping_address JSON,
    payment_method JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

## Product Search Implementation

### Elasticsearch Integration
```python
from elasticsearch import Elasticsearch
from typing import List, Dict

class ProductSearchService:
    def __init__(self, es_host: str):
        self.es = Elasticsearch([es_host])
        self.index_name = 'products'

    def index_product(self, product: Dict):
        """Index a product in Elasticsearch"""
        self.es.index(
            index=self.index_name,
            id=product['id'],
            body={
                'name': product['name'],
                'description': product['description'],
                'category': product['category'],
                'brand': product['brand'],
                'price': product['price'],
                'rating': product.get('rating', 0),
                'in_stock': product.get('in_stock', True)
            }
        )

    def search_products(self, query: str, filters: Dict = None,
                       page: int = 1, limit: int = 20) -> Dict:
        """Search products with filters"""
        search_body = {
            'query': {
                'bool': {
                    'must': [
                        {
                            'multi_match': {
                                'query': query,
                                'fields': ['name^3', 'description^2', 'brand', 'category']
                            }
                        }
                    ],
                    'filter': []
                }
            },
            'sort': [
                {'_score': 'desc'},
                {'rating': 'desc'},
                {'price': 'asc'}
            ],
            'from': (page - 1) * limit,
            'size': limit
        }

        # Add filters
        if filters:
            if 'category' in filters:
                search_body['query']['bool']['filter'].append({
                    'term': {'category': filters['category']}
                })
            if 'brand' in filters:
                search_body['query']['bool']['filter'].append({
                    'term': {'brand': filters['brand']}
                })
            if 'price_min' in filters:
                search_body['query']['bool']['filter'].append({
                    'range': {'price': {'gte': filters['price_min']}}
                })
            if 'price_max' in filters:
                search_body['query']['bool']['filter'].append({
                    'range': {'price': {'lte': filters['price_max']}}
                })

        result = self.es.search(index=self.index_name, body=search_body)

        return {
            'products': [hit['_source'] for hit in result['hits']['hits']],
            'total': result['hits']['total']['value'],
            'page': page,
            'limit': limit
        }

# Usage
search_service = ProductSearchService('localhost:9200')

# Index products
product = {
    'id': 'prod_123',
    'name': 'Gaming Laptop',
    'description': 'High-performance gaming laptop',
    'category': 'electronics',
    'brand': 'TechBrand',
    'price': 1299.99,
    'rating': 4.5,
    'in_stock': True
}
search_service.index_product(product)

# Search products
results = search_service.search_products(
    query='gaming laptop',
    filters={'category': 'electronics', 'price_max': 1500},
    page=1,
    limit=10
)
print(f"Found {results['total']} products")
```

## Recommendation Engine

### Collaborative Filtering
```python
import numpy as np
from typing import List, Dict, Tuple
from collections import defaultdict

class RecommendationEngine:
    def __init__(self):
        self.user_item_matrix = defaultdict(dict)
        self.item_user_matrix = defaultdict(dict)

    def add_rating(self, user_id: str, item_id: str, rating: float):
        """Add user-item rating"""
        self.user_item_matrix[user_id][item_id] = rating
        self.item_user_matrix[item_id][user_id] = rating

    def get_similar_users(self, user_id: str, k: int = 5) -> List[Tuple[str, float]]:
        """Find k most similar users"""
        if user_id not in self.user_item_matrix:
            return []

        user_ratings = self.user_item_matrix[user_id]
        similarities = []

        for other_user, other_ratings in self.user_item_matrix.items():
            if other_user == user_id:
                continue

            similarity = self._cosine_similarity(user_ratings, other_ratings)
            similarities.append((other_user, similarity))

        # Sort by similarity (descending)
        similarities.sort(key=lambda x: x[1], reverse=True)
        return similarities[:k]

    def recommend_items(self, user_id: str, k: int = 10) -> List[Tuple[str, float]]:
        """Recommend items for user"""
        if user_id not in self.user_item_matrix:
            return []

        similar_users = self.get_similar_users(user_id, k=20)
        user_rated_items = set(self.user_item_matrix[user_id].keys())

        item_scores = defaultdict(float)
        item_weights = defaultdict(float)

        for similar_user, similarity in similar_users:
            for item, rating in self.user_item_matrix[similar_user].items():
                if item not in user_rated_items:
                    item_scores[item] += rating * similarity
                    item_weights[item] += similarity

        # Calculate weighted average ratings
        recommendations = []
        for item in item_scores:
            if item_weights[item] > 0:
                score = item_scores[item] / item_weights[item]
                recommendations.append((item, score))

        # Sort by score (descending)
        recommendations.sort(key=lambda x: x[1], reverse=True)
        return recommendations[:k]

    def _cosine_similarity(self, ratings1: Dict[str, float],
                          ratings2: Dict[str, float]) -> float:
        """Calculate cosine similarity between two rating vectors"""
        common_items = set(ratings1.keys()) & set(ratings2.keys())

        if not common_items:
            return 0.0

        # Calculate dot product
        dot_product = sum(ratings1[item] * ratings2[item] for item in common_items)

        # Calculate magnitudes
        magnitude1 = np.sqrt(sum(rating**2 for rating in ratings1.values()))
        magnitude2 = np.sqrt(sum(rating**2 for rating in ratings2.values()))

        if magnitude1 == 0 or magnitude2 == 0:
            return 0.0

        return dot_product / (magnitude1 * magnitude2)

# Usage
rec_engine = RecommendationEngine()

# Add sample ratings
rec_engine.add_rating('user_1', 'prod_1', 5.0)
rec_engine.add_rating('user_1', 'prod_2', 4.0)
rec_engine.add_rating('user_2', 'prod_1', 5.0)
rec_engine.add_rating('user_2', 'prod_3', 4.0)
rec_engine.add_rating('user_3', 'prod_2', 3.0)
rec_engine.add_rating('user_3', 'prod_3', 5.0)

# Get recommendations
recommendations = rec_engine.recommend_items('user_1', k=5)
print("Recommended products:", recommendations)
```

## Inventory Management

### Optimistic Concurrency Control
```python
import psycopg2
from psycopg2.extras import RealDictCursor
from typing import Optional

class InventoryService:
    def __init__(self, db_connection_string: str):
        self.conn = psycopg2.connect(db_connection_string)

    def check_inventory(self, product_id: str) -> int:
        """Check current inventory for product"""
        with self.conn.cursor(cursor_factory=RealDictCursor) as cursor:
            cursor.execute(
                "SELECT stock_quantity FROM products WHERE id = %s",
                (product_id,)
            )
            result = cursor.fetchone()
            return result['stock_quantity'] if result else 0

    def reserve_inventory(self, product_id: str, quantity: int) -> bool:
        """Reserve inventory with optimistic locking"""
        with self.conn.cursor() as cursor:
            # Check current stock
            cursor.execute(
                "SELECT stock_quantity, version FROM products WHERE id = %s",
                (product_id,)
            )
            result = cursor.fetchone()

            if not result:
                return False

            current_stock, version = result

            if current_stock < quantity:
                return False

            # Attempt to update with version check
            cursor.execute(
                """
                UPDATE products
                SET stock_quantity = stock_quantity - %s, version = version + 1
                WHERE id = %s AND version = %s
                """,
                (quantity, product_id, version)
            )

            if cursor.rowcount == 0:
                # Version mismatch - concurrent update occurred
                self.conn.rollback()
                return False

            self.conn.commit()
            return True

    def release_inventory(self, product_id: str, quantity: int):
        """Release reserved inventory"""
        with self.conn.cursor() as cursor:
            cursor.execute(
                "UPDATE products SET stock_quantity = stock_quantity + %s WHERE id = %s",
                (quantity, product_id)
            )
            self.conn.commit()

# Usage
inventory_service = InventoryService("postgresql://user:pass@localhost/ecommerce")

# Check inventory
stock = inventory_service.check_inventory('prod_123')
print(f"Current stock: {stock}")

# Reserve inventory for order
if inventory_service.reserve_inventory('prod_123', 2):
    print("Inventory reserved successfully")
    # Process order...
else:
    print("Insufficient inventory or concurrent update")
```

## Real-World Examples

### Amazon Architecture
```
Components:
- Microservices architecture
- Real-time inventory tracking
- Machine learning recommendations
- Multi-region deployment
- Advanced search capabilities

Technologies:
- Java for core services
- DynamoDB for product catalog
- Elasticsearch for search
- S3 for image storage
- Redshift for analytics
```

### Shopify Architecture
```
Components:
- Multi-tenant platform
- Theme customization
- App marketplace
- Payment gateway integration
- Merchant analytics

Technologies:
- Ruby on Rails
- PostgreSQL for data storage
- Redis for caching
- Sidekiq for background jobs
- AWS infrastructure
```

## Interview Tips

### Common Questions
1. "How would you design the product search system?"
2. "What's your approach to inventory management?"
3. "How would you implement recommendations?"
4. "How would you handle flash sales?"
5. "How would you scale the checkout process?"

### Answer Framework
1. **Requirements Clarification**: Scale, latency, consistency requirements
2. **High-Level Design**: Microservices, data flow, key components
3. **Deep Dive**: Search indexing, inventory concurrency, recommendation algorithms
4. **Scalability**: Caching, load balancing, data partitioning
5. **Edge Cases**: Stockouts, payment failures, high concurrency

### Key Points to Emphasize
- Search performance and relevance
- Inventory consistency challenges
- Recommendation algorithm effectiveness
- Payment security and compliance
- Scalability for peak traffic

## Practice Problems

1. **Design product search with filters**
2. **Implement inventory management with concurrency**
3. **Create recommendation engine**
4. **Design checkout with payment processing**
5. **Handle flash sales and high traffic**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Papers**: "Amazon Architecture", "Shopify Engineering Blog"
- **Concepts**: Search Engines, Recommendation Systems, Inventory Management
