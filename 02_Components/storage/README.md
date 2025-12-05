# Storage Systems

## Definition

Storage systems are technologies and architectures designed to store, manage, and retrieve digital data efficiently, reliably, and at scale. They form the foundation of data persistence in modern applications.

## Storage Hierarchy

### Storage Pyramid
```
┌─────────────────────────────────────┐
│        CPU Registers                │  ← Fastest, Smallest
│        (KB, <1ns)                   │
├─────────────────────────────────────┤
│        CPU Cache                    │
│        (MB, ~1ns)                   │
├─────────────────────────────────────┤
│        RAM (Main Memory)            │
│        (GB, ~100ns)                 │
├─────────────────────────────────────┤
│        SSD Storage                  │
│        (TB, ~100μs)                 │
├─────────────────────────────────────┤
│        HDD Storage                  │
│        (10TB, ~10ms)                │
├─────────────────────────────────────┤
│        Tape Storage                 │
│        (PB, ~100s)                  │ ← Slowest, Largest
└─────────────────────────────────────┘
```

### Performance Characteristics
| Storage Type | Capacity  | Latency  | Throughput  | Cost/GB  | Use Case                  |
|--------------|-----------|----------|-------------|----------|---------------------------|
| RAM          | GB        | 100ns    | 50GB/s      | High     | Hot data, caching         |
| NVMe SSD     | TB        | 100μs    | 5GB/s       | Medium   | Databases, analytics      |
| SATA SSD     | TB        | 500μs    | 500MB/s     | Low      | Boot drives, applications |
| HDD          | 10TB      | 10ms     | 200MB/s     | Very Low | Archives, backups         |
| Tape         | PB        | 100s     | 300MB/s     | Lowest   | Long-term archives        |

## Storage Types

### Block Storage
```
Structure:
┌───────────────────────────────────────┐
│ Block 1  │ Block 2  │ Block 3  │ ...  │
│ 512B     │ 512B     │ 512B     │      │
└───────────────────────────────────────┘

Access:
Application → File System → Block Device → Storage
    ↓           ↓              ↓           ↓
  Files     Inodes        Blocks     Physical
```

**Characteristics:**
- **Fixed Size Blocks**: Typically 512B or 4KB
- **Low Latency**: Direct block access
- **File Systems**: Can be formatted with any file system
- **Raw Access**: Can be used directly by databases

**Use Cases:**
- Database storage
- Virtual machine disks
- High-performance applications
- Boot volumes

**Examples:**
- **AWS EBS**: Elastic Block Store
- **Azure Disk Storage**: Premium SSD
- **Google Persistent Disk**: Standard and SSD
- **Ceph**: Distributed block storage

### Object Storage
```
Structure:
Object = Data + Metadata + ID
┌─────────────────────────────────────┐
│  Object ID: "photos/12345.jpg"      │
│  Data: [Binary Image Data]          │
│  Metadata:                          │
│    - Size: 2.5MB                    │
│    - Type: image/jpeg               │
│    - Created: 2024-01-01            │
│    - Owner: user123                 │
└─────────────────────────────────────┘

Access:
Application → HTTP API → Object Storage
    ↓           ↓            ↓
  GET/PUT    REST API     Objects
```

**Characteristics:**
- **HTTP Interface**: RESTful API access
- **Metadata Rich**: Custom metadata support
- **Unlimited Scale**: Flat namespace
- **Durability**: Multi-AZ replication
- **Cost Effective**: Cheaper than block storage

**Use Cases:**
- Media files (images, videos)
- Backup and archive
- Big data analytics
- Static website hosting

**Examples:**
- **Amazon S3**: Simple Storage Service
- **Azure Blob Storage**: Blob storage
- **Google Cloud Storage**: Object storage
- **MinIO**: Open-source object storage

### File Storage
```
Structure:
Directory Tree:
/
├── home/
│   ├── user1/
│   │   ├── documents/
│   │   └── photos/
│   └── user2/
├── shared/
│   ├── projects/
│   └── resources/
└── backup/

Access:
Application → Network Protocol → File Server → File System
    ↓              ↓                ↓              ↓
  SMB/NFS        TCP/IP           Files         Directories
```

