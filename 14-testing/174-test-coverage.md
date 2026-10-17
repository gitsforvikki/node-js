# Lesson 174 — Test Coverage

## 1. What is test coverage?

**Code coverage** measures how much code executes while tests run. It helps you notice logic that **no test reaches**.

It does **not** measure correctness. A line can execute while a test asserts the wrong thing.

## 2. Four common coverage metrics

| Metric | Question |
| --- | --- |
| Statement | Did this statement execute? |
| Line | Did this source line execute? |
| Function | Was this function called? |
| Branch | Did both sides of an `if`, ternary, etc. execute? |

## 3. Example: branch coverage

```js
export function shippingFee(total) {
  if (total >= 999) return 0;
  return 80;
}
```

If the only test calls `shippingFee(1000)`, the free-shipping branch executes, but the other branch doesn't. Add a test for `shippingFee(500)` and boundaries such as `998`, `999`.

```js
import { test, expect } from "vitest";
import { shippingFee } from "../src/shipping.js";

test("free shipping at threshold", () => {
  expect(shippingFee(999)).toBe(0);
});

test("paid shipping below threshold", () => {
  expect(shippingFee(998)).toBe(80);
});
```

## 4. Vitest coverage setup

```bash
npm install -D vitest @vitest/coverage-v8
npx vitest run --coverage
```

You can configure coverage thresholds in `vitest.config.js` according to your project's risk. Use a compatible version of the coverage package.

## 5. Built-in Node test coverage

```bash
node --test --experimental-test-coverage
```

Support/flags differ by Node version. Check the documentation for your installed Node version and inspect the generated coverage summary.

## 6. Reading a report

Imagine:

```text
File                Statements  Branches  Functions
shipping.js         100%        50%       100%
```

All statements and functions may execute while some branch outcomes remain untested. Inspect the uncovered branch rather than celebrating the 100% statement number.

## 7. Why 100% coverage is not enough

```js
test("calls shippingFee", () => {
  shippingFee(1000); // executes code but asserts NOTHING
});
```

This might increase coverage while finding no bug. Good tests assert behavior. Also check invalid inputs, security, concurrency and database interactions.

## 8. What to prioritize

- Critical payment, login, authorization and data-changing rules.
- Branches with expensive failure consequences.
- Edge cases and error paths.
- Known production bugs (write a regression test for each).
- API contracts and integration boundaries.

Don't write meaningless assertions solely to hit a percentage.

## 9. CI example

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx vitest run --coverage
```

This is a teaching workflow. Pin third-party actions appropriately for production, set minimal permissions, and ensure your project's package versions and Node compatibility are correct.

## 10. Mutation testing

Mutation testing deliberately introduces small code changes (e.g., `>=` becomes `>`) and checks whether tests fail. It can reveal weak assertions even when line coverage is high. Tools such as Stryker can help.

## 11. Common mistakes and practice

Mistakes: chasing 100% without meaningful assertions, excluding difficult business logic, treating coverage as a security guarantee, and ignoring integration behavior.

Practice: write a shipping-fee test for both branches, generate a coverage report, then deliberately introduce an off-by-one bug and confirm tests catch it.

**Interview answer:** Coverage reports what code and branches tests execute, not whether assertions are good. Use it to find untested paths, but prioritize meaningful behaviors, boundaries and failures over a raw percentage.
