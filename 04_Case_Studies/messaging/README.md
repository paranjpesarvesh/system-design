# Messaging System (WhatsApp/Telegram)

## Problem Statement

Design a real-time messaging platform like WhatsApp or Telegram that supports instant messaging, group chats, media sharing, and end-to-end encryption.

## Requirements

### Functional Requirements
- User registration and authentication
- One-on-one and group messaging
- Real-time message delivery
- Media file sharing (images, videos, documents)
- End-to-end encryption
- Message history and search
- Online status and presence
- Push notifications

### Non-Functional Requirements
- **Scale**: 2 billion daily active users, 100 million concurrent users
- **Latency**: < 100ms for message delivery
- **Availability**: 99.99% uptime
- **Consistency**: Eventual consistency for message ordering
- **Security**: End-to-end encryption, data privacy

## Scale Estimation

### Traffic Estimates
```
Daily Active Users: 2 billion
Daily Messages: 100 billion
Peak Concurrent Users: 100 million
Messages per Second: 1.2 million
Media Uploads per Day: 10 billion files
Storage Growth: 50TB/day
```

### Storage Estimates
```
User Data: 2B × 1KB = 2TB
Message Data: 100B × 1KB = 100TB/day = 36PB/year
Media Storage: 10B × 1MB = 10PB/day
Index Data: 100B × 100B = 10PB
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
│  │   User      │  │   Chat      │  │   Media     │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│  User DB        │  Message DB    │  Media DB            │
│  (Cassandra)    │  (Cassandra)   │  (PostgreSQL)        │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Real-time Services                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Presence  │  │   Push      │  │   Search    │      │
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
  "phone": "+1234567890",
  "name": "John Doe",
  "verification_code": "123456"
}

POST /api/v1/users/login
{
  "phone": "+1234567890",
  "verification_code": "123456"
}

GET /api/v1/users/contacts
Response:
{
  "contacts": [
    {
      "user_id": "user_123",
      "name": "Jane Smith",
      "phone": "+1234567891",
      "status": "online",
      "last_seen": "2024-01-01T10:00:00Z"
    }
  ]
}
```

### Chat APIs
```http
POST /api/v1/chats
{
  "type": "direct",
  "participants": ["user_123", "user_456"]
}

POST /api/v1/messages
{
  "chat_id": "chat_789",
  "content": "Hello, how are you?",
  "type": "text",
  "timestamp": "2024-01-01T10:00:00Z"
}

GET /api/v1/chats/{chat_id}/messages?page=1&limit=50
Response:
{
  "messages": [
    {
      "id": "msg_123",
      "sender_id": "user_123",
      "content": "Hello!",
      "type": "text",
      "timestamp": "2024-01-01T10:00:00Z",
      "status": "delivered"
    }
  ],
  "has_more": true
}
```

### Media APIs
```http
POST /api/v1/media/upload
Content-Type: multipart/form-data
{
  "file": <image_file>,
  "chat_id": "chat_789",
  "type": "image"
}

GET /api/v1/media/{media_id}
Response: Binary media content
```

## Database Design

### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255),
    profile_pic_url VARCHAR(500),
    status ENUM('online', 'offline', 'away') DEFAULT 'offline',
    last_seen TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_phone (phone),
    INDEX idx_status (status)
);
```

### Chats Table
```sql
CREATE TABLE chats (
    id UUID PRIMARY KEY,
    type ENUM('direct', 'group') NOT NULL,
    name VARCHAR(255),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_type (type),
    INDEX idx_created_by (created_by)
);
```

### Chat Participants Table
```sql
CREATE TABLE chat_participants (
    chat_id UUID REFERENCES chats(id),
    user_id UUID REFERENCES users(id),
    role ENUM('admin', 'member') DEFAULT 'member',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (chat_id, user_id),
    INDEX idx_user_id (user_id)
);
```

### Messages Table
```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    chat_id UUID REFERENCES chats(id),
    sender_id UUID REFERENCES users(id),
    content TEXT,
    type ENUM('text', 'image', 'video', 'file') DEFAULT 'text',
    media_url VARCHAR(500),
    encrypted_content BLOB,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('sent', 'delivered', 'read') DEFAULT 'sent',
    INDEX idx_chat_id (chat_id),
    INDEX idx_sender_id (sender_id),
    INDEX idx_timestamp (timestamp),
    INDEX idx_status (status)
);
```

## Real-Time Message Delivery

### WebSocket Implementation
```python
import asyncio
import websockets
import json
from typing import Dict, Set
from dataclasses import dataclass

