# Relyr Technical Architecture

**Version:** 1.0
**Last Updated:** November 2025
**Status:** MVP Design

---

## Table of Contents

1. [System Overview](#system-overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [Database Schema](#database-schema)
6. [Security Architecture](#security-architecture)
7. [Integration Architecture](#integration-architecture)
8. [Infrastructure Architecture](#infrastructure-architecture)
9. [Scalability Considerations](#scalability-considerations)
10. [Appendix](#appendix)

---

## System Overview

Relyr is a hybrid payment platform that bridges traditional finance (fiat) and blockchain (stablecoins) to enable instant, low-cost cross-border payments for Nigerian freelancers and businesses.

### Core Capabilities
- **Dual Wallet System:** NGN (via Anchor BaaS) + USDC (via Polygon blockchain)
- **Instant Transfers:** Wallet-to-wallet transfers in <2 minutes
- **Currency Conversion:** Real-time USDC ↔ NGN conversion
- **Funding & Withdrawal:** Card/bank funding via Paystack, bank withdrawals via Anchor
- **Compliance:** KYC via Smile Identity, bank verification via Mono

### Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Mobile | React Native + Expo |
| Backend | NestJS + TypeScript |
| Database | PostgreSQL 15 |
| Cache/Queue | Redis + BullMQ |
| Blockchain | Polygon (USDC via Circle API) |
| BaaS | Anchor (virtual NGN accounts) |
| Payments | Paystack (card/bank payments) |
| Open Banking | Mono (bank verification) |
| KYC | Smile Identity |
| Cloud | AWS (Fargate, RDS, ElastiCache) |
| Monitoring | Sentry, CloudWatch |

---

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        MobileApp[React Native Mobile App]
        AdminWeb[Admin Dashboard - Next.js]
    end

    subgraph "API Gateway & Load Balancer"
        ALB[AWS Application Load Balancer]
        APIGateway[NestJS API Gateway]
    end

    subgraph "Application Layer"
        AuthService[Auth Service]
        UserService[User Service]
        WalletService[Wallet Service]
        TransactionService[Transaction Service]
        KYCService[KYC Service]
        NotificationService[Notification Service]
    end

    subgraph "Integration Layer"
        AnchorClient[Anchor Client]
        CircleClient[Circle Client]
        PaystackClient[Paystack Client]
        MonoClient[Mono Client]
        SmileClient[Smile Identity Client]
        PolygonClient[Polygon Web3 Client]
    end

    subgraph "Data Layer"
        PostgreSQL[(PostgreSQL)]
        Redis[(Redis Cache)]
        BullQueue[BullMQ Job Queue]
    end

    subgraph "External Services"
        Anchor[Anchor BaaS API]
        Circle[Circle USDC API]
        Polygon[Polygon Network]
        Paystack[Paystack API]
        Mono[Mono API]
        Smile[Smile Identity API]
    end

    subgraph "Infrastructure"
        S3[AWS S3 - File Storage]
        CloudWatch[CloudWatch - Logging]
        Sentry[Sentry - Error Tracking]
        SES[AWS SES - Email]
    end

    MobileApp --> ALB
    AdminWeb --> ALB
    ALB --> APIGateway

    APIGateway --> AuthService
    APIGateway --> UserService
    APIGateway --> WalletService
    APIGateway --> TransactionService
    APIGateway --> KYCService
    APIGateway --> NotificationService

    AuthService --> PostgreSQL
    UserService --> PostgreSQL
    WalletService --> PostgreSQL
    TransactionService --> PostgreSQL
    KYCService --> PostgreSQL

    WalletService --> Redis
    TransactionService --> Redis

    WalletService --> AnchorClient
    WalletService --> CircleClient
    TransactionService --> AnchorClient
    TransactionService --> CircleClient
    TransactionService --> PolygonClient
    TransactionService --> PaystackClient
    KYCService --> MonoClient
    KYCService --> SmileClient

    AnchorClient --> Anchor
    CircleClient --> Circle
    PolygonClient --> Polygon
    PaystackClient --> Paystack
    MonoClient --> Mono
    SmileClient --> Smile

    NotificationService --> BullQueue
    BullQueue --> Redis

    KYCService --> S3
    APIGateway --> CloudWatch
    APIGateway --> Sentry
    NotificationService --> SES
```

---

## Component Architecture

### 1. Mobile Application (React Native + Expo)

```
mobile/
├── src/
│   ├── screens/              # Screen components
│   │   ├── auth/
│   │   │   ├── WelcomeScreen.tsx
│   │   │   ├── LoginScreen.tsx
│   │   │   └── RegisterScreen.tsx
│   │   ├── kyc/
│   │   │   ├── KYCIntroScreen.tsx
│   │   │   ├── IDUploadScreen.tsx
│   │   │   ├── SelfieScreen.tsx
│   │   │   └── BankLinkScreen.tsx
│   │   ├── wallet/
│   │   │   ├── DashboardScreen.tsx
│   │   │   ├── WalletDetailScreen.tsx
│   │   │   └── CreateWalletScreen.tsx
│   │   ├── transaction/
│   │   │   ├── SendMoneyScreen.tsx
│   │   │   ├── ReceiveMoneyScreen.tsx
│   │   │   ├── TransactionListScreen.tsx
│   │   │   ├── TransactionDetailScreen.tsx
│   │   │   └── ConvertScreen.tsx
│   │   ├── funding/
│   │   │   └── FundWalletScreen.tsx
│   │   └── profile/
│   │       ├── ProfileScreen.tsx
│   │       ├── SettingsScreen.tsx
│   │       └── ReferralScreen.tsx
│   ├── components/           # Reusable components
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Card.tsx
│   │   │   └── Header.tsx
│   │   ├── wallet/
│   │   │   ├── WalletCard.tsx
│   │   │   └── BalanceDisplay.tsx
│   │   └── transaction/
│   │       ├── TransactionItem.tsx
│   │       └── TransactionStatus.tsx
│   ├── navigation/           # Navigation setup
│   │   ├── AppNavigator.tsx
│   │   ├── AuthStack.tsx
│   │   └── MainStack.tsx
│   ├── services/             # API clients
│   │   ├── api.ts           # Axios instance
│   │   ├── authApi.ts
│   │   ├── walletApi.ts
│   │   ├── transactionApi.ts
│   │   └── kycApi.ts
│   ├── hooks/                # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useWallet.ts
│   │   └── useTransactions.ts
│   ├── store/                # State management (Zustand)
│   │   ├── authStore.ts
│   │   ├── walletStore.ts
│   │   └── userStore.ts
│   ├── theme/                # Design system
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   └── spacing.ts
│   └── utils/                # Utilities
│       ├── validation.ts
│       ├── formatting.ts
│       └── constants.ts
└── app.config.js
```

**Key Technologies:**
- **Navigation:** React Navigation 6
- **State Management:** Zustand
- **Data Fetching:** TanStack Query (React Query)
- **Forms:** React Hook Form
- **UI Library:** React Native Paper / NativeBase
- **Camera:** Expo Camera
- **Secure Storage:** expo-secure-store

---

### 2. Backend API (NestJS)

```
backend/
├── src/
│   ├── auth/                 # Authentication module
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.module.ts
│   │   ├── strategies/
│   │   │   └── jwt.strategy.ts
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   └── dto/
│   │       ├── register.dto.ts
│   │       └── login.dto.ts
│   ├── users/                # User management
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.module.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   └── dto/
│   │       └── update-user.dto.ts
│   ├── wallets/              # Wallet management
│   │   ├── wallets.controller.ts
│   │   ├── wallets.service.ts
│   │   ├── wallets.module.ts
│   │   ├── entities/
│   │   │   └── wallet.entity.ts
│   │   └── dto/
│   │       └── create-wallet.dto.ts
│   ├── transactions/         # Transaction processing
│   │   ├── transactions.controller.ts
│   │   ├── transactions.service.ts
│   │   ├── transactions.module.ts
│   │   ├── entities/
│   │   │   └── transaction.entity.ts
│   │   ├── dto/
│   │   │   ├── send-transaction.dto.ts
│   │   │   └── convert-currency.dto.ts
│   │   └── processors/
│   │       ├── ngn-transfer.processor.ts
│   │       ├── usdc-transfer.processor.ts
│   │       └── conversion.processor.ts
│   ├── kyc/                  # KYC/Compliance
│   │   ├── kyc.controller.ts
│   │   ├── kyc.service.ts
│   │   ├── kyc.module.ts
│   │   ├── entities/
│   │   │   └── kyc-record.entity.ts
│   │   └── dto/
│   │       └── verify-kyc.dto.ts
│   ├── notifications/        # Notifications
│   │   ├── notifications.service.ts
│   │   ├── notifications.module.ts
│   │   └── processors/
│   │       ├── email.processor.ts
│   │       └── push.processor.ts
│   ├── webhooks/             # External webhooks
│   │   ├── webhooks.controller.ts
│   │   ├── webhooks.service.ts
│   │   ├── webhooks.module.ts
│   │   └── handlers/
│   │       ├── anchor.handler.ts
│   │       ├── circle.handler.ts
│   │       └── paystack.handler.ts
│   ├── integrations/         # Third-party integrations
│   │   ├── anchor/
│   │   │   ├── anchor.service.ts
│   │   │   └── anchor.module.ts
│   │   ├── circle/
│   │   │   ├── circle.service.ts
│   │   │   └── circle.module.ts
│   │   ├── paystack/
│   │   │   ├── paystack.service.ts
│   │   │   └── paystack.module.ts
│   │   ├── mono/
│   │   │   ├── mono.service.ts
│   │   │   └── mono.module.ts
│   │   ├── smile/
│   │   │   ├── smile.service.ts
│   │   │   └── smile.module.ts
│   │   └── polygon/
│   │       ├── polygon.service.ts
│   │       └── polygon.module.ts
│   ├── admin/                # Admin endpoints
│   │   ├── admin.controller.ts
│   │   ├── admin.service.ts
│   │   └── admin.module.ts
│   ├── common/               # Shared utilities
│   │   ├── decorators/
│   │   ├── filters/
│   │   ├── interceptors/
│   │   ├── pipes/
│   │   └── utils/
│   ├── config/               # Configuration
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   └── app.config.ts
│   ├── app.module.ts
│   └── main.ts
├── test/
├── migrations/
├── .env.example
├── package.json
├── tsconfig.json
└── Dockerfile
```

**Key Technologies:**
- **Framework:** NestJS 10
- **ORM:** TypeORM
- **Validation:** class-validator, class-transformer
- **Queue:** BullMQ
- **Caching:** ioredis
- **Documentation:** Swagger/OpenAPI
- **Testing:** Jest, Supertest

---

### 3. Database Schema (PostgreSQL)

#### Entity Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ WALLET : has
    USER ||--o{ KYC_RECORD : has
    USER ||--o{ BANK_ACCOUNT : links
    USER ||--o{ REFERRAL : refers
    WALLET ||--o{ TRANSACTION : from
    WALLET ||--o{ TRANSACTION : to
    TRANSACTION ||--o{ WEBHOOK_EVENT : triggers

    USER {
        uuid id PK
        string email UK
        string phone UK
        string password_hash
        string first_name
        string last_name
        string country
        enum kyc_status
        string referral_code UK
        uuid referred_by FK
        timestamp email_verified_at
        timestamp created_at
        timestamp updated_at
    }

    WALLET {
        uuid id PK
        uuid user_id FK
        enum currency
        enum wallet_type
        decimal balance
        string external_id
        json metadata
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    TRANSACTION {
        uuid id PK
        uuid from_wallet_id FK
        uuid to_wallet_id FK
        enum type
        enum status
        decimal amount
        string currency
        decimal fee
        decimal exchange_rate
        string reference UK
        string external_reference
        string recipient_address
        json metadata
        timestamp completed_at
        timestamp created_at
        timestamp updated_at
    }

    KYC_RECORD {
        uuid id PK
        uuid user_id FK
        enum provider
        enum status
        string verification_id
        json submitted_data
        json verification_result
        string rejection_reason
        timestamp verified_at
        timestamp created_at
        timestamp updated_at
    }

    BANK_ACCOUNT {
        uuid id PK
        uuid user_id FK
        string account_number
        string account_name
        string bank_name
        string bank_code
        enum provider
        string external_id
        boolean is_verified
        timestamp created_at
        timestamp updated_at
    }

    WEBHOOK_EVENT {
        uuid id PK
        uuid transaction_id FK
        enum provider
        string event_type
        json payload
        enum status
        int retry_count
        timestamp processed_at
        timestamp created_at
    }

    REFERRAL {
        uuid id PK
        uuid referrer_id FK
        uuid referee_id FK
        decimal reward_amount
        enum status
        timestamp rewarded_at
        timestamp created_at
    }

    AUDIT_LOG {
        uuid id PK
        uuid user_id FK
        string action
        string entity_type
        uuid entity_id
        json changes
        string ip_address
        timestamp created_at
    }
```

#### Table Definitions

**users**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    country VARCHAR(2) DEFAULT 'NG',
    kyc_status VARCHAR(20) DEFAULT 'pending',
    referral_code VARCHAR(10) UNIQUE NOT NULL,
    referred_by UUID REFERENCES users(id),
    email_verified_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_referral_code ON users(referral_code);
```

**wallets**
```sql
CREATE TABLE wallets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    currency VARCHAR(10) NOT NULL,
    wallet_type VARCHAR(20) NOT NULL,
    balance DECIMAL(20, 8) DEFAULT 0,
    external_id VARCHAR(255),
    metadata JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, currency, wallet_type)
);

CREATE INDEX idx_wallets_user_id ON wallets(user_id);
CREATE INDEX idx_wallets_external_id ON wallets(external_id);
```

**transactions**
```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_wallet_id UUID REFERENCES wallets(id),
    to_wallet_id UUID REFERENCES wallets(id),
    type VARCHAR(30) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    amount DECIMAL(20, 8) NOT NULL,
    currency VARCHAR(10) NOT NULL,
    fee DECIMAL(20, 8) DEFAULT 0,
    exchange_rate DECIMAL(20, 8),
    reference VARCHAR(50) UNIQUE NOT NULL,
    external_reference VARCHAR(255),
    recipient_address VARCHAR(255),
    metadata JSONB,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_from_wallet ON transactions(from_wallet_id);
CREATE INDEX idx_transactions_to_wallet ON transactions(to_wallet_id);
CREATE INDEX idx_transactions_reference ON transactions(reference);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);
```

**kyc_records**
```sql
CREATE TABLE kyc_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    verification_id VARCHAR(255),
    submitted_data JSONB,
    verification_result JSONB,
    rejection_reason TEXT,
    verified_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_kyc_user_id ON kyc_records(user_id);
CREATE INDEX idx_kyc_status ON kyc_records(status);
```

---

## Data Flow Diagrams

### 1. User Registration & KYC Flow

```mermaid
sequenceDiagram
    participant User
    participant Mobile
    participant API
    participant DB
    participant Smile
    participant Mono

    User->>Mobile: Enter registration details
    Mobile->>API: POST /auth/register
    API->>DB: Create user record
    API->>DB: Generate referral code
    API-->>Mobile: JWT token + user data

    User->>Mobile: Upload ID + Selfie
    Mobile->>API: POST /kyc/verify
    API->>DB: Create KYC record
    API->>Smile: Submit verification
    Smile-->>API: Verification result
    API->>DB: Update KYC status
    API-->>Mobile: KYC approved

    User->>Mobile: Link bank account
    Mobile->>API: POST /kyc/link-bank
    API->>Mono: Initialize Mono Connect
    Mono-->>Mobile: Mono widget URL
    User->>Mono: Authenticate bank
    Mono->>API: Webhook: Account linked
    API->>DB: Store bank account
    API-->>Mobile: Bank linked successfully
```

### 2. NGN Wallet Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Mobile
    participant API
    participant DB
    participant Anchor

    User->>Mobile: Request NGN wallet
    Mobile->>API: POST /wallets (currency: NGN)
    API->>DB: Check user has no NGN wallet
    API->>Anchor: Create virtual account
    Anchor-->>API: Account details (account_number, bank)
    API->>DB: Store wallet (external_id = anchor_account_id)
    API-->>Mobile: Wallet created
    Mobile->>User: Display NGN account number
```

### 3. USDC Wallet Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Mobile
    participant API
    participant DB
    participant Polygon

    User->>Mobile: Request USDC wallet
    Mobile->>API: POST /wallets/usdc
    API->>Polygon: Generate new address
    Polygon-->>API: Polygon address + private key
    API->>DB: Store wallet (address, encrypted private key)
    API->>DB: Monitor address for incoming transfers
    API-->>Mobile: Wallet created
    Mobile->>User: Display USDC address + QR code
```

### 4. Send NGN Transaction Flow

```mermaid
sequenceDiagram
    participant Sender
    participant Mobile
    participant API
    participant DB
    participant Redis
    participant Queue
    participant Anchor

    Sender->>Mobile: Enter amount + recipient
    Mobile->>API: POST /transactions/send
    API->>DB: Validate sender wallet balance
    API->>Redis: Check rate limits
    API->>DB: Create transaction (status: pending)
    API->>Queue: Add to processing queue
    API-->>Mobile: Transaction initiated

    Queue->>Anchor: Transfer funds
    Anchor-->>Queue: Transfer successful
    Queue->>DB: Update transaction (status: completed)
    Queue->>DB: Update wallet balances
    Queue->>Notification: Send push notification
    Notification-->>Mobile: Transaction completed
```

### 5. Send USDC Transaction Flow

```mermaid
sequenceDiagram
    participant Sender
    participant Mobile
    participant API
    participant DB
    participant Queue
    participant Circle
    participant Polygon

    Sender->>Mobile: Enter amount + recipient address
    Mobile->>API: POST /transactions/send
    API->>DB: Validate USDC balance
    API->>DB: Create transaction (status: pending)
    API->>Queue: Add to blockchain queue
    API-->>Mobile: Transaction initiated

    Queue->>Circle: Initiate USDC transfer
    Circle->>Polygon: Submit transaction
    Polygon-->>Circle: Transaction hash
    Circle-->>Queue: Transaction submitted
    Queue->>DB: Update (status: processing, tx_hash)

    Note over Polygon: Wait for confirmations (~2 min)

    Circle->>API: Webhook: Transfer confirmed
    API->>DB: Update (status: completed)
    API->>DB: Update wallet balance
    API->>Notification: Send notification
    Notification-->>Mobile: Transaction completed
```

### 6. USDC → NGN Conversion Flow

```mermaid
sequenceDiagram
    participant User
    participant Mobile
    participant API
    participant DB
    participant Circle
    participant Anchor
    participant ExchangeRate

    User->>Mobile: Request USDC → NGN conversion
    Mobile->>API: GET /rates/usdc-ngn
    API->>ExchangeRate: Fetch current NGN/USD rate
    ExchangeRate-->>API: Rate (e.g., 1 USDC = 1650 NGN)
    API-->>Mobile: Display rate + amount preview

    User->>Mobile: Confirm conversion
    Mobile->>API: POST /transactions/convert
    API->>DB: Validate USDC balance
    API->>DB: Create conversion transaction

    API->>Circle: Transfer USDC to Relyr reserve
    Circle-->>API: Transfer confirmed

    API->>Anchor: Credit NGN wallet
    Anchor-->>API: Credit successful

    API->>DB: Update transaction (status: completed)
    API->>DB: Update both wallet balances
    API-->>Mobile: Conversion completed
```

### 7. Fund Wallet via Paystack (Card)

```mermaid
sequenceDiagram
    participant User
    participant Mobile
    participant API
    participant DB
    participant Paystack
    participant Anchor

    User->>Mobile: Request to fund NGN wallet
    Mobile->>API: POST /payments/fund (amount)
    API->>DB: Create funding transaction
    API->>Paystack: Initialize payment
    Paystack-->>API: Payment authorization URL
    API-->>Mobile: Redirect to Paystack

    Mobile->>Paystack: User enters card details
    Paystack->>User: 3DS/OTP verification
    User->>Paystack: Complete payment

    Paystack->>API: Webhook: Payment successful
    API->>DB: Verify transaction reference
    API->>Anchor: Credit user's virtual account
    Anchor-->>API: Credit successful
    API->>DB: Update wallet balance
    API->>DB: Update transaction (status: completed)
    API->>Notification: Send notification
    Notification-->>Mobile: Wallet funded successfully
```

### 8. Receive USDC from External Wallet

```mermaid
sequenceDiagram
    participant External
    participant Polygon
    participant Circle
    participant API
    participant DB
    participant Mobile

    External->>Polygon: Send USDC to user's address
    Polygon-->>Circle: Emit transfer event
    Circle->>API: Webhook: Incoming transfer detected
    API->>DB: Find wallet by address
    API->>DB: Create transaction (type: receive)
    API->>Circle: Confirm transaction
    Circle-->>API: Transfer confirmed
    API->>DB: Update wallet balance
    API->>DB: Update transaction (status: completed)
    API->>Notification: Send push notification
    Notification-->>Mobile: USDC received
```

---

## Security Architecture

### 1. Authentication & Authorization

**JWT-based Authentication:**
```typescript
// Token structure
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "role": "user",
  "iat": 1234567890,
  "exp": 1234571490  // 1 hour expiry
}
```

**Security Layers:**
1. **Access Token:** Short-lived (1 hour), used for API requests
2. **Refresh Token:** Long-lived (7 days), stored in secure storage, used to get new access tokens
3. **Transaction PIN:** Required for all send/withdraw operations
4. **Biometric Auth:** Optional, for quick transaction approval

**Authorization Strategy:**
- **Public routes:** /auth/login, /auth/register, /health
- **Authenticated routes:** All /users/*, /wallets/*, /transactions/*
- **KYC-required routes:** /transactions/send, /payments/fund
- **Admin routes:** /admin/* (requires admin role)

### 2. Data Encryption

**At Rest:**
- Database: AWS RDS encryption enabled
- Private keys: AES-256 encryption before storing in DB
- PII data: Encrypted columns for sensitive fields
- S3: Server-side encryption (SSE-S3)

**In Transit:**
- All API communication: HTTPS/TLS 1.3
- Mobile ↔ Backend: Certificate pinning
- Backend ↔ Partners: TLS + API key authentication

**Sensitive Data Handling:**
```typescript
// Example: Encrypting private keys
class CryptoService {
  private readonly algorithm = 'aes-256-gcm';

  encrypt(text: string, masterKey: string): EncryptedData {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, masterKey, iv);
    const encrypted = Buffer.concat([cipher.update(text), cipher.final()]);
    const authTag = cipher.getAuthTag();

    return { encrypted, iv, authTag };
  }

  decrypt(encryptedData: EncryptedData, masterKey: string): string {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      masterKey,
      encryptedData.iv
    );
    decipher.setAuthTag(encryptedData.authTag);

    return Buffer.concat([
      decipher.update(encryptedData.encrypted),
      decipher.final()
    ]).toString();
  }
}
```

### 3. API Security

**Rate Limiting:**
```typescript
// Per-endpoint rate limits
const rateLimits = {
  '/auth/login': { window: '15m', max: 5 },        // 5 attempts per 15 min
  '/auth/register': { window: '1h', max: 3 },      // 3 attempts per hour
  '/transactions/send': { window: '1m', max: 10 }, // 10 transactions per min
  'global': { window: '15m', max: 100 }            // 100 requests per 15 min
};
```

**Request Validation:**
- Input sanitization (class-validator)
- SQL injection protection (TypeORM parameterized queries)
- XSS prevention (sanitize user inputs)
- CSRF protection (token validation)

**Security Headers:**
```typescript
app.use(helmet({
  contentSecurityPolicy: true,
  hsts: { maxAge: 31536000, includeSubDomains: true },
  noSniff: true,
  xssFilter: true,
  referrerPolicy: { policy: 'same-origin' }
}));
```

### 4. Webhook Security

**Signature Verification:**
```typescript
class WebhookService {
  verifyPaystackSignature(payload: string, signature: string): boolean {
    const hash = crypto
      .createHmac('sha512', process.env.PAYSTACK_SECRET_KEY)
      .update(payload)
      .digest('hex');

    return hash === signature;
  }

