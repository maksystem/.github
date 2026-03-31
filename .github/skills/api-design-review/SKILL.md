---
name: api-design-review
description: "Use when reviewing REST API design, checking endpoints against MAK API guidelines, auditing a controller or OpenAPI spec, or validating API compliance before PR merge. Trigger phrases: review API, check API design, audit endpoint, validate REST, API compliance check."
---

# API Design Review Skill

When invoked, perform a structured review of the provided REST API code or specification against the MAK REST API Design Guidelines.

## Step 1 — Collect Context

Ask the user (or read from context) what to review:

- A specific controller file
- A DTO or request/response model
- An OpenAPI YAML/JSON spec
- A PR diff containing API changes

Use file search tools to locate the relevant files if not provided.

## Step 2 — Run the Checklist

Evaluate each item. Mark as ✅ Pass, ❌ Fail, or ⚠️ Warning (needs attention).

### URI Design

- [ ] Plural nouns used for collection resources (`/users`, not `/user`)
- [ ] kebab-case used in URI path segments (`/user-profiles`, not `/userProfiles`)
- [ ] No trailing slash in any URI
- [ ] No verbs in URI paths
- [ ] URI nesting does not exceed 2 levels
- [ ] No file extensions in URIs
- [ ] No sensitive data in URI (API keys, passwords, tokens)
- [ ] API version prefix present (`/api/v1/...`)
- [ ] UUIDs used for resource IDs — not auto-increment integers

### HTTP Methods

- [ ] GET endpoints do not modify state
- [ ] POST returns `201 Created` with `Location` header
- [ ] PUT sends full resource replacement
- [ ] PATCH sends only changed fields
- [ ] DELETE returns `204 No Content` with no request body
- [ ] Correct method used for each operation

### Request Body & DTOs

- [ ] Uses DTO — not a Database Entity or generic `Map`
- [ ] `@Valid` or `@Validated` applied in controller
- [ ] All fields have appropriate validation annotations (`@NotNull`, `@Size(max=...)`, `@Email`)
- [ ] String fields have `@Size(max=...)` defined (DoS prevention)
- [ ] Enum types used for status/role/type fields (not raw `String`)
- [ ] UUID used for ID fields (not `Long`)
- [ ] No sensitive fields settable by client (`id`, `createdAt`, `role` unless intended)
- [ ] `consumes = MediaType.APPLICATION_JSON_VALUE` declared on write endpoints

### Response Body

- [ ] Uses `ResponseEntity<T>` — not raw objects
- [ ] Returns DTO — not a Database Entity
- [ ] Consistent wrapper structure used (`status`, `timestamp`, `data` or `content`/`pageable`)
- [ ] `@JsonInclude(NON_NULL)` applied on response DTOs
- [ ] No sensitive fields exposed (`passwordHash`, `internalId`, etc.)
- [ ] No stack traces in error responses
- [ ] camelCase used for all JSON property names

### Status Codes

- [ ] `200` for GET/PUT/PATCH success
- [ ] `201` for POST success
- [ ] `204` for DELETE success
- [ ] `400` for validation failures
- [ ] `401` for authentication failures (generic — no hint about what was wrong)
- [ ] `403` for authorization failures
- [ ] `404` for missing resources
- [ ] `409` for conflicts/duplicates
- [ ] `413` for oversized payloads
- [ ] `429` for rate limit exceeded
- [ ] `500` for unexpected server errors

### Pagination (Collection Endpoints)

- [ ] `Pageable` parameter supported on all collection endpoints
- [ ] Default page size configured (`@PageableDefault`)
- [ ] Max page size enforced (reject `size > 100`)
- [ ] Response includes `totalElements`, `totalPages`, `pageNumber`, `pageSize`

### Security

- [ ] Spring Security authorization applied at controller level
- [ ] No auto-increment integer IDs used
- [ ] No sensitive data in URLs or query strings
- [ ] OWASP sanitization applied to string inputs
- [ ] No plain-text passwords/PII stored or returned
- [ ] Parameterized queries used — no string concatenation into SQL
- [ ] Request body size limit configured
- [ ] Rate limiting applied (100 req/min for public, 1000 req/min for internal)
- [ ] Security headers configured for public APIs

### Error Handling

- [ ] Global `@ControllerAdvice` used — no try/catch in controllers
- [ ] Standard error structure returned (`timestamp`, `status`, `errorCode`, `message`, `fieldErrors`)
- [ ] Auth errors return generic `401` — no username/password hints
- [ ] No stack traces or internal class names in responses
- [ ] Errors logged server-side with full detail

### Versioning

- [ ] API version in URL path (`/api/v1/`, `/api/v2/`)
- [ ] No breaking changes introduced in existing version
- [ ] Deprecated endpoints have `X-API-Warn` header
- [ ] Separate packages/classes per major API version

### Documentation

- [ ] `@Operation`, `@ApiResponse`, `@Parameter` annotations present (springdoc-openapi)
- [ ] All response status codes documented
- [ ] Request/response examples included
- [ ] Sensitive fields marked appropriately in spec

## Step 3 — Generate Report

Produce a summary report:

```
## API Design Review Report
**File(s) reviewed:** [list files]
**Date:** [current date]

### Summary
- ✅ Passed: X items
- ❌ Failed: Y items
- ⚠️ Warnings: Z items

### Failures (must fix before merge)
[List each ❌ with file location and specific violation]

### Warnings (should fix)
[List each ⚠️ with recommendation]

### Passed
[Brief confirmation of key passing items]
```

## Step 4 — Suggest Fixes

For each ❌ failure, provide a concrete code fix example using the MAK guidelines.

**Example fix for missing pagination:**

```java
// Before
@GetMapping("/users")
public ResponseEntity<List<UserDto>> getUsers() { ... }

// After
@GetMapping("/users")
public ResponseEntity<Page<UserDto>> getUsers(
    @PageableDefault(size = 20) Pageable pageable) { ... }
```

**Example fix for wrong status code on POST:**

```java
// Before
return ResponseEntity.ok(createdUser);

// After
URI location = URI.create("/api/v1/users/" + createdUser.getId());
return ResponseEntity.created(location).body(createdUser);
```