@dataclass
class ConnectedUser:
    user_id: str
    websocket: websockets.WebSocketServerProtocol
    last_ping: float

class WebSocketServer:
    def __init__(self):
        self.connected_users: Dict[str, ConnectedUser] = {}
        self.user_chats: Dict[str, Set[str]] = {}

    async def handle_connection(self, websocket, path):
        """Handle WebSocket connection"""
        user_id = None
        try:
            # Authentication
            auth_message = await websocket.recv()
            auth_data = json.loads(auth_message)
            user_id = auth_data.get('user_id')

            if not user_id:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': 'Authentication required'
                }))
                return

            # Register user
            self.connected_users[user_id] = ConnectedUser(
                user_id=user_id,
                websocket=websocket,
                last_ping=asyncio.get_event_loop().time()
            )

            await websocket.send(json.dumps({
                'type': 'connected',
                'user_id': user_id
            }))

            # Message handling loop
            async for message in websocket:
                await self.handle_message(user_id, message)

        except websockets.exceptions.ConnectionClosed:
            pass
        finally:
            if user_id and user_id in self.connected_users:
                del self.connected_users[user_id]

    async def handle_message(self, user_id: str, message: str):
        """Handle incoming message"""
        try:
            data = json.loads(message)
            message_type = data.get('type')

            if message_type == 'send_message':
                await self.handle_send_message(user_id, data)
            elif message_type == 'join_chat':
                await self.handle_join_chat(user_id, data)
            elif message_type == 'ping':
                await self.handle_ping(user_id)

        except json.JSONDecodeError:
            await self.send_to_user(user_id, {
                'type': 'error',
                'message': 'Invalid message format'
            })

    async def handle_send_message(self, user_id: str, data: dict):
        """Handle message sending"""
        chat_id = data.get('chat_id')
        content = data.get('content')
        message_type = data.get('message_type', 'text')

        if not chat_id or not content:
            await self.send_to_user(user_id, {
                'type': 'error',
                'message': 'Missing chat_id or content'
            })
            return

        # Save message to database
        message_id = await self.save_message(chat_id, user_id, content, message_type)

        # Broadcast to chat participants
        await self.broadcast_to_chat(chat_id, {
            'type': 'new_message',
            'message_id': message_id,
            'chat_id': chat_id,
            'sender_id': user_id,
            'content': content,
            'message_type': message_type,
            'timestamp': asyncio.get_event_loop().time()
        })

    async def broadcast_to_chat(self, chat_id: str, message: dict):
        """Broadcast message to all chat participants"""
        # Get chat participants (in real implementation, query database)
        participants = await self.get_chat_participants(chat_id)

        for participant_id in participants:
            if participant_id in self.connected_users:
                await self.send_to_user(participant_id, message)

    async def send_to_user(self, user_id: str, message: dict):
        """Send message to specific user"""
        if user_id in self.connected_users:
            try:
                await self.connected_users[user_id].websocket.send(
                    json.dumps(message)
                )
            except websockets.exceptions.ConnectionClosed:
                del self.connected_users[user_id]

    async def save_message(self, chat_id: str, sender_id: str,
                          content: str, message_type: str) -> str:
        """Save message to database (placeholder)"""
        # In real implementation, save to Cassandra/DynamoDB
        message_id = f"msg_{int(asyncio.get_event_loop().time() * 1000)}"
        return message_id

    async def get_chat_participants(self, chat_id: str) -> list:
        """Get chat participants (placeholder)"""
        # In real implementation, query database
        return ['user_123', 'user_456']  # Example

# Usage
server = WebSocketServer()

start_server = websockets.serve(
    server.handle_connection,
    "localhost",
    8765
)

asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

