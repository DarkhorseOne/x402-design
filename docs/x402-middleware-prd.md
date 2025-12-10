下面是你要的 **《DarkhorseOne x402 Middleware — 完整 PRD》Markdown 文件内容**。
已根据你接受的策略（Phase 0 → Phase 2）重写成正式可用的 PRD，可直接保存为 `x402-middleware-prd.md`。

---

# DarkhorseOne x402 Middleware

### Product Requirements Document (PRD)

**Version:** 1.0
**License:** MIT (© DarkhorseOne Limited)

---

# 1. Overview

DarkhorseOne x402 Middleware is a TypeScript-based implementation of the **x402 protocol** designed to integrate into **NestJS** and **Next.js** API servers. Its purpose is to enforce **per-call or usage-based micro-payments** using the x402 payment standard (via Coinbase facilitator or compatible services).
Middleware operates as a **Guard / Interceptor** (NestJS) or a **Route Wrapper** (Next.js) and returns **HTTP 402 Payment Required** when payment is missing, invalid, or insufficient.

This PRD defines product goals, functional specifications, non-functional requirements, fallback strategy, extensibility, and phased rollout.

---

# 2. Goals & Non-Goals

## 2.1 Goals

* Provide a production-ready, open-source **MIT-licensed** x402 enforcement layer for API endpoints.
* Support **NestJS** (Guard + Interceptor + Decorator) and **Next.js** (route wrapper) with identical behaviors.
* Integrate with Coinbase’s official x402 facilitator SDK.
* Provide minimal optional abstractions for:

  * local storage (usage logs, payment receipts)
  * caching (Redis)
  * asynchronous processing (RabbitMQ or similar)
* Maintain strict security while remaining easy to integrate.

## 2.2 Non-Goals (v1)

* No built-in credit system (only Phase 2 optional).
* No requirement for Redis, database, or MQ in v1.
* API client-side SDK is not included (may be a future project).

---

# 3. System Architecture (High-Level)

