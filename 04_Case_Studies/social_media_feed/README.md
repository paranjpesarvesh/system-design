# Social Media Feed (Twitter/Instagram)

## Problem Statement

Design a social media feed system that generates personalized timelines for millions of users, showing posts from people they follow in reverse chronological order.

## Requirements

### Functional Requirements
- Users can post updates (text, images, videos)
- Users can follow/unfollow other users
- Generate personalized feed for each user
- Support likes, comments, shares
- Real-time feed updates
- Search and discovery features

### Non-Functional Requirements
- Low latency (feed generation < 200ms)
- High availability (99.9%)
- Scalability to 500M+ users
- Real-time updates
- Support for viral content

## Scale Estimation

### Traffic Estimates
- **Daily Active Users**: 500 million
- **Posts per day**: 500 million
- **Average followers per user**: 500
- **Feed requests per second**: 100,000
- **Peak QPS**: 500,000

### Storage Estimates
- **Post size**: Average 1KB (text + metadata)
- **Media size**: Average 500KB per post
- **Daily storage**: 500M × 1KB = 500GB (text)
- **Daily media storage**: 500M × 500KB = 250TB
- **Annual storage**: ~90PB total

## High-Level Design

```
Client → Load Balancer → API Gateway
                    ↓
                [Feed Service] [Post Service] [User Service]
                    ↓                ↓              ↓
                [Timeline Cache]  [Post DB]    [User DB]
                    ↓                ↓              ↓
                [Redis Cluster]  [PostgreSQL]  [PostgreSQL]
```

## API Design

### Create Post
```
POST /api/v1/posts
Content-Type: application/json

{
  "content": "Hello world!",
  "media_urls": ["https://cdn.example.com/image.jpg"],
  "visibility": "public"
}

Response:
{
  "post_id": "post_123456",
  "user_id": "user_789",
  "content": "Hello world!",
  "media_urls": ["https://cdn.example.com/image.jpg"],
  "created_at": "2024-01-01T12:00:00Z",
  "like_count": 0,
  "comment_count": 0
}
```

### Get Feed
```
GET /api/v1/feed?limit=20&before=cursor_123

Response:
{
  "posts": [
    {
      "post_id": "post_123456",
      "user": {
        "user_id": "user_789",
        "username": "john_doe",
        "avatar": "https://cdn.example.com/avatar.jpg"
      },
      "content": "Hello world!",
      "media_urls": ["https://cdn.example.com/image.jpg"],
      "created_at": "2024-01-01T12:00:00Z",
      "like_count": 150,
      "comment_count": 23,
      "is_liked": false
    }
  ],
  "next_cursor": "cursor_456",
  "has_more": true
}
```

### Follow/Unfollow User
```
POST /api/v1/users/{user_id}/follow
DELETE /api/v1/users/{user_id}/follow
```

## Database Design

### Posts Table
```sql
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    media_urls JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    like_count BIGINT DEFAULT 0,
    comment_count BIGINT DEFAULT 0,
    share_count BIGINT DEFAULT 0,
    visibility VARCHAR(20) DEFAULT 'public',
    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_created (created_at DESC),
    INDEX idx_trending (created_at DESC, like_count DESC)
);
```

### Users Table
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    bio TEXT,
    follower_count BIGINT DEFAULT 0,
    following_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username),
    INDEX idx_follower_count (follower_count DESC)
);
```

### Follows Table
```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    INDEX idx_follower (follower_id),
    INDEX idx_following (following_id),
    FOREIGN KEY (follower_id) REFERENCES users(id),
    FOREIGN KEY (following_id) REFERENCES users(id)
);
```

### Feed Generation Strategies

### 1. Fan-out on Write (Push Model)
```
User A posts → Timeline Cache
     ↓
[User B Timeline] [User C Timeline] [User D Timeline]
```

**Implementation:**
```python
class FeedService:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.fanout_queue = Queue()
    
    def create_post(self, user_id, content):
        # Create post in database
        post_id = self.database.create_post(user_id, content)
        
        # Get followers
        followers = self.database.get_followers(user_id)
        
        # Fan-out to followers' timelines
        for follower_id in followers:
            self.add_to_timeline(follower_id, post_id)
        
        return post_id
    
    def add_to_timeline(self, user_id, post_id):
        timeline_key = f"timeline:{user_id}"
        self.redis_client.lpush(timeline_key, post_id)
        self.redis_client.ltrim(timeline_key, 0, 999)  # Keep last 1000 posts
```

**Pros:**
- Fast feed reads
- Real-time updates
- Simple read logic

**Cons:**
- High write amplification
- Storage overhead
- Not suitable for users with millions of followers

### 2. Pull Model (Read-time Generation)
```
User requests feed → Get followed users → Fetch recent posts → Merge and sort
```

**Implementation:**
```python
class PullFeedService:
    def generate_feed(self, user_id, limit=20):
        # Get followed users
        followed_users = self.database.get_followed_users(user_id)
        
        # Fetch recent posts from followed users
        posts = []
        for followed_id in followed_users:
            user_posts = self.database.get_recent_posts(followed_id, limit)
            posts.extend(user_posts)
        
        # Sort by creation time
        posts.sort(key=lambda x: x['created_at'], reverse=True)
        
        return posts[:limit]
