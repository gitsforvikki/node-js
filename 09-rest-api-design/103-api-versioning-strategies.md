# Lesson 103 — API Versioning Strategies

## Why this lesson matters

APIs change over time.

The important interview question is not:

> How do you put v1 in a URL?

The real questions are:

- What counts as breaking?
- When do you need a new version?
- How do old and new clients coexist?
- Where should version differences live?
- How do you deprecate an old version safely?

---

## 1. What is API versioning?

API versioning is a compatibility strategy that allows multiple API contracts to coexist.

Example:

```text
/api/v1/users
/api/v2/users
```

---

## 2. Breaking vs non-breaking changes

Breaking:

- rename field
- remove field
- change field type
- change semantics
- change auth model
- make optional input required

Usually non-breaking:

- add optional field
- add new endpoint
- add optional query param

But context matters.

---

## 3. URL path versioning

Example:

```text
/api/v1/users
/api/v2/users
```

Advantages:
- obvious
- easy to test
- easy to document
- easy to route
- client-friendly

Disadvantages:
- version visible in URI
- can encourage duplicated code if designed poorly

---

## 4. Header versioning

Example:

```http
Accept: application/vnd.example.v2+json
```

Advantages:
- URLs remain stable
- version treated as representation concern

Disadvantages:
- less discoverable
- harder to test manually
- more operational complexity

---

## 5. Custom header versioning

Example:

```http
X-API-Version: 2
```

Simple but proprietary.

Use only when your ecosystem supports it clearly.

---

## 6. Query parameter versioning

Example:

```text
/users?version=2
```

Easy to implement but usually less preferred for major API contract identity.

---

## 7. Media type versioning

Version can be embedded in Accept/Content-Type media types.

Useful in mature public APIs, but more complex.

---

## 8. Which strategy is best?

There is no universal winner.

For most REST APIs, path versioning is the simplest and most explicit.

Public APIs may choose more formal media-type/header strategies.

Internal APIs may rely on coordinated deployment with shorter migration windows.

---

## 9. Version the contract, not everything

Bad:

```text
v1Service
v2Service
v3Service
```

when business logic is identical.

Better:

```text
v1 Controller
      |
      v
Shared Service

v2 Controller
      |
      v
Shared Service
```

---

## 10. Version-specific serializers

Example:

Domain model:

```js
{
  firstName: "Vikash",
  lastName: "Kumar"
}
```

v1:

```json
{
  "name": "Vikash Kumar"
}
```

v2:

```json
{
  "firstName": "Vikash",
  "lastName": "Kumar"
}
```

Only response mapping changes.

---

## 11. Backward-compatible evolution first

Before adding v2, ask:

> Can I make this change without breaking old clients?

Example:

Instead of removing `name`, temporarily return:

```json
{
  "name": "Vikash Kumar",
  "firstName": "Vikash",
  "lastName": "Kumar"
}
```

Then deprecate `name` gradually.

---

## 12. Deprecation lifecycle

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
migration period
   |
   v
sunset
   |
   v
v1 removed
```

---

## 13. Communicating deprecation

Use:
- documentation
- changelog
- email/developer notices
- response headers where appropriate
- dashboards/usage reports

Never silently remove a public contract.

---

## 14. Sunset strategy

Track who still uses old versions.

Example:

```text
v1 traffic = 3%
```

Then:
- contact consumers
- set deadline
- monitor adoption
- remove only after migration window

---

## 15. Long-lived clients

Mobile apps may remain old for months.

Versioning is especially important when clients cannot update instantly.

---

## 16. Versioning authentication

Changing:

```text
JWT auth
```

to:

```text
OAuth flow
```

may affect:
- routes
- headers
- token semantics
- error behavior

This can be a true version boundary.

---

## 17. Versioning vs feature flags

Feature flag:

```text
same contract
different behavior rollout
```

API version:

```text
different client contract
```

Do not use versioning for every rollout experiment.

---

## 18. Versioning vs DB migrations

API version:
- client contract

DB migration:
- internal storage schema

They are independent concerns.

---

## 19. Versioning and documentation

Each active version should have:
- OpenAPI spec
- examples
- changelog
- deprecation notes

Do not let docs drift from implementation.

---

## 20. Testing versions

Test:
- v1 behavior
- v2 behavior
- shared service correctness
- serialization differences
- deprecation headers

Contract tests are valuable here.

---

## 21. Common mistakes

### Mistake 1
Versioning every small change.

### Mistake 2
Duplicating full backend logic.

### Mistake 3
No deprecation plan.

### Mistake 4
Removing old version too quickly.

### Mistake 5
Mixing versioning with feature flags.

---

## 22. Interview questions

### When should you create a new API version?

When you need a breaking contract change that cannot reasonably be introduced backward-compatibly.

### Which versioning strategy is most common?

URL path versioning.

### Should business logic be duplicated per version?

Usually no.

### How do you deprecate v1?

Announce it, provide migration guidance, monitor usage, set a sunset date, and remove only after a migration window.

---

## 23. Strong interview answer

> I version an API only when a breaking contract change cannot be introduced backward-compatibly. Path versioning such as /api/v1 is usually the simplest strategy. I keep business services shared and isolate version differences in controllers, serializers, or validators. When introducing v2, I deprecate v1 explicitly, monitor remaining usage, publish a migration path, and remove it only after a defined sunset window.

---

## Interview-Ready Summary

```text
Version when:
breaking contract

Strategies:
URL path
headers
media type
query param

Best practice:
shared business logic
versioned contract layer
deprecation plan
sunset monitoring
```

## Practice Task

Design v1 and v2 for an order API where v2 changes:

- address shape
- payment status names
- response pagination format

Decide which logic should stay shared.
