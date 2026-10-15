# Lesson 125 — Refresh Tokens

## Start here — Refresh Token from zero (React + Express)

Is section ko **sequence me padho**. Har next step pichhle step par based hai. Pehle problem samjhenge, phir token generation, database mapping, cookie storage, refresh request, rotation, page reload aur logout. Neeche existing advanced sections bhi preserved hain.

### Step 1 — Refresh Token ki zarurat kyu hai?

Imagine tumne React + Express shopping app banaya. User login karta hai, server **15-minute access token** deta hai. Access token protected APIs (`/api/profile`, `/api/orders`) call karne ka short-term pass hai.

**Problem:** 15 minutes ke baad access token expire ho gaya. Kya user ko baar-baar password enter karna hoga?

**Solution:** Server ek alag **refresh token** bhi issue karta hai. Client usse **sirf refresh endpoint** par bhejta hai, aur server valid refresh session ke basis par **naya access token** issue karta hai.

```text
LOGIN (email + password)
         |
         v
Server verifies password hash
         |
         +----> ACCESS TOKEN (example: 15 min) --> Protected API requests
         |
         +----> REFRESH TOKEN (example: 7 days) --> Only /auth/refresh
                                                         |
                                                         v
                                              New access token
```

**Simple meaning:** Access token = API use karne ka temporary pass. Refresh token = naya temporary pass lene ka protected credential.

### Step 2 — Dono tokens kaise generate hote hain?

**Access token** JWT ho sakta hai. **Refresh token** JWT bhi ho sakta hai, lekin hum yahan **opaque random token** use karenge. Opaque ka matlab: token ke andar readable user ID ya role nahi hai.

```js
import jwt from "jsonwebtoken";
import crypto from "node:crypto";

// Assume password has already been checked against saved hash.
const accessToken = jwt.sign(
  { sub: String(user.id) },
  process.env.ACCESS_TOKEN_SECRET,
  { algorithm: "HS256", expiresIn: "15m" }
);

const refreshToken = crypto.randomBytes(32).toString("base64url");
```

`randomBytes(32)` secure 256-bit random value banata hai. Har login/session ko alag random refresh token milna chahiye.

**Important doubt:** “Refresh token me `userId` nahi hai, to server ko kaise pata chalega ye kis user ka hai?” **Answer: Database mapping.** Agla step dekho.

### Step 3 — Refresh token ko user se kaise connect karte hain?

Login successful hote hi server refresh token ka **SHA-256 hash** nikal kar database me user ID ke saath save karta hai:

```js
import crypto from "node:crypto";

function hashToken(token) {
  return crypto.createHash("sha256").update(token).digest("hex");
}

const tokenHash = hashToken(refreshToken);

await db.refreshSessions.create({
  userId: user.id,
  tokenHash,
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  revokedAt: null,
});
```

`db.refreshSessions.create` example database-repository function hai, built-in Node.js method nahi.

**Database example:**

| sessionId | userId | tokenHash | expiresAt | revokedAt |
| --- | --- | --- | --- | --- |
| S1 | 101 | hash-of-R1 | 7 days later | null |
| S2 | 205 | hash-of-R2 | 7 days later | null |

```text
Raw Refresh Token R1  (browser)
          |
          v
     SHA-256(R1)
          |
          v
DB: tokenHash matches S1
          |
          v
DB record says userId = 101
          |
          v
Now server knows which user/session it belongs to
```

**Token me user information hona zaruri nahi.** Database mapping trusted user identity batati hai. Lekin valid token **jiske paas bhi hai woh use kar sakta hai**—isliye theft protection important hai.

**Why SHA-256 here?** Random refresh token high-entropy hai, isliye SHA-256 hash lookup practical hai. **Human passwords** ke liye SHA-256 alone mat use karo; Argon2id/bcrypt jaise password-hashing algorithms use karo.

### Step 4 — Refresh token kaha store karna chahiye?

Do jagah alag cheezein store hoti hain:

```text
BROWSER                           SERVER DATABASE
---------------------------       ----------------------------
Raw Refresh Token R1              hash(R1)
Secure + HttpOnly cookie           userId = 101
                                  expiresAt, revokedAt, etc.

Access token: React memory (in this example)
```

Express login response me cookie set karta hai:

```js
res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: true,      // production HTTPS
  sameSite: "lax",  // choose according to frontend/backend sites
  path: "/auth",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});

res.json({ accessToken });
```