**Characteristics:**
- **Hierarchical**: Directory tree structure
- **Network Access**: SMB, NFS protocols
- **Shared Access**: Multiple users/applications
- **File Locking**: Concurrent access control

**Use Cases:**
- Home directories
- Shared project files
- Content management
- Development environments

**Examples:**
- **AWS EFS**: Elastic File System
- **Azure Files**: File shares
- **Google Filestore**: Managed file storage
- **NFS Server**: Network File System

## Storage Architectures

### Direct Attached Storage (DAS)
```
Server
┌─────────────────────────────────────┐
│  CPU    │  RAM    │  Storage        │
│         │         │  ┌─────────┐    │
│         │         │  │ HDD/SSD │    │
│         │         │  └─────────┘    │
└─────────────────────────────────────┘

Characteristics:
- Direct connection to server
- High performance
- Limited scalability
- Single point of failure
```

### Network Attached Storage (NAS)
```
Network
    ↓
┌─────────────────────────────────────┐
│           NAS Device                │
│  ┌─────────────────────────────┐    │
│  │      File System            │    │
│  │  ┌─────┐ ┌─────┐ ┌─────┐    │    │
│  │  │HDD 1│ │HDD 2│ │SSD 1│    │    │
│  │  └─────┘ └─────┘ └─────┘    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
    ↓
Multiple Servers/Clients

Characteristics:
- Network access (Ethernet)
- File-level access
- Easy sharing
- Centralized management
```

### Storage Area Network (SAN)
```
Network
    ↓
┌─────────────────────────────────────┐
│           SAN Fabric                │
│                                     │
│  ┌─────────┐  ┌─────────┐           │
│  │Switch 1 │──│Switch 2 │           │
│  └─────────┘  └─────────┘           │
│       ↓            ↓                │
│  ┌─────────┐  ┌─────────┐           │
│  │Array 1  │  │Array 2  │           │
│  │(Disks)  │  │(Disks)  │           │
│  └─────────┘  └─────────┘           │
└─────────────────────────────────────┘
    ↓
Multiple Servers (Block Access)

Characteristics:
- High-speed network (Fiber Channel)
- Block-level access
- High performance
- Complex management
```

## Distributed Storage Systems

### Ceph Architecture
```
Ceph Cluster:
┌─────────────────────────────────────────────────────────┐
│                  MON (Monitor)                          │
│              (Cluster State)                            │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│                  MDS (Metadata)                         │
│              (File System Metadata)                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│                  OSD (Object Storage)                   │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                │
│  │OSD 1│ │OSD 2│ │OSD 3│ │OSD 4│ │OSD 5│                │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘                │
└─────────────────────────────────────────────────────────┘

Data Flow:
Client → RADOS Gateway → OSDs → Replicated Objects
```

**Components:**
- **MON (Monitor)**: Cluster state management
- **MDS (Metadata Server)**: File system metadata
- **OSD (Object Storage Daemon)**: Data storage
- **RADOS Gateway**: S3/Swift compatible interface

### GlusterFS Architecture
```
GlusterFS Cluster:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Server 1      │  │   Server 2      │  │   Server 3      │
│                 │  │                 │  │                 │
│ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
│ │   Brick 1   │ │  │ │   Brick 2   │ │  │ │   Brick 3   │ │
│ │ (Storage)   │ │  │ │ (Storage)   │ │  │ │ (Storage)   │ │
│ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
│                 │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
    ↓                         ↓                    ↓
                      Distributed Volume
                              ↓
                        Client Access
```

**Volume Types:**
- **Distributed**: Files spread across bricks
- **Replicated**: Files replicated across bricks
- **Distributed-Replicated**: Combination of both
- **Striped**: Files split across bricks

## Cloud Storage Services

### Amazon S3
```
S3 Architecture:
┌─────────────────────────────────────┐
│           S3 Bucket                 │
│                                     │
│  Object 1  Object 2  Object 3       │
│    ↓         ↓         ↓            │
│  ┌─────┐  ┌─────┐  ┌─────┐          │
│  │ AZ1 │  │ AZ2 │  │ AZ3 │          │
│  └─────┘  └─────┘  └─────┘          │
│                                     │
│  Versioning, Lifecycle, Encryption  │
└─────────────────────────────────────┘

Storage Classes:
- Standard: Frequent access
- Intelligent-Tiering: Automatic optimization
- Infrequent Access: Less frequent access
- Glacier: Long-term archive
- Deep Archive: Very long-term archive
```

