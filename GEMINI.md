# GEMINI.md

## Purpose

This file defines how Gemini should operate within this repository.

It enforces consistent behavior across a Turborepo-based monorepo containing an API and a web application, communicating internally via RPC over the Cloudflare network.

Gemini must follow these rules before making changes.

---

## Architecture Constraint

- Frontend and backend communicate inside the Cloudflare network using RPC
- Internal communication must use typed RPC contracts
- Do not introduce ad hoc internal REST calls where RPC is intended
- External APIs (e.g., public endpoints) may still use REST where appropriate
- Internal RPC contracts must be shared and consistent across workspaces

---

## Repository Structure (Turborepo)

This repository uses a monorepo structure:

- apps/api → backend API / RPC service (Hono, TypeScript)
- apps/web → frontend application
- packages/* → shared code (types, config, RPC contracts, utilities)

### Rules

- Do not move files across workspaces without explicit approval
- Do not create new packages unless clearly necessary
- Prefer reusing existing packages over duplication
- Keep boundaries between apps and packages clean
- Shared RPC contracts must live in a reusable package when appropriate

---

## Core Stack

- TypeScript (strict mode)
- Hono (API / RPC handlers)
- Drizzle ORM + PostgreSQL (Neon)
- Zod (validation)
- Biome (linting/formatting)
- Ultracite (code quality)
- Vitest (testing)
- Turborepo (monorepo orchestration)
- Cloudflare (deployment / internal networking)
- Facebook API (external data source)

All generated code must follow these constraints.

---

## Gemini Behavior Rules

### Plan Before Acting

- For any non-trivial change, Gemini must:
  - Explain the plan
  - Identify affected files
  - Wait for confirmation if the change is large

### Scope Control

- Prefer small, focused changes
- Avoid touching multiple apps/packages unless necessary
- Do not refactor unrelated code

### Ask Before Risky Changes

Gemini must ask before:

- Adding dependencies
- Modifying shared packages
- Changing Turborepo configuration
- Editing root config files
- Performing cross-workspace refactors
- Changing database schema or migrations
- Introducing new architectural patterns
- Modifying RPC contracts used across apps

---

## Coding Rules

- MUST use TypeScript strict mode
- MUST NOT use `any`
- MUST follow Biome formatting rules
- MUST pass Ultracite quality checks
- Keep functions small and focused
- Prefer explicit types over implicit inference
- Avoid unnecessary abstractions

---

## API & Backend Rules (apps/api)

- Follow `docs/api-conventions.md` for external-facing APIs
- Use typed DTOs and typed RPC contracts
- Validate all inputs using Zod
- Keep handlers thin (routes or RPC)
- Move business logic into services
- Use Drizzle ORM for all database access
- Do not write raw SQL in handlers

### Internal Communication Rules

- All internal communication must use RPC over the Cloudflare network
- Do not create duplicate internal data shapes if shared contracts exist
- Separate external REST API contracts from internal RPC contracts

---

## Frontend Rules (apps/web)

- Communicate with backend using the RPC layer inside the Cloudflare network
- Do not bypass RPC contracts with manual fetch calls unless explicitly required
- Prefer shared RPC types and clients from packages
- Do not duplicate backend request/response shapes manually
- Keep UI logic separate from RPC/data-fetching logic

---

## Shared Package Rules

Shared packages may include:

- packages/types
- packages/config
- packages/rpc
- packages/ui
- packages/tsconfig

### Rules

- Store shared RPC contracts in a dedicated package (e.g., packages/rpc)
- Keep shared code generic and reusable
- Do not duplicate contracts or types across apps
- Avoid coupling unrelated packages

---

## Database & Migrations

- Schema lives in /db/schema
- Use Drizzle ORM and Drizzle Kit
- Never edit migration files manually
- Always generate migrations via:
  - npx drizzle-kit generate
- Review migrations before applying

Gemini must not modify schema or migrations without approval.

---

## Turborepo Rules

- Respect existing turbo.json configuration
- Do not modify Turborepo pipelines without approval
- Keep tasks cacheable when possible
- Avoid unnecessary dependencies between workspaces

---

## Testing

- Run tests via:
  - npm run test
  - npm run test:ui

### Rules

- Add tests for new logic when appropriate
- Do not call real external APIs in tests
- Mock Facebook API and other services
- Ensure tests pass before suggesting completion

---

## External Integrations

- Facebook API is the primary data source
- Future integrations (e.g., TikTok) must be abstracted

### Rules

- Do not tightly couple logic to a single provider
- Validate and sanitize all external data
- Handle rate limits and failures gracefully

---

## Security

- Never expose API keys, tokens, or secrets
- Never log sensitive data
- Use environment variables for configuration
- Validate all inputs and external responses

---

## Deployment

- Services run within the Cloudflare network (Workers / microservices)
- Do not hardcode environment-specific values
- Do not modify deployment configuration without approval

---

## Consistency Rules

- Internal communication must use the Cloudflare RPC pattern
- External APIs may use REST where appropriate
- Do not mix RPC and ad hoc REST internally
- Prefer shared contracts and type reuse
- Follow existing project structure and naming
- Prefer consistency over cleverness

---

## When in Doubt

Gemini must:

- Ask clarifying questions
- Prefer safe, minimal changes
- Avoid assumptions about architecture
- Defer decisions that impact multiple systems