  verifyCircleSignature(payload: string, signature: string): boolean {
    const publicKey = process.env.CIRCLE_PUBLIC_KEY;
    const verify = crypto.createVerify('RSA-SHA256');
    verify.update(payload);

    return verify.verify(publicKey, signature, 'base64');
  }
}
```

### 5. Transaction Security

**Idempotency:**
- All transaction endpoints require idempotency keys
- Prevent duplicate transactions on retries
- Store idempotency keys in Redis (24-hour TTL)

**Transaction Validation:**
```typescript
class TransactionValidator {
  async validateSend(dto: SendTransactionDto, user: User): Promise<void> {
    // Check KYC status
    if (user.kycStatus !== 'approved') {
      throw new ForbiddenException('KYC verification required');
    }

    // Check wallet balance
    const wallet = await this.walletService.findById(dto.fromWalletId);
    if (wallet.balance < dto.amount + dto.fee) {
      throw new BadRequestException('Insufficient balance');
    }

    // Check transaction limits
    const dailyTotal = await this.getDailyTotal(user.id);
    if (dailyTotal + dto.amount > user.dailyLimit) {
      throw new BadRequestException('Daily limit exceeded');
    }

    // Fraud detection
    const isSuspicious = await this.fraudDetection.check(dto, user);
    if (isSuspicious) {
      await this.flagForReview(dto, user);
      throw new BadRequestException('Transaction requires manual review');
    }
  }
}
```

---

## Integration Architecture

### 1. Anchor BaaS Integration

**Purpose:** Virtual NGN bank accounts, local transfers, bank withdrawals

**API Endpoints Used:**
- `POST /accounts` - Create virtual account
- `GET /accounts/:id` - Get account details
- `GET /accounts/:id/balance` - Get balance
- `POST /transfers` - Initiate transfer
- `GET /transactions` - List transactions

**Webhook Events:**
- `account.credited` - Incoming payment received
- `transfer.completed` - Outgoing transfer completed
- `transfer.failed` - Transfer failed

**Configuration:**
```typescript
@Injectable()
export class AnchorService {
  private readonly baseUrl = 'https://api.getanchor.co/v1';
  private readonly apiKey = process.env.ANCHOR_API_KEY;