## End-to-End Encryption

### Signal Protocol Implementation
```python
import os
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import x25519
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from typing import Tuple, Optional
from dataclasses import dataclass

@dataclass
class KeyPair:
    private_key: x25519.X25519PrivateKey
    public_key: x25519.X25519PublicKey

@dataclass
class SessionKeys:
    root_key: bytes
    chain_key: bytes
    message_key: bytes

class E2EEncryption:
    def __init__(self):
        self.key_length = 32

    def generate_keypair(self) -> KeyPair:
        """Generate X25519 key pair"""
        private_key = x25519.X25519PrivateKey.generate()
        public_key = private_key.public_key()
        return KeyPair(private_key=private_key, public_key=public_key)

    def derive_shared_secret(self, private_key: x25519.X25519PrivateKey,
                           public_key: x25519.X25519PublicKey) -> bytes:
        """Derive shared secret using X25519"""
        shared_key = private_key.exchange(public_key)
        return shared_key

    def kdf_rk(self, rk: bytes, dh_out: bytes) -> Tuple[bytes, bytes]:
        """Key derivation function for root key"""
        hkdf = HKDF(
            algorithm=hashes.SHA256(),
            length=self.key_length * 2,
            salt=rk,
            info=b"WhisperText",
        )
        derived = hkdf.derive(dh_out)
        return derived[:self.key_length], derived[self.key_length:]

    def kdf_ck(self, ck: bytes) -> Tuple[bytes, bytes]:
        """Key derivation function for chain key"""
        hkdf = HKDF(
            algorithm=hashes.SHA256(),
            length=self.key_length + 32,  # 32 for message key
            salt=None,
            info=b"WhisperMessageKeys",
        )
        derived = hkdf.derive(ck)
        return derived[:self.key_length], derived[self.key_length:]

    def encrypt_message(self, plaintext: str, message_key: bytes) -> Tuple[bytes, bytes]:
        """Encrypt message using AES"""
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

        # Generate random IV
        iv = os.urandom(16)

        # Create cipher
        cipher = Cipher(algorithms.AES(message_key), modes.CBC(iv))
        encryptor = cipher.encryptor()

        # Pad plaintext to block size
        padded_plaintext = self._pad(plaintext.encode('utf-8'))

        # Encrypt
        ciphertext = encryptor.update(padded_plaintext) + encryptor.finalize()

        return ciphertext, iv

    def decrypt_message(self, ciphertext: bytes, iv: bytes, message_key: bytes) -> str:
        """Decrypt message using AES"""
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

        # Create cipher
        cipher = Cipher(algorithms.AES(message_key), modes.CBC(iv))
        decryptor = cipher.decryptor()

        # Decrypt
        padded_plaintext = decryptor.update(ciphertext) + decryptor.finalize()

        # Unpad
        plaintext = self._unpad(padded_plaintext)

        return plaintext.decode('utf-8')

    def _pad(self, data: bytes, block_size: int = 16) -> bytes:
        """PKCS7 padding"""
        padding_length = block_size - (len(data) % block_size)
        padding = bytes([padding_length]) * padding_length
        return data + padding

    def _unpad(self, data: bytes) -> bytes:
        """PKCS7 unpadding"""
        padding_length = data[-1]
        return data[:-padding_length]

# Usage
encryption = E2EEncryption()

# Alice generates keypair
alice_keys = encryption.generate_keypair()

# Bob generates keypair
bob_keys = encryption.generate_keypair()

# Alice and Bob exchange public keys
alice_shared = encryption.derive_shared_secret(
    alice_keys.private_key, bob_keys.public_key
)
bob_shared = encryption.derive_shared_secret(
    bob_keys.private_key, alice_keys.public_key
)

# Verify shared secrets are equal
assert alice_shared == bob_shared

# Alice encrypts message
message = "Hello, Bob!"
ciphertext, iv = encryption.encrypt_message(message, alice_shared[:32])

# Bob decrypts message
decrypted = encryption.decrypt_message(ciphertext, iv, bob_shared[:32])
print(f"Decrypted message: {decrypted}")
```

## Message Search and Indexing

