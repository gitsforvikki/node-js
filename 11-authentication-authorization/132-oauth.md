# Lesson 132 — OAuth Fundamentals

## Why OAuth matters

OAuth is one of the most misunderstood authentication-related topics in backend interviews.

A common mistake is saying:

> OAuth is login.

OAuth is primarily an **authorization framework**.

It allows one application to access resources on behalf of a user **without asking for the user's password for that resource server**.

Example:

```text
"Continue with Google"
```

often uses OAuth 2.0 together with OpenID Connect.

---

# 1. What problem does OAuth solve?

Without OAuth:

```text
Third-party app asks for your Google password
```

This is dangerous.

With OAuth:

```text
User
  |
  v
Google Authorization Server
  |
  v
grants limited permission
  |
  v
Third-party app receives token
```

The third-party app never needs the user's Google password.

---

# 2. OAuth Roles

Four important roles:

### Resource Owner

Usually the user.

### Client

The application requesting access.

### Authorization Server

Authenticates user and issues tokens.

### Resource Server

API holding protected data.

---

## Mental Model

```text
User
  |
  v
Client Application
  |
  v
Authorization Server
  |
  v
Access Token
  |
  v
Resource Server
```

---

# 3. Example

Suppose your app wants to read Google Calendar.

```text
CareerLoop
  -> Client

User
  -> Resource Owner

Google Auth
  -> Authorization Server

Google Calendar API
  -> Resource Server
```

---

# 4. OAuth Is Delegated Authorization

The user delegates permission.

Example scope:

```text
calendar.readonly
```

The client gets only that permission.

This is better than giving full account credentials.

---

# 5. Access Token

OAuth authorization server issues an access token.

Client sends:

```http
Authorization: Bearer <access-token>
```

to the resource server.

---

# 6. Scope

Scope limits what token can do.

Example:

```text
profile
email
calendar.readonly
```

Least privilege applies.

Request only what the application needs.

---

# 7. OAuth vs Authentication

OAuth:

```text
authorization
```

OpenID Connect:

```text
authentication layer on top of OAuth 2.0
```

This distinction is extremely important.

---

# 8. "Login with Google"

A modern login flow usually uses:

```text
OAuth 2.0
+
OpenID Connect
```

OpenID Connect provides identity information via:

```text
ID Token
```

---

# 9. Access Token vs ID Token

Access token:

```text
used to access API
```

ID token:

```text
contains identity claims
used by client to understand authentication event
```

Do not send ID token to unrelated APIs as a generic access token.

---

# 10. Authorization Code Flow

Most important OAuth flow for web apps.

```text
Client
  |
  v
redirect user to authorization server
  |
  v
user authenticates/consents
  |
  v
authorization code returned
  |
  v
client exchanges code
  |
  v
access token
```

---

# 11. Why Use Authorization Code?

The browser receives only a short-lived code.

The actual access token exchange happens in a trusted client/backend context when appropriate.

This reduces token exposure.

---

# 12. PKCE

PKCE stands for:

```text
Proof Key for Code Exchange
```

It protects authorization code flow from code interception.

---

# 13. PKCE Flow

Client creates:

```text
code_verifier
```

Then derives:

```text
code_challenge
```

Authorization request sends challenge.

Token exchange sends verifier.

Authorization server checks:

```text
verifier -> challenge
```

---

# 14. Why PKCE Matters

Suppose attacker steals authorization code.

Without PKCE:

```text
attacker may exchange code
```

With PKCE:

```text
attacker lacks code_verifier
```

so intercepted code is far less useful.

---

# 15. State Parameter

OAuth authorization request includes:

```text
state
```

Purpose:

- bind request/response
- reduce CSRF/login-flow attacks

Generate cryptographically random state.

Verify exact match on callback.

---

# 16. Nonce

OpenID Connect commonly uses:

```text
nonce
```

to bind the ID token to the authentication request and reduce replay risks.

---

# 17. Redirect URI

Authorization server redirects to registered URI:

```text
https://app.example.com/auth/callback
```

Redirect URIs should be strictly registered and validated.

Open redirect behavior is dangerous.

---

# 18. Authorization Code Example

Step 1:

```text
GET /authorize?
client_id=...
&redirect_uri=...
&response_type=code
&scope=openid profile email
&state=...
&code_challenge=...
```

Step 2:

```text
callback?code=abc&state=xyz
```

Step 3:

Backend exchanges:

```text
code
+
code_verifier
+
client credentials where applicable
```

for tokens.

---

# 19. Client Secret

Confidential clients may have:

```text
client_secret
```

Never expose client secret in browser JavaScript.

A SPA cannot safely keep a secret.

---

# 20. Public vs Confidential Client

Public client:

- SPA
- mobile app

Cannot safely hold secret.

Confidential client:

- backend server
- server-side web app

Can protect client secret.

---

# 21. Client Credentials Flow

Used for machine-to-machine authorization.

```text
Service A
  |
  | client_id + client_secret
  v
Authorization Server
  |
  v
Access Token
  |
  v
Service B
```

No end user involved.

---

# 22. Resource Owner Password Flow

Historically existed:

```text
client collects username/password directly
```

Modern OAuth guidance generally avoids this pattern.

It defeats much of OAuth's delegated-security model.

---

# 23. Implicit Flow

Older browser flow returned access token directly through redirect.

Modern OAuth practice generally favors Authorization Code + PKCE instead.

Know this for interviews.

---

# 24. Refresh Tokens in OAuth

Authorization server may issue refresh token.

Used to obtain new access token without asking user to authorize again.

Refresh-token security concepts remain:

- rotation
- revocation
- secure storage
- replay detection

---

# 25. Token Introspection

Opaque access token may require resource server to ask authorization server:

```text
Is this token active?
What scopes does it have?
```

This is token introspection.

---

# 26. JWT Access Tokens

Some OAuth providers issue JWT access tokens.

Resource server can verify locally.

But OAuth access tokens are not required to be JWTs.

---

# 27. Scope-based Authorization

Example token:

```text
scope:
orders.read
orders.write
```

API middleware:

```js
requireScope(
  "orders.write"
)
```

---

# 28. OAuth Consent

Authorization server may show:

```text
CareerLoop wants:
- your email
- your basic profile
```

User approves requested scopes.

---

# 29. OpenID Connect Claims

ID token commonly includes:

```text
sub
iss
aud
exp
iat
nonce
```

Validate:

- signature
- issuer
- audience
- expiration
- nonce

---

# 30. Social Login Backend Flow

```text
User clicks Google login
       |
       v
Redirect to Google
       |
       v
Google authenticates user
       |
       v
Authorization code
       |
       v
Backend exchanges code
       |
       v
Validate ID token
       |
       v
Find/create local user
       |
       v
Create local session/token
```

Important:

Your app typically creates its **own** local session after provider authentication.

---

# 31. Account Linking

Suppose:

```text
local account email:
vikash@example.com

Google login email:
vikash@example.com
```

Do not automatically merge accounts solely because emails match unless provider verification and your account-linking policy make it safe.

Account takeover risks exist.

---

# 32. Provider Subject

Use stable provider identity:

```text
provider = google
providerUserId = sub
```

not only email.

Email can change.

---

# 33. OAuth Security Mistakes

### Mistake 1
No state validation.

### Mistake 2
No PKCE for public clients.

### Mistake 3
Client secret exposed in frontend.

### Mistake 4
Trusting ID token without verification.

### Mistake 5
Loose redirect URI validation.

### Mistake 6
Requesting unnecessary scopes.

### Mistake 7
Confusing ID token and access token.

---

# 34. OAuth vs JWT

OAuth:

```text
authorization framework
```

JWT:

```text
token format
```

OAuth can use JWT tokens, but the concepts are not equivalent.

---

# 35. OAuth vs OpenID Connect

OAuth:

```text
Can this client access resource?
```

OIDC:

```text
Who authenticated?
```

---

# 36. Interview Questions

## What is OAuth?

An authorization framework that allows a client to obtain limited access to resources on behalf of a user without receiving the user's resource-server password.

## What is PKCE?

Proof Key for Code Exchange, used to bind an authorization code to the client that initiated the flow.

## What does state protect against?

Request/callback mix-up and CSRF-style authorization attacks.

## OAuth vs OIDC?

OAuth is authorization; OpenID Connect adds authentication/identity.

## Access token vs ID token?

Access token is for APIs; ID token describes authentication/identity to the client.

---

# 37. Strong Interview Answer

> OAuth 2.0 is a delegated authorization framework. The client redirects the resource owner to an authorization server, which authenticates the user and issues a short-lived authorization code. The client exchanges that code for an access token, typically using PKCE for public clients. OAuth itself is not an authentication protocol; login experiences such as Sign in with Google usually use OpenID Connect on top of OAuth. I validate state, PKCE, redirect URIs, issuer, audience, token signatures, and request only the scopes the application actually needs.

---

# Interview-Ready Summary

```text
OAuth Roles:
Resource Owner
Client
Authorization Server
Resource Server

Modern web flow:
Authorization Code
+
PKCE
+
state

OIDC:
authentication layer
ID token

OAuth:
delegated authorization
access token
```

## Practical Task

Design "Continue with Google":

- Authorization Code + PKCE
- state validation
- callback handling
- ID token verification
- account creation/linking
- local session creation
