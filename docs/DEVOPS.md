# SkillChain — DevOps Architecture Blueprint
**Containerization:** Docker · **Orchestration (local):** Docker Compose · **CI/CD:** GitHub Actions
**Stack under management:** NestJS API · Next.js Frontend · PostgreSQL · Redis · Hardhat contracts

This document defines the operational backbone that ships and runs everything specified in the backend, database, frontend, and UI/UX blueprints. Config snippets shown are **illustrative structure**, not final production files — they exist to pin down shape and intent for whoever writes the real manifests.

---

## 1. DevOps Architecture Overview

```mermaid
graph TB
    Dev["Developer"] -->|push/PR| GH["GitHub Repository"]
    GH --> Actions["GitHub Actions"]

    subgraph CI["CI Pipeline"]
        Lint["Lint + Typecheck"]
        Test["Unit + Integration Tests"]
        ContractTest["Hardhat Contract Tests"]
        Build["Build Docker Images"]
        Scan["Security/Dependency Scan"]
    end

    Actions --> CI
    CI --> Registry["Container Registry (GHCR)"]

    Registry --> CD["CD Pipeline"]
    subgraph CD_Detail["CD Pipeline"]
        Migrate["Run Prisma Migrations"]
        DeployStaging["Deploy to Staging (Electroneum Testnet)"]
        SmokeTest["Smoke Tests"]
        Approval["Manual Approval Gate"]
        DeployProd["Deploy to Production (Electroneum Mainnet)"]
    end
    CD --> CD_Detail

    DeployProd --> Infra["Production Infrastructure"]
    subgraph Infra_Detail["Production Infrastructure"]
        LB["Load Balancer / Ingress"]
        API["NestJS API (N replicas)"]
        Workers["BullMQ Workers (N replicas)"]
        Web["Next.js Frontend"]
        PG[("Managed PostgreSQL")]
        Redis[("Managed Redis")]
        RPC["Electroneum RPC Provider"]
    end
    Infra --> Infra_Detail

    Infra_Detail --> Observability["Monitoring / Logging / Alerting Stack"]
```

