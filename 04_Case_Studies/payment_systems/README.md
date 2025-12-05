# Payment Systems (Stripe/PayPal)

## Problem Statement

Design a secure and scalable payment processing platform like Stripe or PayPal that handles online transactions, fraud detection, and compliance.

## Requirements

### Functional Requirements
- Merchant onboarding and verification
- Payment processing (credit cards, digital wallets)
- Fraud detection and prevention
- Settlement and payouts
- Recurring billing and subscriptions
- Multi-currency support
- PCI DSS compliance
- Dispute management

### Non-Functional Requirements
- **Scale**: 1 billion transactions per year, 100 million monthly active users
- **Latency**: < 500ms for payment authorization
- **Availability**: 99.999% uptime
- **Security**: End-to-end encryption, PCI compliance
- **Consistency**: Strong consistency for transactions

## Scale Estimation

### Traffic Estimates
```
Monthly Active Users: 100 million
Daily Transactions: 10 million
Peak Transactions per Second: 10,000
Merchant Onboarding: 1 million/month
Fraud Checks per Second: 50,000
```

### Storage Estimates
```
Transaction Data: 10M × 2KB = 20GB/day = 7.3TB/year
User Data: 100M × 1KB = 100GB
Merchant Data: 10M × 5KB = 50GB
Fraud Data: 10M × 500B = 5GB/day = 1.8TB/year
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
│  │   Merchant  │  │   Payment   │  │   Fraud     │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│  Merchant DB  │  Transaction      │  Fraud DB           │
│  (PostgreSQL) │  DB (PostgreSQL)  │  (PostgreSQL)       │
│               │                   │                     │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Core Services                              │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Settlement│  │   Compliance│  │   Analytics │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Data Stores                                │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Redis     │  │   Kafka     │  │   Vault     │      │
│  │   (Cache)   │  │   (Events)  │  │   (Secrets) │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
```

## API Design

### Merchant APIs
```http
POST /api/v1/merchants
{
  "business_name": "Acme Corp",
  "email": "contact@acme.com",
  "tax_id": "12-3456789",
  "bank_account": {
    "routing_number": "123456789",
    "account_number": "9876543210"
  }
}

GET /api/v1/merchants/{merchant_id}/dashboard
Response:
{
  "merchant_id": "merch_123",
  "balance": 125000.50,
  "pending_settlements": 25000.00,
  "monthly_volume": 500000.00,
  "transactions_today": 1250
}
```

### Payment APIs
```http
POST /api/v1/payments
{
  "amount": 99.99,
  "currency": "USD",
  "merchant_id": "merch_123",
  "payment_method": {
    "type": "card",
    "card_number": "4242424242424242",
    "exp_month": 12,
    "exp_year": 2025,
    "cvc": "123"
  },
  "description": "Order #12345"
}

GET /api/v1/payments/{payment_id}
Response:
{
  "payment_id": "pay_456",
  "status": "succeeded",
  "amount": 99.99,
  "currency": "USD",
  "merchant_id": "merch_123",
  "created_at": "2024-01-01T10:00:00Z",
  "captured_at": "2024-01-01T10:00:05Z"
}
```

### Subscription APIs
```http
POST /api/v1/subscriptions
{
  "customer_id": "cus_789",
  "merchant_id": "merch_123",
  "amount": 29.99,
  "currency": "USD",
  "interval": "month",
  "payment_method_id": "pm_101"
}

POST /api/v1/subscriptions/{subscription_id}/cancel
{
  "cancel_at_period_end": true
}
```

## Database Design

### Merchants Table
```sql
CREATE TABLE merchants (
    id VARCHAR(50) PRIMARY KEY,
    business_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    tax_id VARCHAR(50),
    status ENUM('pending', 'active', 'suspended', 'terminated') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_status (status)
);
```

