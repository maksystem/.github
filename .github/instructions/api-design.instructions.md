---
applyTo: "**/*Controller.java,**/api/**/*.java,**/rest/**/*.java,**/controller/**/*.java,**/dto/**/*.java,**/model/**/*.java,**/endpoint/**/*.java,**/openapi/**/*.yaml,**/openapi/**/*.yml,**/api-spec/**/*.yaml,**/api-spec/**/*.yml"
---

# MAK REST API Design Rules — File-Level Enforcement

These rules apply when working on REST controllers, DTOs, endpoint definitions, and OpenAPI specifications.

## Controllers

### URI Design

- Class-level `@RequestMapping` must use **plural noun**, **kebab-case**, and **API version prefix**: `@RequestMapping("/api/v1/user-profiles")`
- Never use verbs in endpoint paths: `/api/v1/users` ✓ not `/api/v1/getUsers` ✗
- Nesting limit: max 2 levels — `/users/{user-id}/orders/{order-id}` ✓, `/users/{id}/orders/{id}/items/{id}` ✗
- No trailing slash, no file extension in the URI

### HTTP Method Mapping

- `@GetMapping` — read-only, must NOT modify state
- `@PostMapping` — create only, must return **`201 Created`** + `Location` header pointing to new resource
- `@PutMapping` — full replace, requires full object in request body
- `@PatchMapping` — partial update, only changed fields in body
- `@DeleteMapping` — remove resource, return **`204 No Content`**, no request body

### Response

- Always return `ResponseEntity<T>` — never raw objects
- Never return Database Entities — always map to a response DTO
- Use `@JsonInclude(JsonInclude.Include.NON_NULL)` on response DTOs
- Never expose: `passwordHash`, `internalId`, raw stack traces, or internal class names

### Content Type

- Always declare `consumes = MediaType.APPLICATION_JSON_VALUE` on write endpoints (`POST`, `PUT`, `PATCH`)
- Always declare `produces = MediaType.APPLICATION_JSON_VALUE`

### Security

- Every endpoint needs Spring Security authorization at the controller level
- Public endpoints must be explicitly permitted — default to deny
- Rate limiting must be configured for all public (internet-facing) endpoints

## Java DTOs (Data Transfer Objects)/Models Rules

- **File Pattern**: `src/main/java/**/*DTO.java`, `src/main/java/**/model/*.java`
- When adding new properties, always use **camelCase**.
- If a user starts typing a field with a hyphen (kebab-case), suggest the camelCase alternative immediately.
- Use `@JsonProperty("camelCaseName")` if Jackson is present.

### Validation (JSR-303/380)

- Always annotate request DTOs with validation constraints:
  - `@NotNull` / `@NotBlank` for required fields
  - `@Size(max = ...)` on ALL String fields — prevents DoS from oversized payloads
  - `@Email` for email fields
  - `@Min` / `@Max` for numeric range constraints
  - `@Pattern` for constrained string formats
- Trigger validation with `@Valid` or `@Validated` in controller method signature

### Security on DTOs

- Use `Enum` types instead of raw `String` for status/role/type fields
- Use `UUID` instead of `Long` for IDs — never expose auto-increment integers
- Never store or accept passwords in plain text — use secure tokens
- Sanitize all `String` fields with OWASP Java HTML Sanitizer to prevent XSS

### Prohibited Fields in Request DTOs

- Never allow clients to set: `id`, `createdAt`, `updatedAt`, `createdBy`, `role` (unless role assignment endpoint)
- Use Jackson's `@JsonIgnore` or separate create/update DTOs to enforce this

## Response Structure

### Standard Success Response (single resource)

```java
{
  "status": "success",
  "timestamp": "ISO-8601",
  "data": { /* resource fields */ }
}
```

### Standard Collection Response

```java
{
  "content": [ /* resource array */ ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "totalElements": 150,
    "totalPages": 8,
    "last": false
  }
}
```

### Standard Error Response

```java
{
  "status": 400,
  "errorCode": "VALIDATION_FAILED",
  "message": "Human-readable message — no internal details",
  "timestamp": "ISO-8601",
  "fieldErrors": [
    { "field": "email", "rejectedValue": "...", "message": "..." }
  ]
}
```

## IDs

- All resource IDs must be `UUID` — annotate path variables with `@PathVariable UUID id`
- Validate UUID format at the controller level — catch `MethodArgumentTypeMismatchException` and return `400`

## Pagination

- All collection endpoints **must** support pagination via `Pageable` parameter
- Default: `page=0`, `size=20` — configure `@PageableDefault(size = 20)`
- Response must always include `totalElements`, `totalPages`, `pageNumber`, `pageSize`
- Reject requests with `size > 100` to prevent abuse

## Error Handling

- Use a global `@ControllerAdvice` / `@RestControllerAdvice` for exception handling — no try/catch in controllers
- Return generic `401 Unauthorized` for auth failures — never reveal if username or password was wrong
- Return `404 Not Found` (not `403`) when a resource doesn't exist for a user without access (prevents resource enumeration)
- Log full exception details server-side only — never include stack traces in API responses

## OpenAPI Specification Files

- Every endpoint must be documented with `@Operation`, `@ApiResponse`, and `@Parameter` annotations (springdoc-openapi)
- Document all possible HTTP status code responses
- Include request/response schema examples
- Mark sensitive fields with `format: password` in the spec
- Use `$ref` components for reusable schemas — avoid duplication
