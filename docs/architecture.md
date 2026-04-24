# Architecture

## Overview

This repository is a Node.js API service designed for team collaboration and AI-assisted development.

Its primary responsibility is to ingest, process, and expose social media data from Facebook, with a design that supports future expansion to additional platforms such as TikTok.

The system prioritizes:
- Maintainability
- Type safety
- Clear separation of concerns
- Scalability and extensibility

---

## Backend

- Framework: Hono (Node.js)
- Language: TypeScript (strict mode)
- Runtime: Node.js
- Application type: REST API service

### Design Principles

- Keep route handlers thin
- Encapsulate business logic in services
- Use typed DTOs for all data flow
- Enforce validation at system boundaries
- Favor explicit and readable code over abstraction

---

## Database

- Database: PostgreSQL
- Hosting: Neon
- ORM: Drizzle ORM
- Migrations: Drizzle Kit

### Data Access Pattern

- Schema definitions live in `/db/schema`
- Queries are handled via Drizzle ORM
- Business logic must not directly access the database
- Use a service or repository layer for all database interactions

### Migration Strategy

- Schema is the source of truth
- All changes go through generated migrations
- Migrations are reviewed before being applied
- No manual database changes in production

---

## Social Integration

- Primary source: Facebook API
- Future sources: TikTok and other platforms

### Data Types

- Posts
- Likes
- Followers
- Audience metrics

### Integration Model

- Data is collected via:
  - Scheduled jobs, or
  - On-demand API triggers

- External data is:
  - Validated
  - Normalized into internal DTOs
  - Stored in the database

### Design Considerations

- Abstract platform-specific logic behind service layers
- Avoid coupling core logic to a single provider
- Prepare for multi-platform support

---

## Testing

- Backend tests: `npm run test`
- UI / integration tests: `npm run test:ui` (Vitest)

### Testing Strategy

- Unit tests for business logic
- Integration tests for API endpoints
- Mock external services (e.g., Facebook API)
- Do not call real third-party APIs during tests
- Ensure all tests pass before merging changes

---

## Deployment

- Prefer automated deployment pipelines over manual deployment
- Do not store secrets or credentials in source control
- Use environment variables or a secrets manager

### Deployment Requirements

- Validate all environment variables before startup
- Apply database migrations in a controlled environment
- Monitor deployment for errors and rollback if necessary

---

## Scalability & Future Considerations

- Design services to be platform-agnostic where possible
- Keep integrations modular and replaceable
- Ensure database schema can evolve without breaking existing data
- Plan for increased data volume and API usage

---

## Project Structure (Guideline)

/routes     → API endpoints  
/services   → business logic  
/db         → schema and database access  
/schemas    → validation schemas  

Maintain clear separation of concerns across all layers.