- `HttpOnly`: React JavaScript cookie ko directly read nahi kar sakta.
- `Secure`: HTTPS par cookie send hoti hai (production).
- `SameSite`: cross-site cookie sending limit karta hai; **CSRF controls phir bhi consider karo**.
- `path: "/auth"`: matching `/auth` routes par cookie send hoti hai.
- `maxAge`: browser cookie ka lifetime. **DB expiry bhi independently check karna zaruri hai.**

Frontend/backend cross-site hone par `SameSite=None; Secure`, correct CORS credentials aur stronger CSRF/origin controls ki zarurat ho sakti hai. **Raw refresh token ko localStorage, URL, logs ya analytics me mat rakho.**

### Step 5 — Access token expire hone par exactly kya hota hai?

```text
React -> GET /api/profile (Bearer access token)
                         |
                         v
                Access token expired
                         |
                         v
                    401 response
                         |
                         v
React -> POST /auth/refresh (credentials: "include")
                         |
                         v
Browser sends HttpOnly refresh cookie automatically
                         |
                         v
Express hashes raw refresh token
                         |
                         v
DB finds matching session -> checks expiry + revoked + user status
                         |
                  +------+------+
                  |             |
                VALID         INVALID
                  |             |
                  v             v
         Rotate refresh       401 -> login
         Issue access token
                  |
                  v
React saves new access token in memory
                  |
                  v
Retry original protected request once
```

React request:

```js
async function requestNewAccessToken() {
  const response = await fetch("/auth/refresh", {
    method: "POST",
    credentials: "include",
  });

  if (!response.ok) throw new Error("Please log in again");

  const data = await response.json();
  return data.accessToken;
}
```

React cookie read nahi karta. Browser cookie attach karta hai, agar origin/site/cookie rules allow karte hain.

### Step 6 — Express refresh endpoint user kaise validate karta hai?

Pehle **simple verification logic** samjho. Agle step me production rotation add karenge.

```js
import jwt from "jsonwebtoken";

// Requires cookie-parser middleware: app.use(cookieParser())
export async function refreshController(req, res) {
  const rawToken = req.cookies?.refreshToken;
  if (!rawToken) return res.sendStatus(401);

  const session = await db.refreshSessions.findByTokenHash(
    hashToken(rawToken)
  );

  if (!session || session.revokedAt || session.expiresAt <= new Date()) {
    return res.sendStatus(401);
  }

  // userId comes from trusted database session, NOT from opaque token.
  const user = await db.users.findById(session.userId);
  if (!user || user.isDisabled) return res.sendStatus(401);

  const accessToken = jwt.sign(
    { sub: String(user.id) },
    process.env.ACCESS_TOKEN_SECRET,
    { algorithm: "HS256", expiresIn: "15m" }
  );

  return res.json({ accessToken });
}
```

`db.refreshSessions.findByTokenHash` and `db.users.findById` are example repository methods. This is a **learning-only validation example**; a production refresh endpoint should also rotate tokens atomically, handle reuse, and enforce CSRF/origin protection.

**Fake token?** `hash(fakeToken)` ka matching active DB record nahi milega -> **401**. **Stolen real token?** Matching session mil sakta hai; isliye rotation, secure storage aur theft detection important hain.

### Step 6A — Express me practical implementation: 3 connected steps

Ab pichhle concepts ko **ek hi example** me connect karte hain. Yeh woh **three practical steps** hain: **(1) login par token generate/save, (2) client refresh request, (3) server token verify karke access token reissue**.

**Assumptions:** `cookie-parser` installed hai, `app.use(cookieParser())` configured hai, login password verification already successful hai, aur `db` tumhara database repository layer hai. Neeche repository methods **illustrative** hain; apne database ke hisaab se implement karne honge.

#### Practical Step 1 — Login ke time Refresh Token generate, DB me map aur cookie me save

```js
import crypto from "node:crypto";
import jwt from "jsonwebtoken";

function createRefreshToken() {
  return crypto.randomBytes(32).toString("base64url");
}

function hashToken(token) {
  return crypto.createHash("sha256").update(token).digest("hex");
}

// After login password verification:
const accessToken = jwt.sign(
  { sub: String(user.id) },
  process.env.ACCESS_TOKEN_SECRET,
  { algorithm: "HS256", expiresIn: "15m" }
);

const refreshToken = createRefreshToken();

await db.refreshSessions.create({
  userId: user.id,
  tokenHash: hashToken(refreshToken),
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  revokedAt: null,
});

res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/auth",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});

return res.json({ accessToken });
```

