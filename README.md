# DarkhorseOne x402 Middleware

TypeScript middleware implementing the x402 payment standard for API servers. Provides NestJS guards/interceptors and Next.js route wrappers that enforce per-call or usage-based micro-payments, returning HTTP 402 when payment is missing, invalid, or insufficient.

## Packages (planned)
- `packages/x402-core`: Framework-agnostic x402 logic, types, and facilitator wiring.
- `packages/x402-nest`: NestJS guard + interceptor + decorator (`@X402Charge`).
- `packages/x402-next`: Next.js App Router/Pages Router `x402Route` wrapper.
- `packages/x402-storage-*`: Optional adapters (Postgres/Prisma, Redis cache, MQ sinks) for Phase 1/2 features.
- `examples/`: Runnable samples for NestJS and Next.js.

## Status & Roadmap
- **Phase 0 (current)**: Stateless, strict verification via facilitator; no DB/Redis/MQ required; fallback mode `deny` by default.
- **Phase 1**: Optional storage/caching, observability, usage logs.
- **Phase 2**: Credit-based fallback, async reconciliation via MQ for enterprise tenants.

## Quick Start (planned APIs)
NestJS:
```ts
@Controller('report')
export class ReportController {
  @Get()
  @X402Charge({ price: '0.02', asset: 'USDC', network: 'base-mainnet' })
  handle() {
    return { status: 'ok' };
  }
}
```

Next.js (App Router):
```ts
export const POST = x402Route(
  { price: '0.05', asset: 'USDC', network: 'base-mainnet' },
  async () => new Response('hello world')
);
```

## Configuration (core)
```ts
interface X402Config {
  facilitatorUrl: string;
  apiKey?: string;
  defaultNetwork: string;
  defaultAsset: string;
  sellerWallet: string;
  fallbackMode: 'deny' | 'temporary_allow' | 'credit_based';
  tenantResolver?: (req) => string | null;
  storage?: X402Storage; // optional Phase 1+
  cache?: X402Cache;     // optional Phase 1+
}
```

## Development
- Install deps: `npm install` (single root lockfile).
- Lint/format: `npm run lint` then `npm run format`.
- Type-check: `npm run typecheck`.
- Tests: `npm test` or `npm run test:watch`; mock facilitator/network calls.
- Local samples: `npm run dev:nest`, `npm run dev:next` (see `examples/`).
- Build packages: `npm run build` before publish.

## Security & Reliability
- Enforce HTTPS to facilitator; never commit secrets—use `.env.example` with `FACILITATOR_URL`, `SELLER_WALLET`, etc.
- Implement replay protection via cache/DB (paymentId dedupe).
- Use schema validation (e.g., Zod) for request payloads and facilitator responses.
- Default fallback is `deny`; other modes require storage/MQ and should be gated per tenant.

## License
MIT (© DarkhorseOne Limited).
