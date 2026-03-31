# MAK REST API Design Guidelines

These rules apply to **all REST API development** across MAK System repositories.
Follow these guidelines whenever creating, reviewing, or suggesting changes to REST APIs.

---

## Core Principles

- APIs must be **resource-oriented** — use nouns, not verbs: `/users`, `/orders`, not `/getUsers`, `/createOrder`
- APIs must be **stateless** — every request must contain all information needed to process it; no server-side session state
- **OpenAPI Specification (OAS) is the source of truth** — prefer Contract-First (Design-First): draft the OpenAPI spec before writing any code
- All inputs are **untrusted by default** — validate and sanitize everything (Zero Trust)

---

## Naming Conventions

- Use **plural nouns** for collections: `/users`, `/products`, `/orders`
- Use **kebab-case** (lowercase + hyphens) for URIs: `/user-profiles` ✓ not `/userProfiles` or `/user_profiles` ✗
- **No trailing slashes**: `/users/123` ✓ not `/users/123/` ✗
- Use **forward slashes** for hierarchy only: `/users/{user-id}/orders/{order-id}`
- **Max 2 levels of nesting** — deeper nesting signals a resource modeling problem; flatten or introduce a dedicated endpoint
- **No file extensions** in URIs — use `Accept` header instead: `GET /reports` with `Accept: application/pdf`
- **Never include sensitive data** (API keys, passwords, tokens) in URLs

---

## HTTP Methods

| Method   | Purpose                                        | Request Body                   |
| -------- | ---------------------------------------------- | ------------------------------ |
| `GET`    | Retrieve resource(s) — must never modify state | None                           |
| `POST`   | Create a new resource                          | Required (new object)          |
| `PUT`    | Replace the entire resource                    | Required (full object)         |
| `PATCH`  | Update only specific fields                    | Required (changed fields only) |
| `DELETE` | Remove a resource                              | None — use path params         |

---

## Request Parameters

### Path Parameters

- Use **only for resource identification**: `/users/{user-id}`, `/orders/{order-id}`
- Always **required** — they point to a specific resource instance
- **Never** put sensitive info (passwords, API keys) in path parameters
- Always **validate and sanitize** to prevent SQL injection and path traversal attacks
- Use **UUID or Snowflake IDs** — never auto-incrementing integers (prevents enumeration attacks)

### Query Parameters

- Use for **filtering**, **sorting**, and **pagination**: `?status=active&page=0&size=20`
- Should be **optional** with sensible defaults applied server-side
- Use only for **safe, read-only operations** (GET) — never to trigger state changes or destructive actions
- **Never** include sensitive data (API keys, PII, passwords) — query strings are logged in plain text
- Multi-value syntax: `?tags=tech,news` or `?tag=tech&tag=news`

### Mandatory Pagination for Collections

```
GET /users?page=0&size=20&sortBy=name&sortDir=ASC
```

Response **must** include: `totalElements`, `totalPages`, `pageNumber`, `pageSize`

### Sorting

```
?sort=fieldName,asc
?sort=field1,asc&sort=field2,desc
```

### Field Selection (reduces payload size)

```
?fields=id,name,email
```

---

## HTTP Headers

- Format: `Name: Value` with **hyphen-separated** names: `Rate-Limit-Remaining` ✓
- **Do not use** `X-` prefixed custom headers (deprecated since 2012) — use descriptive names
- Prefer **IANA-registered headers** before creating custom ones
- Max total header size: 8–16KB (exceeding causes `431 Request Header Fields Too Large`)

### Required Security Headers (Public APIs)

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Cache-Control: no-store, no-cache, must-revalidate, proxy-revalidate
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## Request Body

- Use **`application/json`** as the default content type — explicitly set `consumes = MediaType.APPLICATION_JSON_VALUE`
- Use **specific DTOs** (Data Transfer Objects) — **never** expose Database Entities directly (prevents Mass Assignment attacks)
- Apply **JSR-303/380 validation** annotations: `@NotNull`, `@Size`, `@Email`, `@Min`, `@Max`
- Trigger validation with `@Valid` or `@Validated` in the controller
- Use **OWASP Java HTML Sanitizer** (or equivalent) to strip XSS from all string inputs
- Use **Enums or UUIDs** for IDs and status fields — not raw `String`
- **Never** store passwords or PII in plain text — use JWT or asymmetric encryption
- Define **`@Size(max=...)`** on all inputs to prevent DoS from oversized payloads
- Set a **maximum request body size** (2MB for JSON, 10MB+ for files) — reject with `413 Content Too Large`
- Always use **Parameterized Queries** (Spring Data JPA) — never concatenate user input into SQL

### Valid Request Body Example

```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane.doe@example.com",
  "role": "EDITOR",
  "preferences": {
    "darkMode": true,
    "notificationsEnabled": false
  }
}
```

---

## Response Body

Use a **consistent wrapper structure** for all responses:

### Success (single resource)

```json
{
  "status": "success",
  "timestamp": "2024-03-27T10:15:30Z",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane.doe@example.com",
    "createdAt": "2024-03-27T10:15:30Z"
  }
}
```

### Success (collection)

```json
{
  "content": [
    { "id": 1, "name": "Item A" },
    { "id": 2, "name": "Item B" }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "totalElements": 150,
    "totalPages": 8,
    "last": false
  }
}
```

### Error

```json
{
  "status": 400,
  "errorCode": "VALIDATION_FAILED",
  "message": "Input validation errors occurred",
  "timestamp": "2024-03-27T10:16:00Z",
  "fieldErrors": [
    {
      "field": "email",
      "rejectedValue": "invalid-email",
      "message": "Must be a well-formed email address"
    }
  ]
}
```

### Rules

