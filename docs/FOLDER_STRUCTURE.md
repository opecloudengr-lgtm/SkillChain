# SkillChain Production Monorepo Architecture

### Version 1.0 (Domain-Driven, Microservices-Ready)

This monorepo uses a **modular monolith with explicit bounded contexts**, structured so it can run as a single deployable today and split into independent microservices later — without a rewrite. It applies Clean Architecture layering inside every domain, keeps shared packages narrow and stable, and gives every business capability a clear owner.

Stack: Next.js 15, NestJS, Prisma, PostgreSQL, Hardhat, Solidity, ethers.js, Docker, GitHub Actions, Turborepo, pnpm.

---

## Design Principles

1. **Domain-driven, not layer-driven.** Code is grouped by business capability (Identity, Certificates, Marketplace, Wallet) not by technical type (controllers, services, DTOs at the top level).
2. **Clean Architecture inside every domain.** Domain logic never depends on infrastructure. Application use cases depend on ports (interfaces); infrastructure implements those ports as adapters.
3. **Deployable-ready boundaries.** `apps/services/*` map 1:1 onto `domains/*`. Extracting a microservice later means giving a folder its own pipeline and database — not moving code.
4. **Narrow, stable shared packages.** Nothing is shared "just in case." Shared kernel types are the few primitives every domain genuinely needs (Money, Result, Pagination). Everything else lives with its domain.
5. **Contracts over coupling.** Domains talk to each other through versioned `api-contracts` (sync) and `event-contracts` (async), never by importing each other's internals.
6. **Gateway as the stable frontend seam.** `apps/web` and `apps/admin` never call domain services directly — they call `apps/gateway`, which can keep its external contract constant even as the backend decomposes.
7. **Ownership is structural.** Folder boundaries map directly to `CODEOWNERS`, so review load and accountability scale with the org instead of everyone owning everything.

---

## High-Level Repository Structure

```text
skillchain/

├── apps/
│   ├── web/
│   ├── gateway/
│   ├── admin/
│   ├── explorer/
│   ├── docs-site/
│   └── services/
│       ├── identity/
│       ├── certificates/
│       ├── marketplace/
│       ├── wallet/
│       └── worker/
├── domains/
│   ├── identity/
│   ├── certificates/
│   ├── marketplace/
│   └── wallet/
├── packages/
│   ├── shared-kernel/
│   ├── ui/
│   ├── config/
│   ├── auth/
│   ├── blockchain/
│   ├── sdk/
│   ├── observability/
│   ├── feature-flags/
│   ├── event-contracts/
│   ├── api-contracts/
│   ├── validation/
│   └── testing/
├── blockchain-contracts/
├── infrastructure/
├── docker/
├── scripts/
├── docs/
├── tests/
├── .github/
├── .husky/
├── .vscode/
├── prisma/
├── config/
├── assets/
├── tmp/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
├── .editorconfig
├── .eslintrc
├── .prettierrc
├── .gitignore
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

# 1. `apps/` — Deployables

Purpose: everything that actually gets built, containerized, and shipped.

```text
apps/

├── web/
├── gateway/
├── admin/
├── explorer/
├── docs-site/
└── services/
    ├── identity/
    ├── certificates/
    ├── marketplace/
    ├── wallet/
    └── worker/
```

## apps/web/

Purpose: main Next.js frontend. Talks only to `apps/gateway` — never directly to a domain service.

```text
web/

├── app/
├── components/
├── hooks/
├── providers/
├── lib/
├── services/          # thin API clients calling the gateway, typed via packages/api-contracts
├── styles/
├── public/
├── middleware/
├── types/
├── constants/
├── utils/
├── store/
├── features/          # feature-based grouping: wallet, dashboard, learning, marketplace, certs
├── layouts/
└── tests/
```

Best practices
- Organize by feature, not by page.
- Keep UI components presentation-only; no business logic in components.
- All network calls go through `services/` clients built on `packages/api-contracts`.
- Prefer Server Components; use Client Components only when interactivity requires it.

## apps/gateway/

Purpose: single entry point (BFF / API Gateway) for web, admin, and future mobile clients. This is the seam that lets the backend decompose into microservices without breaking any client.

```text
gateway/

