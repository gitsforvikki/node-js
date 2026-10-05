# Lesson 130 — Role-Based Access Control (RBAC)

## Why RBAC matters

Authentication answers:

```text
Who are you?
```

RBAC helps answer:

```text
What are users with your role allowed to do?
```

RBAC is one of the most common authorization models in backend systems.

Examples:

- user
- admin
- moderator
- support
- manager

But good RBAC design is more than checking:

```js
if (user.role === "admin")
```

---

# 1. What is RBAC?

RBAC stands for:

```text
Role-Based Access Control
```

Permissions are assigned to roles.

Users receive permissions through their assigned role(s).

---

# 2. Core Model

```text
User
  |
  v
Role
  |
  v
Permissions
```

Example:

```text
Vikash
  |
  v
ADMIN
  |
  +--> users:read
  +--> users:update
  +--> users:delete
```

---

# 3. Simple Role Model

User:

```json
{
  "id": "usr_1",
  "role": "admin"
}
```

Authorization:

```js
if (
  req.user.role !==
  "admin"
) {
  return res
    .status(403)
    .json({
      message:
        "Forbidden",
    });
}
```

Works for small systems.

---

# 4. Why simple role checks become limited

Suppose roles:

```text
user
support
editor
manager
admin
super-admin
```

Then route logic becomes:

```js
if (
  role === "admin" ||
  role === "manager" ||
  role === "support"
)
```

This becomes hard to maintain.

---

# 5. Permission-based RBAC

Better:

```text
ADMIN
  -> user.read
  -> user.update
  -> user.delete

SUPPORT
  -> user.read

MANAGER
  -> user.read
  -> report.read
```

Then authorization checks permission, not role name.

---

# 6. Database Model

Typical relational model:

```text
users
roles
permissions
user_roles
role_permissions
```

Architecture:

```text
User
  |
  v
user_roles
  |
  v
Role
  |
  v
role_permissions
  |
  v
Permission
```

---

# 7. Single-role vs multi-role

Simple app:

```text
one user
one role
```

Complex system:

```text
user
  |
  +--> support
  +--> billing
  +--> team-admin
```

Choose based on actual requirements.

---

# 8. Permission Naming

Good permission names should be explicit.

Examples:

```text
users.read
users.create
users.update
users.delete

orders.read
orders.cancel

products.manage
```

Avoid vague:

```text
power_user
special_access
```

---

# 9. Action + Resource model

Common pattern:

```text
resource.action
```

Example:

```text
orders.read
orders.update
orders.cancel
```

This scales better than arbitrary permission naming.

---

# 10. RBAC Middleware

Example:

```js
function requireRole(
  ...roles
) {
  return (
    req,
    res,
    next
  ) => {
    if (
      !roles.includes(
        req.user.role
      )
    ) {
      return res
        .status(403)
        .json({
          error: {
            code:
              "FORBIDDEN",
          },
        });
    }

    next();
  };
}
```

---

# 11. Permission Middleware

Better for larger apps:

```js
function requirePermission(
  permission
) {
  return (
    req,
    res,
    next
  ) => {
    if (
      !req.user.permissions
        .includes(permission)
    ) {
      return res
        .status(403)
        .json({
          error: {
            code:
              "FORBIDDEN",
          },
        });
    }

    next();
  };
}
```

---

# 12. Route Example

```js
router.delete(
  "/users/:id",
  authenticate,
  requirePermission(
    "users.delete"
  ),
  deleteUser
);
```

This is cleaner than checking role names inside controller.

---

# 13. Authentication before RBAC

Order:

```text
Request
  |
  v
Authentication
  |
  v
Load roles/permissions
  |
  v
RBAC
  |
  v
Controller
```

You cannot authorize an unknown identity.

---

# 14. RBAC Is Not Enough for Ownership

Critical interview point.

Suppose:

```text
role = user
```

User can read orders.

Does that mean every order?

No.

You still need:

```text
order belongs to this user
```

---

# 15. RBAC + Ownership

Example:

```text
permission:
orders.read

AND

resource:
order.userId === req.user.id
```

Or admins may bypass ownership.

---

# 16. Policy Example

```js
function canReadOrder(
  user,
  order
) {
  if (
    user.role === "admin"
  ) {
    return true;
  }

  return (
    order.userId ===
    user.id
  );
}
```

---

# 17. RBAC vs ABAC

RBAC:

```text
decision based primarily on role
```

ABAC:

```text
decision based on attributes
```

Examples:
- department
- location
- resource owner
- time
- subscription tier

---

# 18. RBAC vs ACL

ACL:

```text
resource
  |
  +--> user A can read
  +--> user B can edit
```

RBAC:

```text
role
  |
  +--> permissions
```

ACL is resource-specific.

---

# 19. Hierarchical RBAC

Example:

```text
SUPER_ADMIN
   |
   v
ADMIN
   |
   v
MANAGER
   |
   v
USER
```