**Samjho:** Browser ko **raw** refresh token cookie me milta hai; DB me **sirf hash + userId** save hota hai. Access token JSON response me aata hai, jise React memory me rakh sakti hai. Token generate karna aur user ke saath map karna **login time** hota hai.

#### Practical Step 2 — React access token renew karne ki request bhejti hai

Jab access token expire ho, protected API typically **401** deti hai. React:

```js
async function getNewAccessToken() {
  const response = await fetch("/auth/refresh", {
    method: "POST",
    credentials: "include",
  });

  if (!response.ok) throw new Error("Please login again");

  const { accessToken } = await response.json();
  return accessToken; // save in React memory
}
```

**Samjho:** React refresh token ko `localStorage` se nahi nikaalti. `credentials: "include"` ke saath browser matching **HttpOnly cookie automatically send** karta hai. Is request ko CSRF/origin checks se protect karo.

#### Practical Step 3 — Express refresh cookie validate karta hai aur access token reissue karta hai

```js
export async function refreshController(req, res) {
  const rawToken = req.cookies?.refreshToken;
  if (!rawToken) return res.sendStatus(401);

  const session = await db.refreshSessions.findByTokenHash(
    hashToken(rawToken)
  );
  if (!session || session.revokedAt || session.expiresAt <= new Date()) {
    return res.sendStatus(401);
  }

  // Opaque token me userId nahi: DB mapping se userId milta hai.
  const user = await db.users.findById(session.userId);
  if (!user || user.isDisabled) return res.sendStatus(401);

  const accessToken = jwt.sign(
    { sub: String(user.id) },
    process.env.ACCESS_TOKEN_SECRET,
    { algorithm: "HS256", expiresIn: "15m" }
  );

  return res.json({ accessToken });
}
```

**Samjho:** Server `hash(rawToken)` calculate karta hai → matching **DB session** find karta hai → us record se **userId** milta hai → user active hai ya nahi check karta hai → tabhi naya access token deta hai. **Fake token** ka matching DB session nahi milega, isliye 401.

```text
LOGIN (Step 1)
user 101 -> generate R1 -> DB saves hash(R1) + userId=101
                        -> browser receives R1 cookie

REFRESH (Step 2)
React -> POST /auth/refresh -> browser sends R1 cookie

VERIFY + REISSUE (Step 3)
Express -> hash(R1) -> DB record -> userId 101 -> check user
        -> new access token A2 -> React
```

**Important:** Step 3 deliberately **basic learning version** hai. Production me old R1 ko **atomically consume** karke R2 issue karo; next section me rotation samjhenge. Ek hi handler me Step 3 ke last `res.json` ko rotation ke baad call karna hoga.

### Step 7 — Refresh Token Rotation kya hai?

Har successful refresh ke baad **old refresh token invalidate** karo aur **new refresh token issue** karo.

```text
LOGIN          -> Access A1 + Refresh R1
First refresh  -> invalidate R1 -> Access A2 + Refresh R2
Second refresh -> invalidate R2 -> Access A3 + Refresh R3

Old R1/R2 must NOT work again.
```

**Simple implementation idea:**

```js
const nextRefreshToken = crypto.randomBytes(32).toString("base64url");
const nextHash = hashToken(nextRefreshToken);

// IMPORTANT: DB consume-old + create-new MUST be one atomic transaction.
// db.rotateRefreshSession is a placeholder for that transaction.
await db.rotateRefreshSession({
  oldSessionId: session.id,
  oldTokenHash: hashToken(rawToken),
  newTokenHash: nextHash,
  userId: user.id,
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
});

res.cookie("refreshToken", nextRefreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/auth",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});

return res.json({ accessToken });
```

This snippet shows **the rotation step** and replaces the last `res.json` in Step 6; it is **not** a standalone handler. Real DB transaction must ensure **only one request can consume R1**, check revocation/expiry inside the transaction, and save the replacement before sending cookie/response.

**Reuse detection:** Agar already-consumed R1 dobara aaye, possible theft/replay hai. System token family revoke kar sakta hai. Simultaneous browser requests bhi duplicate refresh kar sakte hain, so use atomic rotation + client refresh deduplication.

### Step 8 — Page reload par React memory clear ho gayi, ab kya?

