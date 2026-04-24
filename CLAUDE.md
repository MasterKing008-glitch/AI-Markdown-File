# CLAUDE.md

## Project Overview

This repository supports AI-assisted development in a Turborepo-based monorepo.
All contributors (human or AI) must follow these guidelines before introducing any changes.

> **AI Safety Rule:** Claude must ask before modifying shared packages, RPC contracts, database schema, dependencies, Turborepo config, or performing cross-workspace refactors. When in doubt, ask.

---

## Architecture

### Monorepo Structure (Turborepo)

```
apps/
  api/        → backend API / RPC service (Hono on Node.js)
  web/        → frontend application
packages/
  types/      → shared TypeScript types
  config/     → shared configuration
  rpc/        → typed RPC contracts (source of truth for internal comms)
  providers/  → social platform provider abstractions
  utils/      → shared utilities
```

### Core Stack

| Layer | Technology |
|---|---|
| Backend | Hono on Node.js, TypeScript strict mode |
| Database | PostgreSQL (Neon) via Drizzle ORM |
| Validation | Zod |
| Linting / Formatting | Biome + Ultracite |
| Testing | Vitest |
| Deployment | Cloudflare Workers / microservices |
| Observability | Sentry |
| Integrations | Facebook API (TikTok planned) |

---

## TypeScript Rules

- **MUST** use strict mode — `"strict": true` in tsconfig
- **MUST NOT** use `any` — use `unknown` and narrow explicitly
- Prefer explicit types over inference when the type is non-obvious
- Always infer types from Drizzle schema — never duplicate manually

```ts
// ❌ Wrong
async function getUser(id: any) { ... }

// ✅ Correct
async function getUser(id: string): Promise<typeof users.$inferSelect> { ... }
```

---

## Code Style

- Use Biome for all formatting and linting — do not introduce Prettier or ESLint
- Run `npm run lint` before committing
- Keep functions small and focused (single responsibility)
- Write self-documenting code; only comment non-obvious logic
- Avoid unnecessary abstractions

---

## Project Structure & Separation of Concerns

```
/routes (or RPC handlers)   → HTTP/RPC handling only — no business logic, no DB queries
/services                   → business logic — calls /db helpers, not ORM directly in-line
/db                         → Drizzle schema definitions + query helper functions
/schemas                    → Zod validation schemas (not co-located with routes)
/packages                   → shared code (types, RPC contracts, providers, config)
```

**Layer rules:**
- Handlers call services. Services call db helpers. Never skip layers.
- ❌ Do not write Drizzle queries inside route handlers
- ❌ Do not import route-level concerns into services

```ts
// ❌ Wrong — query inside handler
app.get('/users/:id', async (c) => {
  const user = await db.select().from(users).where(eq(users.id, c.req.param('id')));
  return c.json(user);
});

// ✅ Correct — handler → service → db helper
app.get('/users/:id', async (c) => {
  const user = await userService.getById(c.req.param('id'));
  return c.json(user);
});
```

---

## Validation

- Validate ALL inputs (body, query, params) with Zod
- Reject invalid input with `400 Bad Request`
- Never trust external API responses — validate them too
- Zod schemas live in `/schemas`, not co-located with routes

```ts
// ✅ Example
const CreatePostSchema = z.object({
  title: z.string().min(1).max(255),
  content: z.string().min(1),
});

app.post('/posts', async (c) => {
  const body = await c.req.json();
  const parsed = CreatePostSchema.safeParse(body);
  if (!parsed.success) return c.json({ error: { code: 'VALIDATION_ERROR', message: parsed.error.message } }, 400);
  return c.json(await postService.create(parsed.data));
});
```

---

## API Conventions

- Follow `docs/api-conventions.md` for full detail
- RESTful and predictable endpoints (for external APIs)
- Use typed DTOs for all request/response contracts
- Return consistent error responses:

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": "optional"
  }
}
```

- Use correct HTTP status codes: `400`, `401`, `403`, `404`, `500`
- Never expose stack traces in production responses

---

## Internal Communication (Cloudflare RPC)

- Frontend ↔ backend communicate via **typed RPC contracts** over the Cloudflare network
- RPC contracts live in `packages/rpc/` — this is the single source of truth
- Do not introduce ad hoc REST calls where RPC is intended

```ts
// packages/rpc/src/posts.ts — shared contract
export type GetPostRequest = { id: string };
export type GetPostResponse = { id: string; title: string; content: string };

// ❌ Wrong — ad hoc fetch inside web app
const res = await fetch('/api/posts/123');