  async createVirtualAccount(userId: string): Promise<VirtualAccount> {
    const response = await axios.post(
      `${this.baseUrl}/accounts`,
      {
        customer_id: userId,
        currency: 'NGN',
        type: 'virtual'
      },
      {
        headers: { 'Authorization': `Bearer ${this.apiKey}` }
      }
    );

    return {
      accountNumber: response.data.account_number,
      bankName: response.data.bank_name,
      accountName: response.data.account_name,
      anchorId: response.data.id
    };
  }
}
```

### 2. Circle USDC Integration

**Purpose:** USDC minting, blockchain transfers, wallet management

**API Endpoints Used:**
- `POST /wallets` - Create blockchain wallet
- `GET /wallets/:id` - Get wallet details
- `POST /transfers` - Transfer USDC
- `GET /transfers/:id` - Get transfer status

**Webhook Events:**
- `transfers.created` - Transfer initiated
- `transfers.completed` - Transfer confirmed on blockchain
- `transfers.failed` - Transfer failed

**Configuration:**
```typescript
@Injectable()
export class CircleService {
  private readonly baseUrl = 'https://api.circle.com/v1';
  private readonly apiKey = process.env.CIRCLE_API_KEY;

  async createWallet(): Promise<CircleWallet> {
    const response = await axios.post(
      `${this.baseUrl}/wallets`,
      {
        idempotencyKey: uuidv4(),
        blockchains: ['MATIC']  // Polygon
      },
      {
        headers: { 'Authorization': `Bearer ${this.apiKey}` }
      }
    );

    return {
      walletId: response.data.wallet.walletId,
      address: response.data.wallet.address,
      blockchain: 'MATIC'
    };
  }