React memory me access token tha, reload se lost ho gaya. Refresh cookie browser me abhi bhi ho sakti hai.

```js
async function restoreSession() {
  const response = await fetch("/auth/refresh", {
    method: "POST",
    credentials: "include",
  });

  if (!response.ok) return null;

  const { accessToken } = await response.json();
  return accessToken; // store in app memory
}
```

App startup par once call karke authenticated state restore kar sakte ho. Valid refresh session nahi mila to user ko login karna hoga. Har API request par refresh call mat karo.

### Step 9 — Logout me kya karna hai?

```text
User clicks Logout
       |
       v
POST /auth/logout (refresh cookie sent)
       |
       v
Server finds and revokes refresh session in DB
       |
       v
Server clears refresh cookie
       |
       v
React clears access token from memory
```

Cookie clear example:

```js
res.clearCookie("refreshToken", {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/auth",
});
```

**Only cookie clear karna enough nahi hai.** DB session revoke bhi karo. Aur pehle issued JWT access token apni expiry tak valid reh sakta hai unless separate access-token revocation check hota hai.

### Step 10 — Quick comparison

| Question | Access Token | Refresh Token |
| --- | --- | --- |
| Why? | Protected API access | New access token obtain |
| Example lifetime | 15 minutes | 7 days |
| Format | JWT or opaque | JWT or opaque |
| Browser storage in this design | React memory | HttpOnly Secure cookie |
| Server storage in this design | JWT verification (no DB token row required) | Hashed DB session record |
| Sent to | Protected APIs | Refresh/logout endpoint |
| Rotation | New token on refresh | Replace old token every use |

### Step 11 — Interview questions (simple answers)

**Q. Refresh token me userId nahi hai, user kaise identify hoga?**  
Server refresh token ka hash nikalta hai, DB session lookup karta hai, aur matching record se `userId` read karta hai.

**Q. Refresh token kaise generate hota hai?**  
`crypto.randomBytes(32).toString("base64url")` se secure random opaque token bana sakte hain.

**Q. Client aur server par kya store hota hai?**  
Client: raw token in Secure HttpOnly cookie. Server: token ka hash + userId + expiry/revocation in DB.

**Q. Access token kab regenerate hota hai?**  
When it expires/is near expiry, or on page reload if app uses memory and needs to restore session. Client valid refresh credential ke saath refresh endpoint call karta hai.

**Q. Kya random fake token se access token mil jayega?**  
Nahi, because uska hash DB me active session se match nahi karega.

**Q. Kya valid stolen refresh token use ho sakta hai?**  
Haan. Bearer credential chori ho jaye to misuse possible hai. Rotation, secure cookies, CSRF controls, revocation aur monitoring se risk reduce karte hain.

**Q. JWT refresh token compulsory hai?**  
Nahi. Opaque random token + DB mapping ek common, straightforward option hai.

**Q. Rotation ka kya matlab hai?**  
Har successful refresh par old refresh token invalidate karna aur naya issue karna.

### One-page mental model

```text
LOGIN -> Verify password -> Issue A1 + R1
                         -> DB saves userId + hash(R1)
                         -> Browser saves R1 in HttpOnly cookie
                         -> React memory saves A1

API -> React sends A1 -> Server verifies -> protected response

A1 expires -> React POST /auth/refresh
           -> Browser sends R1 cookie
           -> Server hash(R1) -> DB session -> userId
           -> Validate session + user
           -> Atomically invalidate R1, save hash(R2)
           -> Return A2 + set R2 cookie

LOGOUT -> Revoke refresh session + clear cookie + clear memory
```

**Remember:** Refresh token me user info hona zaruri nahi. **Database mapping user ko identify karti hai; secure token possession refresh request ko authenticate karti hai.**

---

## Further concepts and production notes

## Why refresh tokens matter

Refresh tokens solve a key authentication problem:

> How can access tokens stay short-lived without forcing the user to log in every 15 minutes?

A refresh token is a long-lived credential used to obtain a new access token.

Because it lives longer, it must be protected much more carefully.

---

## 1. Basic flow

```text
Login
  |
  v
Access Token + Refresh Token
  |
  v
Access Token expires
  |
  v
Client sends Refresh Token
  |
  v
Server validates refresh session
  |
  v
New Access Token
```

---

## 2. Access vs refresh token

```text
Access Token
  short TTL
  sent often
  accesses APIs

Refresh Token
  long TTL
  sent rarely
  only used at refresh endpoint
```

