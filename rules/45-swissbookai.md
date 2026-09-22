## The rules of swissbookai

The product is a self-hosted bookkeeping platform for Swiss fiduciary offices, a monorepo of
`apps/api` (NestJS on Fastify), `apps/web` (Next.js) and `apps/worker` (BullMQ) with the packages
under `packages/@swissbooks/*`. These rules come from the repository's own `CLAUDE.md` and
`ARCHITECTURE.md`; where one of them is decided differently there, that file is updated in the
same change.

- **The database engine stays replaceable.** No trigger, no view, no load-bearing row-level
  security; every invariant (the double-entry balance, `updated_at`, the numbering) lives in the
  application layer, and a service depends on a port from `@swissbooks/ports`, never on Drizzle
  directly. [review]
- **The core knows no currency and no country.** No `CHF` and no "Swiss" in a core name or in core
  logic; money arithmetic is named neutrally (`round`, `sum`, `add`, `sub`, `mul`, `cmp`), and what
  is Swiss (VAT, the SME chart of accounts, the cantons, the QR bill) lives in `@swissbooks/swiss`.
  [review]
- **Money is `NUMERIC(15,2)` in the database and `decimal.js` in code**, a string over the wire and
  `moneySchema` in a schema; never `float`, never `number`. [review]
- **A schema change is a migration, and data never disappears in silence.** `pnpm db:generate`
  writes the migration, which is read and committed with the change; `check-migrations` stops a
  migration that carries `DROP TABLE`, `DROP COLUMN`, `TRUNCATE` or a type change unless
  `I_ACCEPT_DATA_LOSS=yes` says the loss is meant; `pnpm db:push` and `pnpm db:reset` run against
  a local database only, never against one that holds somebody's data. [machine · review]
- **`.env.example` changes in the same step as the configuration it mirrors**: `apps/api/src/config/env.ts`,
  `apps/worker/src/config.ts` and `apps/web/next.config.mjs`. It is the canonical, copy-ready
  template. [review]
- **Operator billing is an overview, never a paywall on the core.** VAT settlement, the annual
  statement and the PDF exports are part of every plan and never appear as paid add-ons; only the
  bankability report, the company valuation and the exit preparation are paid; payment processing
  stays out. [review]
- **A model list comes live from the provider, never from code.** `AiService.listModels` behind
  `GET /ai/models` is the only source; the `PRICES` table in `packages/ai-prompts/src/models.ts`
  serves the cost metric alone, and an unpriced model costs 0 there. [review]
- **`@swissbooks/schemas` is the one source of DTOs and enums**, imported by the frontend and the
  backend alike; a new API input is a Zod schema there, made a DTO in the controller with
  `createZodDto`. [review]
- **One feature is one NestJS module** under `src/modules/<feature>/` with a thin controller and the
  logic in the service; an injectable is imported with a plain `import`, never `import type`,
  because the runtime metadata the injection needs is not emitted otherwise. [review]
- **Every call from the web app to the API goes through `apps/web/lib/api.ts`.** [review]
- **The four locales carry identical keys**, `packages/i18n/src/messages/{de,en,fr,it}.ts`: a key is
  added or removed in all four, and these files are edited surgically, never formatted with
  `prettier`. [machine · review]
- **A sensitive field at rest is encrypted through `@swissbooks/secrets`** (AES-256-GCM under
  `APP_ENCRYPTION_KEY`), as the SMTP URL and the cloud OAuth tokens are. [review]
- **Identifiers are English; texts a person reads are German**: UI texts, error messages and
  comments, the project's standard. For this repository this rule replaces the sentence of
  the section "Documentation and comments" that asks for English on disk; the rest of that section
  holds. [review]
- **A commit subject is a Conventional Commit** (`feat(scope): ...`, `fix(...)`, `refactor(...)`,
  `chore(...)`) and, as everywhere, names its issue `#<n>`. [review]
