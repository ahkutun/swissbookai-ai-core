<!-- ai-core map: generated 2026-09-22 from 0db4830; the section "Rules of this repository" is written by people and kept -->
# swissbookai — the map

## What it is
SwissBooks AI is a self-hosted accounting platform for Swiss fiduciary offices: receipt OCR, double-entry bookkeeping with maker-checker review, VAT and financial reports, bank import (camt.053), cloud receipt sync, and LLM-assisted posting. It is used by multi-staff trustee firms managing many client mandates (Mandanten), by the clients through a portal, and by the operator through a billing overview. It is not a payment processor and not tied to one database engine: services depend on ports, never directly on Drizzle.

## Shape
- `apps/api/` — NestJS 11 on Fastify, prefix `/api/v1`; entry `apps/api/src/main.ts`, modules wired in `apps/api/src/app.module.ts`, one feature per `apps/api/src/modules/<feature>/`
- `apps/web/` — Next.js 15 App Router; route groups `apps/web/app/(staff)`, `(client)`, `(auth)`; all API calls through `apps/web/lib/api.ts`
- `apps/worker/` — BullMQ processors; entry `apps/worker/src/main.ts`, one queue per `apps/worker/src/processors/*.processor.ts`
- `packages/schemas/` — Zod DTOs and enums, the single source of truth for API and web
- `packages/db/` — Drizzle schema `packages/db/src/schema/*`, migrations `packages/db/drizzle/`, RLS and seed under `packages/db/src/scripts/` and `packages/db/sql/`
- `packages/ports/` — persistence contracts (`KontoRepository`, `NumberAllocator`); adapters live in `packages/db/src/repositories/`
- `packages/swiss/` — CH locale: money arithmetic, VAT, chart of accounts, cantons, QR-bill, eCH-0217, pain.001
- `packages/finance/` — country-neutral analysis (scoring, valuation, forecast, reconciliation)
- `packages/llm/`, `packages/ai-prompts/` — multi-provider LLM client; prompts, output schemas, `AiTask`, `TASK_TIER` and prices in `packages/ai-prompts/src/models.ts`
- `packages/i18n/` — messages for de, en, fr, it in `packages/i18n/src/messages/`
- `packages/{cloud,email,secrets,storage,config}/` — OAuth, mail, AES-256-GCM at rest, FS/S3 storage, shared lint config
- `deploy/`, `release/`, `.github/workflows/release.yml` — Helm chart, Caddy, compose; `release/release.sh` mints the release tag and pushes the deploy ref
- `scripts/` — `check.sh` (`check.ps1` on Windows) is the one check entry point; `dev-up.mjs` (pnpm start), `smoke.mjs`, `backup.sh`
- `docs/`, `ARCHITECTURE.md`, `CLAUDE.md`, `README.md` — decisions, rules, setup

## Build, check, run
docker compose up -d
pnpm install
cp .env.example .env
pnpm db:reset
pnpm dev
scripts/check.sh
pnpm typecheck
pnpm lint
pnpm test
pnpm build
pnpm smoke
pnpm db:generate
pnpm db:migrate
pnpm db:deploy
pnpm --filter @swissbooks/api test:e2e
pnpm --filter @swissbooks/web test:e2e
There is no CI check pipeline. `scripts/check.sh` runs the four gates (typecheck, lint, test, build) in order, stops at the first red one, and is what the pre-push hook runs. `pnpm db:reset` drops the schema and is not a check. The only workflow is `release.yml`, dispatched manually with version, channel and stage.

## Where to add things
| what is added | where it goes and where it is registered |
|---|---|
| API endpoint | Zod schema in `packages/schemas/src/`, DTO via `createZodDto` in the module controller, logic in the module service; mandate-bound routes under `mandanten/:mandantId/...` with `MandantAccessGuard` |
| API feature module | `apps/api/src/modules/<feature>/{controller,service,module}.ts`, imported in `apps/api/src/app.module.ts` |
| DB table or column | `packages/db/src/schema/<aggregate>.ts` exported from `schema/index.ts`; run `pnpm db:generate`, read and commit the file in `packages/db/drizzle/`; `pnpm db:rls` after any push |
| New aggregate needing engine independence | port in `packages/ports/src/`, Postgres and Mongo adapters in `packages/db/src/repositories/`, pattern `KontoRepository` |
| Background job | `apps/worker/src/processors/<name>.processor.ts` exporting `create<Name>Worker`, started in `apps/worker/src/main.ts`; enqueue method in `apps/api/src/queue/queue.service.ts`; queue names in `packages/schemas/src/jobs.ts` |
| AI feature | prompt and output schema in `packages/ai-prompts/src/`, task in `AiTask` and `TASK_TIER` in `packages/ai-prompts/src/models.ts`, called via `AiService.run` or `.complete` in `apps/api/src/modules/ai/` |
| Environment variable | `apps/api/src/config/env.ts` (and `apps/worker/src/config.ts`, `apps/web/next.config.mjs` if used there) plus `.env.example` in the same change |
| Web page | `apps/web/app/(staff)/<route>/page.tsx` or `(client)`; data through `apps/web/lib/api.ts` and TanStack Query |
| UI text | same key in all four files under `packages/i18n/src/messages/`; edit surgically, never run prettier on them |
| Swiss-specific rule (VAT, cantons, accounts) | `packages/swiss/src/`; core code stays currency and country neutral |
| Secret stored at rest | encrypt through `@swissbooks/secrets` via `apps/api/src/crypto/crypto.service.ts` |
| Shared enum or DTO | `packages/schemas/src/enums.ts` or the domain file next to it |

## Rules of this repository
(none yet: written by people, kept on every regeneration)
