# Lesson 175 — End-to-End (E2E) Testing Concepts

## 1. What is E2E testing?

An **end-to-end test** checks a **complete real user journey**, from the application entry point through the services it depends on.

For a React/Next.js + Express shop:

```text
Browser -> React UI -> Express API -> MongoDB
  -> authenticated session -> order created -> UI confirmation
```

A unit test checks one function. An API test checks an HTTP contract. An E2E test checks the **whole workflow**.

## 2. Real use cases

- Signup → verification → login.
- Add product to cart → checkout → confirmation.
- User A attempts User B's private order (must fail).
- Upload profile image → view updated profile.
- Logout → protected page is inaccessible.

You generally need **fewer E2E tests** than unit tests because they are slower and more fragile.

## 3. E2E vs integration vs API

| Level | Main focus | Often uses |
| --- | --- | --- |
| Unit | Small isolated rule | Fake repository |
| Integration | Joined backend components | Test DB |
| API | HTTP request/response contract | Supertest |
| E2E | Full user journey | Browser + app + test backend |

An API test can also be integration-level. Categories overlap; choose based on the purpose of the test.

## 4. Playwright browser test example

Install Playwright for a project with a running web application:

```bash
npm install -D @playwright/test
npx playwright install
```

**playwright.config.js**

```js
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  use: { baseURL: "http://127.0.0.1:3000" },
  webServer: {
    command: "npm run dev",
    url: "http://127.0.0.1:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

This assumes your web app has `npm run dev` and renders an accessible login screen at `/login`.

**e2e/login.spec.js**

```js
import { test, expect } from "@playwright/test";

test("user can log in", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("tester@example.com");
  await page.getByLabel("Password").fill("example-test-password");
  await page.getByRole("button", { name: "Log in" }).click();

  await expect(page.getByRole("heading", { name: "Dashboard" }))
    .toBeVisible();
});
```

**Important:** The test requires a *test-only user account*, a working isolated backend, and predictable test data. Do not use real production credentials. Use stable accessible labels instead of fragile CSS selectors.

## 5. What happens during this test?

Playwright opens a browser → navigates to login → fills form → clicks → browser calls your backend → session is created → UI changes → assertion verifies the dashboard appears.

If the backend or database is down, the test can fail even though the React component alone works.

## 6. A full-stack test environment

```text
Playwright browser
   |
   +--> Next.js frontend (test port)
   |
   +--> Express backend (test port)
             |
             +--> isolated test MongoDB
             +--> fake payment/email providers
```

Use real internal components where valuable and **safe sandbox/fake versions of external payment/email systems**. External providers' test modes may be suitable for selected integration scenarios.

## 7. Make E2E tests reliable

- Seed/reset test data predictably.
- Isolate users/accounts per test worker.
- Avoid arbitrary `waitForTimeout(5000)`; use Playwright's built-in auto-wait assertions.
- Keep network dependencies stable and controllable.
- Use test IDs/accessible labels for selectors.
- Collect screenshots, traces and logs when tests fail.
- Don't run destructive tests against production.

## 8. Authenticated user setup

Repeated login through the UI slows every test. Playwright supports setup projects and stored authentication state for test-only users.

Protect these state files: they can contain sensitive cookies/tokens. Keep them out of Git and invalidate test sessions after use.

## 9. What to test in the checkout flow

1. Add product to cart.
2. See the correct total.
3. Sign in if required.
4. Submit checkout with a payment sandbox.
5. Verify order confirmation in the UI.
6. Verify an order exists in isolated test DB.
7. Try double-submit and verify idempotency protections where applicable.

## 10. Common mistakes

Running E2E against production, unstable selectors, dependence on test order, real payment/email calls, sleeping instead of waiting for conditions, and testing every minor edge case only with slow browser tests.

## 11. Practice

Build a demo login page and API. Add one Playwright happy-path login test, one invalid-password test, and one logout/unauthorized-access test.

**Interview answer:** E2E testing checks a complete workflow as users experience it, often through a real browser and isolated backend services. It catches integration issues across UI, API and database, but should be limited to critical flows and designed for reliability.
