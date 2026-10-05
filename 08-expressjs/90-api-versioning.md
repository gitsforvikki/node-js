# Lesson 90 — API Versioning

## Why API versioning matters

APIs evolve.

At some point you may need to make a breaking change.

Examples:

- rename fields
- change response shape
- change validation rules
- remove endpoints
- change business semantics

Versioning lets old clients continue working while new clients adopt the new contract.

---

## 1. What is a breaking change?

Suppose v1 returns:

```json
{
  "name": "Vikash"
}
```

v2 returns:

```json
{
  "firstName": "Vikash",
  "lastName": "Kumar"
}
```

Clients expecting `name` may break.

That is a breaking change.

---

## 2. URL versioning

Most common approach:

```text
/api/v1/users
/api/v2/users
```

Express:

```js
app.use(
  "/api/v1",
  v1Router
);

app.use(
  "/api/v2",
  v2Router
);
```

---

## 3. Advantages of URL versioning

- obvious
- easy to debug
- easy to document
- easy for clients
- easy to route

This is why it is widely used.

---

## 4. Header versioning

Example concept:

```http
Accept: application/vnd.myapp.v2+json
```

Advantages:
- URLs stay clean

Disadvantages:
- harder to discover
- harder to test manually
- more complex documentation

---

## 5. Custom header versioning

Example:

```http
X-API-Version: 2
```

Simple, but less standardized.

---

## 6. Query versioning

Example:

```text
/users?version=2
```

Usually less preferred for major API contracts because query params are generally better for resource retrieval options than protocol version identity.

---

## 7. Do not version every change

Non-breaking changes often do not require a new version.

Examples:

- adding optional response fields
- adding new endpoints
- adding optional query parameters

Versioning should be reserved for compatibility boundaries.

---

## 8. Version router architecture

```text
routes/
├── v1/
│   ├── user.routes.js
│   └── order.routes.js
└── v2/
    ├── user.routes.js
    └── order.routes.js
```

But duplicating everything can become expensive.

---

## 9. Share business logic

Do not copy entire services for every API version if business rules are unchanged.

Better:

```text
v1 controller
      |
      v
shared service

v2 controller
      |
      v
shared service
```

Only the API contract layer may differ.

---

## 10. Version transformation layer

Example:

Service returns:

```js
{
  firstName:
    "Vikash",
  lastName:
    "Kumar",
}
```

v1 controller transforms:

```js
{
  name:
    "Vikash Kumar"
}
```

v2 returns newer shape.

This avoids duplicating business logic.

---

## 11. Deprecation

Version lifecycle:

```text
v1 active
   |
   v
v2 released
   |
   v
v1 deprecated
   |
   v
migration window
   |
   v
v1 removed
```

Communicate deprecation clearly.

---

## 12. Deprecation headers

APIs can communicate lifecycle metadata using response headers where appropriate.

Examples may include:
- deprecation indicators
- sunset dates
- documentation links

Exact strategy depends on your API standards.

---

## 13. Backward compatibility

Before creating v2, ask:

> Can this change be made backward-compatible?

Example:

Instead of removing:

```json
{
  "name": "Vikash"
}
```

you may temporarily add:

```json
{
  "name": "Vikash",
  "firstName": "Vikash"
}
```

Then migrate clients gradually.

---

## 14. Internal vs public APIs

Public APIs usually require stronger versioning discipline because external consumers cannot be upgraded instantly.

Internal APIs may use:
- coordinated deployments
- contract testing
- shorter migration windows

Still, breaking changes should be controlled.

---

## 15. Database versioning is different

API version:

```text
/client contract
```

Database migration version:

```text
/schema evolution
```

Do not confuse them.

---

## 16. Common mistakes

### Mistake 1
Creating v2 for every small change.

### Mistake 2
Duplicating all business logic between versions.

### Mistake 3
Removing v1 immediately.

### Mistake 4
No deprecation plan.

### Mistake 5
Changing response shape silently.

---

## 17. Interview questions

### Why version an API?

To introduce breaking changes without immediately breaking existing clients.

### Most common versioning strategy?

URL versioning such as `/api/v1`.

### Should every new field require v2?

Usually no, if the change is backward-compatible.

### Where should version differences live?

Preferably in API contract/controllers/serializers, while sharing business logic where possible.

---

## 18. Strong interview answer

> API versioning is used when a breaking contract change cannot be made backward-compatible. The most common approach is URL versioning such as /api/v1 and /api/v2 because it is explicit and easy to document. I avoid duplicating business logic across versions; instead, I keep services shared and version the controller or response-mapping layer. I also plan deprecation and migration rather than removing older versions abruptly.

---

## Interview-Ready Summary

```text
API Versioning
   |
   +--> breaking changes
   +--> backward compatibility
   +--> /api/v1
   +--> /api/v2
   +--> deprecation
   +--> shared services

Version contract,
not every tiny change.
```

## Practice Task

Design v1 and v2 for a user endpoint where v1 returns:

```json
{
  "name": "Vikash Kumar"
}
```

and v2 returns separate first/last names.

Share the same service layer.