```

**Pros:**
- No write amplification
- Always up-to-date
- Storage efficient

**Cons:**
- Slow feed generation
- High read load
- Not suitable for real-time

### 3. Hybrid Approach
```
Normal users: Fan-out on Write
Celebrities: Pull on Read
```

**Implementation:**
```python
class HybridFeedService:
    def create_post(self, user_id, content):
        post_id = self.database.create_post(user_id, content)
        followers = self.database.get_followers(user_id)
        
        # Check if user is celebrity (many followers)
        if len(followers) > 10000:
            # Use pull model for celebrities
            self.mark_as_celebrity_post(post_id)
        else:
            # Use push model for normal users
            for follower_id in followers:
                self.add_to_timeline(follower_id, post_id)
        
        return post_id
    
    def generate_feed(self, user_id, limit=20):
        # Get timeline from cache
        timeline_key = f"timeline:{user_id}"
        cached_posts = self.redis_client.lrange(timeline_key, 0, limit-1)
        
        # Fill with celebrity posts if needed
        if len(cached_posts) < limit:
            celebrity_posts = self.get_celebrity_posts(user_id, limit - len(cached_posts))
            cached_posts.extend(celebrity_posts)
        
        return cached_posts
```

## Caching Strategy

### Multi-level Caching
```
Client → CDN → Application Cache → Database
```

### Timeline Cache Design
```python
class TimelineCache:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.cache_ttl = 3600  # 1 hour
    
    def get_timeline(self, user_id, limit=20):
        cache_key = f"timeline:{user_id}"
        
        # Get from cache
        post_ids = self.redis_client.lrange(cache_key, 0, limit-1)
        
        if not post_ids:
            # Cache miss - generate timeline
            post_ids = self.generate_timeline(user_id, limit)
            self.cache_timeline(user_id, post_ids)
        
        return self.get_posts_by_ids(post_ids)
    
    def add_post_to_timeline(self, user_id, post_id):
        cache_key = f"timeline:{user_id}"
        
        # Add to beginning of timeline
        self.redis_client.lpush(cache_key, post_id)
        
        # Keep only recent posts
        self.redis_client.ltrim(cache_key, 0, 999)
        
        # Set expiration
        self.redis_client.expire(cache_key, self.cache_ttl)
```

## Real-time Updates

### WebSocket Implementation
```python
class FeedWebSocket:
    def __init__(self):
        self.connections = {}  # user_id -> websocket connection
    
    def handle_connection(self, websocket, user_id):
        self.connections[user_id] = websocket
        
        try:
            while True:
                message = websocket.receive()
                self.handle_message(user_id, message)
        except:
            del self.connections[user_id]
    
    def broadcast_new_post(self, post):
        # Get followers
        followers = self.database.get_followers(post['user_id'])
        
        # Send real-time update to connected followers
        for follower_id in followers:
            if follower_id in self.connections:
                try:
                    self.connections[follower_id].send({
                        'type': 'new_post',
                        'post': post
                    })
                except:
                    # Connection closed
                    del self.connections[follower_id]
```

### Push Notifications
```python
class PushNotificationService:
    def __init__(self):
        self.fcm_service = FCMService()
        self.apns_service = APNSService()
    
    def send_new_post_notification(self, post, followers):
        for follower in followers:
            if follower['push_enabled']:
                device_tokens = follower['device_tokens']
                
                notification = {
                    'title': f"New post from {post['username']}",
                    'body': post['content'][:100],
                    'data': {
                        'post_id': post['id'],
                        'type': 'new_post'
                    }
                }
                
                # Send based on device platform
                for token in device_tokens:
                    if token['platform'] == 'ios':
                        self.apns_service.send(token['token'], notification)
                    else:
                        self.fcm_service.send(token['token'], notification)
```

## Media Storage

### Object Storage Architecture
```
User Upload → CDN Edge → Processing Pipeline → Object Storage
                    ↓
                [Thumbnail Generation]
                [Content Moderation]
                [Virus Scanning]
```

### Media Processing Pipeline
```python
class MediaProcessor:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.image_processor = ImageProcessor()
        self.video_processor = VideoProcessor()
    
    def process_media(self, user_id, media_file):
        # Upload original to S3
        original_key = f"originals/{user_id}/{generate_uuid()}"
        self.s3_client.upload_fileobj(media_file, 'social-media', original_key)
        
        # Generate thumbnails
        if media_file.type.startswith('image/'):
            thumbnail_key = self.image_processor.generate_thumbnail(original_key)
        elif media_file.type.startswith('video/'):
            thumbnail_key = self.video_processor.generate_thumbnail(original_key)
        
        # Run content moderation
        moderation_result = self.moderate_content(original_key)
        
        return {
            'original_url': f"https://cdn.example.com/{original_key}",
            'thumbnail_url': f"https://cdn.example.com/{thumbnail_key}",
            'moderation_status': moderation_result
        }
