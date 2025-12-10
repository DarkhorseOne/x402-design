# `@darkhorseone/x402-core`

## Development Standards (Phase 0)

---

# 1. Language & Runtime

### **Language**

* **TypeScript only**
* Version: **TS ≥ 5.0**

### **Node.js Target**

* **Node.js ≥ 20**

### **Output Format**

* ES2022 modules
* Compiler config:

```json
{
  "target": "ES2022",
  "moduleResolution": "bundler",
  "strict": true,
  "noImplicitAny": true
}
```

---

# 2. Coding Style

### **General**

* Functional and stateless design
* Prefer pure functions when possible
* Avoid side effects unless necessary (e.g., logging)
* Avoid complex inheritance; prefer composition and interfaces

### **Naming Conventions**

* **Classes**: `PascalCase`
* **Interfaces**: `PascalCase` + prefix with `I` only if distinction needed
* **Functions**: `camelCase`
* **Constants**: `UPPER_SNAKE_CASE`

### **File Structure**

* One class/interface per file
* Group by domain (`/config`, `/errors`, `/payment`, `/facilitator`)

### **Error Handling**

* Always throw custom error classes from `/errors`
* Never throw raw strings or generic `Error` unless unavoidable

---

# 3. Documentation Standards

### **DocBlocks**

Every public class, interface, and function must include:

* Description
* Parameter descriptions
* Return type descriptions
* Error behavior

Example:

```ts
/**
 * Validates an x402 payment credential using the facilitator.
 * @throws PaymentNetworkError if facilitator is unreachable
 */
```

### **README Requirements**

* Clear explanation of purpose
* Usage examples
* API documentation for all exported components

---

# 4. Testing Standards

### **Framework**

* Vitest or Jest (project preference: Vitest)

### **Coverage Requirements**

* All core functionality must be covered:

  * Payment requirement generation
  * Payment credential extraction
  * Facilitator verification mapping
  * Error classes

### **Test Style**

* AAA (Arrange, Act, Assert)
* No mocked implementation details unless required (e.g., facilitator mock)

---

# 5. Package Boundaries

### **Forbidden**

* No NestJS imports
* No Next.js imports
* No external dependencies unless absolutely required
* MUST remain framework-agnostic

### **Allowed**

* Optional peer dependency: facilitator SDK

---

# 6. Security & Validation

* Never log API keys or secrets
* Validate all external input
* Ensure PaymentRequirement fields conform to x402 standard
* Ensure nonce generation uses cryptographically secure methods

---

# 7. AI Coding Agent Notes

### **When writing code:**

* Follow interface-first design
* Avoid unnecessary complexity
* Keep modules stateless
* Use explicit return types
* Always include minimal example in JSDoc when possible

### **When generating new files:**

* Use consistent folder naming
* Include exports in `index.ts` automatically
