# 🏗️ ARCHITECTURAL DESIGN DOCUMENT
**Product:** Stripey-x402 – "Stripe for x402"  
**Owner:** Adam Mohib (Founder)  
**Version:** 0.1  
**Date:** December 2025  

---

## 1. System Overview

Stripey-x402 is a developer-first payments and billing platform built on Coinbase's x402 protocol. This document outlines the high-level architecture, component responsibilities, and data flows.

### 1.1 Architecture Principles
- **Non-custodial**: Stripey-x402 never holds customer funds; all payments flow through external facilitators
- **Developer-first**: Minimal integration friction via SDKs and middleware
- **Event-driven**: Asynchronous processing for scalability and resilience
- **Multi-facilitator**: Abstract facilitator complexity to enable choice

---

## 2. High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Node SDK   │  │ Python SDK  │  │   Go SDK    │  │   JS SDK    │         │
│  │ @stripey/   │  │  stripey-   │  │   stripey   │  │ @stripey/   │         │
│  │    node     │  │   python    │  │     -go     │  │    web      │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                │                 │
│         └────────────────┴────────────────┴────────────────┘                 │
│                                   │                                          │
│                          ┌────────▼────────┐                                 │
│                          │   HTTP Client   │                                 │
│                          │  (402 Handler)  │                                 │
│                          └────────┬────────┘                                 │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                          MERCHANT LAYER                                      │
├───────────────────────────────────┼─────────────────────────────────────────┤
│                          ┌────────▼────────┐                                 │
│                          │ Server Middleware│                                │
│                          │ (Express/Fastify/│                                │
│                          │  Django/FastAPI) │                                │
│                          └────────┬────────┘                                 │
│                                   │                                          │
│                          ┌────────▼────────┐                                 │
│                          │  Merchant API   │                                 │
│                          │    Service      │                                 │
│                          └────────┬────────┘                                 │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                       STRIPEY-X402 PLATFORM                                  │
├───────────────────────────────────┼─────────────────────────────────────────┤
│                          ┌────────▼────────┐                                 │
│                          │   API Gateway   │                                 │
│                          │  (Rate Limit/   │                                 │
│                          │   Auth/Route)   │                                 │
│                          └────────┬────────┘                                 │
│                                   │                                          │
│    ┌──────────────────────────────┼──────────────────────────────────┐      │
│    │                              │                                   │      │
│    ▼                              ▼                                   ▼      │
│ ┌──────────┐              ┌──────────────┐                    ┌──────────┐  │
│ │ Billing  │              │   Payment    │                    │ Customer │  │
│ │ Service  │              │   Service    │                    │ Service  │  │
│ └────┬─────┘              └──────┬───────┘                    └────┬─────┘  │
│      │                           │                                  │       │
│      │    ┌──────────────────────┼──────────────────────────┐      │       │
│      │    │                      │                          │      │       │
│      ▼    ▼                      ▼                          ▼      ▼       │
│ ┌─────────────┐           ┌─────────────┐            ┌─────────────┐       │
│ │  Usage &    │           │  Ledger &   │            │  Identity & │       │
│ │  Metering   │           │ Transaction │            │   Account   │       │
│ │   Engine    │           │   Engine    │            │   Engine    │       │
│ └──────┬──────┘           └──────┬──────┘            └──────┬──────┘       │
│        │                         │                          │              │
│        └─────────────────────────┼──────────────────────────┘              │
│                                  │                                         │
│                          ┌───────▼───────┐                                 │
│                          │  Event Bus    │                                 │
│                          │ (Kafka/NATS)  │                                 │
│                          └───────┬───────┘                                 │
│                                  │                                         │
│        ┌─────────────────────────┼─────────────────────────┐              │
│        │                         │                         │              │
│        ▼                         ▼                         ▼              │
│ ┌─────────────┐          ┌─────────────┐          ┌─────────────┐         │
│ │  Webhook    │          │ Analytics & │          │   Alert     │         │
│ │  Service    │          │  Dashboard  │          │  Service    │         │
│ └─────────────┘          └─────────────┘          └─────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                       FACILITATOR LAYER                                      │
├───────────────────────────────────┼─────────────────────────────────────────┤
│    ┌──────────────────────────────┼──────────────────────────────────┐      │
│    │                              │                                   │      │
│    ▼                              ▼                                   ▼      │
│ ┌──────────┐              ┌──────────────┐                    ┌──────────┐  │
│ │ Coinbase │              │   thirdweb   │                    │   AEON   │  │
│ │Facilitator│             │  Facilitator │                    │Facilitator│  │
│ └──────────┘              └──────────────┘                    └──────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                       BLOCKCHAIN LAYER                                       │
├───────────────────────────────────┼─────────────────────────────────────────┤
│                          ┌────────▼────────┐                                 │
│                          │   Base Chain    │                                 │
│                          │  (USDC/ETH)     │                                 │
│                          └─────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Architecture