├── src/
│   ├── routes/            # thin pass-through / aggregation routes per domain
│   ├── aggregation/        # combines multiple domain-service calls into one response
│   ├── auth/                # session/JWT validation at the edge
│   ├── rate-limiting/
│   ├── caching/
│   ├── error-handling/
│   └── main.ts
└── tests/
```

Best practices
- No business logic here — aggregation and translation only.
- Every downstream call is typed against `packages/api-contracts`.
- This is the only app allowed to call multiple domain services directly.

## apps/admin/

Purpose: internal administration portal — user management, analytics, moderation, certificate control, marketplace control, audit logs, support dashboard.

Best practices
- Authenticates independently from the public application.
- Strict role-based access control, enforced at the gateway and re-checked per domain service.
- Every administrative action is logged to an immutable audit trail.

## apps/explorer/

Purpose: blockchain explorer customized for SkillChain — transactions, blocks, certificates, NFTs, wallet lookup, smart contract explorer. Reads on-chain data directly and via `packages/blockchain`; does not depend on domain services.

## apps/docs-site/

Purpose: developer documentation — API docs, SDK docs, smart contract docs, tutorials, architecture guides. Built from `docs/` and `packages/sdk` reference material.

## apps/services/

Purpose: domain services. Each is an independently deployable NestJS app today, and remains one even if run inside a single container during early stages. Each service is a thin transport layer (HTTP/queue controllers) wired onto the corresponding `domains/*/application` use cases — it contains almost no business logic of its own.

```text
services/<domain>/

├── src/
│   ├── http/              # controllers — thin, map requests to use cases
│   ├── events/             # consumers/publishers using packages/event-contracts
│   ├── config/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   ├── pipes/
│   └── main.ts
└── tests/
```

### apps/services/worker/

Purpose: background jobs, queues, cron, and async processing — split out as its own deployable from day one so it can scale and deploy independently of any HTTP service.

```text
worker/

├── src/
│   ├── jobs/
│   ├── queues/
│   ├── schedulers/
│   └── main.ts
└── tests/
```

Best practices for `apps/services/*`
- One bounded context per service — never merge two domains into one service folder.
- Controllers/consumers stay thin: validate transport input, call a use case, map the result.
- No service imports another service's internals. Communication is via `api-contracts` or `event-contracts` only.
- Each service owns and evolves its own schema under its matching `domains/<name>/infrastructure/prisma`.

---

# 2. `domains/` — Business Logic Core (Clean Architecture)

Purpose: framework-agnostic business logic, organized as bounded contexts. This layer has no dependency on NestJS, Express, or any transport framework — it is pure TypeScript plus Prisma adapters.

```text
domains/

├── identity/
├── certificates/
├── marketplace/
└── wallet/
```

## domains/`<context>`/

```text
<context>/

├── domain/
│   ├── entities/
│   ├── value-objects/
│   ├── domain-events/
│   ├── domain-services/
│   └── errors/
├── application/
│   ├── use-cases/          # commands and queries (CQRS-friendly)
│   ├── ports/               # interfaces the infrastructure layer must implement
│   └── dto/                 # internal application DTOs (not the public API contract)
└── infrastructure/
    ├── prisma/
    │   ├── schema.prisma    # this domain OWNS its schema
    │   └── migrations/
    ├── repositories/        # Prisma implementations of application/ports
    ├── external/             # third-party API clients (payment, email, blockchain RPC)
    └── mappers/               # entity <-> persistence model mapping
```

### Bounded contexts

| Context | Owns | Examples of responsibilities |
|---|---|---|
| `identity` | Users, sessions, roles, permissions | Auth, wallet-linked login, RBAC |
| `certificates` | Certificate issuance and verification | Minting, revocation, verification lookups |
| `marketplace` | Listings, purchases, reviews | Catalog, checkout, seller tools |
| `wallet` | On-chain wallet state, balances, transactions | Wallet linking, balance sync, tx history |

Best practices
- The Dependency Rule is enforced strictly: `domain` depends on nothing; `application` depends only on `domain`; `infrastructure` depends on `application`'s ports, never the reverse.
- Domain events are raised in `domain/`, published from `infrastructure/` (or the owning service's `events/` layer) using `packages/event-contracts`.
- Each domain's Prisma schema is physically separate — even while all domains currently share one Postgres instance, this makes a later database-per-service split a deployment change, not a code change.
- Cross-domain reads happen through the owning domain's public use cases or through read models built from consumed events — never through direct repository access.

---

# 3. `packages/` — Shared Libraries

Purpose: reusable code that is genuinely cross-cutting. Kept deliberately narrow to avoid becoming a bottleneck.

```text
packages/

├── shared-kernel/
├── ui/
├── config/
├── auth/
├── blockchain/
├── sdk/
├── observability/
├── feature-flags/
├── event-contracts/
├── api-contracts/
├── validation/
└── testing/
```

## packages/shared-kernel/

Purpose: the *only* types and primitives every domain is allowed to depend on.

Contains: `Money`, `Result`/`Either`, `Pagination`, `EntityId`, base domain error classes.

Best practices
- If a type is specific to one domain, it does not belong here — it belongs in that domain's `application/dto` or in `api-contracts`.
- Changes here require cross-team review, since every domain depends on this package.

## packages/ui/

Purpose: reusable presentation components (buttons, cards, tables, modals, forms, layouts).

Best practices
- Presentation-only. No business logic, no direct API calls, no domain types.

## packages/config/

Purpose: centralized tool configuration — eslint, prettier, typescript, tailwind, commitlint, lint-staged, jest, vitest, hardhat, nest, next presets reused across apps and packages.

## packages/auth/

Purpose: authentication utilities — JWT helpers, wallet-signature verification, role definitions, permission helpers. Consumed by `apps/gateway` at the edge and by `domains/identity` internally.

## packages/blockchain/

Purpose: blockchain integration layer — ethers.js wrappers, contract ABIs, transaction helpers, wallet utilities, event decoders. Consumed by `apps/explorer`, `domains/wallet`, and `domains/certificates`.

## packages/sdk/

Purpose: public JavaScript/TypeScript SDK for third-party developers integrating with SkillChain, built on `packages/api-contracts`.

## packages/observability/

Purpose: structured logging, tracing, and metrics, exporter-agnostic (console, file, cloud, OpenTelemetry-compatible). Replaces a bare "logger" package with the full observability surface needed for a multi-service system.

## packages/feature-flags/

Purpose: centralized feature-flag evaluation, used by web, admin, gateway, and services to roll out changes safely per domain.

## packages/event-contracts/

Purpose: versioned schemas for domain and integration events (e.g. `CertificateIssuedV1`, `MarketplacePurchaseCompletedV1`). This is what makes future microservice extraction incremental instead of a big-bang rewrite — consumers depend on the contract version, not on the producer's internals.

Best practices
- Every event is versioned from day one (`V1`, `V2`, ...).
- Breaking changes ship as a new version; old versions are deprecated, not mutated.

## packages/api-contracts/

Purpose: versioned REST/GraphQL DTOs shared between `apps/web`, `apps/admin`, `apps/gateway`, and `apps/services/*`. The single source of truth for the public shape of each domain's API.

## packages/validation/

Purpose: shared Zod / class-validator schemas, reused between frontend forms and backend request validation to avoid duplication and drift.

## packages/testing/

Purpose: shared testing utilities — mock data, test factories, fixtures, general helpers, blockchain mocks.

---

# 4. `blockchain-contracts/` (formerly `contracts/`)

Purpose: Hardhat smart contract workspace. Renamed from `contracts/` to avoid ambiguity with `packages/api-contracts` and `packages/event-contracts`.

```text
blockchain-contracts/

├── contracts/
├── interfaces/
├── libraries/
├── scripts/
├── deployments/
├── ignition/
├── artifacts/
├── cache/
├── typechain/
├── test/
├── coverage/
├── abi/
└── docs/
```

Best practices
- Separate reusable libraries from application contracts.
- Version contracts carefully; never edit deployed contract source retroactively.
- Keep generated artifacts (`artifacts/`, `cache/`, `typechain/`) out of manual edits and out of code review diffs where possible.
- Publish ABI changes to `packages/blockchain` through a deliberate, versioned step — not an implicit build dependency.

---

# 5. `infrastructure/`

Purpose: Infrastructure as Code, structured so each service can be provisioned and scaled independently.

```text
infrastructure/

├── terraform/
├── kubernetes/        # namespaced per service — ready for independent scaling/splitting
├── nginx/
├── monitoring/
├── logging/
├── secrets/            # references to Vault/SOPS-managed secrets, never raw values
└── environments/
```

Best practices
- `kubernetes/` manifests are organized per `apps/services/<domain>` so any one service can be scaled, redeployed, or extracted without touching the others.
- Secrets are referenced, never committed — `secrets/` holds references/policies only.

---

# 6. `docker/`

Purpose: Docker-related files.

```text
docker/

├── development/
├── production/
├── compose/           # one compose service per apps/{web,gateway,admin,services/*}
└── images/
```

Best practices
- Separate development and production images.
- Keep images minimal, reproducible, and one-per-deployable.
- `compose/` mirrors the `apps/` structure exactly so local dev matches production topology.

---

# 7. `prisma/`

Purpose: root-level Prisma tooling shared across domain schemas (client generation config, shared generators). Individual schemas live with their owning domain in `domains/<context>/infrastructure/prisma/`; this directory does not hold a single monolithic schema.

```text
prisma/

├── generators/
└── seed-orchestration.ts   # coordinates seeding across all domain schemas for local dev
```

---

# 8. `config/`

Purpose: shared configuration consumed via `packages/config` re-exports.

```text
config/

├── eslint/
├── prettier/
├── typescript/
├── tailwind/
├── commitlint/
├── lint-staged/
├── jest/
├── vitest/
├── hardhat/
├── nest/
└── next/
```

Best practices
- Keep all tool configuration centralized and versioned once.
- Apps and packages extend these, they don't redefine them.

---

# 9. `scripts/`

Purpose: automation.

```text
scripts/

├── setup/
├── database/
├── blockchain/
├── deployment/
├── maintenance/
├── generators/          # scaffolds a new domain (domain/application/infrastructure + service)
├── ci/
└── migration/            # orchestrates schema migrations across independently-owned domains
```

Examples
- Local environment setup
- Database reset / seed per domain
- Deploy contracts
- Generate SDK from `packages/api-contracts`
- Scaffold a new bounded context (domain + service + contracts stub)
- Health checks across all `apps/services/*`

Best practices
- Make scripts idempotent where possible.
- Document expected inputs and outputs.
- The domain generator script is the recommended way to add a new bounded context, to keep the domain/application/infrastructure convention consistent.

---

# 10. `docs/`

Purpose: project documentation.

```text
docs/

├── architecture/
├── adr/                  # Architecture Decision Records — one file per significant decision
├── boundaries.md          # explicit bounded-context map and ownership matrix
├── api/
├── blockchain/
├── deployment/
├── database/
├── frontend/
├── backend/
├── security/
├── diagrams/
├── onboarding/
├── operations/
└── roadmap/
```

Best practices
- Keep documentation close to implementation changes.
- Version major architectural decisions as ADRs (e.g. "Why modular monolith before microservices," "Why domain-owned schemas").
- `boundaries.md` is kept current and used as the reference for `CODEOWNERS`.

---

# 11. `tests/`

Purpose: cross-application testing.

```text
tests/

├── e2e/
├── integration/
├── performance/
├── security/
├── load/
├── smoke/
├── fixtures/
└── mocks/
```

Best practices
- Keep end-to-end tests independent of unit tests (unit tests live beside the code they test, inside each `domains/*` and `apps/*`).
- Reuse fixtures and mocks across applications via `packages/testing`.
- Include performance and security testing in CI, gated per affected domain where possible.

---

# 12. `.github/`

Purpose: GitHub automation.

```text
.github/

├── workflows/
├── ISSUE_TEMPLATE/
├── PULL_REQUEST_TEMPLATE/
├── CODEOWNERS
├── dependabot.yml
└── SECURITY.md
```

## workflows/

```text
workflows/

├── ci.yml
├── cd.yml
├── lint.yml
├── test.yml
├── build.yml
├── contracts.yml
├── docker.yml
├── security.yml
├── dependency-review.yml
├── release.yml
├── docs.yml
├── preview.yml
└── codeql.yml
```

### Workflow responsibilities

| Workflow | Purpose |
|---|---|
| `ci.yml` | Validate every pull request: install, lint, type-check, tests, build — scoped to affected packages via Turborepo. |
| `cd.yml` | Deploy approved releases; supports per-service matrix deploys so only changed `apps/services/*` redeploy. |
| `lint.yml` | Run ESLint, formatting, and style checks. |
| `test.yml` | Execute unit, integration, and end-to-end tests, scoped to affected domains. |
| `build.yml` | Verify all applications and packages build successfully. |
| `contracts.yml` | Compile, test, and verify Solidity contracts in `blockchain-contracts/`. |
| `docker.yml` | Build and validate container images, one per deployable. |
| `security.yml` | Run dependency, secret, and vulnerability scans. |
| `dependency-review.yml` | Review dependency changes in pull requests. |
| `release.yml` | Create versioned releases and publish artifacts, including `packages/sdk` and `packages/api-contracts`. |
| `docs.yml` | Generate and publish documentation. |
| `preview.yml` | Create preview deployments for feature branches. |
| `codeql.yml` | Perform static code analysis for security issues. |

### CODEOWNERS

```text
/domains/identity/        @skillchain/identity-team
/domains/certificates/    @skillchain/certificates-team
/domains/marketplace/     @skillchain/marketplace-team
/domains/wallet/          @skillchain/wallet-team
/apps/services/identity/  @skillchain/identity-team
/apps/services/certificates/  @skillchain/certificates-team
/apps/services/marketplace/   @skillchain/marketplace-team
/apps/services/wallet/    @skillchain/wallet-team
/apps/gateway/            @skillchain/platform-team
/packages/shared-kernel/  @skillchain/platform-team
/blockchain-contracts/    @skillchain/blockchain-team
```

Because folder boundaries map directly to bounded contexts, ownership stays accurate as the team grows — no manual re-mapping required.

---

# Supporting Repository Directories

## .husky/

Git hooks for pre-commit linting, formatting, type checking, and commit message validation.

## .vscode/

Workspace settings: extension recommendations, launch configurations, debug settings, workspace preferences.

## assets/

Shared static assets: logos, icons, brand assets, fonts, illustrations.

## tmp/

Temporary files generated during local development. Contains no source code and is excluded from version control.

---

# Architectural Best Practices

- **Domain-driven organization.** Code is grouped by business capability first; technical layering (`domain/application/infrastructure`) exists *within* each domain, not across the whole repo.
- **Clean Architecture dependency rule.** Domain logic depends on nothing. Application depends only on domain. Infrastructure implements application's ports. Transport (`apps/services/*`) depends on application, never the reverse.
- **Bounded contexts are enforced, not just documented.** No domain imports another domain's repository, entity, or Prisma client directly. All cross-context interaction goes through `api-contracts` or `event-contracts`.
- **Deployable-ready from day one.** `apps/services/*` mirror `domains/*` exactly, so extracting a microservice is an infrastructure and deployment change, not a code migration.
- **Narrow shared packages.** `shared-kernel` stays small on purpose; anything domain-specific lives with its domain, not in a shared catch-all.
- **Gateway as a stability seam.** Frontends depend on `apps/gateway`'s contract, which can remain stable even as backend services are added, merged, or split.
- **Contracts are versioned, not mutated.** Both `api-contracts` and `event-contracts` follow explicit versioning so consumers upgrade deliberately.
- **Configuration as code.** Linting, formatting, testing, and build configuration are centralized under `config/` and consumed via `packages/config`.
- **Automation-first.** Scripts, infrastructure, and CI/CD are first-class, including a scaffolding script that generates new bounded contexts with the correct internal structure.
- **Documentation culture.** Architecture decisions are recorded as ADRs; `docs/boundaries.md` is the living source of truth for context ownership.
- **Security built in.** Dependency scanning, static analysis, secret detection, and contract testing run in CI for every affected domain.
- **Observability planned from the start.** `packages/observability` and `packages/feature-flags` exist before they're urgently needed, not after an incident.

---

# Migration Path to Full Microservices

This structure is intentionally a **modular monolith today** with the seams already cut for tomorrow. When a domain needs independent scaling or a dedicated team:

1. Give `apps/services/<domain>` its own CI/CD pipeline (already scoped by Turborepo/workflow matrix).
2. Provision a dedicated database for `domains/<domain>/infrastructure/prisma` (already schema-isolated).
3. Point `apps/gateway` at the new service's network address instead of an in-process call — the contract (`api-contracts`) does not change.
4. Convert any synchronous cross-domain calls that domain made to `event-contracts`-based async communication if tighter coupling was previously tolerated in-process.
5. No changes required in `apps/web`, `apps/admin`, or any other client — they only ever depended on the gateway.

This keeps the day-one repository simple to run (`docker compose up`) while guaranteeing that scaling out later is an infrastructure exercise, not a rewrite.