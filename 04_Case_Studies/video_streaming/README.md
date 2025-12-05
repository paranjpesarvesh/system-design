# Video Streaming Platform (Netflix/YouTube)

## Problem Statement

Design a video streaming platform like Netflix or YouTube that handles video upload, processing, storage, and delivery to millions of concurrent users.

## Requirements

### Functional Requirements
- Video upload and transcoding
- Adaptive bitrate streaming
- User authentication and profiles
- Search and recommendations
- Comments and ratings
- Watch history and progress tracking
- Multiple device support

### Non-Functional Requirements
- **Scale**: 100 million users, 10 million concurrent streams
- **Latency**: < 2 seconds for video start
- **Availability**: 99.99% uptime
- **Storage**: Petabytes of video content
- **Bandwidth**: Support 4K streaming at 60fps

## Scale Estimation

### Traffic Estimates
```
Daily Active Users: 100 million
Concurrent Streams: 10 million peak
Videos Uploaded Daily: 1 million
Average Video Duration: 45 minutes
Video Quality: 1080p (5 Mbps), 4K (25 Mbps)
```

### Storage Estimates
```
Original Videos: 1M × 2GB = 2TB/day = 730TB/year
Transoded Videos: 1M × 4 variants × 1GB = 4TB/day = 1.46PB/year
Thumbnails: 1M × 50KB = 50GB/day = 18TB/year
Total Storage: ~2.2PB/year
```

### Bandwidth Estimates
```
Peak Streaming: 10M × 5Mbps = 50Tbps
Average Streaming: 5M × 5Mbps = 25Tbps
Upload Bandwidth: 1M × 10Mbps = 10Tbps
CDN Bandwidth: 100Tbps (with redundancy)
```

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    Load Balancer                                │
└─────────────────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│                 API Gateway                                     │
└─────────────────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   User      │  │   Video     │  │   Stream    │              │
│  │   Service   │  │   Service   │  │   Service   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────────────┐
│  User DB          │  Video DB             │  Stream DB          │
│ (PostgreSQL)      │  (PostgreSQL)         │  (Redis)            │
└─────────────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────────────┐
│              Object Storage (S3)                                │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Original  │  │   Transcoded│  │   Thumbnails│              │
│  │   Videos    │  │   Videos    │  │             │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    CDN (Edge Locations)                         │
└─────────────────────────────────────────────────────────────────┘
```

## API Design

### Video Upload API
```http
POST /api/v1/videos/upload
Content-Type: multipart/form-data

Request:
- video_file: (binary)
- title: "My Video"
- description: "Video description"
- tags: ["tag1", "tag2"]
- privacy: "public"
- thumbnail_time: 30

Response:
{
  "video_id": "vid_123456",
  "upload_url": "https://upload.example.com/vid_123456",
  "status": "processing",
  "estimated_processing_time": "15 minutes"
}
```

### Streaming API
```http
GET /api/v1/videos/{video_id}/stream
Headers:
  Range: bytes=0-1048575

Response:
Content-Type: video/mp4
Content-Length: 10485760
Accept-Ranges: bytes
Content-Range: bytes 0-10485760/10485760

[Video Data]
```

### Adaptive Streaming API
```http
GET /api/v1/videos/{video_id}/playlist.m3u8