  async transferUSDC(dto: TransferDto): Promise<Transfer> {
    const response = await axios.post(
      `${this.baseUrl}/transfers`,
      {
        idempotencyKey: uuidv4(),
        source: {
          type: 'wallet',
          id: dto.fromWalletId
        },
        destination: {
          type: 'blockchain',
          address: dto.toAddress,
          chain: 'MATIC'
        },
        amount: {
          amount: dto.amount.toString(),
          currency: 'USD'
        }
      },
      {
        headers: { 'Authorization': `Bearer ${this.apiKey}` }
      }
    );

    return response.data.transfer;
  }
}
```

### 3. Paystack Integration

**Purpose:** Card payments, bank transfers for wallet funding

**API Endpoints Used:**
- `POST /transaction/initialize` - Initialize payment
- `GET /transaction/verify/:reference` - Verify payment

**Webhook Events:**
- `charge.success` - Payment successful
- `charge.failed` - Payment failed

**Configuration:**
```typescript
@Injectable()
export class PaystackService {
  private readonly baseUrl = 'https://api.paystack.co';
  private readonly secretKey = process.env.PAYSTACK_SECRET_KEY;

  async initializePayment(amount: number, email: string): Promise<PaymentInit> {
    const response = await axios.post(
      `${this.baseUrl}/transaction/initialize`,
      {
        amount: amount * 100, // Convert to kobo
        email,
        currency: 'NGN',
        callback_url: `${process.env.FRONTEND_URL}/payment/callback`
      },
      {
        headers: { 'Authorization': `Bearer ${this.secretKey}` }
      }
    );

    return {
      authorizationUrl: response.data.data.authorization_url,
      reference: response.data.data.reference
    };
  }
}
```

### 4. Mono Integration

**Purpose:** Bank account verification, transaction history access

**API Endpoints Used:**
- `POST /account/auth` - Initialize Mono Connect
- `GET /accounts/:id` - Get account details
- `GET /accounts/:id/identity` - Get account owner identity

**Webhook Events:**
- `mono.events.account_linked` - Account successfully linked

**Configuration:**
```typescript
@Injectable()
export class MonoService {
  private readonly baseUrl = 'https://api.withmono.com';
  private readonly secretKey = process.env.MONO_SECRET_KEY;