---

## 3. Why not just use one 30-day token?

Because every API request would carry a credential valid for 30 days.

If stolen:

```text
attacker gets long access window
```

Separating tokens reduces exposure.

---

## 4. Refresh token can be opaque

A strong design is:

```text
random 256-bit token
```

Client stores raw token.

Server stores:

```text
hash(refreshToken)
```

This behaves like a session secret.

---

## 5. Refresh token can also be JWT

Possible, but JWT does not remove need for revocation if you want secure logout and rotation.

Many production systems still keep server-side refresh-session state.

---

## 6. Refresh session model

Example DB record:

```text
refresh_sessions

id
user_id
token_hash
expires_at
created_at
revoked_at
replaced_by
user_agent
ip
```

---

## 7. Why store hash instead of raw refresh token?

Suppose DB leaks.

If raw refresh tokens are stored:

```text
attacker can immediately use them
```

If only hashes are stored:

```text
raw token still required
```

This mirrors password/reset-token storage principles.

---

## 8. Refresh endpoint

Example:

```text
POST /auth/refresh
```

Flow:

```text
read refresh token
   |
   v
hash it
   |
   v
find active session
   |
   v
check expiry/revocation
   |
   v
rotate token
   |
   v
issue new access token
```

---

## 9. Refresh token rotation

Rotation means:

> Every time a refresh token is used, invalidate it and issue a new refresh token.

Flow:

```text
RT1 used
  |
  v
RT1 revoked
  |
  v
RT2 issued

RT2 used
  |
  v
RT2 revoked
  |
  v
RT3 issued
```

---

## 10. Why rotation matters

Suppose attacker steals RT1.

User later refreshes and server rotates to RT2.

RT1 becomes invalid.

This limits stolen token lifetime.

---

## 11. Refresh-token reuse detection

Very important advanced concept.

Scenario:

```text
RT1 used legitimately
   |
   v
RT2 issued

Later attacker uses old RT1
```

That suggests token theft.

Server can respond by revoking the whole token family/session chain.

---

## 12. Token family

Conceptually:

```text
Session Family
   |
   +--> RT1
   +--> RT2
   +--> RT3
```

If a revoked ancestor appears again:

```text
possible replay/theft
```

Revoke family.

---

## 13. Rotation race condition

Two browser requests may attempt refresh simultaneously.

```text
Request A uses RT1
Request B uses RT1
```

If both succeed, rotation protection breaks.

Use atomic update/transaction logic.

---

## 14. Atomic rotation concept

```text
UPDATE refresh_session
SET revoked_at = now()
WHERE token_hash = ?
AND revoked_at IS NULL
```

Only one request should succeed.

Then issue replacement.

---

## 15. Expiration

Refresh token lifetime might be:

```text
7 days
30 days
90 days
```

Depends on:
- app sensitivity
- remember-me policy
- device/session management

Longer duration means higher theft risk.

---

## 16. Absolute vs sliding expiration

Absolute:

```text
session expires 30 days after login
no matter how active user is
```

Sliding:

```text
each refresh extends session
```

Sliding sessions need a maximum absolute cap to avoid never-ending sessions.

---

## 17. Logout

Secure logout:

```text
client requests logout
   |
   v
revoke refresh session
   |
   v
clear refresh cookie
```

Access token may still exist briefly until expiration unless explicitly revoked.

---

## 18. Logout all devices

Server can revoke all refresh sessions for user:

```text
UPDATE sessions
SET revoked_at = now()
WHERE user_id = ?
```

Useful after:

- password reset
- security incident
- account recovery

---

## 19. Device sessions

You can show:

```text
Chrome on Linux
Android
iPhone
```

Each device/session has a separate refresh token record.

User can revoke one device independently.

---

## 20. User agent and IP

Useful as metadata:

```text
user_agent
ip
last_used_at
```

But do not rely on IP/user-agent as strong identity.

They can change or be spoofed.

---

## 21. Refresh token storage in browser

Best common browser approach:

```text
HttpOnly
Secure
SameSite
cookie
```

Why?

JavaScript cannot read it directly.

This helps protect the most valuable long-lived token from XSS theft.

---

## 22. Why not localStorage for refresh token?

Refresh token is long-lived.

If XSS steals it:

```text
attacker can keep minting access tokens
```

That is much worse than losing a short-lived access token.

---

## 23. CSRF on refresh endpoint