Response:
#EXTM3U8
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:1800.0
#EXT-X-MEDIA:URI="low.mp4",TYPE="video/mp4",CODECS="avc1.42E01E,mp4a.40.2"
#EXT-X-MEDIA:URI="medium.mp4",TYPE="video/mp4",CODECS="avc1.42E01E,mp4a.40.2"
#EXT-X-MEDIA:URI="high.mp4",TYPE="video/mp4",CODECS="avc1.42E01E,mp4a.40.2"
#EXT-X-MEDIA:URI="ultra.mp4",TYPE="video/mp4",CODECS="hvc1.42E01E,mp4a.40.2"
#EXT-X-STREAM-INF:BANDWIDTH=1280000,RESOLUTION=426x240,low
#EXT-X-STREAM-INF:BANDWIDTH=2560000,RESOLUTION=640x360,medium
#EXT-X-STREAM-INF:BANDWIDTH=5120000,RESOLUTION=1280x720,high
#EXT-X-STREAM-INF:BANDWIDTH=10240000,RESOLUTION=1920x1080,ultra
#EXTINF
#EXT-X-ENDLIST
```

## Database Design

### Videos Table
```sql
CREATE TABLE videos (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    duration_seconds INTEGER NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    resolution VARCHAR(20),
    codec VARCHAR(50),
    upload_status ENUM('uploading', 'processing', 'completed', 'failed') DEFAULT 'uploading',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_id (user_id),
    INDEX idx_upload_status (upload_status),
    INDEX idx_created_at (created_at)
);
```

### Video Variants Table
```sql
CREATE TABLE video_variants (
    id BIGINT PRIMARY KEY,
    video_id BIGINT NOT NULL REFERENCES videos(id),
    quality ENUM('low', 'medium', 'high', 'ultra') NOT NULL,
    codec VARCHAR(50) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    bitrate_kbps INTEGER NOT NULL,
    resolution VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_video_id_quality (video_id, quality),
    INDEX idx_file_path (file_path)
);
```

### Streaming Sessions Table
```sql
CREATE TABLE streaming_sessions (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    video_id BIGINT NOT NULL REFERENCES videos(id),
    start_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    end_time TIMESTAMP,
    watch_duration_seconds INTEGER,
    ip_address INET,
    user_agent TEXT,

    INDEX idx_user_id (user_id),
    INDEX idx_video_id (video_id),
    INDEX idx_start_time (start_time)
);
```

## Video Processing Pipeline

### Upload Processing
```python
import os
import uuid
import boto3
from typing import Dict, Any

class VideoUploadService:
    def __init__(self, s3_client, upload_bucket):
        self.s3_client = s3_client
        self.upload_bucket = upload_bucket
        self.processing_queue = []

    def generate_upload_url(self, video_id: str, file_extension: str) -> Dict[str, Any]:
        """Generate pre-signed URL for video upload"""
        object_key = f"uploads/{video_id}/original.{file_extension}"

        url = self.s3_client.generate_presigned_url(
            ClientMethod='put',
            Params={'Bucket': self.upload_bucket, 'Key': object_key},
            ExpiresIn=3600  # 1 hour
        )

        return {
            'upload_url': url,
            'object_key': object_key,
            'video_id': video_id
        }

    def process_uploaded_video(self, video_id: str, object_key: str):
        """Process uploaded video through transcoding pipeline"""
        # Add to processing queue
        self.processing_queue.append({
            'video_id': video_id,
            'object_key': object_key,
            'status': 'queued',
            'timestamp': time.time()
        })

        # Update database status
        self._update_video_status(video_id, 'processing')

        # Trigger transcoding pipeline
        self._start_transcoding(video_id, object_key)

    def _update_video_status(self, video_id: str, status: str):
        """Update video processing status in database"""
        # Database update logic here
        pass

    def _start_transcoding(self, video_id: str, input_key: str):
        """Start video transcoding process"""
        # This would trigger AWS Elemental, or custom transcoding service
        transcoding_job = {
            'input_key': input_key,
            'output_key_prefix': f"processed/{video_id}/",
            'qualities': ['low', 'medium', 'high', 'ultra'],
            'formats': ['mp4', 'webm']
        }

        # Submit transcoding job
        self._submit_transcoding_job(transcoding_job)

    def _submit_transcoding_job(self, job_config: Dict[str, Any]):
        """Submit job to transcoding service"""
        # Integration with transcoding service
        pass

# Usage
s3_client = boto3.client('s3')
upload_service = VideoUploadService(s3_client, 'video-uploads')

# Generate upload URL
video_id = str(uuid.uuid4())
upload_info = upload_service.generate_upload_url(video_id, 'mp4')

print(f"Upload URL: {upload_info['upload_url']}")
print(f"Video ID: {upload_info['video_id']}")
```

### Adaptive Streaming
```python
import os
import subprocess
from typing import List, Dict