**Operating principles**
- **Everything reproducible from source control.** Dockerfiles, Compose files, GitHub Actions workflows, and infra-as-code (if adopted) all live in the repo — no manually-clicked infrastructure that can't be reconstructed from a commit.
- **The pipeline enforces the same gates for every change**: lint → test → contract test → build → scan → deploy-staging → smoke-test → manual-approve → deploy-production. No path skips a gate, including hotfixes (hotfixes get an expedited approval, not a skipped pipeline).
- **Staging always targets Electroneum testnet; production always targets mainnet.** This binding is enforced at the environment-variable/config-validation level (per the backend blueprint's `ConfigModule` fail-fast rule) so a misconfigured deploy cannot accidentally point production traffic at a testnet contract, or vice versa.
- **Secrets never enter the container image or the repo.** They are injected at runtime by the deployment platform's secret store, full stop (§9).

---

## 2. Docker

### 2.1 Image Strategy
Every deployable service (API, frontend, worker) uses a **multi-stage build**: a heavier `build` stage compiles TypeScript and installs dev dependencies, and a minimal `runtime` stage copies only the compiled output and production dependencies — keeping final images small and reducing attack surface (no compilers, no dev tooling, no source maps unless explicitly needed for error tracking).

```mermaid
graph LR
    Base["node:20-alpine (base)"] --> Deps["deps stage: install prod deps only"]
    Base --> Build["build stage: install all deps, compile TS, build Next.js"]
    Deps --> Runtime["runtime stage: copy dist + prod node_modules"]
    Build --> Runtime
    Runtime --> Image["Final Image (non-root user, read-only fs where possible)"]
```

### 2.2 Illustrative Dockerfile Shape — NestJS API

```dockerfile
# ---- deps stage ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ---- build stage ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

# ---- runtime stage ----
FROM node:20-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
RUN addgroup -S skillchain && adduser -S skillchain -G skillchain
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
USER skillchain
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s \
  CMD node dist/health-check.js || exit 1
CMD ["node", "dist/main.js"]
```

**Key hardening choices**
- **Non-root user** (`skillchain`) — the process never runs as `root` inside the container, limiting blast radius of a container-escape vulnerability.
- **Alpine base** — minimal OS surface; pinned to a specific Node major version, never `:latest`.
- **`HEALTHCHECK` built in** — orchestrators (Compose, Kubernetes, ECS) can detect an unresponsive container without depending on external tooling.
- **`.dockerignore`** excludes `node_modules`, `.env*`, `.git`, test files, and Hardhat artifacts not needed at runtime — keeps build context small and prevents accidental secret leakage into image layers.

### 2.3 Illustrative Dockerfile Shape — Next.js Frontend
Same multi-stage pattern, with Next.js's `output: 'standalone'` build mode used specifically so the runtime stage only needs the standalone server bundle plus static assets — not the full `node_modules` tree, meaningfully shrinking the image versus a naive copy.

### 2.4 Image Tagging & Provenance
- Tags: `{service}:{git-sha}` (immutable, always used for actual deploys) and `{service}:{semver}` for releases — `latest` is never deployed, only used as a convenience tag for local dev pulls.
- Images are pushed to **GitHub Container Registry (GHCR)**, scoped to the repository, with retention policy pruning untagged/old-SHA images beyond a rolling window to control storage cost.
- Each image is built with an embedded `BUILD_INFO` (git SHA, build timestamp) surfaced on the `/health` endpoint — makes "what's actually running in production right now" answerable in one request, not a guess.

### 2.5 Hardhat/Contract Tooling Image
A separate, non-runtime image (used only in CI, never deployed) bundles Hardhat and contract dependencies for compiling and running contract tests — kept isolated from the API/frontend images so contract tooling never bloats production artifacts.

---

## 3. Docker Compose (Local Development)

### 3.1 Purpose & Scope
Docker Compose is the **local development and integration-testing environment only** — it is explicitly not the production orchestration mechanism (that's the deployment platform, §6). Its job is to let a developer run the full stack (API, frontend, Postgres, Redis, a local Hardhat node) with one command and realistic service-to-service networking.

### 3.2 Illustrative Compose Shape

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: skillchain
      POSTGRES_PASSWORD: local_dev_only
      POSTGRES_DB: skillchain_dev
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U skillchain"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5

  hardhat-node:
    build: ./contracts
    command: npx hardhat node
    ports: ["8545:8545"]

  api:
    build:
      context: ./apps/api
      target: build   # dev target: includes dev deps, hot reload
    env_file: .env.api.local
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      hardhat-node: { condition: service_started }
    ports: ["3000:3000"]
    volumes: ["./apps/api:/app", "/app/node_modules"]
    command: npm run start:dev

  web:
    build:
      context: ./apps/web
      target: build
    env_file: .env.web.local
    depends_on: [api]
    ports: ["3001:3000"]
    volumes: ["./apps/web:/app", "/app/node_modules"]
    command: npm run dev

  worker:
    build:
      context: ./apps/api
      target: build
    env_file: .env.api.local
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    command: npm run start:worker

volumes:
  pg_data:
```

### 3.3 Design Decisions
- **`depends_on` with `condition: service_healthy`**, not just startup ordering — the API must not begin accepting traffic (or fail confusingly) before Postgres/Redis are actually ready to accept connections, not merely "the container process has started."
- **Local Hardhat node included** — developers mint/revoke against a fully local, disposable chain by default, so local development never depends on testnet RPC availability or spends real/test-faucet funds during routine iteration.
- **Bind-mounted source + `node_modules` volume trick** — live code editing with hot reload, while still isolating `node_modules` inside the container (avoids host/container platform binary mismatches).
- **`docker-compose.override.yml`** (optional, gitignored) allows individual developers to tweak ports/volumes locally without touching the committed base file.
- **A second Compose file, `docker-compose.test.yml`**, spins up an ephemeral Postgres for integration tests (used by both local `npm run test:integration` and CI) — separate from the dev database so running tests never touches a developer's working dev data.

---

## 4. GitHub Actions

### 4.1 Workflow Inventory

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | Every PR, every push to `main` | Lint, typecheck, unit + integration tests, contract tests, build validation |
| `security-scan.yml` | Every PR + nightly schedule | Dependency vulnerability scan, container image scan, secret-leak scan |
| `docker-publish.yml` | Push to `main`, tagged release | Build + push versioned images to GHCR |
| `deploy-staging.yml` | Successful `docker-publish` on `main` | Auto-deploy to staging (testnet) |
| `deploy-production.yml` | Manual dispatch or tagged release + approval | Deploy to production (mainnet) |
| `db-migrate-check.yml` | Every PR touching `prisma/schema.prisma` | Validates migration is generated, reviews SQL diff, checks for unsafe operations (dropped columns/tables) |
| `contract-audit.yml` | PR touching `/contracts` | Slither/static analysis on Solidity changes, gas-report diff |

### 4.2 Illustrative `ci.yml` Shape

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  unit-tests:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  integration-tests:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: skillchain_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U test" --health-interval 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx prisma migrate deploy
        env: { DATABASE_URL: postgresql://test:test@localhost:5432/skillchain_test }
      - run: npm run test:integration
        env: { DATABASE_URL: postgresql://test:test@localhost:5432/skillchain_test }

  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - working-directory: ./contracts
        run: |
          npm ci
          npx hardhat compile
          npx hardhat test

  build:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, contract-tests]
    steps:
      - uses: actions/checkout@v4
      - run: docker build -f apps/api/Dockerfile -t skillchain-api:ci .
      - run: docker build -f apps/web/Dockerfile -t skillchain-web:ci .
```

### 4.3 Branch Protection & Merge Policy
- `main` requires: all `ci.yml` jobs green, `security-scan.yml` green (or explicitly waived with a documented exception), at least one approving code review, and a linear history (squash merge only) — enforced via GitHub branch protection rules, not convention.
- `db-migrate-check.yml` specifically fails the PR (not just warns) on a detected `DROP COLUMN`/`DROP TABLE` without an accompanying `EXPAND_CONTRACT_APPROVED` label — operationalizes the expand/contract migration discipline from the database blueprint (§8.2 of that doc) as an actual CI gate, not just a written convention.

### 4.4 Caching Strategy
- `npm ci` dependency cache keyed on `package-lock.json` hash, shared across jobs via `actions/setup-node`'s built-in cache.
- Docker layer caching via GitHub Actions cache backend (`type=gha`) on the `docker-publish.yml` build steps — meaningfully speeds up rebuilds when only application code (not dependencies) changed.

---

## 5. CI/CD Pipeline (End-to-End)

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant PR as Pull Request
    participant CI as CI (ci.yml, security-scan.yml)
    participant Main as main branch
    participant Publish as docker-publish.yml
    participant Staging as Staging Env (testnet)
    participant Gate as Manual Approval
    participant Prod as Production Env (mainnet)

    Dev->>PR: Open PR
    PR->>CI: Trigger lint/test/build/scan
    CI-->>PR: Status checks
    Dev->>PR: Address review + CI feedback
    PR->>Main: Merge (squash)
    Main->>Publish: Build + tag + push images (git-sha)
    Publish->>Staging: Auto-deploy (migrate -> deploy -> smoke test)
    Staging-->>Dev: Staging URL, smoke test results
    Dev->>Gate: Request production release (tag/dispatch)
    Gate-->>Gate: Human reviews staging results, changelog
    Gate->>Prod: Approve
    Prod->>Prod: migrate deploy -> rolling deploy -> canary check -> full rollout
```

### 5.1 Staging Deploy (Automatic)
Every merge to `main` that passes CI is automatically deployed to staging: run `prisma migrate deploy` against the staging DB → deploy new container images → run an automated smoke-test suite (health endpoints, a scripted login + course-enrollment + certificate-verification happy path) → report status back to the team (Slack/Teams webhook notification).

### 5.2 Production Deploy (Gated)
Production deploys are **never automatic**. A release is cut (git tag or manual workflow dispatch), which requires a named approver in the GitHub Environment protection rules before `deploy-production.yml` proceeds. The approval step surfaces: the staging smoke-test results, the diff/changelog since the last production release, and an explicit checkbox-equivalent confirming database migrations (if any) have been reviewed per the expand/contract policy.

### 5.3 Deployment Mechanics
- **Rolling deploy** for the API and frontend (multiple replicas, new version brought up and health-checked before old replicas are drained) — chosen over blue/green initially for lower infra cost, with blue/green as a documented upgrade path if zero-downtime guarantees need to tighten further.
- **Canary check window**: after the first replica of a rolling deploy is healthy, an automated check watches error rate/latency for a short window (e.g., 5 minutes) before proceeding to replace remaining replicas — an automatic rollback triggers if error rate crosses a threshold during this window.
- **Workers deploy independently** from the API — a BullMQ worker processing an in-flight mint job is allowed to drain its current job before terminating (graceful shutdown handling), rather than being hard-killed mid-transaction-submission.

### 5.4 Rollback
- Rollback is a **re-deploy of the previous known-good image tag** (immutable git-SHA tags make this trivial and unambiguous), not a `git revert` + rebuild cycle — this keeps rollback fast (minutes, not a full pipeline re-run) for genuine incidents.
- Database migration rollback is handled separately and more carefully per the expand/contract discipline (destructive migrations are never bundled with a release that can't tolerate being rolled back at the app layer while the DB stays on the new schema).

---

## 6. Environment Variables

### 6.1 Environment Tiers
`local` → `staging` (Electroneum testnet) → `production` (Electroneum mainnet). Each tier has its own isolated variable set — no shared `.env` file crosses a tier boundary, and each service validates its full required-variable set at boot (fail-fast), per the backend blueprint's `ConfigModule` convention.

### 6.2 Variable Catalog (representative, grouped)

| Category | Variable | Present in | Notes |
|---|---|---|---|
| **Core** | `NODE_ENV` | all | `development` \| `staging` \| `production` |
| | `PORT` | all | Service listen port |
| | `LOG_LEVEL` | all | `debug` locally, `info` in staging/prod |
| **Database** | `DATABASE_URL` | all | Includes connection pool params |
| | `DATABASE_READ_REPLICA_URL` | staging/prod (optional) | For read-heavy verification traffic |
| **Cache/Queue** | `REDIS_URL` | all | Cache + BullMQ backend |
| **Auth** | `JWT_ACCESS_SECRET` | all | Secret-managed, never plaintext in prod |
| | `JWT_REFRESH_SECRET` | all | Distinct from access secret |
| | `JWT_ACCESS_TTL` / `JWT_REFRESH_TTL` | all | e.g. `15m` / `30d` |
| **Blockchain** | `CHAIN_ID` | all | Electroneum testnet vs. mainnet ID — validated against `NODE_ENV` at boot |
| | `RPC_URL` | all | Provider endpoint, secret-managed if API-key-bound |
| | `CONTRACT_ADDRESS` | all | Deployed certificate contract, per-environment |
| | `ISSUER_WALLET_KEYSTORE_REF` | staging/prod | Reference/path into KMS/secrets manager — never the raw key |
| | `GAS_PRICE_CAP` | staging/prod | Safety ceiling for auto-submitted transactions |
| **Storage** | `S3_BUCKET` / `S3_REGION` | all | Object storage for uploads |
| | `IPFS_PINNING_API_KEY` | staging/prod | Secret-managed |
| **Frontend (public)** | `NEXT_PUBLIC_API_BASE_URL` | all | Safe to expose to browser |
| | `NEXT_PUBLIC_CHAIN_ID` | all | Safe to expose |
| | `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID` | all | Public by design (WalletConnect's model) |
| **Observability** | `SENTRY_DSN` | staging/prod | Error tracking |
| | `OTEL_EXPORTER_ENDPOINT` | staging/prod | Tracing/metrics export |

### 6.3 Naming & Validation Conventions
- `NEXT_PUBLIC_*` prefix is the **only** mechanism by which a variable is exposed to the browser bundle — every other variable is server-only by construction (Next.js's own convention), preventing accidental leakage of a secret into client JS.
- All variables are declared in a typed schema (e.g., Zod-validated config module) per service — an unset required variable fails the service at startup with a clear error, never a silent `undefined` propagating into runtime logic.
- `.env.example` files (committed, no real values) document every variable per service — kept in sync with the validation schema via a CI check that fails if they drift.

### 6.4 Where Values Actually Live
- **Local:** `.env.local` files, gitignored, populated from `.env.example` by each developer.
- **CI:** GitHub Actions **encrypted secrets** (repo or environment-scoped) injected as job env vars — never echoed to logs (GitHub Actions automatically masks known secret values in log output).
- **Staging/Production:** injected by the deployment platform's secret store at container start (§9) — not baked into the Docker image at build time, ever.

---

## 7. Deployment

### 7.1 Target Topology

```mermaid
graph TB
    Users["Users / Verifiers"] --> CDN["CDN / Edge Cache (static assets, ISR pages)"]
    CDN --> WebSvc["Next.js Frontend Service (N instances)"]
    Users --> LB["Load Balancer"]
    LB --> ApiSvc["NestJS API Service (N instances)"]
    ApiSvc --> PG[("Managed PostgreSQL - Primary")]
    ApiSvc -.->|"read-heavy verification traffic (optional)"| PGReplica[("Read Replica")]
    ApiSvc --> RedisSvc[("Managed Redis")]
    WorkerSvc["BullMQ Worker Service (N instances)"] --> RedisSvc
    WorkerSvc --> PG
    WorkerSvc --> RPCNode["Electroneum RPC Provider"]
    ApiSvc --> RPCNode
    ApiSvc --> ObjStore["S3-compatible Object Storage"]
    ApiSvc --> IPFS["IPFS Pinning Service"]
```

### 7.2 Platform Assumptions
This blueprint is written platform-agnostically (works equivalently on ECS/Fargate, GKE/Cloud Run, or a managed PaaS like Render/Railway) but assumes, at minimum:
- **Managed PostgreSQL and Redis** — not self-hosted database containers in production — offloading patching, failover, and automated backups (§10) to the provider.
- **Container-native compute** capable of running the built Docker images directly from GHCR with rolling deploy support and health-check-gated traffic cutover.
- **A load balancer/ingress** terminating TLS, forwarding to the API and frontend services, with WAF/rate-limiting capability at the edge (per the backend blueprint §25).

### 7.3 Service Separation
- **Frontend, API, and Workers deploy as three independently scalable services** — a traffic spike on the public verification page (frontend + API) doesn't need to scale the mint-processing worker fleet, and a backlog of pending mints doesn't need to scale the frontend.
- **Database migrations run as a one-shot job/step in the pipeline** (§5.1/5.2), never as a side effect of a service container starting — prevents N replicas racing to apply the same migration simultaneously.

### 7.4 Environment Parity
Staging is intentionally a **structurally identical, smaller-scale mirror** of production (same container images, same migration path, same topology, fewer replicas) pointed at Electroneum testnet — this is what makes the staging smoke-test step (§5.1) a meaningful production-readiness signal rather than a formality.

---

## 8. Monitoring

### 8.1 Stack

```mermaid
graph LR
    Services["API / Web / Workers"] --> Metrics["Prometheus-compatible metrics endpoint"]
    Services --> Traces["OpenTelemetry traces"]
    Services --> HealthEP["/health liveness+readiness"]
    Metrics --> Grafana["Grafana Dashboards"]
    Traces --> TraceBackend["Trace Backend (e.g. Tempo/Jaeger-compatible)"]
    HealthEP --> LB2["Load Balancer health checks"]
    Services --> Sentry["Sentry (error tracking)"]
    Grafana --> Alerting["Alertmanager -> PagerDuty/Slack"]
    Sentry --> Alerting
```

### 8.2 What's Monitored

| Layer | Signal | Alert Condition (illustrative) |
|---|---|---|
| **HTTP (API + Web)** | Request rate, p50/p95/p99 latency, 5xx rate | 5xx rate > 2% over 5 min |
| **Database** | Connection pool utilization, query latency, replication lag | Pool utilization > 85% sustained |
| **Redis/Queue** | Queue depth (per queue), job failure rate, oldest-job age | Mint queue depth growing for > 15 min without draining |
| **Blockchain** | Issuer wallet balance, RPC latency/error rate, average confirmation time, stuck-transaction count | Issuer wallet balance below configured threshold (this is a common, easily-missed Web3 outage cause) |
| **Infra** | CPU/memory per service, container restart count | Restart count > N in 10 min (crash-loop signal) |
| **Business/product** | Certificates issued/day, verification requests/day, enrollment completion rate | Sudden drop-off (informational dashboard, not a page-worthy alert) |

### 8.3 Health Check Endpoints
Per the backend blueprint's Terminus setup: `/health/liveness` (process is up), `/health/readiness` (DB, Redis, RPC node reachable — used by the load balancer/orchestrator to gate traffic), `/health/deep` (a real, cheap on-chain read succeeds — used by synthetic monitoring, not the hot path of every deploy's readiness gate, since it's slower and chain-dependent).

### 8.4 Synthetic Monitoring
A scheduled external check (every few minutes, from outside the cluster) hits the public verification page and the health endpoints — this is the monitoring layer that would actually catch "the load balancer itself is misconfigured" or "DNS broke," which internal-only monitoring cannot.

### 8.5 Dashboards
Three tiers of Grafana dashboard, matching the three UI/UX registers: an **executive/product** dashboard (issuance volume, active learners), an **on-call/reliability** dashboard (RED metrics, queue health, error budget), and a **blockchain-specific** dashboard (gas spend trends, confirmation times, wallet balance) — because the on-call engineer debugging a stuck mint needs a very different view than the person checking product health.

---

## 9. Logging

### 9.1 Structure
Structured JSON logging (Pino, per the backend blueprint) shipped from every service — no service logs unstructured plain text in staging/production.

### 9.2 Correlation
Every log line carries the `correlationId` generated at the edge (`CorrelationIdMiddleware`) and propagated through service calls, queue jobs, and even into blockchain transaction submission logs — a single request or a single certificate's entire lifecycle (API request → outbox event → queue job → mint tx → confirmation) is traceable as one thread across otherwise-disconnected log lines, which matters enormously for debugging a stuck or failed certificate issuance.

### 9.3 Log Pipeline

```mermaid
graph LR
    Containers["Container stdout/stderr (JSON lines)"] --> Collector["Log Collector/Agent (e.g. Vector/Fluent Bit)"]
    Collector --> Aggregator["Centralized Log Store (e.g. Loki/ELK)"]
    Aggregator --> GrafanaLogs["Grafana Explore / Kibana"]
    Aggregator --> Retention["Retention & Archival Policy"]
```

- Containers log to `stdout`/`stderr` only (12-factor convention) — never to files inside the container — letting the platform's standard log collection handle shipping uniformly across all services.
- A collector agent tails container output and ships to a centralized store, tagging every line with service name, environment, container/replica ID, and git-SHA (from the image's build metadata) for filterability.

### 9.4 Sensitive Data Handling
- A shared logging serializer **redacts known-sensitive fields** (password hashes, JWTs, full wallet signatures, private key material, PII beyond what's operationally necessary) before a log line is ever emitted — redaction happens at the source, not as a downstream scrubbing step that could be bypassed or lag behind.
- Audit-relevant events (admin actions, certificate revocations) are **dual-written**: once to the structured log stream (operational visibility, short-to-medium retention) and once to the `audit_logs` Postgres table (long-retention, compliance-grade, per the database blueprint) — these serve different purposes and are not treated as substitutes for each other.

### 9.5 Retention
Operational logs: 30–90 days hot in the aggregator, per environment (staging shorter than production). Beyond that window, logs are not needed for day-to-day debugging — the durable compliance record lives in `audit_logs` (database-level retention/archival policy, per the database blueprint §7 and §10.3), not in the log aggregator.

---

## 10. Secrets

### 10.1 Principles
- **No secret ever lives in source control**, a Docker image layer, or a CI log line — enforced structurally (`.gitignore`, GitHub Actions' automatic secret masking, image build steps that never `COPY` an env file) and additionally checked by an automated secret-scanning step (§4.1, `security-scan.yml`) on every PR as a backstop against human error.
- **Least privilege per environment**: staging secrets and production secrets are entirely distinct values (not the same JWT secret reused "for convenience") — a staging compromise must not translate into production access.

### 10.2 Secret Storage by Context

| Context | Storage | Access Pattern |
|---|---|---|
| Local development | `.env.local` (gitignored) | Developer-managed, non-sensitive placeholder values where possible |
| CI (GitHub Actions) | GitHub encrypted secrets, scoped to repo or per-environment | Injected as job env vars, auto-masked in logs |
| Staging/Production runtime | Cloud provider secrets manager / KMS (e.g., AWS Secrets Manager, GCP Secret Manager, or platform-native equivalent) | Injected into the container at start as env vars or mounted files; the app never reads the secrets manager API directly at request-time for latency/availability reasons — it reads its already-injected env at boot |
| Issuer signing key specifically | Dedicated KMS/HSM-backed key, **never a plain env var even in the secrets manager**, per the backend blueprint's key-management stance | `BlockchainService` requests signing operations through the KMS's signing API/SDK rather than ever holding the raw private key in process memory where avoidable |

### 10.3 Rotation
- JWT signing secrets, database credentials, and API keys (S3, IPFS pinning, RPC provider) are rotated on a defined schedule (e.g., quarterly) and immediately on suspected compromise — rotation is a documented runbook, not an ad hoc event, since a botched rotation (e.g., invalidating all active sessions unintentionally) is itself an incident.
- Refresh-token rotation (application-level, per the backend blueprint's auth flow) is a distinct mechanism from infrastructure secret rotation and is not conflated with it here.

### 10.4 Access Control & Audit
- Access to the production secrets manager is scoped to the deployment pipeline's service identity and a minimal set of human operators — not the whole engineering team by default.
- Every secret access/rotation event is itself logged (most cloud secrets managers provide this natively) — secrets management has its own audit trail, separate from but complementary to the application's `audit_logs` table.

---

## 11. Backup

*(This section extends the database blueprint's §10 Backup Strategy to the full infrastructure surface — database specifics are not repeated here beyond a summary.)*

### 11.1 What's Backed Up

| Asset | Method | Frequency / Retention |
|---|---|---|
| **PostgreSQL** | Managed provider automated snapshots + continuous WAL archiving (PITR) | Daily snapshots (14d/8w/12m tiered), PITR window 7–14 days — per database blueprint §10 |
| **Redis** | Periodic RDB snapshot (Redis here is cache + queue, not a system of record) | Best-effort; queue jobs are re-derivable from the outbox/DB state, so Redis loss is a recovery-time concern, not a data-loss concern |
| **Object storage (S3-compatible)** | Provider-native versioning + cross-region replication | Versioning retains prior object versions; lifecycle policy transitions old versions to cold storage |
| **IPFS-pinned certificate metadata** | Pinned via a managed pinning service with its own redundancy, **plus** the CID and a copy of the source metadata retained in `certificate_metadata`/object storage as a fallback re-pin source | Effectively permanent — content-addressed data is re-pinnable from the retained source if a pinning provider is ever lost |
| **Infrastructure/deploy config** | Everything as code in the Git repository (Dockerfiles, Compose, GitHub Actions workflows, IaC if adopted) | Git history is the backup — no separate infra backup mechanism needed for anything expressed as code |
| **Secrets** | Secrets manager's own backup/replication (provider-native), **not** separately exported/archived by SkillChain tooling | Avoids creating a second, less-secured copy of secret material purely for backup purposes |

### 11.2 Disaster Recovery Flow

```mermaid
graph TB
    Incident["Detected data-loss / corruption incident"]
    Incident --> Assess["Assess scope: DB only? Object storage? Full region?"]
    Assess --> RestoreDB["Restore PostgreSQL to PITR target"]
    Assess --> RestoreObj["Restore object storage from versioned backups"]
    RestoreDB --> Reconcile["Reconciliation job: verify certificate state against on-chain truth for the affected time window"]
    RestoreObj --> Reconcile
    Reconcile --> Validate["Validate: spot-check certificates, run smoke tests"]
    Validate --> Resume["Resume traffic / lift maintenance mode"]
```

- Because certificate authenticity is independently verifiable on-chain (per the database blueprint §10.4), a database restore is always followed by the reconciliation job before the system is declared fully recovered — a restored DB is treated as "probably right, verify before trusting" rather than automatically authoritative.
- **Maintenance mode middleware** (per the backend blueprint §9) is engaged during an active restore to prevent writes against a database mid-recovery, then lifted only after reconciliation and smoke tests pass.

### 11.3 Backup Verification
- Scheduled restore drills (quarterly, per the database blueprint) are run against a scratch environment — a backup that has never been test-restored is treated as unverified, not trusted.
- The DR flow itself (§11.2) is exercised as a **game day exercise** periodically, not just the mechanical restore — confirming the runbook, the reconciliation job, and the maintenance-mode procedure all work together under time pressure, not just in isolation.

---

*This document defines DevOps structure, pipeline shape, and operational policy. Illustrative Dockerfile/Compose/GitHub Actions snippets are provided to pin down concrete shape and are expected to be adapted, not copy-pasted verbatim, into the final production configuration.*