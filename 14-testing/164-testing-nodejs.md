# Lesson 164 — Testing Node.js Applications

## 1. Why do developers write tests?
A **test** runs code with a known input and checks whether its actual result matches the expected result. Imagine changing the coupon logic in your Express shop. Without tests, a discount bug might reach production. Tests catch regressions before users do.

**Mental model:** Arrange (prepare), Act (run), Assert (check). A test that fails for a real bug is useful; a test that only checks a function was called may miss the actual behavior.

## 2. Testing levels
| Level | What it tests | Example |
| --- | --- | --- |
| Unit | One small function in isolation | Discount calculator |
| Integration | Components working together | Express route + Mongoose + test DB |
| API | HTTP status/body/headers | POST /products returns 201 |
| End-to-end (E2E) | Full user workflow | Register → login → checkout |

**Test pyramid:** many fast unit tests, fewer integration tests, a smaller number of expensive complete workflows. It is a guide, not a fixed numerical rule.

## 3. Your first test using Node's built-in test runner
Node.js has `node:test`; you can start without Jest.

**src/pricing.js**
```js
export function total(price, quantity) {
  if (price < 0 || !Number.isInteger(quantity) || quantity < 1) {
    throw new Error("Invalid order");
  }
  return price * quantity;
}
```

**test/pricing.test.js**
```js
import test from "node:test";
import assert from "node:assert/strict";
import { total } from "../src/pricing.js";

test("calculates total price", () => {
  assert.equal(total(100, 3), 300);
});

test("rejects invalid quantity", () => {
  assert.throws(() => total(100, 0), /Invalid order/);
});
```

Run `node --test`. For these ES module imports, use `"type": "module"` in `package.json`, or use `.mjs` files.

## 4. What does a test runner do?
It discovers tests, executes them, reports **pass/fail**, and provides diagnostics. An **assertion** compares actual behavior with expected behavior. Test frameworks can also provide spies, mocking, coverage, setup hooks and watch mode.

## 5. The AAA pattern explained
```js
test("discount is applied", () => {
  // Arrange
  const originalPrice = 1000;
  const percentage = 10;
  // Act
  const discounted = originalPrice - originalPrice * percentage / 100;
  // Assert
  assert.equal(discounted, 900);
});
```
**Real tests should call your application function**, not reimplement the same calculation as the code under test. This AAA snippet demonstrates the structure; the preceding pricing example tests a real function.

## 6. Good test cases
For `POST /users`, cover the normal path and important failures: valid data (201), invalid email (400), duplicate email (409), unauthenticated protected action (401), and unexpected server error (safe 500).

Also test **boundaries**: zero, maximum length, empty input, missing fields, duplicate requests, and concurrent requests where correctness matters.

## 7. Deterministic, isolated tests
- A test must not depend on which test ran before it.
- Avoid calling live payment/email services from ordinary automated tests.
- Use fake time or a controllable clock instead of real sleeps.
- Seed and clean test data.
- Keep secrets out of test output.
- Test behavior, not private implementation details.

## 8. Where tests run
Local developer machine → pull-request CI → optional predeployment checks → production monitoring (not a replacement for tests). Add a package script: `"test": "node --test"`.

## 9. Common mistakes
Testing only successful requests; testing against a production database; flaky timing assumptions; too much mocking; asserting only HTTP 200 without checking body; expecting 100% coverage to mean zero bugs.

## 10. Practice
Write `calculateShipping(total)`: free at or above ₹999, otherwise ₹80. Test ₹998, ₹999, ₹1000, and invalid totals.

**Interview answer:** Testing verifies application behavior automatically and helps prevent regressions. Unit tests check isolated logic; integration tests check combined components; API tests verify HTTP contracts; E2E tests verify real workflows.