```

## Search and Discovery

### Elasticsearch Integration
```python
class SearchService:
    def __init__(self):
        self.es_client = Elasticsearch(['localhost:9200'])
    
    def index_post(self, post):
        doc = {
            'post_id': post['id'],
            'user_id': post['user_id'],
            'username': post['username'],
            'content': post['content'],
            'hashtags': self.extract_hashtags(post['content']),
            'mentions': self.extract_mentions(post['content']),
            'created_at': post['created_at'],
            'like_count': post['like_count']
        }
        
        self.es_client.index(
            index='posts',
            id=post['id'],
            body=doc
        )
    
    def search_posts(self, query, filters=None):
        search_body = {
            'query': {
                'bool': {
                    'must': [
                        {'match': {'content': query}}
                    ],
                    'filter': filters or []
                }
            },
            'sort': [
                {'created_at': {'order': 'desc'}},
                {'like_count': {'order': 'desc'}}
            ]
        }
        
        response = self.es_client.search(
            index='posts',
            body=search_body,
            size=20
        )
        
        return [hit['_source'] for hit in response['hits']['hits']]
```

## Performance Optimization

### Database Optimization
```sql
-- Partition posts by date
CREATE TABLE posts_2024_01 PARTITION OF posts
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Materialized view for trending posts
CREATE MATERIALIZED VIEW trending_posts AS
SELECT p.*, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.created_at >= NOW() - INTERVAL '24 hours'
ORDER BY p.like_count DESC, p.created_at DESC;

-- Refresh every 5 minutes
CREATE OR REPLACE FUNCTION refresh_trending()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY trending_posts;
END;
$$ LANGUAGE plpgsql;
```

### Connection Pooling
```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Read replicas for feed generation
read_engine = create_engine(
    'postgresql://user:pass@read-replica:5432/socialmedia',
    poolclass=QueuePool,
    pool_size=50,
    max_overflow=100
)

# Write master for post creation
write_engine = create_engine(
    'postgresql://user:pass@master:5432/socialmedia',
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=40
)
```

## Analytics and Metrics

### Real-time Analytics
```python
class AnalyticsService:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.kafka_producer = KafkaProducer()
    
    def track_post_interaction(self, post_id, user_id, action):
        # Increment counters
        self.redis_client.incr(f"post:{post_id}:{action}")
        self.redis_client.incr(f"user:{user_id}:{action}")
        
        # Send to analytics pipeline
        event = {
            'post_id': post_id,
            'user_id': user_id,
            'action': action,
            'timestamp': time.time()
        }
        
        self.kafka_producer.send('post_interactions', event)
    
    def get_trending_posts(self, time_window=3600):
        # Get posts with most interactions in last hour
        trending_keys = self.redis_client.keys("post:*:like")
        
        post_scores = {}
        for key in trending_keys:
            post_id = key.decode().split(':')[1]
            likes = int(self.redis_client.get(key) or 0)
            post_scores[post_id] = likes
        
        # Sort by score
        return sorted(post_scores.items(), key=lambda x: x[1], reverse=True)[:50]
```

## Deployment Architecture

### Microservices Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feed-service
spec:
  replicas: 50
  selector:
    matchLabels:
      app: feed-service
  template:
    metadata:
      labels:
        app: feed-service
    spec:
      containers:
      - name: feed-service
        image: social-media/feed-service:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: REDIS_URL
          value: "redis://redis-cluster:6379"
        - name: DB_URL
          value: "postgresql://user:pass@postgres:5432/socialmedia"
```

## Interview Discussion Points

### Trade-offs
1. **Fan-out on Write vs Pull on Read**: Speed vs storage efficiency
2. **Real-time vs Eventually Consistent**: User experience vs system complexity
3. **Monolithic vs Microservices**: Development speed vs operational complexity
4. **SQL vs NoSQL**: Consistency vs scalability

### Follow-up Questions
1. How would you handle viral content?
2. What about content moderation?
3. How to implement story features?
4. How to handle GDPR compliance?
5. What about internationalization?

## Extensions

### Advanced Features
- **Stories**: Ephemeral content that disappears after 24 hours
- **Live Streaming**: Real-time video broadcasting
- **Algorithmic Feed**: ML-powered content recommendation
- **Social Graph Analysis**: Friend suggestions, influence scoring
- **Content Moderation**: AI-powered content filtering

### Business Features
- **Advertising**: Targeted ad delivery
- **Analytics Dashboard**: User engagement metrics
- **Creator Tools**: Analytics, scheduling, insights
- **Business Accounts**: Professional features, verification

## Further Reading

- **Papers**: "Feed Ranking: A Large-scale Study" by Facebook
- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Documentation**: Redis Documentation, Elasticsearch Guide