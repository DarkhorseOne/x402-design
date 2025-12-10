# `@darkhorseone/x402-nest`

## Phase 0 — Technical Design & Development Document

### Version: 1.0.0

### License: MIT (© DarkhorseOne Limited)

---

# 1. Project Overview

`@darkhorseone/x402-nest` provides the NestJS adapter for the DarkhorseOne x402 Middleware system. It integrates the **framework-agnostic logic** from `@darkhorseone/x402-core` into the NestJS ecosystem using:

* **Decorator** (`@X402Charge`)
* **Guard** (`X402Guard`)
* **Interceptor** (`X402Interceptor`)
* **NestJS Module** (`X402Module`)

This package enables API endpoints to enforce **per-call payment** via the x402 protocol.
When a client calls an endpoint protected by `@X402Charge`, the guard:

1. Extracts the payment credential
2. Validates it with the facilitator (via x402-core)
3. Either:

   * Allows the request to proceed
   * Or throws a `PaymentRequiredError` → which maps to HTTP **402 Payment Required**

**Phase 0 is strictly stateless:**
❌ No DB
❌ No Redis
❌ No MQ
❌ No credit-based fallback

Strict fallback mode is always **deny**.

---

# 2. Phase 0 Scope & Objectives

## 2.1 Included in Phase 0

* NestJS decorator for declaring chargeable endpoints
* Guard implementing the request-level payment decision
* Interceptor providing logging hooks
* Global module configuration
* Conversion of x402-core errors → NestJS HTTP responses
* Strict-mode enforcement (no fallback)

## 2.2 Excluded from Phase 0 (Future Phases)

* Request logging persistence
* Usage tracking
* Redis caching
* MQ event emitting
* Credit fallback logic
* Tenant-level rules (Phase 1+)
* Token/byte metering (Phase 1+)

Phase 0 keeps the middleware minimal, functional, and secure.

---

# 3. Architecture

```
@darkhorseone/x402-nest
│
├── decorators/
│   └── X402Charge.decorator.ts
│
├── guards/
│   └── X402Guard.ts
│
├── interceptors/
│   └── X402Interceptor.ts
│
├── module/
│   └── X402Module.ts
│
├── adapters/
│   └── NestRequestAdapter.ts
│
├── responses/
│   └── X402ExceptionFilter.ts
│
└── index.ts
```

This package depends on:

* NestJS core packages
* `@darkhorseone/x402-core`

---

# 4. Technical Requirements

## 4.1 NestJS Version Support

* **NestJS v10+**

Should work for both:

* Express adapter
* Fastify adapter

## 4.2 Dependencies

| Type     | Packages                                             |
| -------- | ---------------------------------------------------- |
| Required | `@nestjs/common`, `@nestjs/core`, `reflect-metadata` |
| Required | `@darkhorseone/x402-core`                            |

No optional dependencies in Phase 0.

---

# 5. Public API (Phase 0)

```
@X402Charge(options)        // Decorator for endpoints
X402Guard                   // Guard that checks payment
X402Interceptor             // Interceptor for logging hooks
X402Module                  // Module for configuration
X402ExceptionFilter         // Maps x402-core errors → HTTP responses
```

---

# 6. Detailed Component Design

## 6.1 Decorator: `@X402Charge()`

### Purpose

Used to annotate controller methods or classes.
Defines the **payment requirement** for the endpoint.

### Example

```ts
@Controller("report")
export class ReportController {
  @Get()
  @X402Charge({
    price: "0.02",
    asset: "USDC",
    network: "base-mainnet",
    description: "Generate report"
  })
  generateReport() {
    return { ok: true };
  }
}
```

### Implementation Notes

* Uses `Reflect.defineMetadata()`
* Metadata key: `"x402:charge"`
* Options interface:

```ts
export interface X402ChargeOptions {
  price: string;
  asset?: string;           // Defaults to config.defaultAsset
  network?: string;         // Defaults to config.defaultNetwork
  description?: string;
}
```

The `X402Guard` reads this metadata.

---

## 6.2 Guard: `X402Guard`

### Responsibilities

1. Read metadata from the decorator
2. Extract payment credential via `PaymentCredentialExtractor`
3. Build PaymentRequirement via core
4. Verify payment via facilitator client
5. Handle strict fallback: if facilitator fails → throw PaymentNetworkError
6. Throw `PaymentRequiredError` if payment is missing/invalid
7. Attach payment context to the request

