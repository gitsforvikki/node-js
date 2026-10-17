# Lesson 168 — Jest / Vitest Concepts

## 1. Why do we use testing frameworks?
A framework runs tests and provides assertions, setup hooks, spies, mocks, coverage and watch mode. Popular JavaScript choices include **Jest**, **Vitest**, and Node's built-in `node:test`.

**Choose one runner for a project** unless you have a deliberate migration plan. For an ESM-heavy Node + React workspace, Vitest can be convenient; Jest is also widely used and mature.

## 2. Core words
- `describe`: group related tests.
- `it` / `test`: define a scenario.
- `expect`: assert expected result.
- `beforeEach` / `afterEach`: prepare and clean test state.
- Spy: observe calls to a function.
- Mock: substitute controlled behavior.

## 3. Same test in Jest and Vitest
**src/math.js**
```js
export const add = (a, b) => a + b;
```

**Vitest**
```js
import { describe, it, expect } from "vitest";
import { add } from "../src/math.js";

describe("add", () => {
  it("adds two numbers", () => {
    expect(add(2, 3)).toBe(5);
  });
});
```

**Jest**
```js
import { describe, it, expect } from "@jest/globals";
import { add } from "../src/math.js";

describe("add", () => {
  it("adds two numbers", () => {
    expect(add(2, 3)).toBe(5);
  });
});
```
Jest's ESM configuration may require additional setup depending on Node/Jest versions. The API shown is illustrative; follow the installed version's ESM guidance.

## 4. Installation and running (Vitest)
```bash
npm install -D vitest
npx vitest run
npx vitest
```
`vitest run` runs once; `vitest` usually starts interactive/watch behavior in a local terminal.

## 5. Matchers
```js
import { expect, test } from "vitest";

test("matchers", () => {
  expect(5).toBe(5); // primitive identity
  expect({ ok: true }).toEqual({ ok: true }); // deep comparison
  expect(["a", "b"]).toContain("a");
  expect(() => { throw new Error("bad"); }).toThrow("bad");
});
```

## 6. Async tests
```js
test("async result", async () => {
  await expect(Promise.resolve({ ok: true }))
    .resolves.toEqual({ ok: true });
  await expect(Promise.reject(new Error("failed")))
    .rejects.toThrow("failed");
});
```

**Always `await` async expectations**, or a test can finish before rejection assertions run.

## 7. Hooks, mocks and cleanup
```js
import { beforeEach, test, expect, vi } from "vitest";
let send;
beforeEach(() => { send = vi.fn(); });

test("calls sender", () => {
  send("hello");
  expect(send).toHaveBeenCalledWith("hello");
});
```
Jest equivalents usually use `jest.fn()` and `jest.spyOn()`. Restore real spies/mocks after tests to prevent leakage.

## 8. Timers and snapshots
Fake timers test retry/debounce without real waiting. Snapshot tests record expected serialized output and help detect unintended changes, but blindly approving changed snapshots hides regressions. Prefer direct behavior assertions for business logic.

## 9. Which should you pick?
Vitest: straightforward for modern ESM/Vite projects. Jest: large ecosystem and existing projects. Node built-in: minimal dependency for many backend tests. Verify versions and Node compatibility before configuring.

## 10. Common mistakes and practice
Confusing `toBe` with `toEqual`; forgetting to await; mocking every dependency; depending on shared test order; enabling snapshots without review.

Practice: run the same `add` test with Vitest and compare its syntax to Jest. Test a rejected Promise and an async mock.

**Interview answer:** Jest and Vitest supply runners, matchers, setup hooks, mocks and coverage. Their test syntax is similar; choose based on project ecosystem, module support and maintainability.
