# Stripey-x402 Architecture Design
**Status:** Draft v0.1  
**Date:** December 2025  
**Owner:** Adam Mohib / Stripey-x402 Engineering

---

## 1. Context & Goals
Stripey-x402 delivers a "Stripe-like" developer experience for Coinbase's x402 protocol. Key architectural goals:
- **Simple integration:** SDK-first experience that shields developers from x402 complexity.
- **Deterministic billing:** Accurate metering, pricing, invoicing, and dispute-free ledgers.
- **Facilitator abstraction:** Plug-and-play support for Coinbase, thirdweb, AEON, and future facilitators.
- **Operational transparency:** Real-time dashboards, alerts, and webhooks for developers.
- **Security & compliance:** Non-custodial by default, strong key management, and auditable payment flows.
- **Fiat isolation:** Stripey-x402 never touches fiat currency; users stay in stablecoins, and optional onramp connectors simply hand off to regulated providers to acquire x402-compatible assets.

## 2. High-Level System Overview
```
+------------------------+        +-----------------------------+
|  Client SDKs / Agents  |<------>|   Stripey-x402 Edge APIs    |
|  (Node, Python, Go)    |        |  (Auth, Metering, Pricing)  |
+-----------+------------+        +------------+----------------+
            |                                |
            v                                v
   +--------------------+          +----------------------------+
   |  Usage Metering &  |  <---->  |  Billing & Ledger Service  |
   |  Rate Limit Engine |          |  (Pricing, Invoices, AR)   |
   +---------+----------+          +-----------+----------------+
             |                                 |
             v                                 v
   +--------------------+          +----------------------------+
   | Facilitator Broker | <----->  | Blockchain Facilitators    |
   | (x402 Orchestrator)|          | (Coinbase, thirdweb, etc.) |
   +---------+----------+          +----------------------------+
             |
             v
   +--------------------+
   | Developer Console  |
   | (Next.js Dashboard)|
   +--------------------+
```

## 3. Core Components
1. **Client SDKs**
   - Lightweight wrappers (`stripey.fetch`) that detect HTTP 402, parse x402 instructions, sign + submit payments, and handle retries.
   - Ship as language-native packages (npm, PyPI, Go module) with pluggable wallet providers (EOA, smart contract wallets, managed wallets).

2. **Edge Gateway**
   - REST + WebSocket endpoints for onboarding, route configuration, usage ingestion, and analytics queries.
   - Handles API key auth (JWT-backed), request validation, and rate limiting via Redis.

3. **Usage Metering Service**
   - Captures usage events per route/customer, applies unit conversions (per-call, per-MB, per-model), and persists normalized events to PostgreSQL.
   - Emits events to Kafka/NATS for asynchronous billing, alerting, and dashboards.

4. **Billing & Ledger Service**
   - Processes usage streams, applies pricing rules, aggregates invoices (daily/monthly), and maintains double-entry ledger tables (`transactions`, `balances`).
   - Supports credits, prepayments, spending caps, and duplicate detection (idempotency keys + hash fingerprints).

5. **Facilitator Broker (x402 Orchestrator)**
   - Interprets facilitator instructions from upstream APIs.
   - Manages transaction lifecycle: request signing, submission, confirmation polling, and retry semantics.
   - Abstracts facilitator-specific APIs by exposing a common interface (`preparePayment`, `submitPayment`, `verifySettlement`).

6. **Webhook & Notification Service**
   - Delivers signed webhooks (HMAC + timestamp) for payment success/failure, usage anomalies, budget thresholds, invoice readiness.
   - Supports retries with exponential backoff and dead-letter queues.

7. **Developer Console (Next.js)**
   - Provides dashboards for usage analytics, customer management, facilitator health, API key management, and alert configuration.
   - Consumes internal GraphQL/REST APIs to fetch aggregated metrics.

8. **Data Stores**
   - **PostgreSQL:** System of record for customers, routes, usage events, invoices, facilitator configs.
   - **Redis:** Rate limiting, caching of facilitator instructions, short-lived session data.
   - **Object Storage (S3/R2):** Invoice PDFs, export files, archived logs.

9. **Onramp Connectors (Optional Module)**
   - Hosted flows and API hooks that redirect users to regulated fiat-to-crypto partners (e.g., Coinbase Pay, MoonPay) to purchase USDC/Base or other x402-supported assets.
   - Stripey-x402 never custody fiat or stablecoins; the connectors provide a Stripe-like checkout experience while keeping funds off the platform until they land in the user's wallet.

9. **Observability Stack**
   - OpenTelemetry instrumentation → Loki/Elastic for logs, Prometheus for metrics, Tempo/Jaeger for traces.
   - Alerting via PagerDuty/Slack for latency, failure rate, and settlement anomalies.