### Payments Table
```sql
CREATE TABLE payments (
    id VARCHAR(50) PRIMARY KEY,
    merchant_id VARCHAR(50) REFERENCES merchants(id),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status ENUM('pending', 'succeeded', 'failed', 'canceled', 'refunded') DEFAULT 'pending',
    payment_method_type VARCHAR(20),
    payment_method_id VARCHAR(50),
    description TEXT,
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    captured_at TIMESTAMP,
    INDEX idx_merchant_id (merchant_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

### Customers Table
```sql
CREATE TABLE customers (
    id VARCHAR(50) PRIMARY KEY,
    merchant_id VARCHAR(50) REFERENCES merchants(id),
    email VARCHAR(255),
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_merchant_id (merchant_id),
    INDEX idx_email (email)
);
```

### Payment Methods Table
```sql
CREATE TABLE payment_methods (
    id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) REFERENCES customers(id),
    type ENUM('card', 'bank_account', 'wallet') NOT NULL,
    card_brand VARCHAR(20),
    last4 VARCHAR(4),
    exp_month INTEGER,
    exp_year INTEGER,
    fingerprint VARCHAR(255),  -- For duplicate detection
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_customer_id (customer_id),
    INDEX idx_fingerprint (fingerprint)
);
```

## Payment Processing Flow

### Secure Tokenization
```python
import hashlib
import hmac
import secrets
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from typing import Dict, Tuple
import base64

class TokenizationService:
    def __init__(self, master_key: bytes):
        self.master_key = master_key
        self.token_length = 32

    def tokenize_card(self, card_data: Dict) -> Tuple[str, Dict]:
        """Tokenize credit card data"""
        # Create card fingerprint for duplicate detection
        fingerprint = self._create_fingerprint(card_data)

        # Check if card already tokenized
        existing_token = self._find_existing_token(fingerprint)
        if existing_token:
            return existing_token, {"status": "existing"}

        # Generate new token
        token = self._generate_token()

        # Encrypt sensitive data
        encrypted_data = self._encrypt_card_data(card_data)

        # Store token and encrypted data
        self._store_token_data(token, encrypted_data, fingerprint)

        return token, {"status": "new"}

    def detokenize_card(self, token: str) -> Dict:
        """Retrieve card data from token"""
        encrypted_data = self._retrieve_token_data(token)
        if not encrypted_data:
            raise ValueError("Invalid token")

        return self._decrypt_card_data(encrypted_data)

    def _create_fingerprint(self, card_data: Dict) -> str:
        """Create unique fingerprint for card"""
        # Use card number, expiry, and name for fingerprint
        fingerprint_data = f"{card_data['number']}|{card_data['exp_month']}|{card_data['exp_year']}|{card_data['name']}"

        # Hash with salt
        salt = secrets.token_bytes(16)
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )

        fingerprint = base64.b64encode(kdf.derive(fingerprint_data.encode())).decode()
        return fingerprint

    def _generate_token(self) -> str:
        """Generate secure random token"""
        return secrets.token_urlsafe(self.token_length)

    def _encrypt_card_data(self, card_data: Dict) -> bytes:
        """Encrypt card data using AES"""
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

        # Generate random key for this card
        card_key = secrets.token_bytes(32)

        # Encrypt card key with master key
        encrypted_key = self._encrypt_with_master_key(card_key)

        # Encrypt card data with card key
        iv = secrets.token_bytes(16)
        cipher = Cipher(algorithms.AES(card_key), modes.CBC(iv))
        encryptor = cipher.encryptor()

        data_str = str(card_data)
        padded_data = self._pad(data_str.encode())
        encrypted_data = encryptor.update(padded_data) + encryptor.finalize()

        # Return encrypted key + iv + encrypted data
        return encrypted_key + iv + encrypted_data

    def _decrypt_card_data(self, encrypted_data: bytes) -> Dict:
        """Decrypt card data"""
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

        # Extract components
        encrypted_key = encrypted_data[:48]  # Assuming 48 bytes for encrypted key
        iv = encrypted_data[48:64]
        encrypted_card_data = encrypted_data[64:]

        # Decrypt card key
        card_key = self._decrypt_with_master_key(encrypted_key)

        # Decrypt card data
        cipher = Cipher(algorithms.AES(card_key), modes.CBC(iv))
        decryptor = cipher.decryptor()

        padded_data = decryptor.update(encrypted_card_data) + decryptor.finalize()
        data_str = self._unpad(padded_data).decode()

        # Parse back to dict (simplified)
        return eval(data_str)  # In production, use proper JSON serialization

    def _encrypt_with_master_key(self, data: bytes) -> bytes:
        """Encrypt data with master key"""
        # Simplified - in production use proper key management
        cipher = Cipher(algorithms.AES(self.master_key), modes.ECB())
        encryptor = cipher.encryptor()
        return encryptor.update(self._pad(data)) + encryptor.finalize()

    def _decrypt_with_master_key(self, data: bytes) -> bytes:
        """Decrypt data with master key"""
        cipher = Cipher(algorithms.AES(self.master_key), modes.ECB())
        decryptor = cipher.decryptor()
        return self._unpad(decryptor.update(data) + decryptor.finalize())

    def _pad(self, data: bytes, block_size: int = 16) -> bytes:
        """PKCS7 padding"""
        padding_length = block_size - (len(data) % block_size)
        padding = bytes([padding_length]) * padding_length
        return data + padding

    def _unpad(self, data: bytes) -> bytes:
        """PKCS7 unpadding"""
        padding_length = data[-1]
        return data[:-padding_length]

    def _find_existing_token(self, fingerprint: str) -> str:
        """Check if card already tokenized (placeholder)"""
        # In real implementation, query database
        return None

    def _store_token_data(self, token: str, encrypted_data: bytes, fingerprint: str):
        """Store token data (placeholder)"""
        # In real implementation, store in secure database
        pass

    def _retrieve_token_data(self, token: str) -> bytes:
        """Retrieve token data (placeholder)"""
        # In real implementation, query secure database
        return None

