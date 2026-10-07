# AGENTS.md

NestJS 12 B2B auth boilerplate (`pnpm`, ESM, TypeScript 6 strict). Repo is currently the **stock starter** — the planned architecture lives in `docs/` and is not implemented yet.

## Commands (verified)

```bash
pnpm install
pnpm lint                 # oxlint --type-aware src/ test/ (NOT eslint)
pnpm test                 # unit: only **/*.spec.ts
pnpm test:e2e             # e2e: only **/*.e2e-spec.ts (separate vitest config)
pnpm exec tsc --noEmit    # typecheck — there is NO `typecheck` script
pnpm run build            # nest build -> dist/main.js
pnpm exec prettier --check "src/**/*.ts" "test/**/*.ts"   # `pnpm format` writes, it does not check
```

- Package manager is `pnpm` (`pnpm-lock.yaml`); some docs/Dockerfile snippets show `npm` — ignore them.
- Single test: `pnpm test <path/to/file.spec.ts>`.
- Single e2e: `pnpm exec vitest run --config ./vitest.config.e2e.ts <path/to/file.e2e-spec.ts>`.
- Order before finishing: `lint -> tsc --noEmit -> test -> test:e2e`.

## Spec workflow (repo skills)

- Large features go through `/spec` (asks clarifying questions, writes `specs/NN-slug.md`) then `/spec-impl` (implements it step by step). Skills live in `.agents/skills/`; `skills-lock.json` tracks their upstream hashes.
- `/spec-impl` only runs when the spec state means "Approved"/"Aprobado" and creates branch `spec-NN-slug`. It **never commits** — committing is always the user's call.
- `specs/` does not exist yet; `/spec` seeds it along with `specs/.spec-config.yml` (`AutoCreateBranch: true` default).

## Non-negotiable constraints (from `docs/spec_and_req.md`)

The SRS is binding; violating these is a bug even if code compiles:

- `argon2` for password hashing — **bcrypt is forbidden**. 2FA recovery codes use **SHA-256** (not argon2) to protect the event loop.
- `nestjs-pino` for all logging — native Nest logger forbidden in production; logs must carry `traceId`/`userId`/`organizationId`.
- `@nestjs/throttler` **must** use `nestjs-throttler-storage-redis` — in-memory storage forbidden.
- `npx prisma migrate deploy` never runs at app boot (not in `main.ts`, not in Docker `CMD`) — only CI/CD or InitContainer.
- Swagger only when `NODE_ENV === 'development'`; global `ValidationPipe` with `whitelist`, `forbidNonWhitelisted`, `transform`.
- Tenant isolation is enforced by a Prisma Client Extension + `AsyncLocalStorage` (`x-organization-id`), never by hand-written `where` clauses.
- Multi-table writes go through Prisma `$transaction`. Uploads validated by magic numbers (`file-type`), never by extension.

## Toolchain quirks

- **ESM strict:** `"type": "module"` + `moduleResolution: nodenext`. Every relative import needs a `.js` extension (`./app.module.js`), even from `.ts` files.
- **Lint config:** `.oxlintrc.json` turns `typescript/no-explicit-any` **off** and `no-floating-promises` **error**. Prettier: single quotes, trailing commas.
- Vitest `globals: true` (no import of `describe`/`it` needed); path aliases resolved by `vite-tsconfig-paths`.
- `dist/main.js` is the real build output (`tsconfig.build.json` sets `rootDir: ./src`). `docs/fase-1.md` says `dist/src/main.js` — trust the build, not the doc.

## Docs are the spec (all in Spanish)

- `docs/spec_and_req.md` — SRS: data model, security rules, API standards. Source of truth for requirements.
- `docs/plan.md` — 7-phase execution plan (Fase 1 → 7). Read the phase you are implementing.
- `docs/fase-1.md` — detailed next-work plan: deps, `.env`, `docker-compose.yml` (Postgres 15 + Redis 7), Dockerfile, `main.ts`/`app.module.ts` hardening.
- Known doc conflicts: SRS says unit tests use **Jest**, but the repo runs **Vitest** — use Vitest.

## Current state / gotchas

- No CI, no pre-commit hooks, no `opencode.json`. Nothing enforces checks automatically.
- `src/app.module.ts` ships Observe placeholders (`YOUR_APP_KEY`/`YOUR_APP_SECRET`); `src/main.ts` passes `ObserveInstrument` to `NestFactory.create`. Leave both wiring intact when editing bootstrap.
- `.env` / `.env.example` / `docker-compose.yml` do not exist yet (Fase 1 work). Do not assume env vars are loaded — `ConfigModule` is not installed yet.
- README is stock NestJS text plus a Spanish section on client auth strategy (BFF cookies for web, secure storage for mobile) — that strategy still applies when building `AuthModule`.