## 4. Data Model Snapshot
- `customers (id, wallet_address, status, metadata)`
- `routes (id, customer_id, facilitator, pricing_model, config_json)`
- `usage_events (idempotency_key, route_id, units, cost, status, occurred_at)`
- `transactions (tx_hash, facilitator, amount, asset, state, customer_id)`
- `invoices (id, customer_id, period_start, period_end, subtotal, fees, status)`
- `facilitator_configs (id, type, credentials_ref, chain_id)`
- `webhook_endpoints (id, customer_id, url, secret, status)`

## 5. Key Flows
### 5.1 Protected API Call & Payment Resolution
1. Client hits developer API protected by Stripey-x402 middleware.
2. API returns HTTP 402 with facilitator instructions (signed nonce, amount, asset, destination).
3. Client SDK invokes Edge Gateway `/payments/resolve`.
4. Facilitator Broker signs transaction with selected wallet and submits via facilitator API.
5. Settlement is confirmed → Gateway notifies SDK.
6. SDK retries original request → developer API returns 200.
7. Usage event + transaction recorded; billing engine includes it in invoice.

### 5.2 Monthly Billing & Invoicing
1. Aggregator job pulls confirmed usage for the billing period.
2. Pricing rules applied (tiered, flat, per-unit) → invoice line items.
3. Invoice stored in PostgreSQL and rendered to PDF (S3).
4. Webhook notifies developer; optional CSV/QuickBooks export generated.

### 5.3 Dashboard Analytics
1. Dashboard queries analytics API (Materialized views or Timescale/Postgres hypertables).
2. Pre-aggregated metrics served with caching to keep <200 ms latency.

### 5.4 Cash-to-Crypto Onramp Flow
1. Merchant or end user signals a fiat-intent payment inside the application.
2. Stripey-x402 exposes an onramp connector endpoint that issues a signed redirect/session with a regulated partner (Coinbase Pay, MoonPay, etc.).
3. User completes KYC and purchases USDC/Base; assets settle directly into the user-controlled wallet (or an authorized smart-contract wallet) without Stripey-x402 taking custody.
4. Connector notifies Stripey-x402 Edge APIs via webhook/callback; merchant resumes the protected API call.
5. SDK receives the HTTP 402 challenge, funds are already in the wallet, and the facilitator broker submits the x402 smart-contract payment.
6. Facilitator confirms settlement → merchant receives crypto, and the billing/ledger services record the transaction.

## 6. Technology Choices
- **Languages:** TypeScript (Node.js) for Edge, Metering, Billing; Go for facilitator broker if latency-critical.
- **Frameworks:** Fastify/Express for APIs; Next.js 14 app router for console; Prisma/SQLC for DB access.
- **Message Bus:** NATS JetStream or Kafka (managed) for event streaming.
- **Infra:** Vercel (dashboard) + Fly.io/AWS ECS for services; Terraform for IaC; GitHub Actions CI/CD.
- **Wallet Integration:** Wallet-as-a-Service providers (Coinbase Wallet SDK) + optional managed custodial keys via AWS KMS/HSM.

## 7. Environments & Deployment
- **Local Dev:** Docker Compose with Postgres, Redis, NATS.
- **Staging:** Mirrors production, connected to facilitator sandboxes (Base Sepolia).
- **Production:** Multi-region Edge layer, active-active Postgres (managed), Redis Enterprise, autoscaled services.
- Deployments triggered via GitHub Actions → Terraform apply → service rollouts (blue/green or canary).

## 8. Security & Compliance Considerations
- Non-custodial default: user wallets or delegated signing via secure key vault.
- Explicit no-fiat handling: Stripey-x402 never stores, transmits, or settles fiat currency; onramps are referrals/redirects to licensed partners.
- JWT scopes per API key; rotating secrets and audit logs.
- HMAC + timestamped webhooks, replay detection via Redis TTL.
- PII minimization & encryption at rest (Postgres TDE + KMS managed keys).
- SOC2-ready logging, separation of duties, RBAC in dashboard.

## 9. Open Questions / Next Steps
- Do we bundle managed wallets or rely solely on external wallets in v1?
- Which facilitator SDK offers best latency vs. reliability tradeoff for launch (Coinbase only vs. Coinbase + thirdweb)?
- Billing accuracy testing harness: synthetic traffic vs. on-chain replay?
- Dashboard query engine: continue with Postgres hypertables or introduce dedicated OLAP (ClickHouse) when scale demands?

## 10. Implementation Milestones
1. Scaffold monorepo (services + SDK packages) with shared protobuf/JSON schemas.
2. Build Node SDK + basic Edge Gateway with mocked facilitator.
3. Stand up metering/billing data pipeline with PostgreSQL + Redis.
4. Integrate Coinbase x402 facilitator for settlement.
5. Ship dashboard MVP (usage charts, customer list) and webhook service.
6. Add production hardening: observability, rate limits, HA deployments, disaster recovery playbooks.