```
┌─────────────────────────────────────────────────────────────────┐
│                  DarkhorseOne x402 Middleware                    │
├─────────────────────┬────────────────────┬───────────────────────┤
│ @x402-core          │ @x402-nest         │ @x402-next             │
│ (framework-agnostic)│ (NestJS adapter)   │ (Next.js adapter)      │
├─────────────────────┴────────────────────┴───────────────────────┤
│     Optional Extensions                                          │
│     - Storage (DB)                                               │
│     - Redis cache                                                │
│     - MQ event sink                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

# 4. Phased Development Strategy

## Phase 0 — MVP (Stateless, Strict Mode)

* No database required.
* No Redis required.
* No MQ required.
* Fallback **deny** (facilitator failure = do NOT allow access).
* Payment validated **synchronously** through facilitator API.
* Intended for:

  * public open-source libraries
  * low-risk early-stage deployments

## Phase 1 — Audit & Observability

* Add optional storage layer (`X402Storage` interface).
* Provide an official Postgres/Prisma implementation as a separate package.
* Add optional Redis caching layer.
* Add usage logs, payment logs, and per-wallet histories.

## Phase 2 — Credit & Enterprise Features (Optional)

* Introduce `CreditPolicy`, `CreditAccount`, and `CreditLedger`.
* Implement fallback strategy `credit_based`.
* Provide async reconciliation worker (requires MQ).
* Enabled only for **trusted enterprise tenants**.

---

# 5. Functional Requirements

## 5.1 x402 Enforcement Logic

### When an API endpoint is marked as “chargeable”:

1. If **no valid payment** → return:

   * HTTP 402 Payment Required
   * body: x402 Payment Requirement (amount, asset, network, facilitatorRef, expiration)
2. If payment **is attached**:

   * Validate using facilitator
   * Outcomes:

     * **success → allow**
     * **invalid/expired/insufficient → 402**

### Supported Payment Models (Phase 0)

* **Per-call fixed pricing** (e.g., `$0.01` per call)

### Future Payment Models (Phase 1+)

* Usage metered pricing (e.g., tokens, rows, bytes)
* Hybrid subscription + per-call models

---

## 5.2 NestJS Integration Requirements

### 5.2.1 Decorator

```ts
@X402Charge({
  price: "0.01",
  asset: "USDC",
  network: "base-mainnet",
  settlementMode: "strict",
})
```

### 5.2.2 Guard Responsibilities

* Extract payment credential from headers/body.
* Resolve tenant context.
* If payment missing → generate 402 requirement.
* If payment present → call `verifyPayment()`.

### 5.2.3 Interceptor Responsibilities

* Attach payment result to request object (`req.x402`).
* Emit usage logs (sync or async depending on Phase).
* In Phase 1: write to storage.

### 5.2.4 Module

`X402Module.forRoot(...)` or `.forRootAsync(...)`

---

## 5.3 Next.js Integration Requirements

### API

```ts
export const POST = x402Route(config, handler);
```

### Required Features

* Support Next.js App Router (`app/api/.../route.ts`)
* Support Next.js Pages Router (`pages/api/*.ts`)
* Support Edge runtime mode when possible (optional)

---

## 5.4 Facilitator Integration

### Requirements

* Use Coinbase x402 SDK (if available).
* Must support:

  * PaymentRequirement generation
  * Payment verification
  * Configurable network/asset
  * Merchant/seller wallet binding

### Participant Model

* Buyer Wallet
* Seller Wallet (merchant)
* Facilitator (trusted)
* Resource Server (NestJS/Next.js)

---

# 6. Payment Verification Workflow

## 6.1 Unpaid Request

```
Client → /api/foo → Middleware → 402 + PaymentRequirement
```

## 6.2 Paid Request

```
Client → /api/foo (with payment credential)
→ Middleware → Facilitator API
→ success → handler executes
```

## 6.3 Invalid Payment

```
→ Middleware → Facilitator API → invalid → 402
```

---

# 7. Fallback Strategy

Based on your accepted architectural decision:

## 7.1 fallbackMode Options

### 1. `deny` (default)

* Facilitator timeout / 500 → do NOT allow usage.
* Return error: 503 or 402 with retry hint.
* **This is Phase 0 default**.

### 2. `temporary_allow` (Phase 1 optional)

* Allow once per requestId.
* Mark record as “pending verification.”
* Later reconciliation task resolves status.

### 3. `credit_based` (Phase 2)

* Requires:

  * CreditAccount
  * Monthly credit limit
  * Ledger + reconciliation worker
* Only for B2B or trusted tenants.

---

# 8. Storage Layer (Optional, Phase 1)

## Storage Interface

```ts
export interface X402Storage {
  recordUsage(entry: UsageRecord): Promise<void>;
  recordPayment(receipt: PaymentReceipt): Promise<void>;
  getPaymentById(id: string): Promise<PaymentReceipt | null>;
  getUsageByRequestId(id: string): Promise<UsageRecord | null>;
}
```

## Recommended Prisma Schema (Phase 1)

* PaymentReceipt
* UsageRecord
* TenantConfig

(Full schema delivered separately during implementation phase.)

---

# 9. Redis Layer (Optional)

Used for:

* caching verification results
* caching facilitator keys
* rate limiting (per-wallet, per-IP)
* preventing double-spend per paymentId

Redis is **optional**, not required for Phase 0.

---

# 10. MQ Layer (Optional, Phase 2)

Used for:

* async reconciliation of pending payments
* generating accounting events
* large-scale usage reporting

RabbitMQ, Kafka, or SQS acceptable.

---

# 11. Configuration Specification

## Global Config

```ts
interface X402Config {
  facilitatorUrl: string;
  apiKey?: string;
  defaultNetwork: string;
  defaultAsset: string;
  sellerWallet: string;
  fallbackMode: "deny" | "temporary_allow" | "credit_based";
  tenantResolver?: (req) => string | null;
  storage?: X402Storage;
  cache?: X402Cache;
}
```

---

# 12. Error Handling Requirements

### 402 Payment Required

Returned when:

* missing payment
* insufficient/expired/invalid payment

### 503 / 500

Returned when:

* facilitator unavailable (Phase 0 strict mode)
* internal errors
* misconfiguration

### Error Response Format

Must follow x402 standard with extensions:

```json
{
  "error": "payment_required",
  "payment": { ... requirement ... },
  "message": "Payment required to access this resource."
}
```

---

# 13. Telemetry & Observability

Phase 1 requirements:

* Emit structured logs:

  * walletAddress
  * route
  * amount
  * network
  * status
  * latency
* Allow integration with:

  * OpenTelemetry
  * Prometheus/Grafana

---

# 14. Security Requirements

* All facilitator communications = HTTPS only.
* Do not log API keys or sensitive credentials.
* Enforce replay protection (via paymentId + cache/DB).
* Validate all payloads using schema validator (Zod or equivalent).

---

# 15. Example Developer Experience (DX)

### NestJS Example

```ts
@Controller("report")
export class ReportController {

  @Get()
  @X402Charge({
    price: "0.02",
    asset: "USDC",
    network: "base-mainnet",
  })
  generate() {
    return { status: "ok" };
  }
}
```

### Next.js Example

```ts
export const POST = x402Route(
  { price: "0.05", asset: "USDC", network: "base-mainnet" },
  async (req) => new Response("hello world")
);
```

---

# 16. Open-Source Requirements

* MIT License
* README with:

  * installation
  * usage examples
  * integration guides
* Provide code examples for both frameworks.
* Keep framework-specific packages modular.

---

# 17. Deliverables

## Phase 0 Deliverables

* @x402-core package
* @x402-nest package
* @x402-next package
* Strict-mode-only middleware
* Basic README

## Phase 1 Deliverables

* Prisma/Postgres storage adapter
* Redis adapter
* Usage dashboard example

## Phase 2 Deliverables

* Credit system
* Reconciliation worker
* MQ integration package

---

# 18. Appendix A — Terminology

* **Buyer**: Wallet paying USDC.
* **Seller**: Wallet receiving payment.
* **Facilitator**: Verifies payment & ensures correctness.
* **Resource Server**: API server using this middleware.
* **Payment Requirement**: The “invoice” for an API call.
* **Payment Credential**: Proof of payment included in retry request.

---

# 19. PRD Status

✓ Approved by DarkhorseOne
✓ Suitable for open-source publication
✓ Ready for architecture & code skeleton phase

---

If you want, I can now generate:

✅ The full folder structure (monorepo layout)
✅ The initial TypeScript interfaces
✅ The NestJS/Next.js boilerplate code
✅ The Prisma schema for Phase 1
✅ The fallback strategy implementation blueprint

只需要告诉我：**“生成代码骨架”** 或告诉我要哪个模块开始。
