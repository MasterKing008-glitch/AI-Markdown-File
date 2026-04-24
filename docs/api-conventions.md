# API Conventions

## General Principles

- Design endpoints to be RESTful, predictable, and consistent
- Use clear, stable naming for resources and actions
- Keep request and response contracts explicit and versionable
- Use JSON for all request and response bodies
- Avoid breaking changes; introduce versioning when necessary

---

## Request & Response Contracts

- All endpoints must use typed DTOs
- Validate and sanitize all incoming data before business logic executes
- Never expose internal implementation details in responses
- Keep response structures consistent across endpoints

### Success Response

- Return structured, predictable data
- Avoid unnecessary nesting or inconsistent shapes

---

## Status Codes

Use standard HTTP status codes consistently:

- 200 OK for successful GET, PUT, PATCH, and DELETE operations
- 201 Created for successful resource creation
- 400 Bad Request for validation errors or malformed input
- 401 Unauthorized for missing or invalid authentication
- 403 Forbidden for authenticated requests without permission
- 404 Not Found for missing resources
- 500 Internal Server Error for unexpected server failures

- Do not return 200 OK for error cases
- Do not overload status codes with multiple meanings

---

## Error Handling

All errors must follow a consistent structure:

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "field": "email", "message": "Email is required" }
    ]
  }
}

### Rules

- code must be stable and machine-readable
- message must be human-readable
- details is optional and should contain structured field-level errors or metadata
- Never expose stack traces or internal system details in production responses

---

## Routing & Naming

- Use plural nouns for resources:
  - /posts
  - /followers
  - /pages

- Use nested routes only when the parent-child relationship is clear:
  - /pages/{pageId}/posts

- Avoid verbs in route paths
- Use HTTP methods to express actions

- Use query parameters for:
  - filtering
  - sorting
  - pagination

Examples:
- /posts?status=published
- /posts?sort=created_at
- /posts?page=1&limit=20

---

## Pagination

- Use one pagination strategy consistently across the API
- Prefer cursor-based pagination for scalable list endpoints
- If page-based pagination is used, use it consistently

List responses must include pagination metadata when applicable:

{
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}

---

## Validation

- Validate request body, query parameters, and route parameters
- Reject invalid input with 400 Bad Request
- Use a schema validation library such as Zod
- Do not pass unvalidated data into the service layer
- Do not mutate raw request bodies directly; map them into validated DTOs first

---

## Security

- Never expose API keys, access tokens, or secrets in code or responses
- Store sensitive configuration in environment variables or a secrets manager
- Authenticate all protected routes
- Authorize actions carefully at the middleware or service layer
- Sanitize all user input and external API data
- Avoid leaking internal implementation details in error messages

---

## Data & Architecture Rules

- Keep controllers and route handlers thin
- Delegate business logic to services
- Use a dedicated database access layer
- Do not write raw SQL in request handlers
- Use Drizzle ORM for database access and migrations

### Recommended Flow

1. Request received
2. Validate and sanitize input
3. Map input to DTO
4. Pass DTO to service layer
5. Service interacts with database layer
6. Return serialized response DTO

---

## Consistency Rules

- Do not mix response formats across endpoints
- Do not return different shapes for similar resources
- Do not bypass validation for small or simple endpoints
- Prefer consistency and clarity over cleverness

---

## Versioning

- Introduce versioning for breaking changes
- Use clear versioned paths when needed:
  - /v1/posts
  - /v2/posts

- Do not introduce breaking contract changes without versioning
