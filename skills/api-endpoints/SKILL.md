---
description: Guide the creation of new API endpoints. Use when adding routes, handlers, or controllers to the project.
---

# Adding API Endpoints

Follow these conventions whenever you add or change an API endpoint.

## Where route files live

- Each resource gets its own route file: `src/routes/<resource>.routes.ts` (for example `src/routes/users.routes.ts`).
- Request handling logic goes in a controller: `src/controllers/<resource>.controller.ts`.
- Business logic and data access go in a service: `src/services/<resource>.service.ts`. Controllers never talk to the database directly.
- Request validation schemas live in `src/validators/<resource>.schema.ts`.
- Register every new route file in `src/routes/index.ts`. Don't mount routes anywhere else.
- Tests mirror the source tree: `tests/routes/<resource>.test.ts`.

## Naming conventions

- **URLs:** lowercase, kebab-case, plural nouns for collections (`/api/v1/order-items`, `/api/v1/users/:userId`). No verbs in paths; the HTTP method carries the action.
- **Versioning:** every route is prefixed with `/api/v<major>`.
- **Path and query params:** camelCase (`:userId`, `?pageSize=20`).
- **Controller functions:** verb + resource (`listUsers`, `getUser`, `createUser`, `updateUser`, `deleteUser`).
- **Files:** kebab-case with a role suffix (`order-items.routes.ts`, `order-items.controller.ts`).
- **JSON fields:** camelCase in both requests and responses.

## HTTP methods and status codes

| Action            | Method   | Success status   |
|-------------------|----------|------------------|
| List a collection | `GET`    | `200`            |
| Fetch one item    | `GET`    | `200`            |
| Create            | `POST`   | `201` + `Location` header |
| Full replace      | `PUT`    | `200`            |
| Partial update    | `PATCH`  | `200`            |
| Delete            | `DELETE` | `204` (no body)  |

## Error handling

- Validate every request body, query, and params at the edge with the resource's schema. Reject invalid input with `400` before the controller logic runs.
- Throw typed errors (`NotFoundError`, `ValidationError`, `ConflictError`, `UnauthorizedError`, `ForbiddenError`) from services and controllers. Don't build error responses by hand inside handlers.
- Wrap async handlers so rejected promises reach the central error middleware. Never leave a promise unhandled.
- The central error middleware is the only place that turns errors into HTTP responses.
- Unexpected errors return `500` with a generic message. Log the full error and stack trace on the server, but never send stack traces, SQL, or internal details to the client.
- Status code mapping: validation `400`, unauthenticated `401`, forbidden `403`, not found `404`, conflict `409`, rate limited `429`, unexpected `500`.

## Response shape

Every response uses the same envelope.

Success:

```json
{
  "data": { "id": "123", "name": "Ada" },
  "meta": { "requestId": "abc-123" }
}
```

Paginated list:

```json
{
  "data": [ { "id": "123" } ],
  "meta": { "page": 1, "pageSize": 20, "total": 57, "requestId": "abc-123" }
}
```

Error:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "User 123 was not found.",
    "details": []
  },
  "meta": { "requestId": "abc-123" }
}
```

- `error.code` is an UPPER_SNAKE_CASE constant that clients can branch on. `error.message` is human-readable.
- `error.details` lists per-field problems for validation errors (`{ "field": "email", "message": "must be a valid email" }`).
- Timestamps are ISO 8601 strings in UTC.
- `DELETE` returns `204` with no body.

## Before committing

Don't commit a new or changed endpoint until all of these are true:

- [ ] The route is registered in `src/routes/index.ts` under the correct version prefix.
- [ ] Request input has a validation schema, and invalid input returns `400` in the standard error shape.
- [ ] Authentication and authorization are applied, or the route is explicitly marked public with a comment explaining why.
- [ ] Tests cover the success path, validation failure, not found (where relevant), and auth failure.
- [ ] All tests, lint, and type checks pass.
- [ ] Responses use the standard envelope, and the status codes match the table above.
- [ ] No secrets, stack traces, or internal details leak in responses or logs.
- [ ] API docs (OpenAPI spec or README) are updated with the new endpoint, its parameters, and example responses.