**S3 Implementation:**
```python
import boto3
from botocore.exceptions import ClientError

class S3StorageManager:
    def __init__(self, access_key, secret_key, region='us-east-1'):
        self.s3_client = boto3.client(
            's3',
            aws_access_key_id=access_key,
            aws_secret_access_key=secret_key,
            region_name=region
        )

    def create_bucket(self, bucket_name, region='us-east-1'):
        try:
            if region == 'us-east-1':
                self.s3_client.create_bucket(Bucket=bucket_name)
            else:
                self.s3_client.create_bucket(
                    Bucket=bucket_name,
                    CreateBucketConfiguration={
                        'LocationConstraint': region
                    }
                )
            print(f"Bucket {bucket_name} created successfully")
        except ClientError as e:
            print(f"Error creating bucket: {e}")

    def upload_file(self, bucket_name, file_path, object_name=None):
        if object_name is None:
            object_name = file_path.split('/')[-1]

        try:
            self.s3_client.upload_file(file_path, bucket_name, object_name)
            print(f"File {file_path} uploaded to {bucket_name}/{object_name}")
        except ClientError as e:
            print(f"Error uploading file: {e}")

    def download_file(self, bucket_name, object_name, file_path):
        try:
            self.s3_client.download_file(bucket_name, object_name, file_path)
            print(f"File {object_name} downloaded to {file_path}")
        except ClientError as e:
            print(f"Error downloading file: {e}")

    def list_objects(self, bucket_name):
        try:
            response = self.s3_client.list_objects_v2(Bucket=bucket_name)
            objects = response.get('Contents', [])

            for obj in objects:
                print(f"  {obj['Key']} ({obj['Size']} bytes)")

            return objects
        except ClientError as e:
            print(f"Error listing objects: {e}")
            return []

    def set_lifecycle_policy(self, bucket_name):
        lifecycle_config = {
            'Rules': [
                {
                    'ID': 'LifecycleRule',
                    'Status': 'Enabled',
                    'Transitions': [
                        {
                            'Days': 30,
                            'StorageClass': 'STANDARD_IA'
                        },
                        {
                            'Days': 90,
                            'StorageClass': 'GLACIER'
                        },
                        {
                            'Days': 365,
                            'StorageClass': 'DEEP_ARCHIVE'
                        }
                    ]
                }
            ]
        }

        try:
            self.s3_client.put_bucket_lifecycle_configuration(
                Bucket=bucket_name,
                LifecycleConfiguration=lifecycle_config
            )
            print("Lifecycle policy set successfully")
        except ClientError as e:
            print(f"Error setting lifecycle policy: {e}")

# Usage
s3_manager = S3StorageManager('your-access-key', 'your-secret-key')
s3_manager.create_bucket('my-test-bucket')
s3_manager.upload_file('my-test-bucket', 'local-file.txt')
s3_manager.list_objects('my-test-bucket')
```

### Azure Blob Storage
```python
from azure.storage.blob import BlobServiceClient
from azure.core.exceptions import ResourceExistsError

class AzureBlobManager:
    def __init__(self, connection_string):
        self.blob_service_client = BlobServiceClient.from_connection_string(
            connection_string
        )

    def create_container(self, container_name):
        try:
            container_client = self.blob_service_client.create_container(
                container_name
            )
            print(f"Container {container_name} created successfully")
            return container_client
        except ResourceExistsError:
            print(f"Container {container_name} already exists")
            return self.blob_service_client.get_container_client(container_name)

    def upload_blob(self, container_name, blob_name, file_path):
        blob_client = self.blob_service_client.get_blob_client(
            container=container_name,
            blob=blob_name
        )

        with open(file_path, 'rb') as data:
            blob_client.upload_blob(data)
            print(f"Blob {blob_name} uploaded successfully")

    def download_blob(self, container_name, blob_name, file_path):
        blob_client = self.blob_service_client.get_blob_client(
            container=container_name,
            blob=blob_name
        )

        with open(file_path, 'wb') as download_file:
            download_file.write(blob_client.download_blob().readall())
            print(f"Blob {blob_name} downloaded successfully")

    def list_blobs(self, container_name):
        container_client = self.blob_service_client.get_container_client(
            container_name
        )

        blobs = container_client.list_blobs()
        for blob in blobs:
            print(f"  {blob.name} ({blob.size} bytes)")

        return list(blobs)

# Usage
blob_manager = AzureBlobManager('your-connection-string')
blob_manager.create_container('my-container')
blob_manager.upload_blob('my-container', 'test-blob.txt', 'local-file.txt')
```