### Pseudocode

```ts
canActivate(context) {
  const req = context.switchToHttp().getRequest();

  if (!isChargeableEndpoint(context)) return true;

  const credential = extractor.extract(req);

  if (!credential) {
    const requirement = builder.build(options);
    throw new PaymentRequiredError(requirement);
  }

  const verification = await verifier.verify(credential, requirement);

  if (verification.status !== "success") {
    throw new PaymentRequiredError(requirement);
  }

  req.x402 = { verification, requirement };
  return true;
}
```

### Behavior Rules (Phase 0)

* Facilitator timeout → `PaymentNetworkError`
* Guard must **not** allow execution on any failure
* Errors propagate to `X402ExceptionFilter`

---

## 6.3 Interceptor: `X402Interceptor`

### Responsibilities

* Logging hook (console or custom logger)
* Attach a request ID if one isn't present
* Allow request execution to proceed normally

### Example Behavior

```ts
intercept(context, next) {
  const req = context.switchToHttp().getRequest();
  logger.info("x402 verified", req.x402);
  return next.handle();
}
```

### No persistence in Phase 0.

---

## 6.4 Module: `X402Module`

### Purpose

* Register global X402Config
* Provide dependency injection tokens
* Allow async configuration with `.forRootAsync()`

### Example Usage

```ts
@Module({
  imports: [
    X402Module.forRoot({
      facilitatorUrl: "...",
      sellerWallet: "...",
      defaultAsset: "USDC",
      defaultNetwork: "base-sepolia",
      fallbackMode: "deny"
    })
  ]
})
export class AppModule {}
```

### DI Tokens

```
X402_CONFIG
X402_FACILITATOR_CLIENT
X402_PAYMENT_REQUIREMENT_BUILDER
X402_PAYMENT_CREDENTIAL_EXTRACTOR
```

---

## 6.5 Exception Filter: `X402ExceptionFilter`

Maps x402-core errors → appropriate HTTP responses.

### Rules

| Error type           | HTTP Response               |
| -------------------- | --------------------------- |
| PaymentRequiredError | **402 Payment Required**    |
| PaymentInvalidError  | 402                         |
| PaymentExpiredError  | 402                         |
| PaymentNetworkError  | **503 Service Unavailable** |
| X402ConfigError      | 500                         |

### Response Body Format

```json
{
  "error": "payment_required",
  "payment": { ...payment requirement... },
  "message": "Payment required to access this resource."
}
```

---

# 7. Request Context Extensions

Add typing support:

```ts
declare module "http" {
  interface IncomingMessage {
    x402?: {
      verification: PaymentVerificationResult;
      requirement: PaymentRequirement;
    };
  }
}
```

---

# 8. Development Tasks (for Codex)

## 8.1 Implement Decorator

* Create metadata attachment
* Define `X402ChargeOptions`

## 8.2 Implement Guard

* Inject dependencies
* Extract metadata
* Extract credentials
* Build requirement
* Call core verifier
* Throw proper errors
* Attach `req.x402`

## 8.3 Implement Interceptor

* Logging
* Pass-through routing

## 8.4 Implement Module

* `forRoot` and `forRootAsync`
* Provide DI tokens
* Wire up core services

## 8.5 Implement Exception Filter

* Map errors → HTTP exceptions
* Format JSON body correctly

## 8.6 Add Unit Tests

* Decorator metadata tests
* Guard scenarios:

  * No credential → 402
  * Invalid credential → 402
  * Facilitator timeout → 503
  * Valid → success
* Module config tests
* Exception filter tests

---

# 9. Example Usage (Phase 0)

```ts
@Controller("ai")
export class AIController {
  @Post("generate")
  @X402Charge({ price: "0.05", description: "AI text generation" })
  generate() {
    return { text: "Hello" };
  }
}
```

Middleware behavior:

* Missing payment → 402 with requirement
* Paid request → passes to controller
* Facilitator down → 503 (strict mode)

---

# 10. File Structure

```
/src
  /decorators
  /guards
  /interceptors
  /module
  /responses
  /adapters
  index.ts
/test
package.json
tsconfig.json
README.md
```

---

# 11. License

MIT License
© DarkhorseOne Limited

---

# 12. Phase 0 Status

* ✔ Design complete
* ✔ Architecture stable
* ✔ Safe to begin implementation

