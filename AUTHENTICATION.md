# Authentication & Authorization — Assessment Portal

> Backend half verified directly against `services/auth.service.js`, `middlewares/auth.middleware.js`,
> `controllers/auth.controller.js`, `models/User.js`, and `socket.js` by Claude 2 (2026-08-16).
> Frontend half written from `CodeArena-Frontend-Architecture.md` by Claude 1 (2026-08-16).
> Merged into a single file — this is now the authoritative combined reference for both sides.

---

## 1. JWT flow (Backend)

- **Access token**: signed with `JWT_SECRET`, default expiry `15m` (`JWT_EXPIRES_IN`), issuer `coding-platform`. Payload: `{ id, email, role, collegeId }`.
- **Refresh token**: signed with a **separate secret**, `JWT_REFRESH_SECRET`, default expiry `7d` (`JWT_REFRESH_EXPIRES_IN`), issuer `coding-platform`. Payload: `{ id }` only.
- The refresh token is **persisted on the `User` document** (`refreshToken` field, `select: false`) — this makes refresh tokens **server-side revocable**. Only **one active refresh token per user at a time**; a new login overwrites the old one, effectively logging out any other active session.
- `POST /api/auth/refresh` verifies the refresh JWT **and** checks it matches the token stored on the user's Mongo document before minting a new access token. A stolen-but-superseded refresh token is rejected even though it hasn't expired.
- `POST /api/auth/refresh` returns **only a new access token** — it does not rotate/reissue a new refresh token.
- `POST /api/auth/logout` clears `user.refreshToken` server-side (`save({ validateBeforeSave: false })`).
- **Socket.io also authenticates**: `socket.handshake.auth.token` is verified with the **same access-token verifier** used by HTTP requests, inside `socket.js`'s `io.use()` middleware. There is no refresh-token flow for sockets — a client whose access token expires must reconnect with a freshly-refreshed access token.

## 2. Password hashing (Backend)

- `bcryptjs`, cost factor **12**.
- Hashing happens in a `pre('save')` hook on `User` — runs only `if (this.isModified('password'))`.
- `comparePassword(candidatePassword)` instance method wraps `bcrypt.compare`.

## 3. Middleware — `middlewares/auth.middleware.js` (Backend)