class AdaptiveStreamingService:
    def __init__(self, video_storage_path: str):
        self.storage_path = video_storage_path

    def generate_hls_playlist(self, video_id: str, variants: List[Dict]) -> str:
        """Generate HLS playlist for adaptive streaming"""
        playlist = "#EXTM3U8\n#EXT-X-VERSION:3\n"

        for variant in variants:
            duration = self._get_video_duration(variant['file_path'])
            resolution = variant['resolution']
            bandwidth = variant['bitrate_kbps']

            playlist += f"#EXT-X-STREAM-INF:BANDWIDTH={bandwidth},RESOLUTION={resolution}\n"
            playlist += f"#EXTINF:\n"
            playlist += f"#EXT-X-TARGETDURATION:{duration}\n"
            playlist += f"{variant['file_name']}\n"

        return playlist

    def generate_dash_manifest(self, video_id: str, variants: List[Dict]) -> str:
        """Generate DASH manifest for adaptive streaming"""
        manifest = '<?xml version="1.0" encoding="UTF-8"?>\n'
        manifest += '<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" mediaPresentationDuration="1800.0">\n'
        manifest += '  <Period>\n'
        manifest += '    <AdaptationSet>\n'

        for variant in variants:
            manifest += f'      <Representation id="{variant["quality"]}" '
            manifest += f'mimeType="video/mp4" '
            manifest += f'codecs="avc1.42E01E,mp4a.40.2" '
            manifest += f'width="{variant["width"]}" '
            manifest += f'height="{variant["height"]}" '
            manifest += f'bandwidth="{variant["bitrate_kbps"]}" '
            manifest += f'href="{variant["file_name"]}">\n'
            manifest += '      </Representation>\n'

        manifest += '    </AdaptationSet>\n'
        manifest += '  </Period>\n'
        manifest += '</MPD>'

        return manifest

    def _get_video_duration(self, file_path: str) -> float:
        """Get video duration using ffprobe"""
        try:
            cmd = [
                'ffprobe',
                '-v', 'error',
                '-show_entries', 'format=duration',
                '-of', 'default=noprint_wrappers=1:nokey=1',
                file_path
            ]

            result = subprocess.run(cmd, capture_output=True, text=True)

            if result.returncode == 0:
                for line in result.stdout.split('\n'):
                    if 'duration=' in line:
                        return float(line.split('=')[1])

            return 0.0  # Default duration

        except Exception as e:
            print(f"Error getting video duration: {e}")
            return 0.0

# Usage
streaming_service = AdaptiveStreamingService('/var/www/videos')

variants = [
    {
        'quality': 'low',
        'file_name': 'low.mp4',
        'file_path': '/var/www/videos/low.mp4',
        'bitrate_kbps': 800,
        'resolution': '426x240',
        'width': 426,
        'height': 240
    },
    {
        'quality': 'medium',
        'file_name': 'medium.mp4',
        'file_path': '/var/www/videos/medium.mp4',
        'bitrate_kbps': 1600,
        'resolution': '640x360',
        'width': 640,
        'height': 360
    },
    {
        'quality': 'high',
        'file_name': 'high.mp4',
        'file_path': '/var/www/videos/high.mp4',
        'bitrate_kbps': 3200,
        'resolution': '1280x720',
        'width': 1280,
        'height': 720
    }
]

hls_playlist = streaming_service.generate_hls_playlist('video_123', variants)
dash_manifest = streaming_service.generate_dash_manifest('video_123', variants)

print("HLS Playlist:")
print(hls_playlist)

print("\nDASH Manifest:")
print(dash_manifest)
```

## CDN Integration

### CDN Configuration
```python
import requests
from typing import Dict, List

class CDNManager:
    def __init__(self, cdn_api_key, cdn_base_url):
        self.api_key = cdn_api_key
        self.base_url = cdn_base_url

    def purge_cache(self, urls: List[str]) -> Dict[str, Any]:
        """Purge URLs from CDN cache"""
        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

        data = {
            'urls': urls
        }

        response = requests.post(
            f"{self.base_url}/cache/purge",
            headers=headers,
            json=data
        )

        return {
            'success': response.status_code == 200,
            'response': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
            'status_code': response.status_code
        }

    def configure_cache_rules(self, video_id: str, variants: List[str]) -> Dict[str, Any]:
        """Configure CDN caching rules for video variants"""
        cache_rules = []

        for variant in variants:
            rule = {
                'pattern': f"/videos/{video_id}/{variant}/*",
                'cache_ttl': 86400,  # 24 hours
                'browser_ttl': 3600,  # 1 hour
                'edge_ttl': 7200,  # 2 hours
                'compress': True,
                'gzip': True
            }
            cache_rules.append(rule)

        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

        data = {
            'cache_rules': cache_rules
        }

        response = requests.post(
            f"{self.base_url}/cache/rules",
            headers=headers,
            json=data
        )

        return {
            'success': response.status_code == 200,
            'response': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
            'status_code': response.status_code
        }

    def get_analytics(self, video_id: str, start_date: str, end_date: str) -> Dict[str, Any]:
        """Get CDN analytics for video"""
        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

        params = {
            'video_id': video_id,
            'start_date': start_date,
            'end_date': end_date,
            'metrics': ['bandwidth', 'requests', 'cache_hit_ratio']
        }

        response = requests.get(
            f"{self.base_url}/analytics/video",
            headers=headers,
            params=params
        )

        return {
            'success': response.status_code == 200,
            'response': response.json() if response.headers.get('content-type', '').startswith('application/json') else response.text,
            'status_code': response.status_code
        }