### 3.1 Client SDKs

The SDKs wrap HTTP clients to automatically handle x402 payment flows.

**Responsibilities:**
- Intercept HTTP 402 responses
- Parse x402 payment instructions from response headers
- Sign and submit payment transactions to facilitators
- Retry original requests after successful payment
- Cache payment receipts to prevent duplicates
- Surface developer-friendly errors

**Technology Stack:**
- Node.js/TypeScript for `@stripey/node`
- Python for `stripey-python`
- Go for `stripey-go`
- Browser-compatible TypeScript for `@stripey/web`

### 3.2 Server Middleware

Middleware components protect API routes and enforce pricing.

**Supported Frameworks:**
- Express.js (Node.js)
- Fastify (Node.js)
- Django (Python)
- FastAPI (Python)

**Responsibilities:**
- Validate incoming requests against pricing rules
- Return HTTP 402 with x402 payment instructions
- Verify payment completion via facilitator callbacks
- Record usage events to the metering engine
- Support per-endpoint pricing configuration

### 3.3 Core Platform Services

#### 3.3.1 API Gateway
- Authentication (JWT-based API keys)
- Rate limiting per customer/route
- Request routing to internal services
- TLS termination

#### 3.3.2 Billing Service
- Manages pricing rules and plans
- Calculates charges based on usage
- Generates invoices
- Applies discounts and credits

#### 3.3.3 Payment Service
- Coordinates with facilitators for payment verification
- Tracks payment states (pending, confirmed, failed)
- Handles retry logic for failed payments
- Manages payment receipts

#### 3.3.4 Customer Service
- Customer identity and profile management
- API key generation and rotation
- Wallet address binding
- Usage quota management

### 3.4 Internal Engines

#### 3.4.1 Usage & Metering Engine

**Purpose:** Track and aggregate resource consumption.

**Data Model:**
```typescript
interface UsageEvent {
  id: string;
  customerId: string;
  routeId: string;
  transactionId?: string;  // Optional: may not exist for failed/pending payments
  timestamp: Date;
  quantity: number;
  unit: 'request' | 'byte' | 'token' | 'second';
  metadata: Record<string, any>;
}
```

**Processing:**
1. Receive events from middleware via event bus
2. Validate and deduplicate events
3. Store in time-series optimized storage
4. Aggregate for billing periods

#### 3.4.2 Ledger & Transaction Engine

**Purpose:** Maintain an immutable record of all financial transactions.