  async getAccountDetails(accountId: string): Promise<BankAccount> {
    const response = await axios.get(
      `${this.baseUrl}/accounts/${accountId}`,
      {
        headers: { 'mono-sec-key': this.secretKey }
      }
    );

    return {
      accountNumber: response.data.account.accountNumber,
      accountName: response.data.account.name,
      bankName: response.data.account.institution.name,
      bankCode: response.data.account.institution.bankCode
    };
  }
}
```

### 5. Smile Identity Integration

**Purpose:** KYC verification (ID + Selfie + Liveness check)

**API Endpoints Used:**
- `POST /upload` - Upload ID and selfie
- `GET /job_status/:job_id` - Check verification status

**Configuration:**
```typescript
@Injectable()
export class SmileIdentityService {
  private readonly baseUrl = 'https://api.smileidentity.com/v1';
  private readonly apiKey = process.env.SMILE_API_KEY;
  private readonly partnerId = process.env.SMILE_PARTNER_ID;

  async verifyIdentity(dto: KYCDto): Promise<VerificationResult> {
    const response = await axios.post(
      `${this.baseUrl}/upload`,
      {
        partner_id: this.partnerId,
        job_type: 6, // Enhanced KYC
        id_type: 'BVN', // or 'NIN', 'DRIVERS_LICENSE', etc.
        id_number: dto.idNumber,
        images: [
          { image_type_id: 0, image: dto.selfieBase64 },
          { image_type_id: 1, image: dto.idFrontBase64 }
        ],
        partner_params: {
          user_id: dto.userId,
          job_id: uuidv4()
        }
      },
      {
        headers: { 'Authorization': `Bearer ${this.apiKey}` }
      }
    );

    return {
      jobId: response.data.job_id,
      status: response.data.job_status,
      result: response.data.result
    };
  }
}
```

---

## Infrastructure Architecture

### AWS Architecture Diagram

```mermaid
graph TB
    subgraph "Public Internet"
        Users[Mobile Users]
        Admins[Admin Users]
    end

    subgraph "AWS Cloud"
        subgraph "Edge Layer"
            Route53[Route 53 DNS]
            ACM[AWS Certificate Manager]
            CloudFront[CloudFront CDN - Optional]
        end

        subgraph "Load Balancing"
            ALB[Application Load Balancer]
        end

        subgraph "Compute Layer - VPC Public Subnet"
            ECS[ECS Fargate Cluster]
            Task1[API Task 1]
            Task2[API Task 2]
            Task3[API Task 3]
        end

        subgraph "Data Layer - VPC Private Subnet"
            RDS[(RDS PostgreSQL)]
            ElastiCache[(ElastiCache Redis)]
        end

        subgraph "Storage"
            S3[S3 - File Storage]
        end

        subgraph "Monitoring & Logging"
            CloudWatch[CloudWatch Logs]
            CloudWatchMetrics[CloudWatch Metrics]
            SNS[SNS - Alerts]
        end

        subgraph "Security"
            Secrets[Secrets Manager]
            IAM[IAM Roles]
            SecurityGroups[Security Groups]
        end
    end

    subgraph "External Services"
        Sentry[Sentry]
        SES[AWS SES - Email]
    end

    Users --> Route53
    Admins --> Route53
    Route53 --> ALB
    ACM -.->|SSL Cert| ALB

    ALB --> Task1
    ALB --> Task2
    ALB --> Task3

    Task1 --> RDS
    Task2 --> RDS
    Task3 --> RDS

    Task1 --> ElastiCache
    Task2 --> ElastiCache
    Task3 --> ElastiCache

    Task1 --> S3
    Task2 --> S3
    Task3 --> S3

    Task1 --> CloudWatch
    Task2 --> CloudWatch
    Task3 --> CloudWatch

    Task1 --> Sentry
    Task2 --> Sentry
    Task3 --> Sentry

    Task1 --> SES

    CloudWatchMetrics --> SNS

    Task1 -.->|Fetch Secrets| Secrets
    Task2 -.->|Fetch Secrets| Secrets
    Task3 -.->|Fetch Secrets| Secrets
