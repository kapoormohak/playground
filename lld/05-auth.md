# LLD 05 — Authentication: JWT, Sessions, Remember Me, RBAC

> **Companion HLD:** [hld/05-request-lifecycle.md](../hld/05-request-lifecycle.md)  
> **File:** [`src/shared/middleware/auth.ts`](../../pipeclose-backend/src/shared/middleware/auth.ts)

---

## The Problem Authentication Solves

HTTP is stateless. Every request is a fresh connection with no memory of who sent it. The server has no idea if the person sending `PATCH /leads/123` is the owner of that lead or a random person on the internet.

Authentication answers: **"Who are you?"**
Authorization answers: **"Are you allowed to do this?"**

---

## JWT — JSON Web Token

When a user logs in with their email and password, the server:
1. Verifies the password hash (bcrypt compare)
2. Issues a **JWT** — a signed token the client stores and sends on every subsequent request

### What a JWT Is

A JWT is a base64-encoded string with three parts separated by dots:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header (algorithm + type)
.
eyJpZCI6NDIsImVtYWlsIjoiYWRpdHlhQGFjbWUuY29tIiwiY29tcGFueUlkIjo1fQ  ← Payload
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

The **Payload** (middle part) decodes to:
```json
{
  "id": 42,
  "email": "aditya@acme.com",
  "companyId": 5,
  "role": "admin",
  "isSuperAdmin": false,
  "tokenVersion": 3,
  "iat": 1720000000,
  "exp": 1720604800
}
```

The **Signature** is computed with HMAC-SHA256 using the server's `JWT_SECRET`:
```
HMAC-SHA256(base64(header) + "." + base64(payload), JWT_SECRET)
```

If anyone modifies the payload (e.g., changes `role` from `"user"` to `"admin"`), the signature no longer matches and `verifyToken()` rejects it. The secret never leaves the server.

### Token Lifecycle

```
Login ──► server issues JWT (expires in 7 days by default)
              │
              ▼ browser stores in localStorage or memory
              
HTTP request ──► browser sends `Authorization: Bearer <token>`
                       │
                       ▼
              server calls verifyToken(token):
              1. Checks signature (was this token issued by us?)
              2. Checks expiry (iat + exp)
              3. Checks tokenVersion against DB (see below)
```

### tokenVersion — The Logout Kill Switch

**Problem:** JWTs are stateless — once issued, there's no way to invalidate them before they expire. If a user changes their password or gets compromised, their old JWT still works for up to 7 days.

**Solution:** Store a `tokenVersion` integer on each User row. Include it in the JWT payload. On each request, compare the JWT's `tokenVersion` against the DB:

```typescript
// In authMiddleware:
const dbUser = await prisma.user.findUnique({
  where: { id: decodedToken.id },
  select: { tokenVersion: true },
});

if (dbUser.tokenVersion !== decodedToken.tokenVersion) {
  // Token was invalidated (user logged out, changed password, admin revoked)
  return ResponseHandler.unauthorized(res, 'Session expired');
}
```

When you log out:
```sql
UPDATE "User" SET "tokenVersion" = "tokenVersion" + 1 WHERE id = 42;
```

The user's existing JWT now fails the version check on the next request — effectively immediate revocation without needing to maintain a blocklist.

---

## Three Ways to Authenticate

### 1. Bearer Token (Primary)

Most API requests use this. The frontend stores the JWT in memory or localStorage and sends it as:
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 2. Session Cookie (Browser Requests)

Some browser-native requests don't include custom headers (e.g., `<img src="/api/emails/track/open/123">` for tracking pixels). The browser sends cookies automatically.

On login, the server sets an **HttpOnly session cookie**:
```typescript
res.cookie(SESSION_COOKIE_NAME, jwtToken, {
  httpOnly: true,      // JS cannot read this — XSS protection
  secure: true,        // HTTPS only
  sameSite: 'none',    // Cross-site (needed for subdomain setup)
  maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
});
```

`HttpOnly`: JavaScript on the page cannot access this cookie. This protects against XSS attacks — even if a malicious script runs on the page, it can't steal the session token.

### 3. Remember Me Token (Auto-Login)

When a user checks "Remember Me" on login, a **long-lived opaque token** is stored in the database and a cookie is set:

```
user logs in with "remember me" ──► server generates a random 64-byte token
                                         │
                          stores hash in DB: RememberToken { userId, tokenHash, expiresAt: 90days }
                          sets cookie: remember_token=<raw_token>; HttpOnly; 90 days
```

