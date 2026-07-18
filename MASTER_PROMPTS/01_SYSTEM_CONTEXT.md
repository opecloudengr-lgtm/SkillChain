# SkillChain AI System Context

You are working on **SkillChain**, a production-ready Web3 application built on the **Electroneum Smart Chain (EVM-compatible blockchain)**.

## Project Goal

Build a decentralized certificate issuance and verification platform where:

* Institutions issue certificates on-chain.
* Students own certificates permanently.
* Employers verify certificates instantly.
* All blockchain interactions use the Electroneum Smart Chain.

## Approved Technology Stack

### Frontend

* Next.js 15
* TypeScript
* Tailwind CSS
* shadcn/ui
* React Hook Form
* Zod
* TanStack Query

### Backend

* NestJS
* TypeScript
* Prisma
* PostgreSQL
* JWT Authentication
* Swagger

### Blockchain

* Solidity
* Hardhat
* OpenZeppelin
* ethers.js v6

### DevOps

* Docker
* Docker Compose
* GitHub Actions

## Architecture Rules

* Use modular architecture.
* Keep business logic in services.
* Controllers should be thin.
* Use DTOs for all API input/output.
* Use Prisma for database access.
* Never write raw SQL unless explicitly requested.
* Use UUIDs for all database primary keys.
* Use soft deletes where applicable.
* Follow REST API conventions.
* Use environment variables for all secrets.

## Important

Before generating any code:

1. Explain the files being created.
2. Explain the purpose of each file.
3. Generate production-ready code.
4. Include error handling.
5. Include validation.
6. Include comments only where necessary.
7. Ensure code is consistent with previous modules.
