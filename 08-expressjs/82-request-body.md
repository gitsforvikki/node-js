# Lesson 82 — Request Body in Express

## Why this lesson matters

Express makes request bodies look simple:

```js
req.body
```

But you should still understand:

- where req.body comes from
- which middleware populates it
- body-size limits
- Content-Type
- parsing vs validation
- raw bodies for webhooks
- multipart uploads

---

## 1. req.body does not appear automatically

You need body parsing middleware.

For JSON:

```js
app.use(
  express.json()
);
```

Then:

```js
app.post(
  "/users",
  (req, res) => {
    console.log(
      req.body
    );
  }
);
```

---

## 2. JSON middleware

Recommended:

```js
app.use(
  express.json({
    limit: "1mb",
  })
);
```

This:
- reads body stream
- checks compatible Content-Type
- parses JSON
- populates req.body
- enforces configured size limit

---

## 3. URL-encoded middleware

For form-style bodies:

```js
app.use(
  express.urlencoded({
    extended: true,
  })
);
```

Used with:

```text
application/x-www-form-urlencoded
```

---

## 4. Parsing vs validation

Request:

```json
{
  "email": 123,
  "age": "hello"
}
```

This may be valid JSON.

But invalid application data.

So:

```text
express.json()
   |
   v
parsing

validation middleware
   |
   v
schema correctness
```

Never confuse the two.

---

## 5. Validate body before service layer

Good flow:

```text
Request
   |
   v
express.json
   |
   v
validation middleware
   |
   v
controller
   |
   v
service
```

Do not let arbitrary body shapes reach database logic.

---

## 6. Body-size limits

Why:

```text
attacker
   |
   v
100 MB JSON
   |
   v
memory pressure
```

Use realistic limits per endpoint.

A profile JSON request may only need kilobytes.

A file upload should use streaming/multipart handling instead.

---

## 7. Malformed JSON

If JSON is malformed, parsing middleware raises an error.

Example:

```text
{"name":
```

Your centralized error handler should map parsing errors to a client error rather than generic 500.

---

## 8. Content-Type matters

If client sends:

```http
Content-Type: application/json
```

Express JSON middleware knows how to parse it.

If the Content-Type is incompatible, req.body behavior differs.

API clients should send accurate media types.

---

## 9. Raw body for webhooks

This is one of the most important production topics.

Payment providers often sign the exact raw payload bytes.

Example flow:

```text
Webhook Request
   |
   v
Raw Body
   |
   +--> verify HMAC/signature
   |
   v
Parse/process event
```

If normal JSON middleware modifies the representation before verification, signatures may fail.

Use route-specific raw body handling according to the provider's documentation.

---

## 10. Middleware order for webhook routes

Suppose:

```js
app.use(
  express.json()
);
```

runs before a route needing raw bytes.

That may consume/parse the body first.

For signed webhooks, configure middleware carefully so the raw route receives the required bytes.

This is a common payment-integration bug.

---

## 11. Multipart file uploads

`express.json()` does not parse:

```text
multipart/form-data
```

For file uploads, use a proper multipart parser such as a mature upload middleware/library.

Do not buffer huge files in normal JSON middleware.

---

## 12. Never trust client-calculated values

Example checkout body:

```json
{
  "productId": 123,
  "price": 1
}
```

Do not trust the price.

Server should load authoritative product price from DB.

General rule:

```text
client sends intent
server owns authority
```

---

## 13. Mass assignment risk

Bad:

```js
await User.update(
  req.body
);
```

An attacker may send:

```json
{
  "role": "admin"
}
```

Instead, explicitly pick allowed fields.

```js
const {
  name,
  email,
} = req.body;
```

or use validated DTO/schema output.

---

## 14. Prototype / object safety

Do not blindly merge arbitrary client objects into internal objects.

Use:
- schema validation
- allowed fields
- safe object handling

---

## 15. Controller example

```js
export async function createUser(
  req,
  res,
  next
) {
  try {
    const {
      name,
      email,
    } = req.body;

    const user =
      await userService.create({
        name,
        email,
      });

    res
      .status(201)
      .json(user);
  } catch (error) {
    next(error);
  }
}
```

In production, validation should happen before this controller.

---

## 16. Common mistakes

### Mistake 1
Forgetting express.json.

### Mistake 2
No body-size limit.

### Mistake 3
Treating parsed JSON as validated data.

### Mistake 4
Using global JSON middleware incorrectly for signed webhooks.

### Mistake 5
Trusting client price/role/owner fields.

### Mistake 6
Using req.body directly in DB update objects.

---

## 17. Interview questions

### What populates req.body?

Body parsing middleware such as `express.json()`.

### Does express.json validate business rules?

No.

### Why set a body limit?

To reduce memory abuse and denial-of-service risk.

### Why do some webhooks need raw body?

Because signature verification may depend on the exact original bytes.

### Can express.json parse multipart uploads?

No.

---

## 18. Strong interview answer

> In Express, req.body is populated by body-parsing middleware such as express.json. Parsing only converts the incoming bytes into a JavaScript value; it does not validate business rules, so I validate the body before the controller/service layer. I also configure body-size limits, use multipart-specific parsing for uploads, and preserve raw request bytes for signed webhooks when the provider requires signature verification over the original payload.

---

## Interview-Ready Summary

```text
Request Body
   |
   v
express.json()
   |
   v
req.body
   |
   v
validation
   |
   v
controller
   |
   v
service

Important:
body limits
raw webhook body
multipart != JSON
never trust client authority
```

## Section 8 Progress Map

```text
Why Express
   |
   v
Application Setup
   |
   v
Routing
   |
   +--> route params
   +--> query params
   +--> request body
   |
   v
Next:
Middleware architecture
Built-in middleware
Custom middleware
Error handling
Router architecture
API versioning
Large app structure
```

## Practice Task

Create:

```text
POST /users
PATCH /users/:id
```

Requirements:
- JSON body parser
- 1 MB limit
- schema validation
- allowed-field filtering
- centralized error handling
