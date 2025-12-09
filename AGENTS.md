# Repository Guidelines

## Project Structure & Module Organization
- `design/`: PRDs and architecture notes (see `x402-middleware-prd.md`). Keep specs updated before coding.
- Planned code layout follows the PRD: `packages/x402-core` (framework-agnostic logic), `packages/x402-nest` (NestJS guard/interceptor), `packages/x402-next` (Next.js route wrapper), `packages/x402-storage-*` for adapters. Place shared types in `packages/x402-core/src/types`.
- Use `examples/` for runnable reference apps (NestJS API, Next.js App Router). Keep assets (diagrams, mock payloads) in `design/assets/`.

## Build, Test, and Development Commands
- Install deps: `npm install` (root). Use a single lockfile at the repo root.
- Type-check: `npm run typecheck` (all packages via workspace script).
- Lint & format: `npm run lint` then `npm run format` (eslint + prettier). Fail CI on lint errors.
- Tests: `npm test` runs unit tests (Jest/Vitest). For watch mode, `npm run test:watch`.
- Local dev: `npm run dev:nest` (NestJS sample API) and `npm run dev:next` (Next.js sample route). Use `npm run build` before publishing packages.

## Coding Style & Naming Conventions
- Language: TypeScript `"strict"` mode; prefer functional utilities over inheritance.
- Indentation: 2 spaces; line width 100; use semicolons; single quotes in TS, double in JSON.
- Filenames: kebab-case for files, PascalCase for React components, camelCase for variables/functions, UPPER_SNAKE_CASE for env keys.
- Linting/formatting enforced via ESLint + Prettier; add minimal inline `// eslint-disable-next-line` with justification.

## Testing Guidelines
- Framework: Jest or Vitest with ts-jest/ts-node; place unit tests next to code as `*.spec.ts` or `*.test.ts`.
- Cover critical flows: payment requirement creation, facilitator verification, fallback modes, and NestJS/Next.js wrappers. Mock facilitator/network calls.
- Include integration tests in `packages/*/tests` for decorators/route wrappers. Target >80% coverage on core logic; fail CI below threshold.

## Commit & Pull Request Guidelines
- Follow Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`). Scope by package, e.g., `feat(core): add fallback deny mode`.
- Branch naming: `feature/<short-desc>` or `fix/<issue-id>`; keep PRs small and scoped.
- PR checklist: clear description, linked issue/reference, test plan with command outputs, screenshots for devtools/UI where relevant, note breaking changes and migration steps.
- Require at least one reviewer; do not merge with failing CI.

## Security & Configuration
- Never commit secrets; use `.env.example` with required keys (`FACILITATOR_URL`, `SELLER_WALLET`, etc.).
- Enforce HTTPS for facilitator calls; add replay protection (cache/DB) and input validation (Zod or equivalent).
- For optional Redis/MQ adapters, make external services configurable via env vars and document defaults in README.