## Storage Performance Optimization

### Caching Strategies
```python
import redis
import hashlib
from functools import wraps

class StorageCache:
    def __init__(self, redis_client, storage_backend):
        self.redis = redis_client
        self.storage = storage_backend
        self.default_ttl = 3600  # 1 hour

    def cache_key(self, bucket, object_name):
        return f"storage:{bucket}:{object_name}"

    def get_object(self, bucket, object_name):
        cache_key = self.cache_key(bucket, object_name)

        # Try cache first
        cached_data = self.redis.get(cache_key)
        if cached_data:
            print(f"Cache hit for {bucket}/{object_name}")
            return cached_data

        # Cache miss - get from storage
        print(f"Cache miss for {bucket}/{object_name}")
        data = self.storage.get_object(bucket, object_name)

        if data:
            # Cache the result
            self.redis.setex(cache_key, self.default_ttl, data)

        return data

    def put_object(self, bucket, object_name, data):
        # Store in backend
        self.storage.put_object(bucket, object_name, data)

        # Update cache
        cache_key = self.cache_key(bucket, object_name)
        self.redis.setex(cache_key, self.default_ttl, data)

    def invalidate_cache(self, bucket, object_name):
        cache_key = self.cache_key(bucket, object_name)
        self.redis.delete(cache_key)

# Decorator for caching storage operations
def cached_storage(ttl=3600):
    def decorator(func):
        @wraps(func)
        def wrapper(self, bucket, object_name, *args, **kwargs):
            cache_key = f"storage:{bucket}:{object_name}"

            # Check cache
            cached_result = self.redis.get(cache_key)
            if cached_result:
                return cached_result

            # Execute function
            result = func(self, bucket, object_name, *args, **kwargs)

            # Cache result
            self.redis.setex(cache_key, ttl, result)

            return result
        return wrapper
    return decorator
```

### Multi-part Upload for Large Files
```python
import os
import math

class MultipartUploader:
    def __init__(self, storage_client, part_size=8*1024*1024):  # 8MB
        self.storage = storage_client
        self.part_size = part_size

    def upload_large_file(self, bucket, file_path, object_name=None):
        if object_name is None:
            object_name = os.path.basename(file_path)

        file_size = os.path.getsize(file_path)

        if file_size <= self.part_size:
            # Simple upload for small files
            return self.storage.upload_file(bucket, file_path, object_name)

        # Multipart upload for large files
        upload_id = self.storage.create_multipart_upload(bucket, object_name)

        try:
            parts = []
            with open(file_path, 'rb') as file:
                part_number = 1

                while True:
                    data = file.read(self.part_size)
                    if not data:
                        break

                    # Upload part
                    etag = self.storage.upload_part(
                        bucket, object_name, upload_id, part_number, data
                    )

                    parts.append({
                        'PartNumber': part_number,
                        'ETag': etag
                    })

                    part_number += 1

            # Complete multipart upload
            return self.storage.complete_multipart_upload(
                bucket, object_name, upload_id, parts
            )

        except Exception as e:
            # Abort upload on error
            self.storage.abort_multipart_upload(bucket, object_name, upload_id)
            raise e

# Usage
uploader = MultipartUploader(s3_client)
uploader.upload_large_file('my-bucket', 'large-file.zip')
```

## Storage Security

