# 1. Language & Runtime

### **Language**

* **TypeScript**

### **Framework**

* NestJS **v10+**

### **Runtime**

* Node.js **20+**

---

# 2. Coding Style

### **NestJS Conventions**

* Use Decorators (`@Injectable`, `@Module`, `@UseInterceptors`, etc.)
* Use DI tokens defined in `constants.ts`
* Avoid static logic inside guards/interceptors

### **File Naming**

* Decorators → `*.decorator.ts`
* Guards → `*.guard.ts`
* Interceptors → `*.interceptor.ts`
* Filters → `*.filter.ts`
* Modules → `*.module.ts`

### **Class Organization**

* Constructor should receive all required dependencies via DI
* Avoid constructing core components manually; inject via module

---

# 3. Documentation Standards

### **Required DocBlocks**

Every exported class or function must describe:

* Purpose of component
* Expected behavior
* Errors thrown
* Example usage

Example:

```ts
/**
 * NestJS Guard that enforces x402 payment flow.
 * Throws PaymentRequiredError on invalid/missing payment.
 */
```

### **README Requirements**

* Example NestJS module setup
* Example usage with @X402Charge
* Explanation of strict fallback mode

---

# 4. Testing Standards

### **Framework**

* Jest (Nest default) or Vitest

### **Required Tests**

* Decorator metadata extraction
* Guard behavior:

  * Missing credential → 402
  * Invalid credential → 402
  * Facilitator failure → 503
  * Successful payment passes guard
* ExceptionFilter mapping tests

### **Mocking Rules**

* FacilitatorClient MUST be mocked
* Use NestJS testing module for DI testing

---

# 5. Package Boundaries

### **Forbidden**

* No direct reference to HTTP framework internals
* No business logic inside NestJS adapter
* Do not import Express/Fastify types directly

### **Allowed**

* Import from `@nestjs/common`, `@nestjs/core`
* Import from `@darkhorseone/x402-core`

---

# 6. Error Handling

* Guard must throw **core errors**, not Nest exceptions
* ExceptionFilter is responsible for converting to HTTP responses
* Interceptor should NOT catch or swallow errors

---

# 7. Security & Validation

* Never expose facilitator API keys in logs
* Ensure the module config is validated at bootstrap
* Verify all payment data comes directly from core extraction logic

---

# 8. AI Coding Agent Notes

### **When writing guards/interceptors:**

* Always extract metadata with `Reflector`
* Never construct requirement manually → use core builder
* Never attempt payment verification manually → use injected verifier
* Keep guard logic minimal and deterministic

### **When writing module code:**

* Support `.forRoot()` and `.forRootAsync()`
* Ensure DI providers are correctly mapped
