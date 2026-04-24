# API Conventions

> This document governs all external-facing REST API endpoints in `apps/api`.
> For internal frontend ↔ backend communication, use typed RPC contracts in `packages/rpc/` — not REST.
> Validation schemas live in `/schemas`. Business logic lives in `/services`. See `CLAUDE.md` for full architecture.

---

## General Principles

- Endpoints must be RESTful, predictable, and consistent
- Use JSON for all request and response bodies
- Keep contracts explicit — no implicit or undocumented fields
- Avoid breaking changes; version when necessary
- Thin handlers — delegate all logic to services (see Request Flow below)

---

## Request Flow

Every request must follow this order — no skipping steps:

```
1. Request received by handler
2. Validate & sanitize input (Zod schema from /schemas)
3. Map validated input to typed DTO
4. Pass DTO to service layer
5. Service calls database layer
6. Handler returns serialized response DTO
```

```ts
// ✅ Example: thin handler following the flow
app.post('/posts', async (c) => {
  const body = await c.req.json();

  // Step 2-3: validate and map to DTO
  const parsed = CreatePostSchema.safeParse(body);
  if (!parsed.success) {
    return c.json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid request body',
        details: parsed.error.flatten().fieldErrors,
      }
    }, 400);
  }

  // Step 4-5: delegate to service
  const post = await postService.create(parsed.data);

  // Step 6: return serialized response
  return c.json({ data: post }, 201);
});
```

---

## Response Format

### Success

All successful responses wrap data in a `data` key for consistency.

```json
// Single resource
{
  "data": {
    "id": "abc123",
    "title": "My Post",
    "status": "published",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}

// List resource
{
  "data": [
    { "id": "abc123", "title": "My Post" }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

### Error

All errors must use this shape — no exceptions:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request body",
    "details": [
      { "field": "email", "message": "Email is required" }
    ]
  }
}
```

| Field | Rules |
|---|---|
| `code` | Stable, `UPPER_SNAKE_CASE`, machine-readable. Never change a code once published. |
| `message` | Human-readable summary. Safe to display to users. |
| `details` | Optional. Use for field-level validation errors or structured metadata. |

**Standard error codes:**

| Code | When to use |
|---|---|
| `VALIDATION_ERROR` | Invalid or malformed input |
| `UNAUTHORIZED` | Missing or invalid authentication |
| `FORBIDDEN` | Authenticated but not permitted |
| `NOT_FOUND` | Resource does not exist |
| `CONFLICT` | Resource already exists or state conflict |
| `RATE_LIMITED` | Too many requests |
| `INTERNAL_ERROR` | Unexpected server failure |

- ❌ Never return `200 OK` for an error case
- ❌ Never expose stack traces, SQL errors, or internal messages in production responses
- ❌ Never use `500` for validation errors — that's always a `400`

---

## HTTP Status Codes

| Status | When to use |
|---|---|
| `200 OK` | Successful GET, PUT, PATCH |
| `201 Created` | Successful POST that creates a resource |
| `204 No Content` | Successful DELETE with no response body |
| `400 Bad Request` | Validation failure or malformed input |
| `401 Unauthorized` | Missing or invalid auth token |
| `403 Forbidden` | Authenticated but not authorized |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource or state conflict |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected failure — report to Sentry |

---

## Routing & Naming

- Use **plural nouns** for resource names
- Use **kebab-case** for multi-word segments
- Use **HTTP methods** to express actions — no verbs in paths
- Use **nested routes** only when the parent-child relationship is unambiguous

```
✅  GET    /posts
✅  POST   /posts
✅  GET    /posts/:postId
✅  PATCH  /posts/:postId
✅  DELETE /posts/:postId
✅  GET    /pages/:pageId/posts       ← clear parent-child
✅  GET    /social-accounts           ← kebab-case for multi-word

❌  POST   /createPost                ← verb in path
❌  GET    /post                      ← singular noun
❌  GET    /posts/getAll              ← verb
❌  GET    /pages/:pageId/posts/:postId/comments/:commentId/likes  ← too deeply nested; flatten if needed
```

### Query Parameters

Use query parameters for filtering, sorting, and pagination — never in the path:

```
GET /posts?status=published
GET /posts?sort=created_at&order=desc
GET /posts?page=2&limit=20
GET /posts?status=published&sort=created_at&page=1&limit=20
```

---

## Pagination

Use **cursor-based pagination** for all scalable list endpoints. Use page-based only for admin or low-volume endpoints — but never mix strategies in the same resource.

### Cursor-based (preferred)

```json
{
  "data": [...],
  "meta": {
    "nextCursor": "eyJpZCI6IjEyMyJ9",
    "hasMore": true,
    "limit": 20
  }
}
```

```
GET /posts?cursor=eyJpZCI6IjEyMyJ9&limit=20
```

### Page-based (admin / low-volume only)

```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

- Always include `meta` on list responses — never return a bare array
- Default `limit` to `20`; cap at `100`
- ❌ Do not mix cursor and page-based pagination for the same resource

---

## Validation

- Validate **all** inputs: body, query params, and route params
- All schemas live in `/schemas` — not co-located with handlers
- Use `safeParse` and return structured `400` errors on failure
- Map validated data to typed DTOs before passing to services
- ❌ Never pass raw `req.body` directly into a service or DB query
- ❌ Never skip validation for "simple" endpoints

```ts
// /schemas/post.ts
export const CreatePostSchema = z.object({
  title: z.string().min(1).max(255),
  content: z.string().min(1),
  status: z.enum(['draft', 'published']).default('draft'),
});

export type CreatePostDTO = z.infer<typeof CreatePostSchema>;
```

---

## Data Conventions

### IDs

- Use UUIDs for all resource IDs (not sequential integers)
- Always name the field `id` on responses; use `{resource}Id` in path params (`postId`, `pageId`)

### Dates & Times

- All timestamps in **ISO 8601 UTC** format: `"2024-01-15T10:30:00Z"`
- Always include `createdAt` and `updatedAt` on resource responses
- ❌ Never return Unix timestamps or locale-formatted dates

### Nulls & Optionals

- Return `null` for optional fields that have no value — do not omit the field
- ❌ Do not return `undefined` or missing keys for empty optional fields

```json
// ✅ Correct
{ "id": "abc", "deletedAt": null }

// ❌ Wrong
{ "id": "abc" }
```

---

## Versioning

Version the API when introducing **breaking changes** to an existing contract.

**Breaking changes (must version):**
- Removing or renaming a field in a response
- Changing a field's type
- Removing an endpoint
- Changing required/optional status of a request field

**Non-breaking changes (no versioning needed):**
- Adding a new optional field to a response
- Adding a new endpoint
- Adding a new optional request parameter

### Versioning strategy

Use path-based versioning:

```
/v1/posts
/v2/posts
```

- Start at `v1` — do not version prematurely
- Maintain the previous version until all consumers have migrated
- Document breaking changes in the PR description

---

## Rate Limiting

- All public endpoints must be rate limited
- Return `429 Too Many Requests` when limits are exceeded
- Include rate limit headers in responses:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1705312200
```

- Return a structured error body on `429`:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please try again later.",
    "details": { "retryAfter": 60 }
  }
}
```

---

## Idempotency

- `GET`, `PUT`, `DELETE` must be idempotent
- `POST` is not idempotent by default — for critical operations (e.g. payment, job triggers), support an `Idempotency-Key` header
- ❌ Never produce side effects from `GET` requests

---

## Security

- Authenticate all protected routes at the middleware layer
- Authorize actions in the service layer — not the handler
- ❌ Never expose API keys, tokens, or secrets in responses or logs
- ❌ Never leak SQL errors, stack traces, or internal paths in error messages
- All input and external API data must be sanitized via Zod before use
- See `CLAUDE.md` for full security and logging rules

---

## Consistency Rules

- ❌ Do not mix response shapes across endpoints
- ❌ Do not return bare arrays — always wrap in `{ data: [] }`
- ❌ Do not bypass validation for any endpoint regardless of perceived simplicity
- ❌ Do not use different field names for the same concept across endpoints (`userId` vs `user_id` vs `authorId`)
- Prefer clarity and consistency over brevity