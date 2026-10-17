# Lesson 165 — Unit Testing

## 1. What is a unit test?
A **unit test** checks one small piece of behavior (usually a function or service) with controlled inputs. It should be quick and not need MongoDB, HTTP, network or a real email service.

Example: an e-commerce application computes a discount. You want to check it independently from Express and Mongoose.

## 2. Real JavaScript example
**src/discount.js**
```js
export function calculateFinalPrice(price, discountPercent) {
  if (!Number.isFinite(price) || price < 0) throw new Error("Invalid price");
  if (!Number.isFinite(discountPercent) || discountPercent < 0 || discountPercent > 100) {
    throw new Error("Invalid discount");
  }
  return price * (1 - discountPercent / 100);
}
```

**test/discount.test.js**
```js
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { calculateFinalPrice } from "../src/discount.js";

describe("calculateFinalPrice", () => {
  it("applies a 20% discount", () => {
    assert.equal(calculateFinalPrice(1000, 20), 800);
  });

  it("returns original amount with zero discount", () => {
    assert.equal(calculateFinalPrice(500, 0), 500);
  });

  it("rejects a negative price", () => {
    assert.throws(() => calculateFinalPrice(-1, 10), /Invalid price/);
  });

  it("rejects a discount above 100%", () => {
    assert.throws(() => calculateFinalPrice(500, 150), /Invalid discount/);
  });
});
```
Run `node --test`. Ensure `"type":"module"` in `package.json`.

## 3. Step-by-step execution
Test runner loads test file → `describe` groups cases → each `it` calls real function → `assert` verifies result → runner reports pass/fail.

## 4. Arrange, Act, Assert
```js
it("calculates discount", () => {
  const price = 1000; // Arrange
  const result = calculateFinalPrice(price, 20); // Act
  assert.equal(result, 800); // Assert
});
```

## 5. Testing async functions
```js
export async function getGreeting(name) {
  if (!name) throw new Error("Name required");
  return `Hi ${name}`;
}
```
```js
it("rejects missing name", async () => {
  await assert.rejects(() => getGreeting(""), /Name required/);
});
```
Use `assert.rejects` for rejected Promises, `assert.throws` for synchronous errors.

## 6. Dependency injection for an async service
```js
export async function getUserName(userId, repository) {
  const user = await repository.findById(userId);
  if (!user) throw new Error("User not found");
  return user.name;
}
```
```js
it("returns user name without real DB", async () => {
  const fakeRepo = { findById: async () => ({ name: "Vikash" }) };
  assert.equal(await getUserName("u1", fakeRepo), "Vikash");
});
```
Passing a fake repository lets you test business logic without MongoDB. This is **dependency injection**: the function receives a dependency rather than creating it itself.

## 7. Test design: equivalence classes and boundaries
For an allowed age of 18–120, test 17, 18, 120 and 121—not every possible age. Test null, undefined, wrong types and empty data if your function accepts external input.

## 8. What unit tests cannot prove
A mocked repository doesn't prove Mongoose queries work or the real DB is configured correctly. Use integration tests (Lesson 166).

## 9. Common mistakes
Testing only implementation details; too many mocks; shared mutable state; confusing async and sync assertions; testing a copy of the logic instead of importing it.

## 10. Practice
Write `isEligibleForFreeDelivery(orderTotal)` and test positive, negative and boundary values. Then unit-test an async user service with a fake repository.

**Interview answer:** Unit tests quickly verify small isolated behaviors with predictable inputs. Mock or inject external dependencies and test expected results, errors and edge cases.
