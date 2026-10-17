# Lesson 170 — Mocking, Stubs, Spies, and Fakes

## 1. Why do we mock things?

Imagine your `placeOrder()` function:
1. Checks stock in MongoDB.
2. Calls a payment service.
3. Sends an email.

During a **unit test**, you usually don't want to charge a real card, call the internet, or send an email. You replace the external dependency with **controlled test behavior**.

> A mock is a tool, not the purpose of the test. The purpose is to check your application's behavior.

## 2. Test doubles explained simply

| Term | Meaning | Example |
| --- | --- | --- |
| Dummy | Filler argument, never used | A placeholder config |
| Stub | Returns predefined data | Fake `findById()` returns user |
| Spy | Records calls and arguments | Check whether email was called |
| Fake | Working but simplified implementation | In-memory user repository |
| Mock | Test double with expected interactions | Verify payment called with correct total |

People often casually call all of these "mocks."

## 3. Design code so it can be tested

**src/services/order.service.js**

```js
export async function placeOrder(
  { userId, productId, quantity },
  { products, payments, orders }
) {
  if (!Number.isInteger(quantity) || quantity <= 0) {
    throw new Error("Invalid quantity");
  }

  const product = await products.findById(productId);
  if (!product) throw new Error("Product not found");
  if (product.stock < quantity) throw new Error("Out of stock");

  const total = product.price * quantity;
  const payment = await payments.charge({ userId, amount: total });
  return orders.create({
    userId, productId, quantity, total, paymentId: payment.id,
  });
}
```

Dependencies are passed in, rather than imported as tightly coupled global objects. This is **dependency injection**.

## 4. Real unit test with Vitest

Install `npm install -D vitest`. Save as `test/order.test.js`:

```js
import { test, expect, vi } from "vitest";
import { placeOrder } from "../src/services/order.service.js";

test("charges the correct amount and saves the order", async () => {
  // Arrange
  const products = {
    findById: vi.fn().mockResolvedValue({ price: 500, stock: 10 }),
  };
  const payments = {
    charge: vi.fn().mockResolvedValue({ id: "pay_1" }),
  };
  const orders = {
    create: vi.fn().mockImplementation(async (data) => ({
      id: "ord_1", ...data,
    })),
  };

  // Act
  const order = await placeOrder(
    { userId: "u1", productId: "p1", quantity: 2 },
    { products, payments, orders }
  );

  // Assert
  expect(payments.charge).toHaveBeenCalledWith({
    userId: "u1", amount: 1000,
  });
  expect(orders.create).toHaveBeenCalledWith(
    expect.objectContaining({ total: 1000, quantity: 2 })
  );
  expect(order.id).toBe("ord_1");
});
```

Run `npx vitest run`. All external effects above are replaced.

## 5. Test the failure path too

```js
test("does not charge when stock is insufficient", async () => {
  const products = {
    findById: vi.fn().mockResolvedValue({ price: 500, stock: 1 }),
  };
  const payments = { charge: vi.fn() };
  const orders = { create: vi.fn() };

  await expect(
    placeOrder(
      { userId: "u1", productId: "p1", quantity: 2 },
      { products, payments, orders }
    )
  ).rejects.toThrow("Out of stock");

  expect(payments.charge).not.toHaveBeenCalled();
  expect(orders.create).not.toHaveBeenCalled();
});
```

This test verifies an important **business rule**, not just a function call.

## 6. vi.fn(), vi.spyOn(), and module mocks

- `vi.fn()`: create a controllable mock function.
- `vi.spyOn(object, "method")`: observe or replace an existing object's method.
- `vi.mock("module")`: substitute a module import. Module mocking can be sensitive to import timing; prefer dependency injection for beginner-friendly services.

```js
import { afterEach, test, expect, vi } from "vitest";

afterEach(() => vi.restoreAllMocks());

test("tracks logger calls", () => {
  const logger = { info: (message) => message };
  const spy = vi.spyOn(logger, "info");
  logger.info("Created order");
  expect(spy).toHaveBeenCalledWith("Created order");
});
```

## 7. Mocking time and randomness

Date/time-based code can become flaky. Inject a clock, or use `vi.useFakeTimers()` for code that schedules timers. Restore real timers after each test.

**Important:** For security-sensitive tokens, do not replace cryptographic randomness with predictable values in production. Controlled fake values are only for tests.

## 8. Mocking vs integration testing

A test with a fake `products.findById` **cannot prove** that the MongoDB query or indexes work. Add integration tests against a real isolated database to test those boundaries. Excessive mocking can make broken applications appear well tested.

## 9. Common mistakes

- Mocking the function you are trying to test.
- Writing tests that only verify a mock was called, not the observable result.
- Forgetting to reset spies and fake timers.
- Depending on live external APIs for routine unit tests.
- Believing mocks prove database transactions, HTTP integrations, or credentials work.

## 10. Practice

1. Change the order service to reject quantities greater than stock.
2. Test that payment is **never called** for invalid quantity.
3. Test what happens when `payments.charge` rejects.
4. Explain why at least one integration test should use a real database.

**Interview answer:** Mocking replaces external dependencies with controlled behavior to make unit tests deterministic. Use stubs for data, spies for interactions, and fakes for simplified implementations. Mock at the boundaries; still test real integrations separately.