### Encryption at Rest
```python
from cryptography.fernet import Fernet
import os

class EncryptedStorage:
    def __init__(self, storage_backend, encryption_key):
        self.storage = storage_backend
        self.cipher = Fernet(encryption_key)

    def encrypt_data(self, data):
        if isinstance(data, str):
            data = data.encode()
        return self.cipher.encrypt(data)

    def decrypt_data(self, encrypted_data):
        decrypted = self.cipher.decrypt(encrypted_data)
        return decrypted.decode()

    def put_object(self, bucket, object_name, data):
        # Encrypt data before storing
        encrypted_data = self.encrypt_data(data)

        # Store encrypted data
        self.storage.put_object(bucket, object_name, encrypted_data)

    def get_object(self, bucket, object_name):
        # Get encrypted data
        encrypted_data = self.storage.get_object(bucket, object_name)

        if encrypted_data:
            # Decrypt data
            return self.decrypt_data(encrypted_data)

        return None

    @staticmethod
    def generate_key():
        return Fernet.generate_key()

# Usage
encryption_key = EncryptedStorage.generate_key()
encrypted_storage = EncryptedStorage(s3_client, encryption_key)
encrypted_storage.put_object('secure-bucket', 'secret.txt', 'confidential data')
```

### Access Control
```python
class StorageAccessControl:
    def __init__(self, storage_backend):
        self.storage = storage_backend
        self.policies = {}

    def add_policy(self, policy_name, policy):
        self.policies[policy_name] = policy

    def check_access(self, user, bucket, object_name, action):
        # Check bucket-level policies
        bucket_policy = self.policies.get(f'bucket:{bucket}', {})

        if not self._evaluate_policy(bucket_policy, user, action):
            return False

        # Check object-level policies
        object_policy = self.policies.get(f'object:{bucket}:{object_name}', {})

        return self._evaluate_policy(object_policy, user, action)

    def _evaluate_policy(self, policy, user, action):
        # Simple policy evaluation
        allowed_users = policy.get('allowed_users', [])
        allowed_actions = policy.get('allowed_actions', [])

        if allowed_users and user not in allowed_users:
            return False

        if allowed_actions and action not in allowed_actions:
            return False

        return True

    def put_object_with_acl(self, bucket, object_name, data, acl):
        # Store object
        self.storage.put_object(bucket, object_name, data)

        # Store ACL
        acl_key = f"acl:{bucket}:{object_name}"
        self.storage.put_object('metadata', acl_key, json.dumps(acl))

    def get_object_with_acl_check(self, user, bucket, object_name):
        # Check ACL
        acl_key = f"acl:{bucket}:{object_name}"
        acl_data = self.storage.get_object('metadata', acl_key)

        if acl_data:
            acl = json.loads(acl_data)
            if not self._check_acl(acl, user, 'read'):
                raise PermissionError("Access denied")

        # Get object
        return self.storage.get_object(bucket, object_name)

    def _check_acl(self, acl, user, action):
        # Check owner
        if acl.get('owner') == user:
            return True

        # Check permissions
        permissions = acl.get('permissions', {})
        user_permissions = permissions.get(user, [])

        return action in user_permissions

# Usage
access_control = StorageAccessControl(s3_client)

# Define policies
access_control.add_policy('bucket:public', {
    'allowed_actions': ['read'],
    'allowed_users': ['*']
})

access_control.add_policy('bucket:private', {
    'allowed_actions': ['read', 'write'],
    'allowed_users': ['admin', 'user1']
})

# Check access
if access_control.check_access('user1', 'private-bucket', 'file.txt', 'read'):
    data = access_control.get_object_with_acl_check('user1', 'private-bucket', 'file.txt')
```

## Storage Monitoring