# Usage
cdn_manager = CDNManager('cdn-api-key', 'https://api.cdn-provider.com')

# Purge cache for updated video
purge_result = cdn_manager.purge_cache([
    'https://cdn.example.com/videos/vid_123/medium.mp4',
    'https://cdn.example.com/videos/vid_123/high.mp4'
])

print(f"Cache purge result: {purge_result}")

# Configure caching rules
cache_config = cdn_manager.configure_cache_rules(
    'vid_123',
    ['low.mp4', 'medium.mp4', 'high.mp4', 'ultra.mp4']
)

print(f"Cache configuration result: {cache_config}")
```

## Real-Time Analytics

### Streaming Analytics
```python
import time
import redis
from typing import Dict, Any
from dataclasses import dataclass

@dataclass
class StreamingEvent:
    user_id: str
    video_id: str
    quality: str
    timestamp: float
    bytes_served: int
    buffer_events: int
    playback_position: float

class StreamingAnalyticsService:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.analytics_window = 300  # 5 minutes

    def record_streaming_event(self, event: StreamingEvent):
        """Record streaming event for analytics"""
        # Store in Redis for real-time processing
        event_key = f"streaming:{event.user_id}:{event.video_id}"

        event_data = {
            'user_id': event.user_id,
            'video_id': event.video_id,
            'quality': event.quality,
            'timestamp': event.timestamp,
            'bytes_served': event.bytes_served,
            'buffer_events': event.buffer_events,
            'playback_position': event.playback_position
        }

        # Use Redis stream for time-series data
        self.redis.xadd(
            f"streaming_events:{event.video_id}",
            {
                'user_id': event.user_id,
                'quality': event.quality,
                'timestamp': event.timestamp,
                'bytes_served': event.bytes_served,
                'buffer_events': event.buffer_events,
                'playback_position': event.playback_position
            }
        )

        # Update user session data
        self.redis.hset(
            f"user_session:{event.user_id}",
            f"current_video:{event.video_id}",
            json.dumps(event_data)
        )

        # Set expiration
        self.redis.expire(f"user_session:{event.user_id}", self.analytics_window)

    def get_real_time_metrics(self, video_id: str) -> Dict[str, Any]:
        """Get real-time streaming metrics for a video"""
        current_time = time.time()
        window_start = current_time - self.analytics_window

        # Get events from Redis stream
        events = self.redis.xrange(
            f"streaming_events:{video_id}",
            min=f"{int(window_start * 1000)}-0",
            max="+",
            count=1000
        )

        if not events:
            return {
                'concurrent_viewers': 0,
                'total_bandwidth': 0,
                'average_quality': None,
                'buffer_ratio': 0
            }

        # Process events
        user_sessions = {}
        total_bandwidth = 0
        total_buffer_events = 0
        quality_counts = {}

        for event_id, event_data in events:
            event = event_data[1]  # Extract event data
            user_id = event['user_id']

            if user_id not in user_sessions:
                user_sessions[user_id] = {
                    'start_time': event['timestamp'],
                    'last_update': event['timestamp'],
                    'total_bytes': 0,
                    'buffer_events': 0,
                    'quality': event['quality']
                }

            session = user_sessions[user_id]
            session['last_update'] = event['timestamp']
            session['total_bytes'] += event['bytes_served']
            session['buffer_events'] += event['buffer_events']

            total_bandwidth += event['bytes_served']
            total_buffer_events += session['buffer_events']

            quality = event['quality']
            quality_counts[quality] = quality_counts.get(quality, 0) + 1

        # Calculate metrics
        concurrent_viewers = len(user_sessions)
        average_quality = max(quality_counts.items(), key=lambda x: x[1])[0] if quality_counts else None
        buffer_ratio = total_buffer_events / max(1, sum(event[1]['buffer_events'] for event in events))

        return {
            'concurrent_viewers': concurrent_viewers,
            'total_bandwidth_kbps': total_bandwidth * 8 / self.analytics_window,  # Convert to kbps
            'average_quality': average_quality,
            'buffer_ratio': buffer_ratio,
            'active_sessions': len([s for s in user_sessions.values() if current_time - s['last_update'] < 300])
        }

    def get_quality_distribution(self, video_id: str) -> Dict[str, float]:
        """Get quality distribution for video"""
        metrics = self.get_real_time_metrics(video_id)

        # Get all quality preferences from active sessions
        active_sessions = self.redis.keys(f"user_session:*")
        quality_distribution = {}

        for session_key in active_sessions:
            session_data = self.redis.hgetall(session_key)

            for field, value in session_data.items():
                if field.startswith('current_video:') and value:
                    session_json = json.loads(value)
                    quality = session_json.get('quality')

                    if quality:
                        quality_distribution[quality] = quality_distribution.get(quality, 0) + 1

        total_sessions = sum(quality_distribution.values())

        if total_sessions > 0:
            for quality in quality_distribution:
                quality_distribution[quality] = quality_distribution[quality] / total_sessions

        return quality_distribution