### Elasticsearch for Message Search
```python
from elasticsearch import Elasticsearch
from typing import List, Dict

class MessageSearchService:
    def __init__(self, es_host: str):
        self.es = Elasticsearch([es_host])
        self.index_name = 'messages'

    def index_message(self, message: Dict):
        """Index a message in Elasticsearch"""
        self.es.index(
            index=self.index_name,
            id=message['id'],
            body={
                'message_id': message['id'],
                'chat_id': message['chat_id'],
                'sender_id': message['sender_id'],
                'content': message['content'],
                'type': message['type'],
                'timestamp': message['timestamp'],
                'participants': message.get('participants', [])
            }
        )

    def search_messages(self, user_id: str, query: str,
                       chat_ids: List[str] = None,
                       page: int = 1, limit: int = 20) -> Dict:
        """Search messages for user"""
        search_body = {
            'query': {
                'bool': {
                    'must': [
                        {
                            'multi_match': {
                                'query': query,
                                'fields': ['content']
                            }
                        },
                        {
                            'term': {
                                'participants': user_id
                            }
                        }
                    ],
                    'filter': []
                }
            },
            'sort': [
                {'timestamp': 'desc'}
            ],
            'from': (page - 1) * limit,
            'size': limit
        }

        # Filter by specific chats
        if chat_ids:
            search_body['query']['bool']['filter'].append({
                'terms': {'chat_id': chat_ids}
            })

        result = self.es.search(index=self.index_name, body=search_body)

        return {
            'messages': [hit['_source'] for hit in result['hits']['hits']],
            'total': result['hits']['total']['value'],
            'page': page,
            'limit': limit
        }

# Usage
search_service = MessageSearchService('localhost:9200')

# Index message
message = {
    'id': 'msg_123',
    'chat_id': 'chat_456',
    'sender_id': 'user_789',
    'content': 'Hello, how are you doing today?',
    'type': 'text',
    'timestamp': '2024-01-01T10:00:00Z',
    'participants': ['user_123', 'user_456']
}
search_service.index_message(message)

# Search messages
results = search_service.search_messages(
    user_id='user_123',
    query='hello',
    chat_ids=['chat_456'],
    page=1,
    limit=10
)
print(f"Found {results['total']} messages")
```

## Real-World Examples

### WhatsApp Architecture
```
Components:
- End-to-end encryption
- Real-time message delivery
- Media compression and storage
- Multi-device synchronization
- Status updates and stories

Technologies:
- Erlang for server infrastructure
- FreeBSD for operating system
- Custom protocol for message delivery
- S3 for media storage
- Custom encryption protocols
```

### Telegram Architecture
```
Components:
- Cloud-based architecture
- Secret chats with encryption
- Bot platform
- Channel broadcasting
- File sharing and storage

Technologies:
- Custom MTProto protocol
- Multiple data centers
- Distributed architecture
- Custom encryption
- WebRTC for calls
```

## Interview Tips

### Common Questions
1. "How would you handle real-time message delivery?"
2. "What's your approach to end-to-end encryption?"
3. "How would you scale message storage?"
4. "How would you implement message search?"
5. "How would you handle offline message delivery?"

### Answer Framework
1. **Requirements Clarification**: Scale, latency, security requirements
2. **High-Level Design**: WebSocket connections, message queues, encryption
3. **Deep Dive**: Real-time protocols, encryption algorithms, storage strategies
4. **Scalability**: Sharding, caching, load balancing
5. **Edge Cases**: Network failures, device synchronization, message ordering

### Key Points to Emphasize
- Real-time communication challenges
- Security and privacy concerns
- Scalability for massive user base
- Message delivery guarantees
- Cross-platform compatibility

## Practice Problems

1. **Design real-time chat system**
2. **Implement end-to-end encryption**
3. **Create message search functionality**
4. **Handle offline message delivery**
5. **Scale message storage and retrieval**

## Further Reading

- **Books**: "Designing Data-Intensive Applications" by Martin Kleppmann
- **Papers**: "WhatsApp Architecture", "Signal Protocol"
- **Concepts**: Real-time Systems, Cryptography, Distributed Messaging
