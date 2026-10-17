# Lesson 143 — Security Headers and Helmet

## 1. What are security headers?

**HTTP response headers** are instructions and metadata the server sends to the browser. **Security headers** tell the browser to apply additional restrictions, helping reduce risks such as XSS, clickjacking, MIME sniffing, and insecure transport.

They are **defense in depth**: useful extra protection, but not a substitute for safe code, authorization, or validation.

## 2. What is Helmet?

**Helmet** is an Express middleware package that sets several security-related response headers for you.

Install:

```bash
npm install helmet
```

Use it near the beginning of your Express middleware stack:

```js
import express from "express";
import helmet from "helmet";

const app = express();
app.use(helmet());
app.use(express.json());

app.get("/health", (req, res) => res.json({ ok: true }));
```

Helmet has defaults, but **review them for your app** instead of assuming every setting fits all deployments.

## 3. Important headers explained

| Header | What problem does it help with? |
| --- | --- |
| `Content-Security-Policy` (CSP) | Restricts which scripts, styles, images, and other resources the page may load |
| `Strict-Transport-Security` (HSTS) | Tells browsers to prefer HTTPS for future visits |
| `X-Content-Type-Options: nosniff` | Prevents certain MIME-type guessing |
| `Referrer-Policy` | Controls how much URL/referrer information is sent to other sites |
| `X-Frame-Options` | Helps prevent unwanted framing/clickjacking (CSP `frame-ancestors` is more flexible) |
| `Cross-Origin-Resource-Policy` | Controls which other origins/sites can load certain resources |
| `Cross-Origin-Opener-Policy` | Helps isolate top-level browsing contexts |

## 4. Content Security Policy (CSP) in simple language

Imagine your webpage is allowed to run scripts only from itself. A CSP tells the browser that rule.

```text
Browser loads your page
  -> sees Content-Security-Policy
  -> checks script/style/image source against policy
  -> blocks disallowed resources
```

Example for a **simple Express-served webpage**:

```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      objectSrc: ["'none'"],
      baseUri: ["'self'"],
      frameAncestors: ["'none'"],
    },
  },
}));
```

This example may break apps that need external scripts, inline styles, development tooling, or third-party resources. **Do not copy it blindly into a Next.js app**. Configure CSP to match your deployment and consider nonces/hashes for scripts.

## 5. HSTS and HTTPS

`Strict-Transport-Security` tells browsers to use HTTPS for future requests to a domain for a specified period.

Example conceptual header:

```text
Strict-Transport-Security: max-age=31536000
```

**Important:** HSTS only works when delivered over HTTPS. Don't enable broad `includeSubDomains` or `preload` until **every affected hostname** supports HTTPS reliably. Helmet's default HSTS behavior can be inconvenient in local development.

For local HTTP development only, you may choose:

```js
app.use(helmet({
  strictTransportSecurity: false,
}));
```

Keep HSTS enabled in a correctly configured HTTPS production deployment.

## 6. MIME sniffing and clickjacking

**MIME sniffing:** Some browsers might interpret content differently from the declared `Content-Type`. `X-Content-Type-Options: nosniff` helps prevent dangerous interpretation in certain contexts.

**Clickjacking:** A malicious page frames your site and tricks a user into clicking something unintended. `X-Frame-Options` or CSP `frame-ancestors` helps control who can frame your pages.

For apps that intentionally support embedding, configure these restrictions explicitly instead of disabling every security header.

## 7. Helmet vs CORS vs CSRF

| Mechanism | Main purpose |
| --- | --- |
| **Helmet** | Adds browser security response headers |
| **CORS** | Controls browser cross-origin response access |
| **CSRF protection** | Prevents unwanted state-changing requests made using ambient credentials |
| **Authentication** | Identifies the requester |
| **Authorization** | Checks allowed actions/resources |

**Helmet does not configure CORS or implement CSRF tokens automatically.**

## 8. A practical Express 5 configuration

```js
import express from "express";
import helmet from "helmet";
import cors from "cors";

const app = express();

app.use(helmet());
app.use(cors({
  origin: "https://app.example.com",
  credentials: true,
}));
app.use(express.json({ limit: "16kb" }));

app.get("/api/health", (req, res) => {
  res.json({ status: "ok" });
});
```

**Note:** Replace the example origin with your actual frontend origin. Cookie-based auth also needs secure cookies, authentication, authorization, and CSRF protection where relevant.

## 9. How to verify response headers

Run the app, then:

```bash
curl -I http://localhost:3000/api/health
```

Or open browser DevTools → Network → select request → Response Headers.

Look for headers such as `Content-Security-Policy`, `X-Content-Type-Options`, `Referrer-Policy`, and others enabled by your configuration.

**Caution:** `curl -I` sends a HEAD request. Some routes/middleware behave differently for HEAD; use `curl -i` for a normal GET if needed.

## 10. Common mistakes

1. Thinking Helmet makes an API fully secure.
2. Turning off CSP because one external script fails, without understanding the needed directive.
3. Copying strict CSP into development without testing.
4. Enabling HSTS preload before checking all subdomains.
5. Confusing Helmet headers with CORS headers.
6. Assuming security headers can fix insecure database queries.

## 11. Practice

1. Add Helmet to an Express app and inspect response headers.
2. Explain the purpose of CSP, HSTS, and `nosniff` in one sentence each.
3. Configure a CSP for a simple static page and check blocked resources.
4. Compare Helmet, CORS, and CSRF protection.

## Interview-ready answer

Helmet is Express middleware that adds browser security headers such as CSP, HSTS, `nosniff`, and referrer policy. These headers reduce certain browser-side risks but do not replace secure application logic. Configure and test them for the real deployment, especially CSP and HSTS.