# Usage
redis_client = redis.Redis(host='localhost', port=6379, db=0)
analytics_service = StreamingAnalyticsService(redis_client)

# Simulate streaming events
import random
import time

for i in range(100):
    event = StreamingEvent(
        user_id=f"user_{random.randint(1, 1000)}",
        video_id="video_123",
        quality=random.choice(['low', 'medium', 'high']),
        timestamp=time.time(),
        bytes_served=random.randint(100000, 1000000),
        buffer_events=random.randint(0, 10),
        playback_position=random.uniform(0, 1800)
    )

    analytics_service.record_streaming_event(event)
    time.sleep(0.1)  # Simulate streaming events

# Get real-time metrics
metrics = analytics_service.get_real_time_metrics("video_123")
print(f"Real-time metrics: {metrics}")

# Get quality distribution
quality_dist = analytics_service.get_quality_distribution("video_123")
print(f"Quality distribution: {quality_dist}")
```

## Real-World Examples

### Netflix Architecture
```
Components:
- Open Connect (custom CDN appliances)
- Microservices architecture
- Chaos engineering for resilience
- Machine learning for recommendations
- Multi-region deployment

Key Features:
- Adaptive streaming
- Personalized recommendations
- Real-time analytics
- Global content delivery
```

### YouTube Architecture
```
Components:
- Google Cloud Platform
- Custom transcoding pipeline
- YouTube Data API
- Content ID system
- Monetization platform

Key Features:
- Video processing at scale
- Live streaming capabilities
- Community features
- Global infrastructure
```

### Amazon Prime Video Architecture
```
Components:
- AWS infrastructure
- Elemental MediaConvert for transcoding
- CloudFront for CDN
- DynamoDB for metadata
- Redshift for analytics

Key Features:
- 4K streaming support
- X-Ray for monitoring
- Lambda for serverless processing
- Global content delivery
```

## Interview Tips

### Common Questions
1. "How would you handle video transcoding?"
2. "What's adaptive streaming and how does it work?"
3. "How would you scale to millions of concurrent users?"
4. "How would you implement video recommendations?"
5. "What are the challenges with live streaming?"

### Answer Framework
1. **Requirements Clarification**: Scale, quality, features, latency
2. **High-Level Design**: Components, data flow, technologies
3. **Deep Dive**: Upload, processing, streaming, analytics
4. **Scalability**: CDN, caching, load balancing, auto-scaling
5. **Edge Cases**: Network issues, device compatibility, failures

### Key Points to Emphasize
- Adaptive bitrate streaming importance
- CDN and edge computing
- Video processing pipeline complexity
- Real-time analytics and monitoring
- Cost optimization strategies

## Practice Problems

1. **Design video upload system**
2. **Implement adaptive streaming**
3. **Create video recommendation engine**
4. **Design live streaming platform**
5. **Build video analytics system**

## Further Reading

- **Books**: "Video Streaming: Concepts, Algorithms, and Systems" by Yao Wang
- **Standards**: HLS, MPEG-DASH, HTTP Live Streaming
- **Documentation**: AWS Media Services, Netflix TechBlog, YouTube Engineering Blog
