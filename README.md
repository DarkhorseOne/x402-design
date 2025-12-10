# DarkhorseOne x402 Middleware

https://github.com/DarkhorseOne/x402-design

TypeScript middleware implementing the x402 payment standard for API servers. Provides NestJS guards/interceptors and Next.js route wrappers that enforce per-call or usage-based micro-payments and return HTTP 402 when payment is missing, invalid, or insufficient. All requirements live in `docs/x402-middleware-prd.md`; roadmap phases are mirrored in `docs/Phase0`, `docs/Phase1`, and `docs/Phase2`.

## Packages (planned)
- `packages/x402-core`: Framework-agnostic x402 logic, types, facilitator wiring.
- `packages/x402-nest`: NestJS guard + interceptor + decorator (`@X402Charge`).
- `packages/x402-next`: Next.js App Router/Pages Router `x402Route` wrapper.
- `packages/x402-storage-*`: Optional adapters (Postgres/Prisma, Redis cache, MQ sinks) for Phase 1/2 features.
- `examples/`: Runnable samples for NestJS and Next.js.

## Roadmap (see PRD §4)
- **Phase 0 (current)**: Stateless, strict facilitator verification; no DB/Redis/MQ; fallback `deny`.
- **Phase 1**: Optional storage + caching, observability, usage/payment logs, Prisma adapter.
- **Phase 2**: Credit-based fallback, async reconciliation via MQ, enterprise gating.

## Planned API Examples
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

## Core Configuration (PRD §11)
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

## Security & Reliability (PRD §14)
- Enforce HTTPS to facilitator; never commit secrets—use `.env.example` with `FACILITATOR_URL`, `SELLER_WALLET`, etc.
- Add replay protection via cache/DB (paymentId dedupe) and schema validation (e.g., Zod).
- Default fallback is `deny`; other modes need storage/MQ and tenant gating.

## License
MIT (© DarkhorseOne Limited).
