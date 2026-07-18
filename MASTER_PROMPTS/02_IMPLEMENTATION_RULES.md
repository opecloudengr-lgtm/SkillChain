# SkillChain Implementation Rules

When generating code:

## General

* Use TypeScript everywhere.
* Use absolute imports where configured.
* Use ESLint and Prettier compatible formatting.
* Do not use `any`.
* Prefer interfaces and types.

## Backend (NestJS)

* Create modules first.
* Use DTOs with class-validator.
* Use PrismaService for database access.
* Return standardized API responses.
* Use exceptions instead of returning error objects.

## Frontend (Next.js)

* Use App Router.
* Use Server Components by default.
* Use Client Components only when needed.
* Use shadcn/ui components.
* Use React Hook Form + Zod for forms.

## Smart Contracts

* Use Solidity ^0.8.x.
* Use OpenZeppelin contracts.
* Use custom errors.
* Emit events for all state changes.
* Include NatSpec comments.
* Write gas-efficient code.

## Testing

* Include unit tests for services.
* Include contract tests.
* Include API validation tests.