On a future request where the JWT is missing or expired:
1. Auth middleware reads `remember_token` cookie
2. Validates it against the database (hashed comparison)
3. If valid: generates a new JWT, sets the user on `req.user`, continues

Note: the remember token is NOT rotated on every use (this would cause race conditions — 3 concurrent tab loads would all try to rotate the same token and invalidate each other). It's only rotated on explicit login.

---

## The Full Auth Middleware Logic

**File:** [`src/shared/middleware/auth.ts`](../../pipeclose-backend/src/shared/middleware/auth.ts)

```
Step 1: Try JWT from Authorization header or session cookie
  ├── If found and valid:
  │     Check tokenVersion against DB
  │     If matches: user = decoded token payload
  │     If doesn't match: user = null (token invalidated)
  └── If not found: user = null

Step 2: If user = null, try remember_token cookie
  ├── If cookie found:
  │     Validate against DB (hash comparison)
  │     If valid: fetch user from DB, set user = dbUser
  │     If invalid: clear the remember_token cookie
  └── If cookie not found: skip

Step 3: If user still null → return 401 Unauthorized

Step 4: Fetch company record to check it's active
  ├── Check Redis cache (company:auth:{companyId})
  ├── If cache miss: query PostgreSQL, write to Redis (60s TTL)
  ├── If company not found or inactive → return 401
  └── Cross-tenant check: if req.tenant.id ≠ user.companyId → return 403

Step 5: Attach user to request
  req.user = { id, email, companyId, role, ... }
  
Step 6: Load RBAC permissions (per-request, not from JWT)
  req.user.permissions = await getUserPermissionKeys(userId, companyId)
  
Step 7: Attach trace actor (for observability)
  attachTraceActor({ companyId, userId, method, path })
  
Step 8: next() — continue to controller
```

---

## RBAC — Role-Based Access Control

**Module:** `src/modules/rbac`

Authorization is not just "are you logged in?" — it's "are you allowed to do this specific thing?"

### The Three Tiers of Access

```
SuperAdmin (isSuperAdmin: true)
  └── Can do anything across all companies (Pipeclose staff)
  
Company Admin (role: 'admin')
  └── Full access within their company
  
Custom Role (roleId: N)
  └── Access defined by a set of permissions assigned to that role
```

### Permissions

A permission is a string key like `leads.create`, `deals.delete`, `sequences.manage`. They're stored in the database:

```
Permission { key: 'leads.create', description: '...' }
Role       { name: 'Sales Rep', companyId: 5 }
RolePermission { roleId: 2, permissionKey: 'leads.create' }
UserRole   { userId: 42, roleId: 2 }
```

On every authenticated request, `getUserPermissionKeys(userId, companyId)` fetches the user's permissions from the database. This is not cached per-request (it's loaded fresh each time) — this means if an admin removes a permission from a user, it takes effect on their next request, not after some cache expiry.

```typescript
// In a controller that needs permission check
if (!req.user!.permissions.includes('leads.create')) {
  return ResponseHandler.forbidden(res, 'You do not have permission to create leads');
}
```

### Why Not Embed Permissions in the JWT?

If permissions were stored in the JWT, a revoked permission wouldn't take effect until the token expired (7 days). Per-request DB lookup means revocation is instant.

The trade-off: one extra DB query per request. At scale, this could be cached (e.g., Redis with a short TTL). For now, the straightforward approach is used.

---

## CORS and Credential Sharing

**File:** [`src/shared/middleware/auth.ts` — `corsMiddleware`](../../pipeclose-backend/src/shared/middleware/auth.ts)

CORS (Cross-Origin Resource Sharing) controls which browser origins can make requests to this API.

```
Allowed: app.pipeclose.com, acme.pipeclose.com, globex.pipeclose.com (any *.pipeclose.com subdomain)
Denied: evilsite.com, app-pipeclose.com (look-alike domains)
```

`Access-Control-Allow-Credentials: true` is only sent when the origin is allowed. This is required for the browser to include cookies (like the session cookie) in cross-origin requests. **You can never send this with `Access-Control-Allow-Origin: *`** — browsers prohibit that combination.

---

## Next

- [06-multi-tenancy.md](./06-multi-tenancy.md) — the full tenant isolation picture
- [02-http-layer.md](./02-http-layer.md) — how controllers use `req.user` after auth