- Use **camelCase** for all JSON keys
- Use `@JsonInclude(NON_NULL)` to omit null fields from responses
- **Never** expose sensitive fields (`passwordHash`, `internalId`, etc.)
- **Never** return stack traces or internal class names in responses
- Align response body with the HTTP status code

---

## HTTP Status Codes

| Code  | Name                  | When to Use                                                          |
| ----- | --------------------- | -------------------------------------------------------------------- |
| `200` | OK                    | Successful GET, PUT, PATCH                                           |
| `201` | Created               | Successful POST — include `Location` header pointing to new resource |
| `204` | No Content            | Successful DELETE                                                    |
| `400` | Bad Request           | Validation failure, malformed request                                |
| `401` | Unauthorized          | Missing or invalid authentication token                              |
| `403` | Forbidden             | Authenticated but not authorized for this resource                   |
| `404` | Not Found             | Resource does not exist                                              |
| `409` | Conflict              | Duplicate resource creation attempt                                  |
| `413` | Content Too Large     | Payload exceeds configured size limit                                |
| `429` | Too Many Requests     | Rate limit exceeded                                                  |
| `500` | Internal Server Error | Unexpected server-side failure                                       |
| `503` | Service Unavailable   | Server temporarily unable to handle requests                         |

---

## Security

### HTTPS

- **Always use HTTPS** — HTTP transmits data in plain text
- TLS 1.3 preferred; TLS 1.2 only as fallback for legacy clients (configure in Tomcat)

### Principle of Least Privilege

- Every API consumer only has access to what they need — nothing more
- Endpoints must be protected with access controls restricted to the specific user or role
- Treat every caller as untrusted, even internal services (Zero Trust)

### IDs

- **Use UUID or Snowflake IDs** — never auto-incrementing integers
- Auto-increment IDs allow enumeration attacks that expose sensitive business data

### Authentication — Service-to-Service (M2M)

| Method                             | When to Use                                                                               |
| ---------------------------------- | ----------------------------------------------------------------------------------------- |
| OAuth 2.0 Client Credentials (JWT) | Preferred — exchange `client_id` + `client_secret` via IdP (Keycloak, Microsoft Entra ID) |
| Mutual TLS (mTLS)                  | High-security internal traffic; requires internal PKI                                     |
| Basic Auth                         | Intranet-only endpoints where no IdP is available                                         |

### Authentication — User-Facing

- **OAuth 2.0 Authorization Code Flow + PKCE**
- After login, the IdP sends an authorization code; server exchanges it for ID/Access Tokens
- Validate JWT: verify `exp`, `iss`, `aud` claims against IdP's JWKS endpoint
- Return **generic `401 Unauthorized`** — **never** specify if the username or password was wrong (prevents enumeration)

### Authorization

- Enforce at the **Controller level** using Spring Security
- Users can only perform actions on resources they are authorized for

### Public APIs (Internet-Facing)

- **Rate limit**: 100 requests/minute per client
- Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- All security headers required (see above)

### Internal APIs (Intranet-Only)

- **Rate limit**: 1000 requests/minute
- No internet exposure
- Stack traces in dev/staging only — **never in production**

---

## Error Handling

Always return a **standardized JSON error structure**:

```json
{
  "timestamp": "2026-03-26T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/v1/users",
  "fieldErrors": [{ "field": "email", "message": "Invalid email format" }]
}
```

- **Never** return stack traces or internal class names to clients
- **Never** confirm whether a username or password was wrong — return generic error messages
- Log full error details **server-side**; surface only user-friendly messages in responses

---

## Versioning

### When to Create a New Version

Version on every **breaking change**, including:

- Removing or renaming an endpoint or response field
- Changing a data type (e.g., `int` → `float`)
- Adding a new **required** field to a request

Non-breaking changes (adding optional fields or new endpoints) do **not** require a new version.

### URL Path Versioning (required)

```
GET /api/v1/users
GET /api/v2/users
```

URL path versioning is preferred — the version is visible in every request and easy to route.

### Deprecation Process

1. Add header: `X-API-Warn: This endpoint is deprecated and will be removed on 2025-06-01. Use /v2/users instead.`
2. Notify clients via changelogs and documentation
3. Maintain the deprecated version through the defined deprecation period

### Project Structure per Version

```
src/
  api/
    v1/
      UsersController.java
    v2/
      UsersController.java
```

---

## API Documentation

- Use **SwaggerHub** for centralized, shared API documentation
- **Contract-First (Design-First)** — preferred for public APIs and cross-team projects:
  1. Draft the OpenAPI Specification (YAML/JSON) in SwaggerHub first
  2. Collect feedback and sign-off from frontend, backend, and QA teams
  3. Use SwaggerHub mock server to allow parallel frontend/backend development
  4. Generate server stubs and client SDKs from the finalized contract
  5. Validate implementation against the contract with automated tests
- **Contract-Last (Code-First)** — acceptable for legacy migrations and rapid prototyping:
  - Use `springdoc-openapi` to auto-generate OpenAPI at runtime
  - Import generated YAML/JSON into SwaggerHub
  - Sync automatically via GitHub integration to prevent documentation drift
  - Review generated Swagger UI frequently to catch accidental breaking changes

---

## Java Technology Stack

| Scenario                 | Framework                                   | Notes                                                           |
| ------------------------ | ------------------------------------------- | --------------------------------------------------------------- |
| New projects             | **Spring Boot** (`spring-boot-starter-web`) | Jackson for JSON, `ResponseEntity<T>`, `springdoc-openapi`      |
| Legacy on Tomcat/Jetty   | **Spring MVC**                              | `DispatcherServlet`, same tooling                               |
| SOAP + REST side-by-side | **JAX-RS**                                  | `jersey-spring-boot-starter` or `cxf-spring-boot-starter-jaxrs` |