Higher role may inherit permissions.

Be careful with accidental privilege escalation.

---

# 20. Deny by Default

Best principle:

```text
no explicit permission
      |
      v
deny
```

Do not default to allow.

---

# 21. Least Privilege

Give role only what it needs.

Example:

Support role may need:

```text
users.read
orders.read
```

not:

```text
users.delete
payments.refund
```

---

# 22. Permission Caching

Permissions may be loaded from DB every request.

To reduce cost:

- store in session
- cache in Redis
- include in short-lived JWT

But this creates freshness considerations.

---

# 23. Stale Role Problem

User starts as:

```text
admin
```

Then is demoted.

JWT/session cache still says:

```text
admin
```

Possible solutions:

- short token TTL
- permission version
- DB lookup for sensitive actions
- cache invalidation
- session revocation

---

# 24. Permission Version

User:

```json
{
  "permissionVersion": 7
}
```

Auth session/token:

```json
{
  "permissionVersion": 7
}
```

When role changes:

```text
permissionVersion -> 8
```

Old credentials can be rejected or refreshed.

---

# 25. Multi-tenant RBAC

Very important in SaaS.

User may be:

```text
ADMIN in company A
USER in company B
```

Role cannot always live globally on user.

Better:

```text
membership
  userId
  tenantId
  role
```

---

# 26. Tenant-aware Authorization

Request:

```text
DELETE /companies/123/users/456
```

Authorization must check:

```text
caller belongs to company 123
AND
caller has users.delete permission in company 123
```

Global role alone is unsafe.

---

# 27. Database Example

```text
memberships
----------------
user_id
organization_id
role_id
```

This is common in SaaS.

---

# 28. Super Admin Caution

A global super-admin role is powerful.

Protect it with:

- MFA
- audit logs
- reauthentication
- minimal assignment
- strong monitoring

---

# 29. Audit Authorization Decisions

For sensitive operations, log:

```text
who
what resource
what action
what role
result
requestId
timestamp
```

Useful for security investigations.

---

# 30. Do Not Trust Role From Request

Bad:

```json
{
  "role": "admin"
}
```

Client input never decides authorization.

Role comes from trusted server-side identity/session/database.

---

# 31. Do Not Trust JWT Role Without Verification

Decoded token:

```text
role = admin
```

is meaningless unless signature is verified.

---

# 32. Field-level Authorization

Example:

Support can update:

```text
user.displayName
```

but cannot update:

```text
user.role
```

Authorization must sometimes apply per field.

---

# 33. Mass Assignment Risk

Bad:

```js
await User.update(
  req.params.id,
  req.body
);
```

Attacker could send:

```json
{
  "role": "admin"
}
```

Whitelisting fields is essential.

---

# 34. RBAC in Service Layer

Do not rely exclusively on route middleware if the same service can be called from:

- HTTP route
- background job
- internal function
- queue consumer

Sensitive domain authorization may need to exist in policy/service layer too.

---

# 35. Route Middleware vs Policy Layer

Route middleware:

```text
coarse permission
```

Example:

```text
orders.cancel
```

Policy/service:

```text
is this specific order cancellable by this user?
```

Both can work together.

---

# 36. Common Mistakes

### Mistake 1
Role check only, no ownership check.

### Mistake 2
Trust role from request body.

### Mistake 3
Hardcode role checks everywhere.

### Mistake 4
No tenant context.

### Mistake 5
Cached permissions never invalidated.

### Mistake 6
Default allow.

---

# 37. Interview Questions

## What is RBAC?

Authorization model where permissions are assigned to roles and users receive permissions through roles.

## Role vs permission?

Role is a grouping. Permission represents a specific allowed action.

## Is RBAC enough for user-owned resources?

No. Ownership/object-level authorization is also needed.

## What is multi-tenant RBAC?

Roles are scoped to a tenant/organization rather than globally to the user.

## RBAC vs ABAC?

RBAC uses roles; ABAC uses user/resource/environment attributes.

---

# 38. Strong Interview Answer

> RBAC maps users to roles and roles to permissions. In small applications I may use direct role middleware, but in larger systems I prefer permission-based checks such as orders.cancel or users.delete. RBAC alone is not enough for object-level security, so I combine it with ownership or policy checks. In multi-tenant SaaS systems, roles should usually be scoped through organization memberships rather than stored as one global role on the user.

---

# Interview-Ready Summary

```text
User
  |
  v
Role
  |
  v
Permissions

Examples:
users.read
users.delete
orders.cancel

Need in addition:
ownership checks
tenant scope
field-level rules
least privilege
deny by default
```

## Practical Task

Design RBAC for a SaaS project with:

Roles:
- owner
- admin
- manager
- member

Permissions:
- members.read
- members.invite
- members.remove
- billing.read
- billing.manage
- projects.create
- projects.delete

Make roles tenant-specific.
