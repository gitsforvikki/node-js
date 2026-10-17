# Lesson 207 — Controller–Service–Repository Architecture

## 1. What problem does it solve?
When a controller validates business rules, charges a payment and writes MongoDB queries, every change touches the HTTP layer. Separating responsibilities makes code easier to test.

```text
Client -> Route -> Controller -> Service -> Repository -> Database
                    HTTP        Rules        Queries
```

## 2. Define each layer
- **Controller:** read validated request information, call service, choose response/status.
- **Service:** enforce business rules and coordinate operations.
- **Repository:** read/write data; hide Mongoose/SQL query details.
- **Model:** database schema/representation.
- **Route:** attach middleware and map URL to controller.

## 3. Real example: create product
**product.repository.js**
```js
import { Product } from "./product.model.js";
export const productRepository = {
  findBySku: (sku) => Product.findOne({ sku }),
  create: (data) => Product.create(data),
};
```

**product.service.js**
```js
export function makeProductService(repository) {
  return {
    async createProduct(input) {
      const existing = await repository.findBySku(input.sku);
      if (existing) {
        const error = new Error("SKU already exists");
        error.statusCode = 409;
        throw error;
      }
      return repository.create({
        sku: input.sku,
        name: input.name,
        price: input.price,
      });
    },
  };
}
```

**product.controller.js**
```js
export function makeProductController(service) {
  return async (req, res) => {
    // Assume Zod validation middleware stored trusted input here.
    const product = await service.createProduct(res.locals.validatedBody);
    res.status(201).json({ id: product.id, name: product.name });
  };
}
```

**product.routes.js**
```js
import { Router } from "express";
import { productRepository } from "./product.repository.js";
import { makeProductService } from "./product.service.js";
import { makeProductController } from "./product.controller.js";

const router = Router();
const service = makeProductService(productRepository);
router.post("/", validateProductBody, makeProductController(service));
export default router;
```
`validateProductBody` represents your implemented validation middleware; configure your global error handler to translate known 409 errors to safe responses. The example focuses on **layer boundaries**.

## 4. Follow the execution
POST body → validation → controller passes parsed fields → service checks duplicate SKU → repository runs query → creates document → controller returns 201.

## 5. Why service-level uniqueness checks aren't enough
Two concurrent requests may both see “SKU available.” Also create a **unique MongoDB index**, catch duplicate-key error `11000`, and map it to HTTP 409. Application-level checks improve UX but don't replace database constraints.

## 6. Unit test the service without MongoDB
```js
import assert from "node:assert/strict";
import test from "node:test";

test("rejects duplicate SKU", async () => {
  const fakeRepo = {
    findBySku: async () => ({ id: "existing" }),
    create: async () => { throw new Error("Should not create"); },
  };
  const service = makeProductService(fakeRepo);
  await assert.rejects(
    service.createProduct({ sku: "SKU1", name: "Mouse", price: 500 }),
    /SKU already exists/
  );
});
```

## 7. Common mistakes
Business logic in controllers; repositories calling `res.json`; services depending on `req` objects; unnecessary abstraction for a tiny CRUD route; forgetting transactional consistency across several writes.

## Practice
Implement order cancellation with: controller reads params → service checks ownership and status → repository conditionally updates order.

**Interview answer:** Controller handles HTTP, service handles business rules, repository handles persistence. The separation improves testing and change isolation, but database constraints and transactions still enforce correctness.
