# FRONTEND_BACKEND_FLOW.md

> Cross-checked against the real, source-verified `API_CONTRACT.md` (Claude 2, 2026-08-16) and
> `FRONTEND.md` (Claude 1, from `CodeArena-Frontend-Architecture.md`). Every row below is now a
> real ✅/❌/⚠️ verdict, not "Unverified." Where a row is ⚠️, see the note for what to reconcile.

**Legend:** ✅ matches · ❌ mismatch, needs a fix · ⚠️ works but with a caveat both sides should know · 🔴 broken backend-side, frontend UI can be built but will fail

---

## Route-by-Route Cross-Check

| Route / Feature | Frontend calls | Status | Note |
|---|---|---|---|
| `/login` (`LoginPage`) | `POST /auth/login` | ✅ | Response shape `{ success, data: { accessToken, refreshToken, user } }` matches exactly. |
| Session boot (`fetchMe`) | `GET /auth/me` | ✅ | Matches; response `{ data: <User, collegeId populated> }`. |
| Logout | `POST /auth/logout` | ✅ | Matches. Frontend swallows errors — fine, since backend clears state regardless. |
| `/practice` (`PracticePage`) | `GET /problems` (paged/filtered) | ✅ | Query params `page`, `limit`, `search`, `difficulty` all match backend's actual query destructure. |
| `/problem/:id` | `GET /problems/:id`, `POST /execute`, `POST /submit` | ✅ | `/execute` = sample-only, no DB write. `/submit` = hidden test cases, creates `Submission`. Distinction confirmed real, not assumed. |
| `/submissions` | `GET /submissions`, `/submissions/stats`, `/submissions/:id` | ⚠️ | Base routes match. **`/submissions/stats?collegeId=...` will 500** (TODO #4) — frontend must omit `collegeId` from this call until backend fixes the un-`new`'d `ObjectId` bug. |
| `/leaderboard` | `GET /leaderboard`, `/leaderboard/overall`, `/leaderboard/:collegeId` | ⚠️ | Routes match. **Caveat both sides should know:** Redis is never connected backend-side (TODO #3) — reads silently fall back to Mongo (still correct data, just slower/no cache), and the `rebuild` endpoints report success but are no-ops. Not a frontend bug, but don't expect `rebuild` to actually do anything yet. |
| `/contests` (`ContestListPage`) | `GET /contests` | ⚠️ | Route matches, but **response shape differs from every other list endpoint**: `{ data, total, page, limit }` — NOT the standard `{ data, meta: {...} }` pagination envelope used elsewhere. **Frontend action needed:** verify `ContestListPage`'s pagination logic reads `total`/`page`/`limit` directly rather than expecting a `meta` object, or it will silently break pagination on this one page. |
| `/contests/:id` | `GET /contests/:id`, `POST /contests/:id/join`, `POST /contests/:id/submit/:problemOrder`, `GET /contests/:id/my-submissions` | 🔴 | `GET`, `join`, `my-submissions` all ✅. **`submit/:problemOrder` is completely broken backend-side** (`runSubmission` is not exported — TODO #1). Build the UI; every submit attempt will throw a 500 until the backend fixes this. |
| `/contests/:id` realtime (Socket.io) | Emits `contest:join`/`contest:leave`, listens `leaderboard:update` | 🔴 | **RESOLVED — root cause confirmed.** Frontend reads the JWT from the wrong `localStorage` key (`accessToken` instead of `cp_access_token`), so every socket connects with `token: undefined`. Backend's `socket.js` uses the same strict JWT verifier as HTTP routes and **will reject** this. Live leaderboard updates are broken for 100% of users today. **Fix is frontend-only** — see `AUTHENTICATION.md` §7 and `TODO.md` #2. |
| `/daily` (`DailyChallengePage`) | `GET /daily`, `POST /daily/submit`, `GET /daily/streak`, `GET /daily/calendar` | ⚠️ | Routes match. Two backend caveats to know: (1) "today" is hardcoded to IST for all users worldwide, not timezone-aware — worth flagging to frontend if you have non-Indian users; (2) `/daily/submit`'s 10-point award is response-only, not persisted anywhere — don't build UI that implies it accumulates into a persistent score total. |
| `/admin/problems` | `POST/PUT/DELETE /problems[/:id]`, `GET /problems/:id/admin` | ✅ | All confirmed, including the `/admin` suffix pattern (it's a real, separate route — `GET /problems/:id/admin`, not a query param) — your original worry about this was reasonable to flag, but it checks out as written. |
| `/admin/students` | `GET/POST /students`, `GET/PUT/DELETE /students/:id` | ✅ | Matches, including that `GET /students/:id` also allows self-access (not just admin). |
| `/admin/colleges` | `GET/POST /colleges`, `/:id`, `/:id/students` | ⚠️ | Matches, but **`PUT /colleges/:id` has no backend-side Joi validation** (TODO #12) — raw `req.body` passed through (Mongoose schema validators still apply, so this is a minor/soft risk, not a functional bug). |
| `/admin/submissions` | Reuses `GET /submissions`, admin-scoped | ✅ | Matches — admin can filter by `userId`/`collegeId`, students cannot. |
| `/admin/leaderboard` | `POST /leaderboard/:collegeId/rebuild`, `/overall/rebuild` | ⚠️ | Routes exist and return success — but see the Redis caveat above; these are currently no-ops. |
| `/admin/contests` | `POST/PUT/DELETE /contests[/:id]` | ⚠️ | Routes match, but two backend quirks: (1) `POST /contests` returns **HTTP 200, not 201** as documented — a `sendSuccess()` argument bug (TODO #11); if frontend checks `response.status === 201` anywhere for this call, it will incorrectly treat success as failure — check for this. (2) `DELETE /contests/:id` is a **hard delete**, unlike every other admin delete in the app (which soft-delete). Any "past contests" list or submission-history view must handle a `contestId` that 404s outright. |
| `/admin/daily` | `POST /daily/schedule`, `DELETE /daily/:date`, `GET /problems` | ✅ | Matches. Confirmed this feature calling `api` directly bypasses `services/` (TODO #23) — functionally fine, just inconsistent styling versus the rest of the app. |

---

## Realtime — Socket.io (RESOLVED)

Previously the highest-priority open question. Now confirmed:

- **Backend behavior:** `socket.js`'s `io.use()` middleware verifies `socket.handshake.auth.token` with the same strict JWT verifier used by HTTP `authenticate` middleware. It does **not** allow degraded/anonymous connections — a missing or invalid token is rejected outright.
- **Frontend bug:** `useContestSocket.js` sends `localStorage.getItem('accessToken')`, which is `undefined` (the real key is `cp_access_token`).
- **Net effect:** live leaderboard updates are broken for all contest participants, right now, in production. No error currently surfaces in the UI for this — it fails silently.
- **Event names** (`contest:join`, `contest:leave`, `leaderboard:update`) are correctly matched on both sides — this part is not the problem.
- **Fix:** one-line change in `useContestSocket.js`. No backend change required.

Full detail: `AUTHENTICATION.md` §7, `TODO.md` #2.

---

## Response Envelope

Confirmed against `API_CONTRACT.md`: the standard envelope is
```json
{ "success": true, "message": "...", "data": {...}, "meta": {...} }
```
Frontend's tolerant unwrapping (`res.data.data ?? res.data`) correctly handles this. **One exception confirmed:** `GET /api/contests` uses `{ data, total, page, limit }` instead of the standard `meta` object — this is the one place frontend's pagination-reading logic needs a specific check (see contests row above), everywhere else the standard envelope holds.

---

## Outstanding items (none are "unverified" anymore, but still open as work)

1. **Fix `useContestSocket.js`'s localStorage key** (frontend, one-line, unblocks live leaderboards) — TODO #2.
2. **Fix `runSubmission` import in `contest.controller.js`** (backend) — TODO #1. Contest submission UI can be built now, but won't work end-to-end until this ships.
3. **Verify `ContestListPage`'s pagination reads `total`/`page`/`limit`, not `meta`** — small frontend check, flagged above.
4. **Check for any `response.status === 201` assumption on contest creation** — will currently see 200, flagged above.
5. **Restrict the contest-submission language picker to `javascript, python, java, cpp, c`** (5, not the full 10 practice-mode supports) until the backend enum is widened or the restriction is made permanent — TODO #5.