// ✅ Correct — use the typed RPC client from packages/rpc
const post = await rpc.posts.get({ id: '123' });
```

- Do not duplicate request/response shapes across apps
- External API contracts must be kept separate from internal RPC contracts

---

## Database & Drizzle ORM

### Schema Conventions

- Singular table names: `user`, `post` (not `users`, `posts`)
- `snake_case` for all column names
- Always include `created_at` and `updated_at` timestamps
- Define relations explicitly

```ts
// ✅ Example schema: /db/schema/user.ts
export const user = pgTable('user', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

### Migration Workflow

1. Edit schema in `/db/schema`
2. Generate migration:
   ```bash
   npx drizzle-kit generate
   ```
3. Review generated SQL carefully
4. Test locally:
   ```bash
   npx drizzle-kit migrate
   ```
5. Commit schema **and** migration together

### Migration Safety Rules

- ❌ Never edit existing migration files
- ❌ Never run manual SQL in production
- ❌ Never run `drizzle-kit migrate` in CI without review
- Prefer additive migrations — avoid destructive changes unless explicitly required
- All migrations must be reviewed before merging

### Query Rules

- All DB access through Drizzle ORM — no raw SQL unless absolutely unavoidable
- Encapsulate queries in `/db` helper functions or `/services`
- ❌ Never write Drizzle queries inline in handlers (see Separation of Concerns)

---

## Social Provider Architecture

All social platforms must follow the shared provider abstraction in `packages/providers/`.

### Shared Interface

```ts
// packages/providers/src/types.ts
export interface SocialProvider {
  getPosts(params: GetPostsParams): Promise<NormalizedPost[]>;
  getFollowers(params: GetFollowersParams): Promise<NormalizedFollower[]>;
  getMetrics(params: GetMetricsParams): Promise<NormalizedMetrics>;
}
```

### Provider Rules

- Each platform implements `SocialProvider` in its own isolated module:
  - `packages/providers/src/facebook/FacebookProvider.ts`
  - `packages/providers/src/tiktok/TikTokProvider.ts` _(future)_
- All external data must be normalized into internal DTOs **inside** the provider — never expose raw API shapes beyond the provider layer
- ❌ Do not tightly couple business logic to a single provider
- New providers must implement the interface and must not require changes to existing providers

---

## External API Usage

- Respect rate limits at all times
- Implement retries with exponential backoff
- Cache responses where appropriate
- Validate all external API responses with Zod before use

---

## Observability & Error Handling

- Use **Sentry** for all error tracking — no `console.log` in production
- Catch errors at boundaries (handlers and services)
- Report unexpected errors to Sentry with useful context (request id, user id if safe, operation name)
- Return sanitized error responses to clients — never expose internal stack traces

```ts
// ✅ Correct error boundary pattern
try {
  return await userService.getById(id);
} catch (err) {
  Sentry.captureException(err, { extra: { userId: id, operation: 'getUser' } });
  return c.json({ error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' } }, 500);
}
```

### Logging Rules

- ❌ Never log tokens, passwords, or personal data
- ❌ Never use `console.log` in production code
- Include request id, safe user context, and operation name in Sentry events
- Do not over-log — only capture meaningful, actionable errors

---

## Testing

```bash
npm run test       # backend tests
npm run test:ui    # UI / Vitest tests
npm run lint       # linting
```

### Structure

- Unit tests for all service and business logic functions
- Integration tests for API/RPC endpoints
- Test files live in `__tests__/` adjacent to the module they test, or co-located as `*.test.ts`
- Mock all external services (Facebook API, etc.) — never call real third-party APIs in tests

```ts
// ✅ Example: mock the provider, test the service
vi.mock('@/providers/facebook/FacebookProvider');
const mockProvider = vi.mocked(FacebookProvider);
mockProvider.getPosts.mockResolvedValue([...]);
```

- Ensure all tests pass locally before pushing

---

## Environment Configuration

- ❌ Never commit `.env` files or secrets
- All variables must be documented in `.env.example`
- Use `UPPER_SNAKE_CASE` for all variable names
- Access env vars through a typed config module — never `process.env.X` scattered across files

---

## Security

- Sanitize all user input and external data (Zod handles this)
- Use least-privilege access for database and APIs
- Never expose internal system details in API responses
- Never log sensitive data (see Observability)

---

## Turborepo Rules

- Respect the existing workspace structure — `apps/*` and `packages/*`
- ❌ Do not move files across workspaces without approval
- ❌ Do not create new packages unless necessary — prefer extending existing shared packages
- ❌ Do not modify `turbo.json` or pipeline config without approval
- Keep all tasks cacheable and deterministic

---

## CI Requirements

All PRs must pass:

- `tsc --noEmit` (zero TypeScript errors)
- `npm run lint` (Biome clean)
- `npm run test` (all tests green)
- Migration validation (if schema changed)

PRs must not be merged if any check fails.

---

## Definition of Done

A change is complete when:

- Code compiles with zero TypeScript errors
- Lint passes with no warnings
- All relevant tests pass; new logic has test coverage
- API/RPC changes are documented
- Schema changes include generated Drizzle migrations (committed together)
- No secrets, sensitive data, or stack traces are exposed
- No security or data risks are introduced

---

## Branch & PR Process

> _This section is for human contributors. AI should not open PRs autonomously._

- Branch naming: `feature/<description>`, `bugfix/<description>`, `hotfix/<description>`
- One PR per change; require at least one human reviewer before merging
- PR description must include: purpose, testing notes, and relevant issue links

---

## AI Usage Guidelines

- AI-generated code must be reviewed before merging
- Do not blindly accept generated DB queries or migrations — review the SQL
- All generated code must follow TypeScript strict rules and project conventions
- Prefer simple, readable solutions over complex generated ones

### AI Must Ask Before

- Modifying any file in `packages/`
- Changing RPC contracts in `packages/rpc/`
- Adding or upgrading dependencies
- Changing database schema in `/db/schema`
- Editing `turbo.json` or pipeline configuration
- Moving files across workspace boundaries
- Performing any cross-workspace refactor