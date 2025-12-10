# `@darkhorseone/x402-core`

## Phase 0 — Technical Design & Development Document

### Version: 1.0.0

### License: MIT (© DarkhorseOne Limited)

---

# 1. Project Overview

`@darkhorseone/x402-core` is the framework-agnostic foundation of the DarkhorseOne x402 Middleware ecosystem.
This package provides all protocol-level logic required to protect API endpoints using the **x402 HTTP payment standard**, including:

* Building **x402 Payment Requirements** for HTTP `402 Payment Required`
* Extracting **payment credentials** from incoming requests
* Verifying payments using a **facilitator API** (e.g., Coinbase x402)
* Defining **shared types** and **error classes**
* A configurable **stateless core service**
* Extensible design for future phases (storage, redis, MQ, credit system)

**Phase 0 is strictly stateless:**
❌ No database
❌ No Redis
❌ No message queue
❌ No credit fallback

This package serves as the minimal functional backbone for all framework adapters (NestJS, Next.js, etc.).

---

# 2. Phase 0 Scope & Objectives

## 2.1 Included in Phase 0

* Protocol-compliant **Payment Requirement generation**
* **Payment credential extraction**
* **Payment verification** using facilitator API/SDK
* Strict fallback mode: **fallbackMode = "deny"**
* Typed error system (PaymentRequiredError, PaymentInvalidError, etc.)
* Optional logger interface
* Config + tenant resolver function
* Clean, framework-independent API

## 2.2 Not Included in Phase 0 (Future Phases)

These features are intentionally excluded in v0:

* Storage / persistence layer (Postgres, etc.)
* Redis caching
* Message queue integrations
* Credit-based fallback system
* Usage metering (token-based, byte-based, row-based)
* Multi-facilitator strategy
* Subscription + pay-per-use hybrid billing

---

# 3. Architecture

```
@darkhorseone/x402-core
│
├── config/
│   └── X402Config.ts
│
├── payment/
│   ├── PaymentRequirementBuilder.ts
│   ├── PaymentCredentialExtractor.ts
│   └── PaymentVerifier.ts
│
├── facilitator/
│   ├── X402FacilitatorClient.ts (Interface)
│   └── HttpFacilitatorAdapter.ts
│
├── errors/
│   ├── X402Error.ts
│   ├── PaymentRequiredError.ts
│   ├── PaymentInvalidError.ts
│   ├── PaymentExpiredError.ts
│   └── PaymentNetworkError.ts
│
├── utils/
│   ├── logger.ts
│   └── request-id.ts
│
└── index.ts
```

All framework-specific implementations (NestJS or Next.js) must consume this package.

---

# 4. Technical Requirements

## 4.1 TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "noImplicitAny": true,
    "esModuleInterop": true
  }
}
```

## 4.2 Node.js Target

* **Node.js ≥ 20**

## 4.3 Package Dependencies (Phase 0)

| Type     | Package                                |
| -------- | -------------------------------------- |
| Required | None (pure TS implementation)          |
| Optional | Coinbase x402 SDK (as peer dependency) |

This package must **not** enforce any external SDK dependency; instead, it should define an interface-based adapter.

---

# 5. Core Functionality Specifications

# 5.1 Configuration Structure

```ts
export interface X402Config {
  facilitatorUrl: string;       // Required
  apiKey?: string;              // Optional (depends on facilitator)
  defaultNetwork: string;       // e.g., "base-mainnet"
  defaultAsset: string;         // e.g., "USDC"
  sellerWallet: string;         // Wallet address to receive payments
  fallbackMode: "deny";         // Phase 0 = strict mode only
  tenantResolver?: (req: unknown) => string | null;
  logger?: X402Logger;
}
```

### Phase 0 Enforcement

`fallbackMode` MUST always be `"deny"`.

---

# 5.2 Payment Requirement Generation

A Payment Requirement is the "invoice" a service returns when rejecting access with HTTP 402.

```ts
export interface PaymentRequirement {
  amount: string;
  asset: string;
  network: string;
  seller: string;
  facilitator: string;
  expiresAt: string;
  nonce: string;
  description?: string;
}
```

Builder:

```ts
export class PaymentRequirementBuilder {
  constructor(private config: X402Config) {}