If refresh token is in cookie, browser automatically sends it.

Protect refresh endpoint with:

- SameSite
- CSRF protection if needed
- Origin checks
- strict CORS

---

## 24. SameSite architecture

SameSite behavior depends on whether frontend/backend are:

- same-site
- cross-site
- separate subdomains

This must be designed deliberately.

---

## 25. Refresh endpoint should be narrow

A refresh token should generally only be accepted at:

```text
/auth/refresh
```

Do not use it to access normal API routes.

---

## 26. Never return refresh token in logs

Do not log:
- request cookies
- refresh token body
- Authorization headers

Redact credentials.

---

## 27. Password change behavior

After password change, decide:

- revoke all refresh sessions
- revoke all except current session
- require login again

For high-security apps, revoke all sessions.

---

## 28. Refresh token theft detection

Signals:

- reuse of old token
- impossible travel
- sudden UA/IP change
- multiple reuse attempts

Advanced systems can trigger:

- session revocation
- user alert
- MFA challenge

---

## 29. Refresh token validation flow

```text
raw refresh token
      |
      v
hash token
      |
      v
find DB session
      |
      +--> missing -> reject
      |
      +--> revoked -> reject / detect reuse
      |
      +--> expired -> reject
      |
      v
rotate atomically
      |
      v
issue access + next refresh token
```

---

## 30. Example refresh service

```js
async function refreshSession(
  rawToken
) {
  const tokenHash =
    hashToken(rawToken);

  const session =
    await sessionRepo
      .consumeActiveToken(
        tokenHash
      );

  if (!session) {
    throw new InvalidRefreshTokenError();
  }

  const nextRefreshToken =
    generateSecureToken();

  await sessionRepo.createReplacement({
    userId:
      session.userId,
    tokenHash:
      hashToken(
        nextRefreshToken
      ),
    parentId:
      session.id,
  });

  const accessToken =
    await signAccessToken({
      sub:
        session.userId,
    });

  return {
    accessToken,
    refreshToken:
      nextRefreshToken,
  };
}
```

---

## 31. Refresh tokens and JWT revocation

Refresh session DB provides server-side control:

```text
logout
password reset
device removal
security incident
```

This is why "JWT means fully stateless" is often a misleading goal.

---

## 32. Access token refresh frequency

Do not refresh on every request.

Refresh only when:

- access token expired
- access token close to expiry
- proactive client strategy requires it

Otherwise refresh endpoint becomes unnecessary load.

---

## 33. Retry behavior

If multiple API calls receive 401 simultaneously, client should avoid triggering ten refresh requests.

Common SPA pattern:

```text
first 401
   |
   v
one refresh promise
   |
   v
other requests wait
   |
   v
retry with new access token
```

This is sometimes called refresh deduplication.

---

## 34. Common mistakes

### Mistake 1
Refresh token never rotates.

### Mistake 2
Store raw refresh tokens in DB.

### Mistake 3
Store long-lived refresh token in localStorage.

### Mistake 4
No logout/revocation state.

### Mistake 5
No reuse detection.

### Mistake 6
Non-atomic rotation.

### Mistake 7
Use refresh token on normal APIs.

---

## 35. Interview questions

### Why use refresh token?

To keep access tokens short-lived without forcing frequent logins.

### Why rotate refresh tokens?

To invalidate previously issued refresh credentials and reduce replay risk.

### What is reuse detection?

Detecting use of an already consumed/revoked refresh token, which can indicate theft.

### Why hash refresh tokens in DB?

To reduce damage if session database leaks.

### Where should browser refresh token live?

Commonly in a Secure HttpOnly cookie.

---

## 36. Strong interview answer

> A refresh token is a long-lived credential used only to obtain new short-lived access tokens. I usually model refresh tokens as server-side sessions: generate a cryptographically random token, store only its hash, keep the raw token in a Secure HttpOnly cookie, and rotate it on every refresh. If an already-used token appears again, I treat it as possible theft and revoke the token family. Rotation must be atomic to prevent concurrent reuse.

---

## Interview-Ready Summary

```text
Refresh Token
   |
   +--> long-lived
   +--> used rarely
   +--> stored securely
   +--> server-side session
   +--> rotated
   +--> revocable

Best practice:
store hash
detect reuse
revoke family
HttpOnly cookie
```

## Practical Task

Design a refresh-token table and implement:

- login
- refresh
- logout
- logout all devices
- token rotation
- reuse detection