# Usage
master_key = secrets.token_bytes(32)
tokenizer = TokenizationService(master_key)

card_data = {
    'number': '4242424242424242',
    'exp_month': 12,
    'exp_year': 2025,
    'cvc': '123',
    'name': 'John Doe'
}

token, status = tokenizer.tokenize_card(card_data)
print(f"Token: {token}, Status: {status}")

# Later, retrieve card data
retrieved_card = tokenizer.detokenize_card(token)
print(f"Retrieved card: {retrieved_card}")
```

## Fraud Detection

### Machine Learning Model
```python
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
from typing import Dict, List
import pickle

class FraudDetectionService:
    def __init__(self):
        self.model = None
        self.scaler = None
        self.feature_names = [
            'amount', 'merchant_category', 'card_age_days',
            'transactions_last_24h', 'avg_transaction_amount',
            'distance_from_home', 'unusual_hour', 'foreign_transaction'
        ]

    def train_model(self, training_data: List[Dict]):
        """Train fraud detection model"""
        # Extract features and labels
        X = []
        y = []

        for transaction in training_data:
            features = self._extract_features(transaction)
            X.append(features)
            y.append(transaction['is_fraud'])

        X = np.array(X)

        # Scale features
        self.scaler = StandardScaler()
        X_scaled = self.scaler.fit_transform(X)

        # Train model
        self.model = RandomForestClassifier(
            n_estimators=100,
            max_depth=10,
            random_state=42
        )
        self.model.fit(X_scaled, y)

    def predict_fraud(self, transaction: Dict) -> Dict:
        """Predict fraud probability for transaction"""
        if not self.model:
            raise ValueError("Model not trained")

        features = self._extract_features(transaction)
        features_scaled = self.scaler.transform([features])

        fraud_probability = self.model.predict_proba(features_scaled)[0][1]
        is_fraud = fraud_probability > 0.5

        return {
            'is_fraud': is_fraud,
            'fraud_probability': fraud_probability,
            'risk_score': self._calculate_risk_score(transaction, fraud_probability)
        }

    def _extract_features(self, transaction: Dict) -> List[float]:
        """Extract features from transaction"""
        features = []

        # Amount (log transformed)
        features.append(np.log(transaction['amount'] + 1))

        # Merchant category (one-hot encoded, simplified)
        category_mapping = {
            'retail': 0, 'restaurant': 1, 'online': 2, 'travel': 3
        }
        features.append(category_mapping.get(transaction.get('merchant_category', 'retail'), 0))

        # Card age in days
        features.append(transaction.get('card_age_days', 0))

        # Transactions in last 24 hours
        features.append(transaction.get('transactions_last_24h', 0))

        # Average transaction amount
        features.append(transaction.get('avg_transaction_amount', 0))

        # Distance from cardholder's home
        features.append(transaction.get('distance_from_home', 0))

        # Unusual hour (1 if transaction at unusual time)
        hour = transaction.get('hour', 12)
        features.append(1 if hour < 6 or hour > 22 else 0)

        # Foreign transaction
        features.append(1 if transaction.get('foreign_transaction', False) else 0)

        return features

    def _calculate_risk_score(self, transaction: Dict, fraud_prob: float) -> float:
        """Calculate comprehensive risk score"""
        base_score = fraud_prob * 100

        # Adjust based on transaction characteristics
        if transaction.get('amount', 0) > 1000:
            base_score += 20

        if transaction.get('foreign_transaction', False):
            base_score += 15

        if transaction.get('unusual_hour', False):
            base_score += 10

        return min(base_score, 100)

    def save_model(self, filepath: str):
        """Save trained model"""
        model_data = {
            'model': self.model,
            'scaler': self.scaler,
            'feature_names': self.feature_names
        }
        with open(filepath, 'wb') as f:
            pickle.dump(model_data, f)

    def load_model(self, filepath: str):
        """Load trained model"""
        with open(filepath, 'rb') as f:
            model_data = pickle.load(f)

        self.model = model_data['model']
        self.scaler = model_data['scaler']
        self.feature_names = model_data['feature_names']

