# API Integration Plan

**Version:** 1.0
**Last Updated:** November 2025
**Status:** Pre-Implementation

---

## Table of Contents

1. [Overview](#overview)
2. [Anchor BaaS Integration](#anchor-baas-integration)
3. [Circle USDC Integration](#circle-usdc-integration)
4. [Paystack Integration](#paystack-integration)
5. [Mono Open Banking Integration](#mono-open-banking-integration)
6. [Smile Identity Integration](#smile-identity-integration)
7. [Polygon Web3 Integration](#polygon-web3-integration)
8. [Integration Testing Strategy](#integration-testing-strategy)
9. [Error Handling & Retry Logic](#error-handling--retry-logic)
10. [Monitoring & Observability](#monitoring--observability)

---

## Overview

This document provides detailed integration specifications for all external APIs used in Relyr MVP. Each integration includes:

- **Setup & Authentication**
- **Core Endpoints & Usage**
- **Webhook Configuration**
- **Error Handling**
- **Testing Strategy**
- **Code Examples**

### Integration Priority

| Partner | Priority | Week | Critical Path |
|---------|----------|------|---------------|
| Anchor | 1 (Highest) | Week 3 | Yes - Core wallet functionality |
| Circle | 2 | Week 4 | Yes - USDC operations |
| Mono | 3 | Week 4 | Yes - KYC bank verification |
| Smile Identity | 4 | Week 4 | Yes - KYC identity verification |
| Paystack | 5 | Week 5 | No - Can defer to Week 6 |
| Polygon | 6 | Week 4 | Indirect - via Circle API |

---

## Anchor BaaS Integration

### Overview

**Purpose:** Virtual NGN bank accounts, local transfers, balance management
**Documentation:** https://docs.getanchor.co
**API Base URL:**
- Sandbox: `https://api.sandbox.getanchor.co/v1`
- Production: `https://api.getanchor.co/v1`

### Setup & Authentication

**1. Account Registration**
```bash
# Register at: https://getanchor.co/signup
# Navigate to: Dashboard → API Keys
# Generate: Sandbox API Key (for development)
```

**2. Authentication Method**
```typescript
// Bearer Token Authentication
headers: {
  'Authorization': 'Bearer YOUR_API_KEY',
  'Content-Type': 'application/json'
}
```

**3. Environment Variables**
```bash
ANCHOR_API_KEY=sk_sandbox_xxxxxxxxxxxxx
ANCHOR_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
ANCHOR_BASE_URL=https://api.sandbox.getanchor.co/v1
```

### Core Endpoints

#### 1. Create Virtual Account

**Endpoint:** `POST /accounts`

**Purpose:** Create a virtual NGN bank account for a user

**Request:**
```typescript
interface CreateAccountRequest {
  customer: {
    name: string;          // Full name
    email: string;         // Email address
    phone_number: string;  // Nigerian phone (+234...)
    identity: {
      type: 'BVN' | 'NIN';
      number: string;
    };
  };
  account_type: 'individual' | 'corporate';
  currency: 'NGN';
}

// Example
const request = {
  customer: {
    name: 'John Doe',
    email: 'john@example.com',
    phone_number: '+2348012345678',
    identity: {
      type: 'BVN',
      number: '22222222222'
    }
  },
  account_type: 'individual',
  currency: 'NGN'
};
```

**Response:**
```typescript
interface CreateAccountResponse {
  id: string;               // Anchor account ID
  account_number: string;   // Virtual account number (10 digits)
  account_name: string;     // Account name
  bank_name: string;        // Bank name (e.g., "Wema Bank")
  bank_code: string;        // Nigerian bank code
  currency: 'NGN';
  balance: number;          // Current balance (starts at 0)
  status: 'active' | 'inactive';
  created_at: string;       // ISO 8601 timestamp
}

// Example Response
{
  "id": "acc_1234567890",
  "account_number": "8012345678",
  "account_name": "John Doe",
  "bank_name": "Wema Bank",
  "bank_code": "035",
  "currency": "NGN",
  "balance": 0,
  "status": "active",
  "created_at": "2025-11-09T10:30:00Z"
}
```

**Implementation:**
```typescript
// src/integrations/anchor/anchor.service.ts

import { Injectable, HttpException } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AnchorService {
  private readonly client: AxiosInstance;

  constructor(private configService: ConfigService) {
    this.client = axios.create({
      baseURL: this.configService.get('ANCHOR_BASE_URL'),
      headers: {
        'Authorization': `Bearer ${this.configService.get('ANCHOR_API_KEY')}`,
        'Content-Type': 'application/json'
      },
      timeout: 30000 // 30 second timeout
    });

    // Add response interceptor for error handling
    this.client.interceptors.response.use(
      response => response,
      error => this.handleError(error)
    );
  }

  async createVirtualAccount(dto: CreateAccountDto): Promise<VirtualAccount> {
    try {
      const response = await this.client.post('/accounts', {
        customer: {
          name: `${dto.firstName} ${dto.lastName}`,
          email: dto.email,
          phone_number: dto.phone,
          identity: {
            type: 'BVN',
            number: dto.bvn
          }
        },
        account_type: 'individual',
        currency: 'NGN'
      });

      return {
        anchorId: response.data.id,
        accountNumber: response.data.account_number,
        accountName: response.data.account_name,
        bankName: response.data.bank_name,
        bankCode: response.data.bank_code,
        balance: response.data.balance
      };
    } catch (error) {
      throw new HttpException(
        `Failed to create Anchor account: ${error.message}`,
        error.status || 500
      );
    }
  }

  private handleError(error: any) {
    const status = error.response?.status;
    const message = error.response?.data?.message || error.message;

    console.error('Anchor API Error:', {
      status,
      message,
      data: error.response?.data
    });

    throw error;
  }
}
```

#### 2. Get Account Balance

**Endpoint:** `GET /accounts/:accountId/balance`

**Request:**
```typescript
// No request body, account ID in URL
GET /accounts/acc_1234567890/balance
```

**Response:**
```typescript
interface BalanceResponse {
  account_id: string;
  currency: 'NGN';
  available_balance: number;
  ledger_balance: number;
  updated_at: string;
}

// Example
{
  "account_id": "acc_1234567890",
  "currency": "NGN",
  "available_balance": 50000.00,
  "ledger_balance": 50000.00,
  "updated_at": "2025-11-09T14:20:00Z"
}
```

**Implementation:**
```typescript
async getBalance(anchorAccountId: string): Promise<number> {
  const response = await this.client.get(`/accounts/${anchorAccountId}/balance`);
  return response.data.available_balance;
}
```

#### 3. Transfer Funds

**Endpoint:** `POST /transfers`

**Purpose:** Send money from one Anchor account to another or to external bank account

**Request:**
```typescript
interface TransferRequest {
  source_account_id: string;
  destination: {
    type: 'anchor_account' | 'bank_account';
    account_number: string;
    bank_code?: string;      // Required if type = bank_account
    account_name?: string;   // Optional, for validation
  };
  amount: number;
  currency: 'NGN';
  narration: string;
  reference: string;         // Unique transaction reference
}

// Example
const transferRequest = {
  source_account_id: 'acc_1234567890',
  destination: {
    type: 'bank_account',
    account_number: '0123456789',
    bank_code: '058',
    account_name: 'Jane Smith'
  },
  amount: 5000,
  currency: 'NGN',
  narration: 'Payment for services',
  reference: 'TXN_20251109_001'
};
```

**Response:**
```typescript
interface TransferResponse {
  id: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  amount: number;
  fee: number;
  reference: string;
  created_at: string;
  completed_at?: string;
}
```

**Implementation:**
```typescript
async transfer(dto: TransferDto): Promise<Transfer> {
  try {
    const response = await this.client.post('/transfers', {
      source_account_id: dto.fromAnchorId,
      destination: {
        type: dto.destinationType,
        account_number: dto.toAccountNumber,
        bank_code: dto.bankCode,
        account_name: dto.accountName
      },
      amount: dto.amount,
      currency: 'NGN',
      narration: dto.narration || 'Relyr transfer',
      reference: dto.reference
    });

    return {
      transferId: response.data.id,
      status: response.data.status,
      amount: response.data.amount,
      fee: response.data.fee,
      reference: response.data.reference
    };
  } catch (error) {
    throw new HttpException(
      `Transfer failed: ${error.message}`,
      error.status || 500
    );
  }
}
```

#### 4. Get Transaction History

**Endpoint:** `GET /accounts/:accountId/transactions`

**Query Parameters:**
```typescript
interface TransactionQueryParams {
  start_date?: string;  // ISO 8601 date
  end_date?: string;
  limit?: number;       // Max 100
  offset?: number;
  type?: 'credit' | 'debit';
}
```

**Response:**
```typescript
interface TransactionHistoryResponse {
  transactions: Array<{
    id: string;
    type: 'credit' | 'debit';
    amount: number;
    balance_after: number;
    narration: string;
    reference: string;
    created_at: string;
  }>;
  pagination: {
    total: number;
    limit: number;
    offset: number;
  };
}
```

**Implementation:**
```typescript
async getTransactions(
  anchorAccountId: string,
  params: TransactionQueryParams
): Promise<Transaction[]> {
  const response = await this.client.get(
    `/accounts/${anchorAccountId}/transactions`,
    { params }
  );

  return response.data.transactions.map(tx => ({
    id: tx.id,
    type: tx.type,
    amount: tx.amount,
    balanceAfter: tx.balance_after,
    narration: tx.narration,
    reference: tx.reference,
    createdAt: new Date(tx.created_at)
  }));
}
```

### Webhook Configuration

**Webhook Events:**
1. `account.credited` - Funds received
2. `transfer.completed` - Transfer successful
3. `transfer.failed` - Transfer failed

**Setup:**
```bash
# In Anchor Dashboard:
# Settings → Webhooks → Add Endpoint
# URL: https://api.relyr.app/webhooks/anchor
# Events: Select all
```

**Webhook Handler:**
```typescript
// src/webhooks/handlers/anchor.handler.ts

@Injectable()
export class AnchorWebhookHandler {
  constructor(
    private readonly transactionService: TransactionService,
    private readonly walletService: WalletService,
    private readonly notificationService: NotificationService
  ) {}

  async handleWebhook(payload: any, signature: string): Promise<void> {
    // 1. Verify webhook signature
    if (!this.verifySignature(payload, signature)) {
      throw new UnauthorizedException('Invalid webhook signature');
    }

    const event = payload.event;
    const data = payload.data;

    // 2. Handle event based on type
    switch (event) {
      case 'account.credited':
        await this.handleAccountCredited(data);
        break;

      case 'transfer.completed':
        await this.handleTransferCompleted(data);
        break;

      case 'transfer.failed':
        await this.handleTransferFailed(data);
        break;

      default:
        console.log(`Unhandled Anchor webhook event: ${event}`);
    }
  }

  private verifySignature(payload: any, signature: string): boolean {
    const secret = process.env.ANCHOR_WEBHOOK_SECRET;
    const hash = crypto
      .createHmac('sha256', secret)
      .update(JSON.stringify(payload))
      .digest('hex');

    return hash === signature;
  }

  private async handleAccountCredited(data: any): Promise<void> {
    const { account_id, amount, reference } = data;

    // Find wallet by Anchor account ID
    const wallet = await this.walletService.findByExternalId(account_id);
    if (!wallet) {
      console.error(`Wallet not found for Anchor account: ${account_id}`);
      return;
    }

    // Create transaction record
    await this.transactionService.create({
      toWalletId: wallet.id,
      type: 'RECEIVE',
      amount,
      currency: 'NGN',
      status: 'COMPLETED',
      reference,
      externalReference: data.transaction_id
    });

    // Update wallet balance
    await this.walletService.updateBalance(wallet.id, amount);

    // Send notification
    await this.notificationService.sendPush(wallet.userId, {
      title: 'Money Received',
      body: `You received ₦${amount.toLocaleString()} in your NGN wallet`
    });
  }

  private async handleTransferCompleted(data: any): Promise<void> {
    const { reference } = data;

    // Update transaction status
    await this.transactionService.updateByReference(reference, {
      status: 'COMPLETED',
      completedAt: new Date()
    });
  }

  private async handleTransferFailed(data: any): Promise<void> {
    const { reference, reason } = data;

    // Update transaction status and reverse balance
    const transaction = await this.transactionService.findByReference(reference);

    await this.transactionService.updateByReference(reference, {
      status: 'FAILED',
      metadata: { failureReason: reason }
    });

    // Refund the sender
    await this.walletService.updateBalance(
      transaction.fromWalletId,
      transaction.amount // Add back to balance
    );
  }
}
```

### Error Handling

**Common Errors:**

| Status | Error Code | Meaning | Action |
|--------|-----------|---------|--------|
| 400 | invalid_request | Missing/invalid parameters | Check request body |
| 401 | unauthorized | Invalid API key | Verify API key |
| 403 | forbidden | Insufficient permissions | Check account status |
| 404 | not_found | Account/resource not found | Verify IDs |
| 409 | duplicate_reference | Reference already used | Generate new reference |
| 429 | rate_limit_exceeded | Too many requests | Implement exponential backoff |
| 500 | server_error | Anchor server error | Retry with backoff |

**Retry Strategy:**
```typescript
async transferWithRetry(dto: TransferDto, maxRetries = 3): Promise<Transfer> {
  let lastError: any;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await this.transfer(dto);
    } catch (error) {
      lastError = error;
      const status = error.response?.status;

      // Don't retry client errors (4xx except 429)
      if (status >= 400 && status < 500 && status !== 429) {
        throw error;
      }

      // Exponential backoff: 2^attempt seconds
      const delayMs = Math.pow(2, attempt) * 1000;
      console.log(`Anchor transfer attempt ${attempt} failed, retrying in ${delayMs}ms...`);
      await new Promise(resolve => setTimeout(resolve, delayMs));
    }
  }

  throw lastError;
}
```

### Testing Strategy

**1. Sandbox Testing**
```typescript
// test/integration/anchor.service.spec.ts

describe('AnchorService', () => {
  let service: AnchorService;

  beforeEach(() => {
    // Use sandbox API key
    process.env.ANCHOR_API_KEY = 'sk_sandbox_test';
    service = new AnchorService(configService);
  });

  it('should create virtual account', async () => {
    const result = await service.createVirtualAccount({
      firstName: 'Test',
      lastName: 'User',
      email: 'test@example.com',
      phone: '+2348012345678',
      bvn: '22222222222'
    });

    expect(result.accountNumber).toHaveLength(10);
    expect(result.bankName).toBeDefined();
  });

  it('should get account balance', async () => {
    const balance = await service.getBalance('acc_sandbox_123');
    expect(typeof balance).toBe('number');
  });

  it('should handle transfer', async () => {
    const result = await service.transfer({
      fromAnchorId: 'acc_sandbox_123',
      toAccountNumber: '0123456789',
      bankCode: '058',
      amount: 1000,
      reference: `TEST_${Date.now()}`
    });

    expect(result.status).toBe('pending');
  });
});
```

**2. Mock Data for Development**
```typescript
// Create mock Anchor service for local development
export class MockAnchorService implements AnchorService {
  async createVirtualAccount(dto: any): Promise<VirtualAccount> {
    return {
      anchorId: `acc_mock_${Date.now()}`,
      accountNumber: Math.random().toString().slice(2, 12),
      accountName: `${dto.firstName} ${dto.lastName}`,
      bankName: 'Mock Bank',
      bankCode: '999',
      balance: 0
    };
  }

  async getBalance(id: string): Promise<number> {
    return Math.random() * 100000;
  }
}
```

---

## Circle USDC Integration

### Overview

**Purpose:** USDC wallet creation, blockchain transfers, payment tracking
**Documentation:** https://developers.circle.com
**API Base URL:**
- Sandbox: `https://api-sandbox.circle.com/v1`
- Production: `https://api.circle.com/v1`

### Setup & Authentication

**1. Account Setup**
```bash
# Register at: https://console.circle.com
# Create App: Dashboard → Apps → Create App
# Get API Key: API Keys → Generate API Key
```

**2. Authentication**
```typescript
headers: {
  'Authorization': 'Bearer YOUR_API_KEY',
  'Content-Type': 'application/json'
}
```

**3. Environment Variables**
```bash
CIRCLE_API_KEY=SAND_API_KEY:xxxxxxxxxxxxxxxx
CIRCLE_ENTITY_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
CIRCLE_WEBHOOK_PUBLIC_KEY="-----BEGIN PGP PUBLIC KEY BLOCK-----..."
CIRCLE_BASE_URL=https://api-sandbox.circle.com/v1
```

### Core Endpoints

#### 1. Create Wallet

**Endpoint:** `POST /wallets`

**Request:**
```typescript
interface CreateWalletRequest {
  idempotencyKey: string;  // UUID for idempotency
  blockchains: ['MATIC'];  // Polygon network
}

const request = {
  idempotencyKey: crypto.randomUUID(),
  blockchains: ['MATIC']
};
```

**Response:**
```typescript
interface CreateWalletResponse {
  data: {
    walletId: string;
    entityId: string;
    type: 'end_user_wallet';
    description: string;
    balances: Array<{
      amount: string;
      currency: 'USD';
    }>;
  };
}
```

**Implementation:**
```typescript
// src/integrations/circle/circle.service.ts

@Injectable()
export class CircleService {
  private readonly client: AxiosInstance;

  constructor(private configService: ConfigService) {
    this.client = axios.create({
      baseURL: this.configService.get('CIRCLE_BASE_URL'),
      headers: {
        'Authorization': `Bearer ${this.configService.get('CIRCLE_API_KEY')}`,
        'Content-Type': 'application/json'
      },
      timeout: 30000
    });
  }

  async createWallet(): Promise<CircleWallet> {
    try {
      const response = await this.client.post('/wallets', {
        idempotencyKey: crypto.randomUUID(),
        blockchains: ['MATIC']  // Polygon
      });

      const wallet = response.data.data;

      return {
        walletId: wallet.walletId,
        entityId: wallet.entityId,
        balances: wallet.balances
      };
    } catch (error) {
      throw new HttpException(
        `Failed to create Circle wallet: ${error.message}`,
        error.status || 500
      );
    }
  }
}
```

#### 2. Get Wallet Address

**Endpoint:** `POST /wallets/:walletId/addresses`

**Request:**
```typescript
interface GenerateAddressRequest {
  idempotencyKey: string;
  currency: 'USD';
  chain: 'MATIC';  // Polygon
}

const request = {
  idempotencyKey: crypto.randomUUID(),
  currency: 'USD',
  chain: 'MATIC'
};
```

**Response:**
```typescript
interface GenerateAddressResponse {
  data: {
    address: string;        // Polygon address (0x...)
    currency: 'USD';
    chain: 'MATIC';
  };
}

// Example
{
  "data": {
    "address": "0x1234567890abcdef1234567890abcdef12345678",
    "currency": "USD",
    "chain": "MATIC"
  }
}
```

**Implementation:**
```typescript
async generateAddress(walletId: string): Promise<string> {
  const response = await this.client.post(`/wallets/${walletId}/addresses`, {
    idempotencyKey: crypto.randomUUID(),
    currency: 'USD',
    chain: 'MATIC'
  });

  return response.data.data.address;
}
```

#### 3. Get Wallet Balance

**Endpoint:** `GET /wallets/:walletId`

**Response:**
```typescript
interface WalletBalanceResponse {
  data: {
    walletId: string;
    balances: Array<{
      amount: string;       // e.g., "100.50"
      currency: 'USD';
    }>;
  };
}
```

**Implementation:**
```typescript
async getBalance(walletId: string): Promise<number> {
  const response = await this.client.get(`/wallets/${walletId}`);
  const usdBalance = response.data.data.balances.find(
    b => b.currency === 'USD'
  );
  return parseFloat(usdBalance?.amount || '0');
}
```

#### 4. Transfer USDC

**Endpoint:** `POST /transfers`

**Request:**
```typescript
interface TransferRequest {
  idempotencyKey: string;
  source: {
    type: 'wallet';
    id: string;           // Source wallet ID
  };
  destination: {
    type: 'blockchain';
    address: string;      // Recipient Polygon address
    chain: 'MATIC';
  };
  amount: {
    amount: string;       // e.g., "10.50"
    currency: 'USD';
  };
}

const transferRequest = {
  idempotencyKey: crypto.randomUUID(),
  source: {
    type: 'wallet',
    id: 'wallet-id-123'
  },
  destination: {
    type: 'blockchain',
    address: '0xRecipientAddress...',
    chain: 'MATIC'
  },
  amount: {
    amount: '10.00',
    currency: 'USD'
  }
};
```

**Response:**
```typescript
interface TransferResponse {
  data: {
    id: string;           // Transfer ID
    source: { type: string; id: string; };
    destination: { type: string; address: string; chain: string; };
    amount: { amount: string; currency: string; };
    status: 'pending' | 'complete' | 'failed';
    transactionHash?: string;
    createDate: string;
  };
}
```

**Implementation:**
```typescript
async transfer(dto: CircleTransferDto): Promise<Transfer> {
  try {
    const response = await this.client.post('/transfers', {
      idempotencyKey: crypto.randomUUID(),
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
        amount: dto.amount.toFixed(2),
        currency: 'USD'
      }
    });

    const transfer = response.data.data;

    return {
      transferId: transfer.id,
      status: transfer.status,
      amount: parseFloat(transfer.amount.amount),
      transactionHash: transfer.transactionHash,
      createdAt: new Date(transfer.createDate)
    };
  } catch (error) {
    throw new HttpException(
      `Circle transfer failed: ${error.message}`,
      error.status || 500
    );
  }
}
```

#### 5. Get Transfer Status

**Endpoint:** `GET /transfers/:transferId`

**Response:**
```typescript
interface TransferStatusResponse {
  data: {
    id: string;
    status: 'pending' | 'complete' | 'failed';
    errorCode?: string;
    transactionHash?: string;
  };
}
```

**Implementation:**
```typescript
async getTransferStatus(transferId: string): Promise<TransferStatus> {
  const response = await this.client.get(`/transfers/${transferId}`);
  const transfer = response.data.data;

  return {
    id: transfer.id,
    status: transfer.status,
    transactionHash: transfer.transactionHash,
    errorCode: transfer.errorCode
  };
}
```

### Webhook Configuration

**Webhook Events:**
1. `transfers.created` - Transfer initiated
2. `transfers.completed` - Transfer confirmed on blockchain
3. `transfers.failed` - Transfer failed

**Setup:**
```bash
# In Circle Console:
# Webhooks → Create Subscription
# URL: https://api.relyr.app/webhooks/circle
# Events: Select all transfer events
```

**Webhook Handler:**
```typescript
// src/webhooks/handlers/circle.handler.ts

@Injectable()
export class CircleWebhookHandler {
  constructor(
    private readonly transactionService: TransactionService,
    private readonly walletService: WalletService
  ) {}

  async handleWebhook(payload: any, signature: string): Promise<void> {
    // Verify signature using Circle's PGP public key
    if (!this.verifySignature(payload, signature)) {
      throw new UnauthorizedException('Invalid signature');
    }

    const eventType = payload.Type;
    const notification = payload.Message;

    switch (eventType) {
      case 'transfers.created':
        await this.handleTransferCreated(notification);
        break;

      case 'transfers.completed':
        await this.handleTransferCompleted(notification);
        break;

      case 'transfers.failed':
        await this.handleTransferFailed(notification);
        break;
    }
  }

  private verifySignature(payload: string, signature: string): boolean {
    const publicKey = process.env.CIRCLE_WEBHOOK_PUBLIC_KEY;
    // Use PGP library to verify signature
    // Implementation depends on Circle's signature scheme
    return true; // Simplified for example
  }

  private async handleTransferCompleted(data: any): Promise<void> {
    const { id: transferId, transactionHash, destination } = data;

    // Find wallet by address
    const wallet = await this.walletService.findByAddress(destination.address);
    if (!wallet) return;

    // Update transaction with tx hash
    await this.transactionService.updateByExternalRef(transferId, {
      status: 'COMPLETED',
      metadata: { transactionHash },
      completedAt: new Date()
    });

    // Update wallet balance
    const amount = parseFloat(data.amount.amount);
    await this.walletService.updateBalance(wallet.id, amount);
  }
}
```

### Testing Strategy

```typescript
describe('CircleService', () => {
  it('should create wallet', async () => {
    const wallet = await service.createWallet();
    expect(wallet.walletId).toBeDefined();
  });

  it('should generate address', async () => {
    const address = await service.generateAddress('wallet-id');
    expect(address).toMatch(/^0x[a-fA-F0-9]{40}$/);
  });

  it('should transfer USDC', async () => {
    const transfer = await service.transfer({
      fromWalletId: 'wallet-123',
      toAddress: '0x...',
      amount: 10
    });
    expect(transfer.status).toBe('pending');
  });
});
```

---

## Paystack Integration

### Overview

**Purpose:** Card payments, bank transfers for funding wallets
**Documentation:** https://paystack.com/docs
**API Base URL:** `https://api.paystack.co`

### Setup & Authentication

**1. Account Setup**
```bash
# Register at: https://paystack.com/signup
# Verify business: Complete verification process
# Get API Keys: Settings → API Keys & Webhooks
```

**2. Authentication**
```typescript
headers: {
  'Authorization': 'Bearer YOUR_SECRET_KEY',
  'Content-Type': 'application/json'
}
```

**3. Environment Variables**
```bash
PAYSTACK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxx
PAYSTACK_PUBLIC_KEY=pk_test_xxxxxxxxxxxxxxxx
PAYSTACK_CALLBACK_URL=https://api.relyr.app/payments/callback
```

### Core Endpoints

#### 1. Initialize Payment

**Endpoint:** `POST /transaction/initialize`

**Request:**
```typescript
interface InitializePaymentRequest {
  email: string;
  amount: number;        // In kobo (100 = ₦1)
  currency: 'NGN';
  reference?: string;    // Optional unique reference
  callback_url?: string;
  metadata?: object;     // Custom data
  channels?: string[];   // ['card', 'bank', 'ussd']
}

const request = {
  email: 'user@example.com',
  amount: 500000,        // ₦5,000 in kobo
  currency: 'NGN',
  reference: `PAY_${Date.now()}`,
  callback_url: 'https://app.relyr.app/payment/success',
  metadata: {
    user_id: 'user-123',
    wallet_id: 'wallet-456'
  },
  channels: ['card', 'bank']
};
```

**Response:**
```typescript
interface InitializePaymentResponse {
  status: boolean;
  message: string;
  data: {
    authorization_url: string;  // Redirect user here
    access_code: string;
    reference: string;
  };
}

// Example
{
  "status": true,
  "message": "Authorization URL created",
  "data": {
    "authorization_url": "https://checkout.paystack.com/xxxxxxxxx",
    "access_code": "xxxxxxxxx",
    "reference": "PAY_1699520400000"
  }
}
```

**Implementation:**
```typescript
// src/integrations/paystack/paystack.service.ts

@Injectable()
export class PaystackService {
  private readonly client: AxiosInstance;

  constructor(private configService: ConfigService) {
    this.client = axios.create({
      baseURL: 'https://api.paystack.co',
      headers: {
        'Authorization': `Bearer ${this.configService.get('PAYSTACK_SECRET_KEY')}`,
        'Content-Type': 'application/json'
      }
    });
  }

  async initializePayment(dto: InitializePaymentDto): Promise<PaymentInit> {
    try {
      const response = await this.client.post('/transaction/initialize', {
        email: dto.email,
        amount: dto.amount * 100, // Convert to kobo
        currency: 'NGN',
        reference: dto.reference || `PAY_${Date.now()}`,
        callback_url: process.env.PAYSTACK_CALLBACK_URL,
        metadata: {
          user_id: dto.userId,
          wallet_id: dto.walletId,
          type: 'wallet_funding'
        },
        channels: ['card', 'bank', 'ussd']
      });

      const data = response.data.data;

      return {
        authorizationUrl: data.authorization_url,
        accessCode: data.access_code,
        reference: data.reference
      };
    } catch (error) {
      throw new HttpException(
        `Failed to initialize payment: ${error.message}`,
        error.status || 500
      );
    }
  }
}
```

#### 2. Verify Payment

**Endpoint:** `GET /transaction/verify/:reference`

**Response:**
```typescript
interface VerifyPaymentResponse {
  status: boolean;
  message: string;
  data: {
    id: number;
    reference: string;
    amount: number;          // In kobo
    currency: 'NGN';
    status: 'success' | 'failed' | 'abandoned';
    paid_at: string;
    channel: 'card' | 'bank' | 'ussd';
    fees: number;
    customer: {
      email: string;
    };
    metadata: object;
  };
}
```

**Implementation:**
```typescript
async verifyPayment(reference: string): Promise<PaymentVerification> {
  const response = await this.client.get(`/transaction/verify/${reference}`);
  const data = response.data.data;

  return {
    reference: data.reference,
    amount: data.amount / 100, // Convert from kobo to naira
    status: data.status,
    paidAt: data.paid_at ? new Date(data.paid_at) : null,
    channel: data.channel,
    fees: data.fees / 100,
    metadata: data.metadata
  };
}
```

### Webhook Configuration

**Webhook Events:**
1. `charge.success` - Payment successful
2. `charge.failed` - Payment failed
3. `transfer.success` - Payout successful
4. `transfer.failed` - Payout failed

**Setup:**
```bash
# In Paystack Dashboard:
# Settings → API Keys & Webhooks → Add Webhook URL
# URL: https://api.relyr.app/webhooks/paystack
```

**Webhook Handler:**
```typescript
@Injectable()
export class PaystackWebhookHandler {
  async handleWebhook(payload: any, signature: string): Promise<void> {
    // Verify signature
    if (!this.verifySignature(payload, signature)) {
      throw new UnauthorizedException('Invalid signature');
    }

    const event = payload.event;
    const data = payload.data;

    switch (event) {
      case 'charge.success':
        await this.handleChargeSuccess(data);
        break;

      case 'charge.failed':
        await this.handleChargeFailed(data);
        break;
    }
  }

  private verifySignature(payload: string, signature: string): boolean {
    const secret = process.env.PAYSTACK_SECRET_KEY;
    const hash = crypto
      .createHmac('sha512', secret)
      .update(payload)
      .digest('hex');

    return hash === signature;
  }

  private async handleChargeSuccess(data: any): Promise<void> {
    const { reference, amount, metadata } = data;

    // Credit user's wallet
    await this.walletService.updateBalance(
      metadata.wallet_id,
      amount / 100 // Convert from kobo
    );

    // Update transaction status
    await this.transactionService.updateByReference(reference, {
      status: 'COMPLETED',
      completedAt: new Date()
    });
  }
}
```

### Testing

```typescript
describe('PaystackService', () => {
  it('should initialize payment', async () => {
    const result = await service.initializePayment({
      email: 'test@example.com',
      amount: 5000,
      userId: 'user-123',
      walletId: 'wallet-456'
    });

    expect(result.authorizationUrl).toContain('paystack.com');
    expect(result.reference).toBeDefined();
  });

  it('should verify payment', async () => {
    const verification = await service.verifyPayment('PAY_123');
    expect(verification.status).toBe('success');
  });
});
```

---

## Mono Open Banking Integration

### Overview

**Purpose:** Bank account verification, transaction history, identity verification
**Documentation:** https://docs.mono.co
**API Base URL:** `https://api.withmono.com`

### Setup & Authentication

**1. Account Setup**
```bash
# Register at: https://app.withmono.com/signup
# Create App: Dashboard → Apps → Create New App
# Get Keys: Settings → Keys
```

**2. Authentication**
```typescript
headers: {
  'mono-sec-key': 'YOUR_SECRET_KEY',
  'Content-Type': 'application/json'
}
```

**3. Environment Variables**
```bash
MONO_SECRET_KEY=live_sk_xxxxxxxxxxxxxxxx
MONO_PUBLIC_KEY=live_pk_xxxxxxxxxxxxxxxx
```

### Core Endpoints

#### 1. Get Account Details

**Endpoint:** `GET /accounts/:accountId`

**Purpose:** Get linked bank account information after user connects via Mono Connect widget

**Response:**
```typescript
interface AccountDetailsResponse {
  account: {
    _id: string;
    name: string;              // Account holder name
    accountNumber: string;
    type: 'savings' | 'current';
    balance: number;
    currency: 'NGN';
    bvn: string;
    institution: {
      name: string;            // e.g., "Access Bank"
      bankCode: string;
      type: 'bank';
    };
  };
}
```

**Implementation:**
```typescript
// src/integrations/mono/mono.service.ts

@Injectable()
export class MonoService {
  private readonly client: AxiosInstance;

  constructor(private configService: ConfigService) {
    this.client = axios.create({
      baseURL: 'https://api.withmono.com',
      headers: {
        'mono-sec-key': this.configService.get('MONO_SECRET_KEY')
      }
    });
  }

  async getAccountDetails(accountId: string): Promise<BankAccount> {
    try {
      const response = await this.client.get(`/accounts/${accountId}`);
      const account = response.data.account;

      return {
        monoAccountId: account._id,
        accountNumber: account.accountNumber,
        accountName: account.name,
        accountType: account.type,
        balance: account.balance,
        bankName: account.institution.name,
        bankCode: account.institution.bankCode,
        bvn: account.bvn
      };
    } catch (error) {
      throw new HttpException(
        `Failed to get Mono account: ${error.message}`,
        error.status || 500
      );
    }
  }
}
```

#### 2. Get Account Identity

**Endpoint:** `GET /accounts/:accountId/identity`

**Purpose:** Get verified identity information for KYC

**Response:**
```typescript
interface AccountIdentityResponse {
  identity: {
    fullName: string;
    email: string;
    phone: string;
    gender: 'male' | 'female';
    dob: string;           // Date of birth
    bvn: string;
    addressLine1: string;
    addressLine2: string;
    city: string;
    state: string;
    country: string;
  };
}
```

**Implementation:**
```typescript
async getAccountIdentity(accountId: string): Promise<Identity> {
  const response = await this.client.get(`/accounts/${accountId}/identity`);
  const identity = response.data.identity;

  return {
    fullName: identity.fullName,
    email: identity.email,
    phone: identity.phone,
    gender: identity.gender,
    dateOfBirth: new Date(identity.dob),
    bvn: identity.bvn,
    address: {
      line1: identity.addressLine1,
      line2: identity.addressLine2,
      city: identity.city,
      state: identity.state,
      country: identity.country
    }
  };
}
```

#### 3. Unlink Account

**Endpoint:** `POST /accounts/:accountId/unlink`

**Purpose:** Remove account connection

**Implementation:**
```typescript
async unlinkAccount(accountId: string): Promise<void> {
  await this.client.post(`/accounts/${accountId}/unlink`);
}
```

### Mono Connect Widget (Frontend)

**Mobile Integration (React Native):**

```typescript
// Install: npm install react-native-mono

import { useMonoConnect } from 'react-native-mono';

function LinkBankScreen() {
  const monoConnect = useMonoConnect({
    publicKey: process.env.MONO_PUBLIC_KEY,
    onSuccess: async (code) => {
      // Send code to backend to exchange for account ID
      const response = await api.post('/kyc/link-bank', { code });
      console.log('Bank linked:', response.data);
    },
    onClose: () => {
      console.log('Widget closed');
    }
  });

  return (
    <Button
      title="Link Bank Account"
      onPress={() => monoConnect.open()}
    />
  );
}
```

**Backend Exchange Code:**
```typescript
// Exchange authorization code for account ID
async exchangeCode(code: string): Promise<string> {
  const response = await this.client.post('/account/auth', {
    code
  });

  return response.data.id; // Account ID to store in database
}
```

### Webhook Configuration

**Webhook Events:**
1. `mono.events.account_linked` - Account successfully linked
2. `mono.events.account_updated` - Account data updated
3. `mono.events.account_reauthorization` - Requires re-auth

**Webhook Handler:**
```typescript
@Injectable()
export class MonoWebhookHandler {
  async handleWebhook(payload: any): Promise<void> {
    const event = payload.event;
    const data = payload.data;

    switch (event) {
      case 'mono.events.account_linked':
        await this.handleAccountLinked(data);
        break;

      case 'mono.events.account_reauthorization':
        await this.handleReauthorization(data);
        break;
    }
  }

  private async handleAccountLinked(data: any): Promise<void> {
    const { account: accountId } = data;

    // Fetch account details
    const account = await this.monoService.getAccountDetails(accountId);

    // Store in database
    await this.bankAccountService.create({
      monoAccountId: accountId,
      accountNumber: account.accountNumber,
      accountName: account.accountName,
      bankName: account.bankName,
      isVerified: true
    });
  }
}
```

---

## Smile Identity Integration

### Overview

**Purpose:** KYC verification (ID + Selfie + Liveness)
**Documentation:** https://docs.usesmileidentity.com
**API Base URL:** `https://api.smileidentity.com/v1`

### Setup & Authentication

**1. Account Setup**
```bash
# Register at: https://portal.usesmileidentity.com/signup
# Create Partner: Dashboard → Create Partner
# Get Credentials: Settings → API Keys
```

**2. Authentication**
Smile uses API Key + Partner ID

**3. Environment Variables**
```bash
SMILE_PARTNER_ID=1234
SMILE_API_KEY=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
SMILE_CALLBACK_URL=https://api.relyr.app/webhooks/smile
```

### Core Endpoints

#### 1. Submit ID Verification

**Endpoint:** `POST /upload`

**Request:**
```typescript
interface IDVerificationRequest {
  partner_id: string;
  partner_params: {
    user_id: string;
    job_id: string;
    job_type: number;      // 6 = Enhanced KYC
  };
  id_info: {
    country: 'NG';
    id_type: 'BVN' | 'NIN' | 'DRIVERS_LICENSE' | 'PASSPORT';
    id_number: string;
    first_name?: string;
    last_name?: string;
    dob?: string;
  };
  images: Array<{
    image_type_id: number;  // 0 = selfie, 1 = ID front, 2 = ID back
    image: string;          // Base64 encoded
  }>;
  timestamp: string;
  signature: string;        // HMAC signature
}

const request = {
  partner_id: process.env.SMILE_PARTNER_ID,
  partner_params: {
    user_id: 'user-123',
    job_id: crypto.randomUUID(),
    job_type: 6  // Enhanced KYC
  },
  id_info: {
    country: 'NG',
    id_type: 'BVN',
    id_number: '22222222222',
    first_name: 'John',
    last_name: 'Doe',
    dob: '1990-01-01'
  },
  images: [
    {
      image_type_id: 0,
      image: 'base64_selfie_data...'
    },
    {
      image_type_id: 1,
      image: 'base64_id_front_data...'
    }
  ],
  timestamp: new Date().toISOString(),
  signature: 'calculated_hmac_signature'
};
```

**Response:**
```typescript
interface IDVerificationResponse {
  success: boolean;
  smile_job_id: string;
  timestamp: string;
  signature: string;
  result: {
    ResultCode: '0000' | '1220' | '1221';  // 0000 = success
    ResultText: string;
    ConfidenceValue: string;  // e.g., "99.87"
    FullName: string;
    DOB: string;
    IDNumber: string;
    Photo: string;            // Base64 photo from ID database
  };
}
```

**Implementation:**
```typescript
// src/integrations/smile/smile.service.ts

@Injectable()
export class SmileIdentityService {
  private readonly client: AxiosInstance;
  private readonly partnerId: string;
  private readonly apiKey: string;

  constructor(private configService: ConfigService) {
    this.partnerId = this.configService.get('SMILE_PARTNER_ID');
    this.apiKey = this.configService.get('SMILE_API_KEY');

    this.client = axios.create({
      baseURL: 'https://api.smileidentity.com/v1',
      timeout: 60000  // 60 seconds for image processing
    });
  }

  async verifyIdentity(dto: KYCVerificationDto): Promise<VerificationResult> {
    const jobId = crypto.randomUUID();
    const timestamp = new Date().toISOString();

    // Calculate signature
    const signature = this.calculateSignature(timestamp, dto.userId, jobId);

    try {
      const response = await this.client.post('/upload', {
        partner_id: this.partnerId,
        partner_params: {
          user_id: dto.userId,
          job_id: jobId,
          job_type: 6  // Enhanced KYC
        },
        id_info: {
          country: 'NG',
          id_type: dto.idType,  // BVN, NIN, etc.
          id_number: dto.idNumber,
          first_name: dto.firstName,
          last_name: dto.lastName,
          dob: dto.dateOfBirth
        },
        images: [
          {
            image_type_id: 0,  // Selfie
            image: dto.selfieBase64
          },
          {
            image_type_id: 1,  // ID front
            image: dto.idFrontBase64
          }
        ],
        timestamp,
        signature
      });

      const result = response.data.result;

      return {
        jobId: response.data.smile_job_id,
        resultCode: result.ResultCode,
        resultText: result.ResultText,
        confidence: parseFloat(result.ConfidenceValue),
        verified: result.ResultCode === '0000',
        fullName: result.FullName,
        dateOfBirth: result.DOB,
        idNumber: result.IDNumber
      };
    } catch (error) {
      throw new HttpException(
        `Smile ID verification failed: ${error.message}`,
        error.status || 500
      );
    }
  }

  private calculateSignature(timestamp: string, userId: string, jobId: string): string {
    const data = `${timestamp}${this.partnerId}${userId}${jobId}`;
    return crypto
      .createHmac('sha256', this.apiKey)
      .update(data)
      .digest('hex');
  }
}
```

#### 2. Get Job Status

**Endpoint:** `POST /job_status`

**Request:**
```typescript
interface JobStatusRequest {
  partner_id: string;
  user_id: string;
  job_id: string;
  timestamp: string;
  signature: string;
}
```

**Implementation:**
```typescript
async getJobStatus(userId: string, jobId: string): Promise<JobStatus> {
  const timestamp = new Date().toISOString();
  const signature = this.calculateSignature(timestamp, userId, jobId);

  const response = await this.client.post('/job_status', {
    partner_id: this.partnerId,
    user_id: userId,
    job_id: jobId,
    timestamp,
    signature
  });

  return {
    jobId: response.data.job_id,
    status: response.data.job_status,  // 'complete', 'processing', 'failed'
    result: response.data.result
  };
}
```

### Mobile Integration

**Capture Selfie with Liveness:**
```typescript
// Use Smile Identity React Native SDK
import { SmileID } from '@smile_identity/react-native';

async function captureSelfie() {
  const result = await SmileID.captureSelfie({
    livenessCheck: true,  // Enable liveness detection
    instructions: true    // Show user instructions
  });

  return result.image;  // Base64 image
}
```

### Testing

```typescript
describe('SmileIdentityService', () => {
  it('should verify identity with BVN', async () => {
    const result = await service.verifyIdentity({
      userId: 'user-123',
      idType: 'BVN',
      idNumber: '22222222222',
      firstName: 'John',
      lastName: 'Doe',
      dateOfBirth: '1990-01-01',
      selfieBase64: 'data:image/jpeg;base64,...',
      idFrontBase64: 'data:image/jpeg;base64,...'
    });

    expect(result.verified).toBe(true);
    expect(result.confidence).toBeGreaterThan(90);
  });
});
```

---

## Polygon Web3 Integration

### Overview

**Purpose:** Direct blockchain interaction (fallback to Circle API)
**Library:** ethers.js or web3.js
**Network:** Polygon Mainnet / Mumbai Testnet

### Setup

**1. Install Dependencies**
```bash
npm install ethers @circle-fin/circle-sdk
```

**2. Environment Variables**
```bash
POLYGON_RPC_URL=https://polygon-rpc.com
POLYGON_TESTNET_RPC_URL=https://rpc-mumbai.maticvigil.com
USDC_CONTRACT_ADDRESS=0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174  # Polygon Mainnet
```

### Implementation

```typescript
// src/integrations/polygon/polygon.service.ts

import { Injectable } from '@nestjs/common';
import { ethers } from 'ethers';

@Injectable()
export class PolygonService {
  private readonly provider: ethers.providers.JsonRpcProvider;
  private readonly usdcContract: ethers.Contract;

  constructor(private configService: ConfigService) {
    this.provider = new ethers.providers.JsonRpcProvider(
      this.configService.get('POLYGON_RPC_URL')
    );

    // USDC contract ABI (simplified)
    const usdcAbi = [
      'function balanceOf(address owner) view returns (uint256)',
      'function transfer(address to, uint256 amount) returns (bool)',
      'event Transfer(address indexed from, address indexed to, uint256 value)'
    ];

    this.usdcContract = new ethers.Contract(
      this.configService.get('USDC_CONTRACT_ADDRESS'),
      usdcAbi,
      this.provider
    );
  }

  async getUSDCBalance(address: string): Promise<number> {
    const balance = await this.usdcContract.balanceOf(address);
    // USDC has 6 decimals
    return parseFloat(ethers.utils.formatUnits(balance, 6));
  }

  async monitorAddress(address: string, callback: (tx: any) => void): Promise<void> {
    // Listen for incoming USDC transfers
    const filter = this.usdcContract.filters.Transfer(null, address);

    this.usdcContract.on(filter, (from, to, amount, event) => {
      callback({
        from,
        to,
        amount: parseFloat(ethers.utils.formatUnits(amount, 6)),
        txHash: event.transactionHash,
        blockNumber: event.blockNumber
      });
    });
  }
}
```

**Note:** For MVP, prefer Circle API over direct Web3 interaction for simplicity and better transaction management.

---

## Integration Testing Strategy

### Test Environments

| Partner | Sandbox/Test Mode | Test Cards/Accounts |
|---------|------------------|---------------------|
| Anchor | Yes - Sandbox API | Provided in docs |
| Circle | Yes - Sandbox API | Test USDC on Mumbai testnet |
| Paystack | Yes - Test mode | Test cards in docs |
| Mono | Yes - Sandbox | Demo bank accounts |
| Smile Identity | Yes - Test mode | Sample IDs provided |

### Integration Test Suite

```typescript
// test/integration/payments.e2e-spec.ts

describe('Payment Flow Integration Tests', () => {
  let app: INestApplication;
  let user: User;
  let wallet: Wallet;

  beforeAll(async () => {
    // Setup test app with sandbox keys
    app = await createTestApp();
    user = await createTestUser();
  });

  describe('NGN Wallet Flow', () => {
    it('should create NGN wallet via Anchor', async () => {
      wallet = await request(app.getHttpServer())
        .post('/wallets')
        .set('Authorization', `Bearer ${user.token}`)
        .send({ currency: 'NGN' })
        .expect(201);

      expect(wallet.accountNumber).toHaveLength(10);
      expect(wallet.bankName).toBeDefined();
    });

    it('should fund wallet via Paystack', async () => {
      const payment = await request(app.getHttpServer())
        .post('/payments/fund')
        .set('Authorization', `Bearer ${user.token}`)
        .send({ amount: 5000, walletId: wallet.id })
        .expect(201);

      expect(payment.authorizationUrl).toContain('paystack');

      // Simulate webhook callback
      await simulatePaystackWebhook({
        event: 'charge.success',
        data: { reference: payment.reference, amount: 500000 }
      });

      // Verify balance updated
      const updatedWallet = await walletService.findById(wallet.id);
      expect(updatedWallet.balance).toBe(5000);
    });

    it('should send NGN to another user', async () => {
      const recipient = await createTestUser();
      const recipientWallet = await createWallet(recipient, 'NGN');

      const transaction = await request(app.getHttpServer())
        .post('/transactions/send')
        .set('Authorization', `Bearer ${user.token}`)
        .send({
          fromWalletId: wallet.id,
          toWalletId: recipientWallet.id,
          amount: 1000
        })
        .expect(201);

      // Wait for Anchor webhook
      await waitForWebhook('transfer.completed');

      expect(transaction.status).toBe('completed');
    });
  });

  describe('USDC Wallet Flow', () => {
    it('should create USDC wallet via Circle', async () => {
      const usdcWallet = await request(app.getHttpServer())
        .post('/wallets/usdc')
        .set('Authorization', `Bearer ${user.token}`)
        .expect(201);

      expect(usdcWallet.address).toMatch(/^0x[a-fA-F0-9]{40}$/);
    });

    it('should receive USDC', async () => {
      // Simulate Circle webhook for incoming transfer
      await simulateCircleWebhook({
        Type: 'transfers.completed',
        Message: {
          destination: { address: usdcWallet.address },
          amount: { amount: '10.00' }
        }
      });

      const balance = await walletService.getBalance(usdcWallet.id);
      expect(balance).toBe(10);
    });
  });
});
```

---

## Error Handling & Retry Logic

### Centralized Error Handler

```typescript
// src/common/filters/integration-error.filter.ts

@Catch()
export class IntegrationErrorFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    // Map integration errors to user-friendly messages
    const errorMap = {
      'ANCHOR_INSUFFICIENT_BALANCE': 'Insufficient balance in your wallet',
      'CIRCLE_INVALID_ADDRESS': 'Invalid recipient address',
      'PAYSTACK_CARD_DECLINED': 'Your card was declined. Please try another card',
      'MONO_ACCOUNT_NOT_FOUND': 'Bank account not found. Please re-link',
      'SMILE_VERIFICATION_FAILED': 'Identity verification failed. Please try again'
    };

    const message = errorMap[exception.code] || 'An error occurred. Please try again';

    response.status(exception.status || 500).json({
      statusCode: exception.status || 500,
      message,
      error: exception.message
    });
  }
}
```

### Retry Logic with Exponential Backoff

```typescript
// src/common/utils/retry.util.ts

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries: number;
    initialDelay: number;
    maxDelay: number;
    shouldRetry?: (error: any) => boolean;
  }
): Promise<T> {
  const { maxRetries, initialDelay, maxDelay, shouldRetry } = options;
  let lastError: any;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Don't retry if custom condition says no
      if (shouldRetry && !shouldRetry(error)) {
        throw error;
      }

      // Don't retry on last attempt
      if (attempt === maxRetries) {
        throw error;
      }

      // Calculate delay with exponential backoff
      const delay = Math.min(
        initialDelay * Math.pow(2, attempt),
        maxDelay
      );

      console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

// Usage
const result = await retryWithBackoff(
  () => anchorService.transfer(dto),
  {
    maxRetries: 3,
    initialDelay: 1000,   // 1 second
    maxDelay: 10000,      // 10 seconds
    shouldRetry: (error) => error.status >= 500 || error.status === 429
  }
);
```

---

## Monitoring & Observability

### Integration Metrics

```typescript
// src/common/interceptors/metrics.interceptor.ts

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const integration = this.getIntegration(request.url);
    const startTime = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        this.recordMetric(integration, 'success', duration);
      }),
      catchError((error) => {
        const duration = Date.now() - startTime;
        this.recordMetric(integration, 'error', duration, error);
        throw error;
      })
    );
  }

  private recordMetric(
    integration: string,
    status: string,
    duration: number,
    error?: any
  ) {
    // Send to CloudWatch or similar
    console.log(`[Metrics] ${integration} - ${status} - ${duration}ms`, {
      integration,
      status,
      duration,
      errorCode: error?.code
    });
  }

  private getIntegration(url: string): string {
    if (url.includes('anchor')) return 'anchor';
    if (url.includes('circle')) return 'circle';
    if (url.includes('paystack')) return 'paystack';
    if (url.includes('mono')) return 'mono';
    if (url.includes('smile')) return 'smile';
    return 'unknown';
  }
}
```

### Health Checks

```typescript
// src/health/integration-health.controller.ts

@Controller('health/integrations')
export class IntegrationHealthController {
  constructor(
    private anchor: AnchorService,
    private circle: CircleService,
    private paystack: PaystackService,
    private mono: MonoService,
    private smile: SmileIdentityService
  ) {}

  @Get()
  async checkHealth(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkAnchor(),
      this.checkCircle(),
      this.checkPaystack(),
      this.checkMono(),
      this.checkSmile()
    ]);

    return {
      status: checks.every(c => c.status === 'fulfilled') ? 'healthy' : 'degraded',
      integrations: {
        anchor: checks[0].status === 'fulfilled' ? 'up' : 'down',
        circle: checks[1].status === 'fulfilled' ? 'up' : 'down',
        paystack: checks[2].status === 'fulfilled' ? 'up' : 'down',
        mono: checks[3].status === 'fulfilled' ? 'up' : 'down',
        smile: checks[4].status === 'fulfilled' ? 'up' : 'down'
      },
      timestamp: new Date().toISOString()
    };
  }

  private async checkAnchor(): Promise<void> {
    // Simple health check - can be a lightweight API call
    await this.anchor.client.get('/health').catch(() => {
      throw new Error('Anchor unavailable');
    });
  }

  // Similar for other integrations...
}
```

---

## Summary

This integration plan provides:

✅ **Complete API specifications** for all 5 partners
✅ **Step-by-step implementation guides** with TypeScript code
✅ **Webhook handling** for real-time events
✅ **Error handling & retry logic** for resilience
✅ **Testing strategies** for each integration
✅ **Monitoring & observability** for production

### Next Steps

1. **Register for sandbox accounts** with all partners (Week 1)
2. **Implement integrations sequentially** (Weeks 3-4)
3. **Test each integration thoroughly** before moving to next
4. **Setup monitoring** from day 1
5. **Document any deviations** from this plan

**Ready to start building!** 🚀