```

### Infrastructure Components

**1. Compute (ECS Fargate)**
```yaml
# ecs-task-definition.json
{
  "family": "relyr-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "relyr-api",
      "image": "ECR_REPO_URL:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:relyr/db-url"
        },
        {
          "name": "ANCHOR_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:relyr/anchor-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/relyr-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

**2. Database (RDS PostgreSQL)**
```
Instance Type: db.t4g.medium (2 vCPUs, 4 GB RAM)
Storage: 100 GB SSD (gp3)
Multi-AZ: Yes (for production)
Backup: Automated daily backups, 7-day retention
Encryption: At rest and in transit
```

**3. Cache (ElastiCache Redis)**
```
Instance Type: cache.t4g.micro
Nodes: 1 (increase for production)
Engine: Redis 7.0
Encryption: In transit and at rest
```

**4. File Storage (S3)**
```
Buckets:
- relyr-kyc-documents (KYC uploads, encrypted)
- relyr-transaction-receipts (optional)
- relyr-backups (database backups)

Lifecycle Policies:
- KYC docs: Transition to Glacier after 90 days
- Backups: Delete after 30 days
```

### Deployment Strategy

**CI/CD Pipeline (GitHub Actions)**
```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: |
          npm ci
          npm test
          npm run test:e2e

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: relyr-api
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster relyr-cluster \
            --service relyr-api-service \
            --force-new-deployment
```

---

## Scalability Considerations

### Horizontal Scaling

**Auto-scaling Configuration:**
```typescript
// ECS Service Auto-scaling
{
  "MinCapacity": 2,
  "MaxCapacity": 10,
  "TargetValue": 70.0,  // CPU utilization target
  "ScaleInCooldown": 300,
  "ScaleOutCooldown": 60
}
```

**Database Scaling:**
- **Read Replicas:** Add 1-2 read replicas for read-heavy operations
- **Connection Pooling:** Limit connections per instance (max 100)
- **Vertical Scaling:** Upgrade instance type as needed

### Caching Strategy

**Redis Caching Layers:**
```typescript
// Cache configuration
const cacheStrategies = {
  // User session: 1 hour
  'user:session:{userId}': { ttl: 3600 },

  // Wallet balance: 30 seconds (frequently updated)
  'wallet:balance:{walletId}': { ttl: 30 },

  // FX rates: 1 minute (real-time data)
  'rates:usdc-ngn': { ttl: 60 },

  // User profile: 15 minutes
  'user:profile:{userId}': { ttl: 900 },

  // Transaction history: 5 minutes
  'transactions:list:{userId}': { ttl: 300 }
};
```

### Queue Management

**BullMQ Queue Strategy:**
```typescript
// Queue configuration
const queues = {
  // High priority: Instant execution
  'transactions.send': {
    priority: 1,
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 }
  },

  // Medium priority: Process within seconds
  'notifications.push': {
    priority: 5,
    attempts: 2,
    backoff: { type: 'fixed', delay: 5000 }
  },

  // Low priority: Can be delayed
  'analytics.track': {
    priority: 10,
    attempts: 1,
    backoff: { type: 'fixed', delay: 10000 }
  }
};
```

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| API Response Time (p95) | <500ms | CloudWatch Metrics |
| Transaction Processing | <2 min | End-to-end timing |
| App Startup Time | <3s | Mobile analytics |
| Database Query Time | <100ms | pg_stat_statements |
| Cache Hit Rate | >80% | Redis INFO stats |
| API Uptime | >99.9% | UptimeRobot |

---

## Appendix

### A. Environment Variables

```bash
# .env.example

# Application
NODE_ENV=production
PORT=3000
API_URL=https://api.relyr.app

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/relyr
DATABASE_SSL=true

# Redis
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=your-super-secret-key-change-this
JWT_EXPIRY=1h
JWT_REFRESH_EXPIRY=7d

# Anchor
ANCHOR_API_KEY=sk_live_xxx
ANCHOR_WEBHOOK_SECRET=whsec_xxx

# Circle
CIRCLE_API_KEY=xxx
CIRCLE_WEBHOOK_PUBLIC_KEY=xxx
CIRCLE_ENTITY_ID=xxx

# Paystack
PAYSTACK_SECRET_KEY=sk_live_xxx
PAYSTACK_PUBLIC_KEY=pk_live_xxx

# Mono
MONO_SECRET_KEY=live_sk_xxx
MONO_PUBLIC_KEY=live_pk_xxx

# Smile Identity
SMILE_PARTNER_ID=xxx
SMILE_API_KEY=xxx
SMILE_CALLBACK_URL=https://api.relyr.app/webhooks/smile

# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
S3_BUCKET_KYC=relyr-kyc-documents

# Sentry
SENTRY_DSN=https://xxx@sentry.io/xxx

# Email
SES_FROM_EMAIL=no-reply@relyr.app
SUPPORT_EMAIL=hi@relyr.app

# Encryption
MASTER_ENCRYPTION_KEY=your-master-key-for-encrypting-private-keys

# Rate Limiting
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX=100
```

### B. API Endpoint Reference

**Authentication:**
- `POST /auth/register` - User registration
- `POST /auth/login` - User login
- `POST /auth/refresh` - Refresh access token
- `POST /auth/logout` - Logout user

**Users:**
- `GET /users/me` - Get current user profile
- `PATCH /users/me` - Update profile
- `POST /users/me/pin` - Set transaction PIN
- `PATCH /users/me/pin` - Change transaction PIN

**KYC:**
- `POST /kyc/verify` - Submit KYC documents
- `GET /kyc/status` - Check KYC status
- `POST /kyc/link-bank` - Link bank account (Mono)
- `GET /kyc/bank-accounts` - List linked banks

**Wallets:**
- `POST /wallets` - Create NGN wallet
- `POST /wallets/usdc` - Create USDC wallet
- `GET /wallets` - List user wallets
- `GET /wallets/:id` - Get wallet details
- `GET /wallets/:id/balance` - Get wallet balance

**Transactions:**
- `POST /transactions/send` - Send money
- `POST /transactions/convert` - Convert currency
- `GET /transactions` - List transactions
- `GET /transactions/:id` - Get transaction details

**Payments:**
- `POST /payments/fund` - Fund wallet via Paystack
- `GET /payments/:reference/verify` - Verify payment

**Rates:**
- `GET /rates/usdc-ngn` - Get current USDC/NGN rate

**Webhooks:**
- `POST /webhooks/anchor` - Anchor webhooks
- `POST /webhooks/circle` - Circle webhooks
- `POST /webhooks/paystack` - Paystack webhooks
- `POST /webhooks/mono` - Mono webhooks

**Admin:**
- `GET /admin/users` - List all users
- `PATCH /admin/users/:id/kyc` - Update KYC status
- `GET /admin/transactions` - List all transactions
- `GET /admin/analytics` - Dashboard metrics

### C. Technology Decisions

| Decision | Options Considered | Choice | Rationale |
|----------|-------------------|--------|-----------|
| Mobile Framework | React Native, Flutter, Native | React Native + Expo | Faster development, web reuse potential, larger community |
| Backend Framework | Express, NestJS, Fastify | NestJS | Better structure, TypeScript native, scalable |
| Database | PostgreSQL, MongoDB, MySQL | PostgreSQL | ACID compliance, JSON support, mature ecosystem |
| Blockchain | Polygon, Solana, Ethereum | Polygon | Lower gas fees, Circle native USDC, EVM compatible |
| BaaS Provider | Anchor, Brass, OnePipe | Anchor | YC-backed, good API, regulatory support |
| Payment Gateway | Paystack, Flutterwave | Paystack | Better for Nigeria, Stripe-backed, lower fees |
| Open Banking | Mono, Okra | Mono | Okra shut down, Mono is market leader |
| KYC Provider | Smile Identity, Sumsub, Onfido | Smile Identity | Africa-focused, cheaper, good coverage |
| Cloud Provider | AWS, GCP, Azure | AWS | Mature, best services, good for startups |

---

**Document Maintainer:** Founding Team
**Next Review:** After Week 4 of development
**Feedback:** hi@relyr.app