### `authenticate`
1. Parses `Authorization: Bearer <token>` — `401` "Access token required" if missing/malformed.
2. Verifies the token against `JWT_SECRET` — `401` "Invalid or expired access token" if verification fails.
3. Loads the full `User` document by `decoded.id`, with `+refreshToken` explicitly selected (unnecessary field-selection on every request — see `TODO.md` #14).
4. `401` "User no longer exists" if the user record is gone.
5. `401` "Account has been deactivated" if `isActive === false`.
6. On success, attaches `req.user` (full Mongoose doc), `req.userId` (string), `req.userRole` to the request.

### `authorize(...roles)`
- Factory. Returns `403` if `req.user.role` isn't in the allowed set.
- Used throughout as `authorize('admin')` — no route uses `authorize('student')` exclusively; student-only behavior is expressed as "not admin" logic inside controllers.

### `collegeScope` — **defined but unused**
```js
const collegeScope = (req, res, next) => {
  if (req.user.role === 'admin') return next();
  const requestedCollegeId = req.params.collegeId || req.query.collegeId;
  if (requestedCollegeId && requestedCollegeId !== req.user.collegeId?.toString()) {
    return next(new AppError('Access denied: you can only access your own college data', 403));
  }
  next();
};
```
Exported but never imported/attached to any route. College-tenant isolation is instead achieved **implicitly** — controllers read `req.user.collegeId` (server-trusted) rather than trusting client input. Safe everywhere checked, but any future route that trusts a client-supplied `collegeId` without replicating this pattern would leak cross-tenant data.

## 4. Role model (Backend)

Exactly two roles: **`admin`**, **`student`**. No `instructor`/`ta` intermediate role exists at the data-model level despite the "LMS" framing — would require a schema change plus new `authorize(...)` wiring to add.

## 5. Multi-tenant (college) isolation pattern (Backend)

- Every `User` with `role: 'student'` has a **required** `collegeId`; admins do not need one.
- Student-facing endpoints derive scope from `req.user.collegeId` (server-trusted).
- Admin routes accept an explicit `collegeId` route/query param for cross-tenant access.
- **Contest scoping**: `scope: 'all'` (open to every college) or `scope: 'college'` (gated by `allowedColleges[]`). Both `join` and `submitToContest` check `req.user.collegeId` against `contest.allowedColleges`.

---

## 6. Frontend: Auth Implementation

### Token storage
Three `localStorage` keys, all set/cleared together:
- `cp_access_token`
- `cp_refresh_token`
- `cp_user`

### Login flow
1. `LoginPage` dispatches the `login(credentials)` thunk (`authSlice.js`).
2. Thunk calls `POST /auth/login` via `authService.js`.
3. Response shape: `{ success, data: { accessToken, refreshToken, user } }` — **matches backend contract, confirmed against `API_CONTRACT.md`.**
4. On success, `accessToken` → `cp_access_token`, `refreshToken` → `cp_refresh_token`, `user` → `cp_user`, and the `auth` slice state is populated with the same values.

### Session rehydration
`authSlice` initializes its state **synchronously** by reading `localStorage` at module load — no loading spinner needed on refresh, but a stale/tampered `localStorage` value is trusted at boot until something calls `fetchMe` (`GET /auth/me`).

### Request interceptor (attaching the token)
`services/api.js`'s request interceptor reads `cp_access_token` from `localStorage` on every outgoing request and sets `Authorization: Bearer <token>`.

### Response interceptor (401 handling)
Clears all three `localStorage` keys and hard-redirects (`window.location.href`) to `/login` on **any** 401. Global, not opt-outable per call site. Given the backend's "single active refresh token per device" behavior (§1 above), a second-device login will eventually trigger this same 401→redirect flow on the first device once its access token naturally expires.

### Role model (as seen from the UI)
`useAuth()` derives `isAdmin` / `isStudent` from `role === 'admin' | 'student'` — matches backend's two-role model exactly (§4).

Route-level enforcement:
- **`ProtectedRoute`** — redirects to `/login` if `!isLoggedIn`.
- **`AdminRoute`** — redirects to `/login` if not logged in, or to `/` if `role !== 'admin'`.

Both guards are client-side only, as expected for an SPA — actual authorization is enforced server-side via `authorize('admin')` (§3).

### Logout
`logoutUser` thunk calls `POST /auth/logout` best-effort (errors swallowed) and clears local state/`localStorage` regardless of outcome — consistent with backend's logout behavior (§1).

### Refresh-token flow — not implemented on frontend
`refreshToken` is stored on login and `authService.refresh` exists as a function, but **nothing in the app calls it**. Token expiry is handled purely by the global 401 interceptor forcing a full re-login. Given the backend's 15-minute access-token TTL (§1), this means users are forced to fully re-login roughly every 15 minutes of inactivity-free use unless this gap is closed — flagged in `TODO.md` #7 (FRONTEND) as a real UX cost, now that the short TTL is confirmed.

---

## 7. Socket.io auth — RESOLVED (previously flagged as open in `TODO.md` #3 / `FRONTEND_BACKEND_FLOW.md`)

**Backend behavior, confirmed from source (`socket.js`):** the socket auth middleware verifies `socket.handshake.auth.token` using the **same access-token verifier as HTTP requests** — i.e. it is a real, enforced check, not a permissive/no-op one.

**Frontend bug, confirmed from source (`useContestSocket.js`):** reads the JWT via `localStorage.getItem('accessToken')` — the wrong key. The actual key is `cp_access_token` (written by `authSlice.js`, read everywhere else by `services/api.js`).

**Combined conclusion:** every Socket.io connection is currently initiated with `token: undefined`, and the backend's real JWT verification will **reject** that connection. **Live leaderboard updates (`leaderboard:update`) are fully broken for 100% of contest participants today**, with no visible error surfaced in the current UI (this fails silently from the user's perspective).

**Fix required:** frontend-only code change — correct `useContestSocket.js` to read `cp_access_token` instead of `accessToken`. No backend change needed; the backend's rejection behavior is correct and should be left as-is. Downgraded from "BOTH — needs backend confirmation" to **FRONTEND, confirmed root cause, high severity** — see `TODO.md` #3.

---

## 8. Practical implications for the frontend (combined)

- Store both `accessToken` and `refreshToken` from login; only the access token is short-lived (15m) and needs periodic refresh — currently not implemented (see §6).
- A `401` from any endpoint could mean: missing token, expired/invalid token, deactivated account, or a superseded refresh token — treat all as "force re-login," there's no distinct error code differentiating them.
- Logging in on a second device **invalidates the refresh token on the first device** — the first session's access token still works until it naturally expires (≤15 min), but can't silently refresh past that point.
- Socket.io connections need a **fresh, correctly-keyed access token** at `socket.handshake.auth.token` — currently broken, see §7.
- Do not rely on any server-side college-scope guard beyond what's described in §5 — `collegeScope` is dead code, not active anywhere.