### Performance Metrics
```python
import time
from prometheus_client import Counter, Histogram, Gauge

# Storage metrics
STORAGE_OPERATIONS = Counter('storage_operations_total', 'Total storage operations', ['operation', 'status'])
STORAGE_DURATION = Histogram('storage_operation_duration_seconds', 'Storage operation duration')
STORAGE_SIZE = Gauge('storage_size_bytes', 'Storage size in bytes')
STORAGE_OBJECTS = Gauge('storage_objects_total', 'Total number of objects')

class MonitoredStorage:
    def __init__(self, storage_backend):
        self.storage = storage_backend

    @STORAGE_DURATION.time()
    def put_object(self, bucket, object_name, data):
        start_time = time.time()

        try:
            result = self.storage.put_object(bucket, object_name, data)

            # Record metrics
            STORAGE_OPERATIONS.labels(operation='put', status='success').inc()
            STORAGE_SIZE.inc(len(data))
            STORAGE_OBJECTS.inc()

            return result

        except Exception as e:
            STORAGE_OPERATIONS.labels(operation='put', status='error').inc()
            raise e

    @STORAGE_DURATION.time()
    def get_object(self, bucket, object_name):
        start_time = time.time()

        try:
            result = self.storage.get_object(bucket, object_name)

            # Record metrics
            STORAGE_OPERATIONS.labels(operation='get', status='success').inc()
            if result:
                STORAGE_SIZE.dec(len(result))
                STORAGE_OBJECTS.dec()

            return result

        except Exception as e:
            STORAGE_OPERATIONS.labels(operation='get', status='error').inc()
            raise e

    def collect_storage_stats(self):
        # Collect storage statistics
        stats = self.storage.get_storage_stats()

        STORAGE_SIZE.set(stats['total_size'])
        STORAGE_OBJECTS.set(stats['total_objects'])

        return stats
```

### Health Checks
```python
class StorageHealthChecker:
    def __init__(self, storage_backend):
        self.storage = storage_backend
        self.health_check_file = 'health_check.txt'
        self.health_check_content = f'Health check at {time.time()}'

    def check_write_health(self):
        try:
            # Test write
            self.storage.put_object(
                'health-check',
                self.health_check_file,
                self.health_check_content
            )
            return True, "Write operation successful"
        except Exception as e:
            return False, f"Write operation failed: {e}"

    def check_read_health(self):
        try:
            # Test read
            data = self.storage.get_object(
                'health-check',
                self.health_check_file
            )

            if data == self.health_check_content:
                return True, "Read operation successful"
            else:
                return False, "Read operation returned incorrect data"
        except Exception as e:
            return False, f"Read operation failed: {e}"

    def check_delete_health(self):
        try:
            # Test delete
            self.storage.delete_object(
                'health-check',
                self.health_check_file
            )
            return True, "Delete operation successful"
        except Exception as e:
            return False, f"Delete operation failed: {e}"

    def comprehensive_health_check(self):
        results = {
            'write': self.check_write_health(),
            'read': self.check_read_health(),
            'delete': self.check_delete_health()
        }

        overall_healthy = all(result[0] for result in results.values())

        return {
            'healthy': overall_healthy,
            'checks': results,
            'timestamp': time.time()
        }
```

## Real-World Examples

### Netflix Storage Architecture
```
Components:
- S3 for primary storage
- Glacier for archival
- Edge caching for CDN
- Custom object storage (S3-compatible)

Data Flow:
Upload → S3 → Processing → CDN → Users
Archive → Glacier → Long-term storage
```

### Dropbox Storage Architecture
```
Components:
- Block-based storage system
- Metadata database
- Sync engine
- Version control

Data Flow:
File → Blocks → Storage → Metadata → Sync
```

### Instagram Storage Architecture
```
Components:
- S3 for photo storage
- CDN for delivery
- Redis for caching
- Cassandra for metadata

Data Flow:
Upload → Processing → S3 → CDN → Users
Metadata → Cassandra → Search/Discovery
```

## Interview Tips

### Common Questions
1. "What's the difference between block, object, and file storage?"
2. "How would you design a file storage system?"
3. "What are the trade-offs between different storage types?"
4. "How do you handle large file uploads?"
5. "How would you implement storage encryption?"

### Answer Framework
1. **Understand Requirements**: Access patterns, scale, performance
2. **Choose Storage Type**: Block vs Object vs File based on needs
3. **Design Architecture**: Distribution, replication, caching
4. **Implement Features**: Security, monitoring, optimization
5. **Consider Trade-offs**: Cost vs performance vs complexity

### Key Points to Emphasize
- Access patterns and performance requirements
- Durability and availability needs
- Cost optimization strategies
- Security and compliance requirements
- Scalability considerations

## Practice Problems

1. **Design distributed file storage system**
2. **Implement object storage service**
3. **Create backup and archival system**
4. **Design photo storage for social media**
5. **Build content delivery network**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Documentation**: AWS Storage Services, Azure Storage Documentation
- **Concepts**: CAP Theorem, Consistency Models, Distributed Systems