**Ledger Architecture:**
The ledger follows double-entry accounting principles to ensure financial integrity.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LEDGER STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    ACCOUNTS                              │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  Customer Accounts (Balance/Liability)                   │    │
│  │  Merchant Revenue Accounts (Revenue)                     │    │
│  │  Platform Fee Accounts (Revenue)                         │    │
│  │  Facilitator Settlement Accounts (Asset)                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    ENTRIES                               │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  Entry {                                                 │    │
│  │    id: UUID                                              │    │
│  │    transactionId: UUID                                   │    │
│  │    accountId: UUID                                       │    │
│  │    amount: Decimal                                       │    │
│  │    type: 'DEBIT' | 'CREDIT'                              │    │
│  │    currency: 'USDC' | 'ETH'                              │    │
│  │    createdAt: Timestamp                                  │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    TRANSACTIONS                          │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  Transaction {                                           │    │
│  │    id: UUID                                              │    │
│  │    type: 'PAYMENT' | 'REFUND' | 'FEE' | 'SETTLEMENT'     │    │
│  │    txHash: string (blockchain reference)                 │    │
│  │    status: 'PENDING' | 'CONFIRMED' | 'FAILED'            │    │
│  │    entries: Entry[]                                      │    │
│  │    metadata: JSON                                        │    │
│  │    createdAt: Timestamp                                  │    │
│  │  }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**Ledger Workflow:**

1. **Payment Initiated**
   ```
   Customer calls API → 402 returned → SDK initiates payment
   ```

2. **Payment Recorded (Pending)**
   ```
   Debit: Customer Balance Account    $0.05
   Credit: Pending Payments Account   $0.05
   Status: PENDING
   ```

3. **Payment Confirmed (On-chain verification)**
   ```
   Debit: Pending Payments Account    $0.05
   Credit: Merchant Revenue Account   $0.045 (90%)
   Credit: Platform Fee Account       $0.005 (10%)
   Status: CONFIRMED
   TxHash: 0x123...abc
   ```

4. **Settlement to Merchant**
   ```
   Debit: Merchant Revenue Account    $0.045
   Credit: Settlement Account         $0.045
   Note: Funds transferred via facilitator
   ```

#### 3.4.3 Identity & Account Engine

**Purpose:** Manage customer identities and access credentials.

**Features:**
- Customer registration and profile management
- API key generation with scoped permissions
- Wallet address verification and binding
- Multi-device support
- Credit/prepayment balance management

---

## 4. Data Flow

### 4.1 Payment Flow (Happy Path)

```
┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌────────────┐    ┌───────────┐
│  Client  │    │ Merchant │    │  Stripey-x402 │    │ Facilitator│    │ Blockchain│
│   SDK    │    │   API    │    │   Platform   │    │ (Coinbase) │    │  (Base)   │
└────┬─────┘    └────┬─────┘    └──────┬───────┘    └─────┬──────┘    └─────┬─────┘
     │               │                 │                  │                 │
     │  1. Request   │                 │                  │                 │
     │──────────────▶│                 │                  │                 │
     │               │                 │                  │                 │
     │               │  2. Check Auth  │                  │                 │
     │               │────────────────▶│                  │                 │
     │               │                 │                  │                 │
     │               │  3. No Payment  │                  │                 │
     │               │◀────────────────│                  │                 │
     │               │                 │                  │                 │
     │  4. HTTP 402  │                 │                  │                 │
     │  + x402 hdrs  │                 │                  │                 │
     │◀──────────────│                 │                  │                 │
     │               │                 │                  │                 │
     │  5. Parse instructions          │                  │                 │
     │─────┐         │                 │                  │                 │
     │     │         │                 │                  │                 │
     │◀────┘         │                 │                  │                 │
     │               │                 │                  │                 │
     │  6. Sign & submit payment       │                  │                 │
     │─────────────────────────────────────────────────▶│                 │
     │               │                 │                  │                 │
     │               │                 │                  │  7. Submit tx   │
     │               │                 │                  │────────────────▶│
     │               │                 │                  │                 │
     │               │                 │                  │  8. Confirm     │
     │               │                 │                  │◀────────────────│
     │               │                 │                  │                 │
     │  9. Payment receipt             │                  │                 │
     │◀─────────────────────────────────────────────────│                 │
     │               │                 │                  │                 │
     │  10. Retry request with receipt │                  │                 │
     │──────────────▶│                 │                  │                 │
     │               │                 │                  │                 │
     │               │  11. Verify     │                  │                 │
     │               │────────────────▶│                  │                 │
     │               │                 │                  │                 │
     │               │  12. Log Usage  │                  │                 │
     │               │────────────────▶│                  │                 │
     │               │                 │                  │                 │
     │  13. HTTP 200 │                 │                  │                 │
     │◀──────────────│                 │                  │                 │
     │               │                 │                  │                 │
```

### 4.2 Webhook Flow

```
┌─────────────────┐    ┌──────────────┐    ┌─────────────┐
│ Event Bus       │    │ Webhook      │    │ Merchant    │
│ (Kafka/NATS)    │    │ Service      │    │ Endpoint    │
└───────┬─────────┘    └──────┬───────┘    └──────┬──────┘
        │                     │                   │
        │  1. Payment Event   │                   │
        │────────────────────▶│                   │
        │                     │                   │
        │                     │  2. Lookup        │
        │                     │  webhook config   │
        │                     │────┐              │
        │                     │    │              │
        │                     │◀───┘              │
        │                     │                   │
        │                     │  3. Sign payload  │
        │                     │  (HMAC-SHA256)    │
        │                     │────┐              │
        │                     │    │              │
        │                     │◀───┘              │
        │                     │                   │
        │                     │  4. POST webhook  │
        │                     │  + signature      │
        │                     │──────────────────▶│
        │                     │                   │
        │                     │  5. 200 OK        │
        │                     │◀──────────────────│
        │                     │                   │
```

---

## 5. Data Model

### 5.1 Core Entities

```typescript
// Customer
interface Customer {
  id: string;
  email: string;
  name: string;
  walletAddresses: WalletAddress[];
  apiKeys: ApiKey[];
  balance: Decimal;
  createdAt: Date;
  updatedAt: Date;
}

// WalletAddress
interface WalletAddress {
  id: string;
  customerId: string;
  address: string;
  chain: 'base' | 'ethereum' | 'polygon';
  isPrimary: boolean;
  verifiedAt: Date;
}

// Route (Protected API Endpoint)
interface Route {
  id: string;
  merchantId: string;
  path: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  pricing: RoutePricing;
  isActive: boolean;
}

// RoutePricing
interface RoutePricing {
  mode: 'per-request' | 'per-byte' | 'per-token' | 'per-second';
  amount: Decimal;
  currency: 'USDC' | 'ETH';
  facilitator: 'coinbase' | 'thirdweb' | 'aeon';
}

// Transaction
interface Transaction {
  id: string;
  customerId: string;
  routeId: string;
  amount: Decimal;
  currency: 'USDC' | 'ETH';
  txHash: string;
  status: 'PENDING' | 'CONFIRMED' | 'FAILED';
  facilitator: string;
  createdAt: Date;
  confirmedAt?: Date;
}

// UsageEvent
interface UsageEvent {
  id: string;
  customerId: string;
  routeId: string;
  transactionId?: string;  // Optional: may not exist for failed/pending payments
  quantity: number;
  unit: 'request' | 'byte' | 'token' | 'second';
  timestamp: Date;
  metadata: Record<string, any>;
}

// InvoiceLineItem
interface InvoiceLineItem {
  id: string;
  routeId: string;
  description: string;
  quantity: number;
  unitPrice: Decimal;
  amount: Decimal;
}

// Invoice
interface Invoice {
  id: string;
  customerId: string;
  period: { start: Date; end: Date };
  lineItems: InvoiceLineItem[];
  subtotal: Decimal;
  fees: Decimal;
  total: Decimal;
  status: 'DRAFT' | 'PENDING' | 'PAID' | 'VOID';
  createdAt: Date;
  paidAt?: Date;
}
```

### 5.2 Database Schema

```sql
-- Customers table
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  balance DECIMAL(20, 8) DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Wallet addresses
CREATE TABLE wallet_addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id),
  address VARCHAR(42) NOT NULL,
  chain VARCHAR(20) NOT NULL,
  is_primary BOOLEAN DEFAULT false,
  verified_at TIMESTAMPTZ,
  UNIQUE(address, chain)
);

-- Routes (protected endpoints)
CREATE TABLE routes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  merchant_id UUID NOT NULL,
  path VARCHAR(500) NOT NULL,
  method VARCHAR(10) NOT NULL,
  pricing_mode VARCHAR(20) NOT NULL,
  pricing_amount DECIMAL(20, 8) NOT NULL,
  pricing_currency VARCHAR(10) NOT NULL,
  facilitator VARCHAR(50) NOT NULL,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Transactions
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id),
  route_id UUID REFERENCES routes(id),
  amount DECIMAL(20, 8) NOT NULL,
  currency VARCHAR(10) NOT NULL,
  tx_hash VARCHAR(66),
  status VARCHAR(20) NOT NULL,
  facilitator VARCHAR(50) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  confirmed_at TIMESTAMPTZ
);

-- Ledger entries (double-entry accounting)
CREATE TABLE ledger_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID REFERENCES transactions(id),
  account_id UUID NOT NULL,
  amount DECIMAL(20, 8) NOT NULL,
  entry_type VARCHAR(10) NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
  currency VARCHAR(10) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usage events (time-series)
CREATE TABLE usage_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id),
  route_id UUID REFERENCES routes(id),
  transaction_id UUID REFERENCES transactions(id),  -- Nullable: may not exist for failed/pending payments
  quantity DECIMAL(20, 8) NOT NULL,
  unit VARCHAR(20) NOT NULL CHECK (unit IN ('request', 'byte', 'token', 'second')),
  timestamp TIMESTAMPTZ NOT NULL,
  metadata JSONB
);

-- Create indexes for performance
CREATE INDEX idx_transactions_customer ON transactions(customer_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_usage_events_timestamp ON usage_events(timestamp);
CREATE INDEX idx_usage_events_customer ON usage_events(customer_id);
CREATE INDEX idx_ledger_entries_transaction ON ledger_entries(transaction_id);
```

---

## 6. Technology Stack

### 6.1 Backend Services

| Component | Technology | Rationale |
|-----------|------------|-----------|
| API Gateway | Kong / AWS API Gateway | Rate limiting, auth, routing |
| Core Services | Node.js + TypeScript | Developer familiarity, async I/O |
| Database | PostgreSQL | ACID compliance, JSONB support |
| Cache | Redis | Rate limits, sessions, idempotency |
| Message Queue | Kafka / NATS | Event streaming, decoupling |
| RPC Client | ethers.js / viem | Blockchain interaction |

### 6.2 Frontend Dashboard

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Framework | Next.js 14 | SSR, API routes, React |
| Styling | Tailwind CSS | Rapid UI development |
| Charts | Recharts | Data visualization |
| State | TanStack Query | Server state management |

### 6.3 Infrastructure

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Hosting | Vercel + Fly.io | Edge deployment, low latency |
| Container | Docker | Consistent environments |
| IaC | Terraform | Infrastructure as code |
| CI/CD | GitHub Actions | Automated deployments |
| Monitoring | Datadog / Prometheus | Metrics, alerting |
| Logging | Elastic Stack | Centralized logging |

---

## 7. Security Architecture

### 7.1 Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────┐
│                   SECURITY LAYERS                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 API Key Authentication               │    │
│  │  - JWT-based tokens with scoped permissions          │    │
│  │  - Automatic key rotation support                    │    │
│  │  - Rate limiting per key                             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Webhook Security                     │    │
│  │  - HMAC-SHA256 signatures                            │    │
│  │  - Timestamped nonces (5-minute window)              │    │
│  │  - IP allowlisting (optional)                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Payment Security                     │    │
│  │  - txHash verification against blockchain            │    │
│  │  - Replay protection via nonce tracking              │    │
│  │  - Amount verification                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Data Security                        │    │
│  │  - Encryption at rest (AES-256)                      │    │
│  │  - Encryption in transit (TLS 1.3)                   │    │
│  │  - Audit logs for all sensitive operations           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Threat Model

| Threat | Mitigation |
|--------|------------|
| Replay attacks | Nonce tracking, txHash verification |
| Double spending | Ledger double-entry, blockchain confirmation |
| API key compromise | Key rotation, scoped permissions, IP restrictions |
| Man-in-the-middle | TLS 1.3 everywhere |
| DDoS | Rate limiting, CDN protection |

---

## 8. Scalability Considerations

### 8.1 Horizontal Scaling

- **Stateless services**: All core services are stateless and can scale horizontally
- **Database sharding**: Shard by customer_id for write-heavy tables
- **Read replicas**: PostgreSQL read replicas for analytics queries
- **Cache layer**: Redis cluster for session and rate limit data

### 8.2 Performance Targets

| Metric | Target |
|--------|--------|
| API latency (p99) | < 150ms |
| Payment verification | < 3s |
| Webhook delivery | < 5s |
| Dashboard load | < 2s |
| Uptime | 99.9% |

---

## 9. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PRODUCTION ENVIRONMENT                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          CDN / Edge                                  │    │
│  │                        (Cloudflare)                                  │    │
│  └──────────────────────────────┬──────────────────────────────────────┘    │
│                                 │                                            │
│                     ┌───────────┴───────────┐                               │
│                     │                       │                               │
│           ┌─────────▼─────────┐   ┌─────────▼─────────┐                     │
│           │   Dashboard       │   │   API Gateway     │                     │
│           │   (Vercel)        │   │   (Fly.io)        │                     │
│           └───────────────────┘   └─────────┬─────────┘                     │
│                                             │                               │
│                         ┌───────────────────┼───────────────────┐           │
│                         │                   │                   │           │
│               ┌─────────▼──────┐  ┌─────────▼──────┐  ┌─────────▼──────┐   │
│               │ Billing Service │  │ Payment Service│  │Customer Service│   │
│               │   (Fly.io)     │  │   (Fly.io)     │  │   (Fly.io)     │   │
│               └────────────────┘  └────────────────┘  └────────────────┘   │
│                                             │                               │
│                                   ┌─────────▼─────────┐                     │
│                                   │   Message Queue   │                     │
│                                   │   (Upstash Kafka) │                     │
│                                   └─────────┬─────────┘                     │
│                                             │                               │
│               ┌─────────────────────────────┼─────────────────────────────┐ │
│               │                             │                             │ │
│     ┌─────────▼─────────┐         ┌─────────▼─────────┐         ┌─────────▼─────────┐
│     │   PostgreSQL      │         │      Redis        │         │   Blob Storage    │
│     │   (Neon.tech)     │         │   (Upstash)       │         │   (R2/S3)         │
│     └───────────────────┘         └───────────────────┘         └───────────────────┘
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Development Roadmap Alignment

| Phase | Architecture Focus |
|-------|-------------------|
| Months 1-2 | SDK core, middleware, basic API services |
| Month 3 | Billing engine, ledger system, webhook service |
| Month 4 | Dashboard frontend, analytics pipeline |
| Month 5 | Multi-facilitator abstraction layer |
| Month 6 | Subscription engine, scaling optimizations |

---

## 11. Appendix

### A. Glossary

- **x402**: HTTP payment protocol using 402 status code
- **Facilitator**: Third-party payment processor (Coinbase, thirdweb, AEON)
- **Ledger**: Double-entry accounting system for transaction records
- **Metering**: Usage tracking and aggregation system

### B. References

- [x402 Protocol Specification](https://www.x402.org)
- [Coinbase Commerce API](https://commerce.coinbase.com/docs)
- [Base Chain Documentation](https://docs.base.org)