  build(params: PaymentParams): PaymentRequirement;
}
```

---

# 5.3 Credential Extraction

Supports extracting from:

* HTTP header `x402-payment`
* JSON body: `payment`
* Query parameter: `payment`

Class:

```ts
export class PaymentCredentialExtractor {
  extract(req: unknown): PaymentCredential | null;
}
```

---

# 5.4 Payment Verification

Interface:

```ts
export interface PaymentVerifier {
  verify(
    credential: PaymentCredential,
    requirement: PaymentRequirement
  ): Promise<PaymentVerificationResult>;
}
```

Result:

```ts
export type PaymentVerificationStatus =
  | "success"
  | "insufficient"
  | "invalid"
  | "expired"
  | "network_error";

export interface PaymentVerificationResult {
  status: PaymentVerificationStatus;
  txHash?: string;
  walletAddress?: string;
}
```

---

# 5.5 Facilitator Client Abstraction

```ts
export interface X402FacilitatorClient {
  verifyPayment(
    credential: PaymentCredential
  ): Promise<PaymentVerificationResult>;

  createRequirement(params: PaymentParams): PaymentRequirement;
}
```

Phase 0 implementation:

```ts
export class HttpFacilitatorAdapter implements X402FacilitatorClient {
  constructor(private config: X402Config) {}

  async verifyPayment(...) { ... }
  createRequirement(...) { ... }
}
```

---

# 6. Error Handling Specification

### Custom Error Classes

| Error Class            | Purpose                                             |
| ---------------------- | --------------------------------------------------- |
| `PaymentRequiredError` | Triggers a 402 response handler in upper frameworks |
| `PaymentInvalidError`  | Payment does not match requirement                  |
| `PaymentExpiredError`  | Payment expired                                     |
| `PaymentNetworkError`  | Facilitator timeout / unavailability                |
| `X402ConfigError`      | Startup misconfiguration                            |

### Phase 0 Rules

* Facilitator failure → **PaymentNetworkError**
* Middleware adapters MUST block request execution
* Framework adapters (Nest/Next) handle translation into HTTP errors

---

# 7. Logging Requirements

Interface:

```ts
export interface X402Logger {
  info(message: string, meta?: any): void;
  warn(message: string, meta?: any): void;
  error(message: string, meta?: any): void;
}
```

Default implementation uses `console`.

---

# 8. Development Plan (for Codex)

## 8.1 Implement All Public Interfaces

* `X402Config`
* `PaymentRequirement`
* `PaymentCredential`
* `PaymentVerificationResult`

## 8.2 Implement Core Classes

* `PaymentRequirementBuilder`
* `PaymentCredentialExtractor`
* `PaymentVerifier` (interface only)
* `HttpFacilitatorAdapter`
* `X402CoreService`

## 8.3 Error Classes

* Implement full list of typed errors

## 8.4 Utility Modules

* `logger.ts`
* `request-id.ts`

## 8.5 Required Unit Tests

* Payment requirement generation
* Credential extraction parsing
* Verification output handling
* Error throwing conditions
* Config validation

## 8.6 Deliverables

```
/src
/tests
/dist
/README.md (or this document)
package.json
tsconfig.json
```

---

# 9. Public API (index.ts)

```ts
export * from "./config/X402Config";
export * from "./payment/PaymentRequirementBuilder";
export * from "./payment/PaymentCredentialExtractor";
export * from "./payment/PaymentVerifier";
export * from "./facilitator/X402FacilitatorClient";
export * from "./errors";
export * from "./utils/logger";
```

---

# 10. License

MIT License
© DarkhorseOne Limited

---

# 11. Phase 0 Status

* ✔ Architecture final
* ✔ Spec ready for implementation
* ✔ Suitable for Codex / Cursor task generation

