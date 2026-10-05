# Lesson 72 — Query Parameters

## Why query parameters matter

Query parameters are one of the most common ways to send optional request data.

They are heavily used for:

- pagination
- filtering
- searching
- sorting
- feature flags
- date ranges
- optional request modifiers

Example:

```text
GET /users?page=2&limit=20&role=admin
```

---

## 1. What are query parameters?

They are key-value pairs that appear after `?` in a URL.

Example:

```text
/products?category=mobile&page=2
```

Here:

```text
category = mobile
page = 2
```

Multiple parameters are separated by `&`.

---

## 2. Query string anatomy

```text
/products?category=mobile&page=2&sort=price
         |-------------------------------|
                  query string
```

The route itself is:

```text
/products
```

The query parameters modify how the resource is retrieved.

---

## 3. Parsing query parameters in raw Node.js

Use the standard `URL` API.

```js
const url =
  new URL(
    req.url,
    `http://${req.headers.host}`
  );

console.log(
  url.searchParams
);
```

Get one value:

```js
const page =
  url.searchParams.get(
    "page"
  );
```

---

## 4. Values are strings

Example:

```text
/users?page=2
```

Then:

```js
const page =
  url.searchParams.get(
    "page"
  );

console.log(typeof page);
```

Output:

```text
string
```

Convert explicitly:

```js
const pageNumber =
  Number(page ?? 1);
```

---

## 5. Missing parameter

```js
const sort =
  url.searchParams.get(
    "sort"
  );
```

If missing:

```text
null
```

Use defaults:

```js
const sort =
  url.searchParams.get(
    "sort"
  ) ?? "createdAt";
```

---

## 6. Repeated parameters

Example:

```text
/products?tag=node&tag=backend
```

Use:

```js
const tags =
  url.searchParams.getAll(
    "tag"
  );
```

Result:

```js
[
  "node",
  "backend"
]
```

---

## 7. has()

```js
if (
  url.searchParams.has(
    "active"
  )
) {
  // ...
}
```

Useful for optional flags.

---

## 8. Iteration

```js
for (
  const [key, value]
  of url.searchParams
) {
  console.log(
    key,
    value
  );
}
```

---

## 9. Pagination example

Request:

```text
GET /users?page=3&limit=20
```

Code:

```js
const page =
  Math.max(
    1,
    Number(
      url.searchParams.get(
        "page"
      ) ?? 1
    )
  );

const limit =
  Math.min(
    100,
    Math.max(
      1,
      Number(
        url.searchParams.get(
          "limit"
        ) ?? 20
      )
    )
  );
```

Why cap `limit`?

Without a limit:

```text
/users?limit=1000000
```

could become expensive.

---

## 10. Filtering example

```text
GET /products?category=laptop&minPrice=50000&maxPrice=100000
```

Conceptually:

```js
const filters = {
  category:
    url.searchParams.get(
      "category"
    ),

  minPrice:
    Number(
      url.searchParams.get(
        "minPrice"
      )
    ),

  maxPrice:
    Number(
      url.searchParams.get(
        "maxPrice"
      )
    ),
};
```

Always validate.

---

## 11. Sorting

Example:

```text
GET /products?sort=price&order=asc
```

Do not blindly trust arbitrary database field names.

Bad:

```js
ORDER BY ${userInput}
```

Use an allowlist:

```js
const allowedSortFields =
  new Set([
    "price",
    "createdAt",
    "name",
  ]);
```

---

## 12. Searching

Example:

```text
GET /users?q=vikash
```

Query parameters are ideal for optional search criteria.

---

## 13. Route param vs query param

```text
/users/123
```

`123` identifies the resource.

```text
/users?page=2
```

`page=2` modifies the collection view.

Mental model:

```text
Route parameter
  -> which resource?

Query parameter
  -> how should it be filtered/viewed?
```

---

## 14. Encoding

Special characters are URL encoded.

Example:

```js
const params =
  new URLSearchParams();

params.set(
  "q",
  "node js & backend"
);
```

Produces safe encoding.

Do not manually concatenate query strings when standard APIs can do it.

---

## 15. Query params are not secure

Never put sensitive data such as:

- passwords
- access tokens
- secret keys

inside query parameters.

Why?

URLs may appear in:
- browser history
- access logs
- analytics
- proxy logs
- referrer data

---

## 16. Common mistakes

### Mistake 1
Assuming numbers are already numeric.

### Mistake 2
No pagination bounds.

### Mistake 3
Trusting arbitrary sort/filter fields.

### Mistake 4
Putting secrets in URLs.

### Mistake 5
Confusing route params with query params.

---

## 17. Interview questions

### What are query parameters?

Optional key-value values in the URL used to modify retrieval behavior such as filtering, pagination, sorting, or search.

### How do you parse them in raw Node.js?

Using the standard `URL` and `URLSearchParams` APIs.

### Are query parameter values typed?

No. They are strings and should be parsed and validated.

### Query param vs route param?

Route params usually identify resources; query params usually modify the result set or representation.

---

## 18. Strong interview answer

> Query parameters are optional URL values typically used for filtering, sorting, search, and pagination. In raw Node.js I parse them using the URL API and validate every value because they arrive as strings. I also cap parameters such as page size and allowlist sort fields to protect the API and database from abusive or invalid input.

---

## Interview-Ready Summary

```text
Query Params
   |
   +--> pagination
   +--> filtering
   +--> sorting
   +--> searching

Node API:
URL
URLSearchParams

Important:
values are strings
validate
cap limits
never put secrets in URL
```

## Practice Task

Implement:

```text
GET /products?page=2&limit=10&category=laptop&sort=price&order=desc
```

Validate all query parameters and reject invalid sort fields.