# Usage
fraud_detector = FraudDetectionService()

# Sample training data
training_data = [
    {
        'amount': 50.00,
        'merchant_category': 'retail',
        'card_age_days': 365,
        'transactions_last_24h': 2,
        'avg_transaction_amount': 45.00,
        'distance_from_home': 5,
        'hour': 14,
        'foreign_transaction': False,
        'is_fraud': 0
    },
    {
        'amount': 5000.00,
        'merchant_category': 'online',
        'card_age_days': 30,
        'transactions_last_24h': 10,
        'avg_transaction_amount': 75.00,
        'distance_from_home': 5000,
        'hour': 3,
        'foreign_transaction': True,
        'is_fraud': 1
    }
]

# Train model
fraud_detector.train_model(training_data)

# Predict fraud for new transaction
new_transaction = {
    'amount': 200.00,
    'merchant_category': 'online',
    'card_age_days': 180,
    'transactions_last_24h': 1,
    'avg_transaction_amount': 150.00,
    'distance_from_home': 100,
    'hour': 15,
    'foreign_transaction': False
}

result = fraud_detector.predict_fraud(new_transaction)
print(f"Fraud prediction: {result}")
```

## Settlement and Payouts

### Automated Settlement Process
```python
import datetime
from typing import List, Dict
from decimal import Decimal

