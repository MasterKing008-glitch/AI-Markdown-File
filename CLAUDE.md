# CLAUDE.md

## Project Overview

This repository supports AI-assisted development workflows. It defines the core constraints, architecture, coding standards, and collaboration rules for contributors and AI tools.

All contributors (human or AI) must follow these guidelines before introducing changes.

---

## Architecture

- **Backend:** Hono on Node.js with TypeScript (strict mode)
- **Database:** PostgreSQL (Neon) with Drizzle ORM
- **Integrations:** Facebook (future: TikTok) APIs for posts, likes, followers, and audience metrics
- **Testing:** Vitest for frontend and integration testing

See:
- `docs/architecture.md` for system overview  
- `docs/api-conventions.md` for API design patterns  

---

## API Conventions

- Follow `docs/api-conventions.md`
- Keep endpoints RESTful and predictable
- Use typed DTOs for all request/response contracts
- Validate and sanitize all inputs
- Return consistent error responses and HTTP status codes

### Error Response Format

All errors must follow this structure:

{
  "error": {
    "code": "string",
    "message": "string",
    "details": "optional"
  }
}

- Do not expose stack traces in production
- Use appropriate HTTP status codes (400, 401, 403, 404, 500, etc.)

---

## Coding Rules

- MUST use **TypeScript strict mode**
- MUST NOT use `any`
- MUST use **Biome** for formatting and linting
- MUST use **Ultracite** for code quality
- Write clear, self-documenting code
- Keep functions small and focused
- Prefer explicit types over implicit inference when unclear
- Avoid unnecessary abstractions

---

## Validation

- All inputs (body, query, params) must be validated
- Use a schema validation library (e.g., Zod)
- Reject invalid input with `400 Bad Request`
- Never trust external API data without validation

---

## Database & Migrations (Drizzle)

- Use **Drizzle ORM** for all database access and schema definitions
- Use **Drizzle Kit** for generating and running migrations

### Rules

- Do not edit existing migration files
- Always generate migrations via Drizzle:
  - `npx drizzle-kit generate`
- Apply migrations via:
  - CI pipeline or approved deployment workflow
- Never run manual SQL changes directly in production
- Keep schema definitions (`/db/schema`) as the source of truth

### Workflow

1. Update schema in `/db/schema`
2. Generate migration:
   npx drizzle-kit generate
3. Review generated SQL carefully
4. Run locally if needed:
   npx drizzle-kit migrate
5. Commit schema + migration together

### Safety

- Review all schema changes before merging
- Avoid destructive changes unless explicitly required
- Use additive migrations when possible

---

## Database Access

- All database queries must go through Drizzle ORM
- Do not write raw SQL unless absolutely necessary
- Encapsulate queries inside `/services` or `/db` modules
- Keep query logic separate from route handlers

### Example Pattern

- `/routes` → HTTP handling
- `/services` → business logic
- `/db` → schema and query helpers

### Type Safety

- Always infer types from Drizzle schema
- Avoid duplicating types manually

---

## Schema Conventions (Drizzle)

- Use singular table names (`user`, `post`)
- Use `snake_case` for column names
- Use `created_at`, `updated_at` timestamps consistently
- Define relations explicitly where applicable

---

## Environment Configuration

- Never commit `.env` files or secrets
- All environment variables must:
  - Be documented in `.env.example`
  - Use `UPPER_SNAKE_CASE`
- Access env vars through a typed config module
- Use secure secret management in production

---

## Testing

Commands:
- `npm run test` → backend tests  
- `npm run test:ui` → UI / Vitest tests  
- `npm run lint` → linting  

### Testing Guidelines

- Write unit tests for business logic
- Write integration tests for API endpoints
- Mock external services (e.g., Facebook APIs)
- Do not call real third-party APIs in tests
- Ensure tests pass locally before pushing

---

## CI Requirements

All pull requests must pass:

- Lint checks
- TypeScript compilation (`tsc --noEmit`)
- All tests
- Migration validation (if applicable)

PRs must not be merged if any checks fail.

---

## Definition of Done

A change is complete when:

- Code compiles with no TypeScript errors
- Lint passes
- All relevant tests pass
- New functionality includes tests where appropriate
- API changes are documented
- Database changes include generated Drizzle migrations
- No security or data risks are introduced

---

## Deployment

- Prefer automated deployment pipelines over manual changes
- Document deployment steps once environments are finalized
- Ensure:
  - Environment variables are configured securely
  - Migrations are applied in a controlled manner

---

## External API Usage

- Respect rate limits (Facebook, future TikTok APIs)
- Implement retry logic with backoff where appropriate
- Cache responses when possible
- Validate all third-party responses before use

---

## Security

- Never log sensitive data (tokens, passwords, personal data)
- Sanitize all user input and external data
- Use least-privilege access for database and APIs
- Avoid exposing internal system details in responses

---

## Project Structure (Guideline)

/routes     → API endpoints
/services   → business logic
/db         → database access and schema
/schemas    → validation schemas

Keep separation of concerns clear and consistent.

---

## Branch & PR Process

- Use descriptive branch names:
  - feature/<description>
  - bugfix/<description>
  - hotfix/<description>

- Open a PR for every change
- Require at least one reviewer before merging
- PR descriptions must include:
  - Purpose of the change
  - Testing notes
  - Relevant links/issues

---

## AI Usage Guidelines

- AI-generated code must be reviewed before merging
- Do not blindly accept generated database queries or migrations
- Ensure all generated code follows:
  - TypeScript strict rules
  - Project conventions
- Prefer simple, readable solutions over complex AI-generated ones

---

## Safety & Collaboration Notes

- Never commit secrets or sensitive data
- Never bypass migration workflows
- Maintain strict typing and code quality standards
- When in doubt, prioritize clarity, safety, and maintainability
