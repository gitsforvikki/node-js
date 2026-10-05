# Lesson 80 — Route Parameters in Express

## What are route parameters?

Route parameters are dynamic values embedded in the path.

Example:

```text
GET /users/123
```

Route definition:

```js
app.get(
  "/users/:id",
  handler
);
```

Access:

```js
req.params.id
```

---

## 1. Basic example

```js
app.get(
  "/users/:id",
  (req, res) => {
    res.json({
      id:
        req.params.id,
    });
  }
);
```

Request:

```text
GET /users/42
```

Result:

```js
{
  id: "42"
}
```

---

## 2. Params are strings

Important:

```js
typeof req.params.id
```

is:

```text
string
```

Convert when necessary.

```js
const id =
  Number(
    req.params.id
  );
```

---

## 3. Multiple params

```js
app.get(
  "/users/:userId/orders/:orderId",
  handler
);
```

Then:

```js
req.params.userId
req.params.orderId
```

---

## 4. Validate params

Never trust route params.

Example numeric validation:

```js
const id =
  Number(
    req.params.id
  );

if (
  !Number.isInteger(id) ||
  id <= 0
) {
  return res
    .status(400)
    .json({
      message:
        "Invalid id",
    });
}
```

---

## 5. UUID/ObjectId validation

If your DB uses:
- UUID
- MongoDB ObjectId
- custom IDs

validate before querying.

Benefits:
- clearer client errors
- fewer invalid DB operations
- reduced exception noise

---

## 6. Ownership and authorization

This is a highly important security concept.

Request:

```text
GET /users/123/orders/456
```

Do not just query:

```text
order id = 456
```

You may need:

```text
order.id = 456
AND
order.userId = 123
```

and also verify the authenticated user is allowed to access user 123.

This prevents broken object-level authorization.

---

## 7. req.params vs req.query

Route param:

```text
/users/123
```

```js
req.params.id
```

Query param:

```text
/users?page=2
```

```js
req.query.page
```

Mental model:

```text
params
  -> identity

query
  -> filtering/modification
```

---

## 8. Controller example

```js
export async function getUser(
  req,
  res,
  next
) {
  try {
    const id =
      req.params.id;

    const user =
      await userService
        .getById(id);

    if (!user) {
      return res
        .status(404)
        .json({
          message:
            "User not found",
        });
    }

    res.json(user);
  } catch (error) {
    next(error);
  }
}
```

---

## 9. Route params are not business validation

Even if ID format is valid:

```text
/users/123
```

you still need to check:
- resource exists
- user is authorized
- state allows operation

Format validation is only the first layer.

---

## 10. Nested route design

Good when relation matters:

```text
/users/:userId/orders
```

But avoid excessive nesting.

Often:

```text
/orders/:id
```

is simpler after the resource has a unique identity.

---

## 11. Param middleware concept

Express supports reusable param handling patterns.

Example concept:

```js
router.param(
  "id",
  async (
    req,
    res,
    next,
    id
  ) => {
    // validate/load
    next();
  }
);
```

Useful in some architectures, but do not overuse implicit behavior.

---

## 12. Common mistakes

### Mistake 1
Using params without validation.

### Mistake 2
Assuming numeric type.

### Mistake 3
Ignoring ownership checks.

### Mistake 4
Duplicating resource ID in route and body unnecessarily.

### Mistake 5
Over-nesting URLs.

---

## 13. Interview questions

### How do you access route params?

Using `req.params`.

### What type are they?

Strings.

### Why validate them?

To reject malformed identifiers before DB access and keep API behavior predictable.

### What security issue is associated with IDs?

Broken object-level authorization if ownership/access is not enforced.

---

## 14. Strong interview answer

> Express exposes dynamic path values through req.params. I validate them before using them, because route params arrive as strings and may contain invalid identifiers. For user-owned resources, I also enforce authorization or ownership constraints in the query rather than treating possession of an ID as permission.

---

## Interview-Ready Summary

```text
/users/:id
       |
       v
req.params.id

Always:
validate
convert if needed
check existence
authorize ownership
```

## Practice Task

Implement:

```text
GET /users/:userId/orders/:orderId
```

Validate both IDs and prevent access to an order that does not belong to the user.