class SettlementService:
    def __init__(self, db_connection, bank_api):
        self.db = db_connection
        self.bank_api = bank_api
        self.settlement_fee = Decimal('0.029')  # 2.9%
        self.fixed_fee = Decimal('0.30')

    def process_daily_settlements(self):
        """Process settlements for all merchants"""
        merchants = self._get_merchants_due_settlement()

        for merchant in merchants:
            try:
                self._settle_merchant(merchant)
            except Exception as e:
                print(f"Failed to settle merchant {merchant['id']}: {e}")
                # Log error and retry later

    def _settle_merchant(self, merchant: Dict):
        """Settle funds for a specific merchant"""
        merchant_id = merchant['id']

        # Calculate settlement amount
        settlement_amount = self._calculate_settlement_amount(merchant_id)

        if settlement_amount <= 0:
            return

        # Create settlement record
        settlement_id = self._create_settlement_record(merchant_id, settlement_amount)

        # Transfer funds via bank API
        transfer_result = self.bank_api.transfer_funds(
            from_account='platform_reserve',
            to_account=merchant['bank_account'],
            amount=settlement_amount,
            reference=f"Settlement {settlement_id}"
        )

        if transfer_result['success']:
            # Update settlement status
            self._update_settlement_status(settlement_id, 'completed')

            # Update merchant balance
            self._update_merchant_balance(merchant_id, -settlement_amount)
        else:
            # Handle transfer failure
            self._update_settlement_status(settlement_id, 'failed')
            # Notify merchant and retry later

    def _calculate_settlement_amount(self, merchant_id: str) -> Decimal:
        """Calculate amount to settle for merchant"""
        # Get pending settlements
        pending_amount = self._get_pending_settlement_amount(merchant_id)

        # Subtract fees
        fees = self._calculate_fees(pending_amount)

        return max(pending_amount - fees, Decimal('0'))

    def _calculate_fees(self, amount: Decimal) -> Decimal:
        """Calculate processing fees"""
        percentage_fee = amount * self.settlement_fee
        total_fee = percentage_fee + self.fixed_fee
        return total_fee

    def _create_settlement_record(self, merchant_id: str, amount: Decimal) -> str:
        """Create settlement record in database"""
        settlement_id = f"sett_{int(datetime.datetime.now().timestamp())}"

        # Insert into database
        self.db.execute("""
            INSERT INTO settlements (id, merchant_id, amount, status, created_at)
            VALUES (?, ?, ?, 'pending', ?)
        """, (settlement_id, merchant_id, str(amount), datetime.datetime.now()))

        return settlement_id

    def _update_settlement_status(self, settlement_id: str, status: str):
        """Update settlement status"""
        self.db.execute("""
            UPDATE settlements SET status = ?, updated_at = ?
            WHERE id = ?
        """, (status, datetime.datetime.now(), settlement_id))

    def _update_merchant_balance(self, merchant_id: str, amount_change: Decimal):
        """Update merchant's available balance"""
        self.db.execute("""
            UPDATE merchants SET balance = balance + ?, updated_at = ?
            WHERE id = ?
        """, (str(amount_change), datetime.datetime.now(), merchant_id))

    def _get_merchants_due_settlement(self) -> List[Dict]:
        """Get merchants due for settlement (simplified)"""
        # In real implementation, check settlement schedule
        return self.db.execute("""
            SELECT id, bank_account FROM merchants
            WHERE balance > 1000  -- Minimum settlement amount
        """).fetchall()

    def _get_pending_settlement_amount(self, merchant_id: str) -> Decimal:
        """Get pending settlement amount for merchant"""
        result = self.db.execute("""
            SELECT COALESCE(SUM(amount), 0) as pending_amount
            FROM merchant_balances
            WHERE merchant_id = ? AND status = 'available'
        """, (merchant_id,)).fetchone()

        return Decimal(result['pending_amount'])

# Usage
class MockBankAPI:
    def transfer_funds(self, from_account, to_account, amount, reference):
        # Simulate bank transfer
        print(f"Transferring {amount} from {from_account} to {to_account}")
        return {'success': True, 'transaction_id': 'txn_123'}

settlement_service = SettlementService(db_connection=None, bank_api=MockBankAPI())
settlement_service.process_daily_settlements()
```

## Real-World Examples

### Stripe Architecture
```
Components:
- API-first design
- Strong security and compliance
- Global payment processing
- Developer-friendly integrations
- Advanced fraud detection

Technologies:
- Ruby for core services
- PostgreSQL for data storage
- Redis for caching
- Kafka for event processing
- Custom security infrastructure
```

### PayPal Architecture
```
Components:
- Mass payment processing
- Risk management systems
- Multi-currency support
- Merchant services
- Digital wallet features

Technologies:
- Java for backend services
- Oracle databases
- Custom encryption
- Real-time fraud detection
- Global data centers
```

## Interview Tips

### Common Questions
1. "How would you design secure payment processing?"
2. "What's your approach to fraud detection?"
3. "How would you handle PCI compliance?"
4. "How would you implement tokenization?"
5. "How would you scale payment processing?"

### Answer Framework
1. **Requirements Clarification**: Security, compliance, scale requirements
2. **High-Level Design**: Tokenization, fraud detection, settlement
3. **Deep Dive**: Encryption, ML models, payment flows
4. **Scalability**: Load balancing, database sharding, caching
5. **Edge Cases**: Chargebacks, failed payments, regulatory compliance

### Key Points to Emphasize
- Security and compliance requirements
- Fraud detection and prevention
- Scalability for transaction volume
- Regulatory compliance (PCI DSS)
- Merchant and customer experience

## Practice Problems

1. **Design secure payment tokenization**
2. **Implement fraud detection system**
3. **Create settlement and payout process**
4. **Handle PCI DSS compliance**
5. **Scale payment processing infrastructure**

## Further Reading

- **Books**: "The Phoenix Project" by Gene Kim
- **Papers**: "Stripe Architecture", "Payment System Security"
- **Concepts**: PCI DSS, Fraud Detection, Financial Regulations
