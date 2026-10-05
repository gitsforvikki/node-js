# Lesson 105 — OpenAPI / Swagger

## Why this lesson matters

A production API needs a contract that humans and tools can understand.

OpenAPI helps describe:

- endpoints
- methods
- parameters
- request bodies
- response schemas
- authentication
- errors

Swagger is commonly associated with tools built around OpenAPI.

---

## 1. OpenAPI vs Swagger

This distinction is important.

```text
OpenAPI
  -> API specification standard

Swagger
  -> ecosystem/tools around that specification
```

Examples:
- Swagger UI
- Swagger Editor
- Swagger Codegen

Do not use the terms as if they are exactly the same thing.

---

## 2. What an OpenAPI document describes

Example concepts:

```text
GET /users/{id}
POST /users

Parameters
Request Body
Responses
Schemas
Security
```

---

## 3. Minimal OpenAPI shape

Example:

```yaml
openapi: 3.1.0

info:
  title: User API
  version: 1.0.0

paths:
  /users:
    get:
      responses:
        "200":
          description: Users returned
```

---

## 4. info section

```yaml
info:
  title: CareerLoop API
  version: 1.0.0
  description: Job tracking API
```

Useful for generated docs.

---

## 5. servers

```yaml
servers:
  - url: https://api.example.com/api/v1
```

You may define:
- production
- staging
- local

Do not expose internal/private infrastructure unnecessarily.

---

## 6. Path parameters

Example:

```yaml
/users/{id}:
  get:
    parameters:
      - in: path
        name: id
        required: true
        schema:
          type: string
```

Path params are required by definition in OpenAPI.

---

## 7. Query parameters

```yaml
parameters:
  - in: query
    name: page
    schema:
      type: integer
      minimum: 1
```

You can document:
- type
- default
- enum
- limits
- description

---

## 8. Request body

Example:

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: "#/components/schemas/CreateUserRequest"
```

---

## 9. Reusable schemas

```yaml
components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
      properties:
        id:
          type: string
        name:
          type: string
```

Reuse avoids duplication.

---

## 10. Response schemas

```yaml
responses:
  "200":
    description: Success
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/User"
```

Document error responses too.

---

## 11. Standard error schema

```yaml
Error:
  type: object
  properties:
    error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        requestId:
          type: string
```

This matches your real API error contract.

---

## 12. Security schemes

Bearer JWT:

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

Then:

```yaml
security:
  - bearerAuth: []
```

---

## 13. API key security

```yaml
type: apiKey
in: header
name: X-API-Key
```

Document authentication explicitly.

---

## 14. Examples

Good API docs include realistic examples.

Example:

```yaml
example:
  id: usr_123
  name: Vikash Kumar
```

Examples make docs much easier to use.

---

## 15. Swagger UI

Swagger UI renders an OpenAPI document as interactive documentation.

Benefits:
- browse endpoints
- inspect schemas
- send test requests
- view auth requirements

---

## 16. Documentation is not enough

An OpenAPI file that is never validated against implementation can drift.

This is one of the biggest real-world problems.

```text
Docs say:
field = name

Implementation returns:
fullName
```

Now the contract is wrong.

---

## 17. Design-first approach

Flow:

```text
OpenAPI contract
     |
     v
review
     |
     v
implementation
     |
     v
generated clients/tests
```

Useful for teams and public APIs.

---

## 18. Code-first approach

Flow:

```text
route/schema code
     |
     v
generate OpenAPI
```

Convenient in application teams.

Risk:
docs become tied to framework annotations/tooling.

---

## 19. Which approach is better?

Neither is always best.

Design-first:
- strong contract discipline
- cross-team collaboration

Code-first:
- less duplication
- faster implementation

Hybrid is common:
- shared schemas
- generated validation/docs

---

## 20. Contract testing

You can validate that implementation matches spec.

Examples:
- response schema checks
- request validation
- generated tests

This reduces drift.

---

## 21. Client generation

OpenAPI can generate:
- TypeScript clients
- Java clients
- SDKs
- models

This is especially useful for frontend-backend integration.

---

## 22. Schema reuse

Avoid defining the same User object in 20 endpoints.

Use:

```text
$ref
```

to shared components.

---

## 23. Versioning OpenAPI

If API has:

```text
v1
v2
```

maintain contracts for each active version.

Do not overwrite v1 docs with v2.

---

## 24. Document pagination

Example:

```yaml
pagination:
  nextCursor:
    type: string
    nullable: true
  hasMore:
    type: boolean
```

Clients should not guess pagination semantics.

---

## 25. Document rate limits

OpenAPI may describe relevant headers and endpoint behavior, but operational rate-limit policy should also be clearly documented.

Example response:

```text
429
```

with retry headers.

---

## 26. Document errors

Do not document only:

```text
200
```

Also include:
- 400
- 401
- 403
- 404
- 409
- 422
- 429
- 500

as appropriate.

---

## 27. OpenAPI is not runtime security

Having:

```yaml
security:
  - bearerAuth: []
```

does not enforce JWT validation.

Your application still needs actual authentication middleware.

The spec documents the contract.

---

## 28. OpenAPI is not validation unless integrated

Likewise, a schema in docs does not automatically validate runtime requests.

You need:
- validator middleware
- generated validator
- shared schema integration

---

## 29. Avoid documenting secrets

Never put:
- real API keys
- tokens
- production credentials

inside examples/spec files committed to public repositories.

---

## 30. Common mistakes

### Mistake 1
Calling OpenAPI and Swagger identical.

### Mistake 2
Documenting only success responses.

### Mistake 3
Docs drift from implementation.

### Mistake 4
No shared schemas.

### Mistake 5
Putting production secrets in examples.

### Mistake 6
Assuming spec automatically enforces security.

---

## 31. Interview questions

### OpenAPI vs Swagger?

OpenAPI is the specification; Swagger refers to tools/ecosystem around it.

### Why use OpenAPI?

For machine-readable API contracts, documentation, client generation, validation, and testing.

### Design-first vs code-first?

Design-first starts from the contract; code-first generates the contract from implementation. Both have trade-offs.

### What is the biggest danger?

Specification drift from the real implementation.

---

## 32. Strong interview answer

> OpenAPI is a machine-readable API contract describing endpoints, parameters, request bodies, responses, schemas, and security requirements. Swagger is the tooling ecosystem around OpenAPI, such as Swagger UI. I use reusable schemas, document both success and error responses, and keep the spec synchronized with implementation through code generation or contract tests. The spec documents security and validation requirements, but it does not enforce them unless integrated with runtime tooling.

---

## Interview-Ready Summary

```text
OpenAPI
   |
   +--> paths
   +--> parameters
   +--> request bodies
   +--> responses
   +--> schemas
   +--> security

Swagger
   -> UI/editor/codegen tools

Best practice:
shared schemas
error docs
contract tests
no drift
```

## Section 9 Final Mental Model

```text
REST Architecture
      |
      v
Resource Design
      |
      v
Methods + Idempotency
      |
      v
Status Codes
      |
      v
Validation
      |
      v
Pagination / Filtering / Sorting / Search
      |
      v
Response + Error Contracts
      |
      v
Versioning
      |
      v
Rate Limiting
      |
      v
OpenAPI Contract
```

Section 9 is now complete as a production REST API design foundation.

## Practice Task

Create an OpenAPI 3.1 document for:

```text
GET    /users
GET    /users/{id}
POST   /users
PATCH  /users/{id}
DELETE /users/{id}
```

Include:
- auth
- validation schemas
- pagination
- error responses
- examples
- rate-limit response
