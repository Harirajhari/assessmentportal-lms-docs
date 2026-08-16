# TODO — Defects, Inconsistencies & Next Steps (Merged)

> Combines Claude 2's backend defect list (verified against `assesmentportal-with-lms-backend-v1`
> source, 2026-08-16) and Claude 1's frontend findings (from `CodeArena-Frontend-Architecture.md`,
> 2026-08-16). Item #3 (socket auth) was originally flagged BOTH/unverified by both sides — it is
> now RESOLVED, see `AUTHENTICATION.md` §7. No code has been changed producing this document.

Tags: **BACKEND**, **FRONTEND**, **BOTH**, **DOCUMENTATION**

---

## 🔴 Critical — breaks a whole feature end-to-end

### 1. [BACKEND] Contest submission is completely broken
`contest.controller.js` imports `runSubmission` from `services/judge0.service.js`, which doesn't export a function by that name (actual exports: `submitToJudge0, pollResult, runSingleTestCase, runMultipleTestCases`). **`POST /api/contests/:id/submit/:problemOrder` throws `TypeError` at call time.** Most significant functional defect found. Frontend can build the contest-submission UI, but it will fail end-to-end until this is fixed backend-side.

### 2. [FRONTEND — RESOLVED ROOT CAUSE] Socket auth reads the wrong `localStorage` key
`features/contest/useContestSocket.js` reads `localStorage.getItem('accessToken')` instead of `cp_access_token` (the real key). **Confirmed:** backend's `socket.js` verifies the token with the same strict JWT check as HTTP requests, so it **rejects** the resulting `token: undefined` connection. **Live leaderboard updates are broken for 100% of contest participants**, silently (no error surfaces in the UI). Fix: correct the localStorage key in `useContestSocket.js`. No backend change needed. See `AUTHENTICATION.md` §7.

### 3. [BACKEND] Redis is never connected
`server.js` has `//connectRedis();` commented out. Every `leaderboard.service.js` call to `getRedis()` throws; reads silently fall back to MongoDB, writes/cache-invalidation/rebuild fail silently (logged, swallowed). `POST /api/leaderboard/:collegeId/rebuild` and `/overall/rebuild` are no-ops that report success. App functions, but purely off MongoDB — Redis provides zero benefit currently.

---

## 🟠 High — breaks a specific request/flow

### 4. [BOTH] `getSubmissionStats` — un-`new`'d `ObjectId` call
`controllers/submission.controller.js` line 78 calls `mongoose.Types.ObjectId(collegeId)` without `new` — throws in Mongoose 8 when `collegeId` is supplied. **`GET /api/submissions/stats?collegeId=...` 500s.** Unfiltered version works fine. Frontend: avoid passing `collegeId` to this endpoint until fixed.

### 5. [BOTH] `ContestSubmission.language` enum narrower than practice mode
Contest submissions only accept `javascript, python, java, cpp, c` (5), while practice-mode/Judge0 config support 10 (`+typescript, go, rust, ruby, csharp`). Submitting an unsupported language to a contest fails schema validation. Frontend: contest language picker should be restricted to the 5 supported languages until the backend enum is widened.

### 6. [BOTH] Contest is hard-deleted while every other resource is soft-deleted
`Contest.findByIdAndDelete`, vs. College/Problem/User all using `isActive=false`. Deleting a contest orphans its `ContestParticipant`/`ContestSubmission` history — no cascade delete. Frontend: any "past contests" or submission-history view must handle a `contestId` that 404s.

---

## 🟡 Medium

### 7. [BACKEND] Two different leaderboard score formulas disagree
`User.getLeaderboardScore()` (model method, appears unused/dead) includes a daily-streak bonus; `leaderboard.service.js: calculateScore()` (the one actually used) does not. Maintenance hazard if the dead method is ever wired up.

### 8. [BACKEND] `collegeScope` middleware defined but never used
Exported, never imported into any route. Isolation currently works via controllers reading `req.user.collegeId` implicitly — safe today, but no safety net for a future route that skips this pattern.

### 9. [BACKEND] No rate limiting on contest/daily submission endpoints
`submissionLimiter` (5/min/user) covers `/api/execute` and `/api/submit` only; contest/daily submit endpoints only get the blanket 200-req/min limiter, despite also triggering Judge0 calls.

### 10. [FRONTEND] No refresh-token flow implemented
`authService.refresh` exists but is never called. Combined with the backend's 15-minute access-token TTL (confirmed, `AUTHENTICATION.md` §1), users are forced to fully re-login roughly every 15 minutes of inactive-token use. Real UX cost, not hypothetical, now that the TTL is confirmed.

### 11. [BACKEND] `sendSuccess()` called with an ignored third argument in `createContest`
`sendSuccess(res, {...}, 201)` — signature has no third param. `201` is silently discarded; response defaults to `200`, inconsistent with every other "create" endpoint.

### 12. [BACKEND] `College` update route has no Joi validation
`PUT /api/colleges/:id` passes raw `req.body` to `findByIdAndUpdate`; Mongoose schema validators still apply, but request shape/whitelisting isn't enforced like on create.

### 13. [FRONTEND] No automated tests, linter, or TypeScript anywhere
No `*.test.*`/`*.spec.*` files, no ESLint/Prettier config, plain JS/JSX throughout.

---

## 🟢 Low — cosmetic, dead code, or minor inefficiency

### 14. [BACKEND] `dailyChallenge.controller.js: getCalendar` uses string date range, not real dates
Builds `to = '${y}-${m}-31'` as a string comparison, not real date arithmetic. Happens to work today due to lexicographic string comparison, but relies on coincidence, not correctness.

### 15. [BACKEND] `problem.controller.js` has two separate `module.exports` statements
Second overwrites the first — not a functional bug, but a maintenance smell from a likely copy-paste edit.

### 16. [BACKEND] `connectRedis` isn't invoked, no readiness gate for Mongo either
No fast-fail 503 behavior if MongoDB is briefly unreachable at startup — falls back to default Mongoose buffering/timeout behavior instead.

### 17. [BACKEND] Auth middleware selects `+refreshToken` unnecessarily on every request
`authenticate` selects `+refreshToken` on every request even though it's not read downstream — minor inefficiency, repeated on every authenticated call.

### 18. [FRONTEND] `Topbar.jsx` fully built but never rendered
`AppLayout` only renders `Sidebar` + routed content. Likely superseded by breadcrumb/streak UI folded into `Sidebar`'s footer.

### 19. [FRONTEND] `components/editor/ProblemPanel.jsx` unused duplicate
The actual problem panel is an inline function inside `CodingWorkspace.jsx`. Standalone file is near-identical but unimported.

### 20. [FRONTEND] `CodeEditor.jsx` reset-to-template button references undefined `TEMPLATES`
`resetCode()` calls `TEMPLATES[language]`, but only `LANGUAGES` and `saveCode` are imported — `TEMPLATES` is never imported. **Confirmed runtime bug — throws on every click.**

### 21. [FRONTEND] `services/Contestservice.js` filename casing mismatch vs. import path
Imported as `contestService` (lowercase), file on disk is `Contestservice.js` (capital C). Works on case-insensitive filesystems (macOS/Windows dev) but **will fail module resolution on case-sensitive filesystems** (Linux, most CI/production Docker images). Verify against actual deploy target before shipping — local dev success gives false confidence.

### 22. [FRONTEND] `DailyStreakWidget`'s 7-day mini-calendar mostly decorative
Only "today" reflects real state; other 6 days hardcoded unfilled despite `dailySlice.calendar` existing and being populated.

### 23. [FRONTEND] Daily-challenge endpoints bypass the `services/` layer
Called directly via `api` from `dailySlice.js` / `AdminDailySchedulePage.jsx` instead of a dedicated `dailyService.js`. Functionally fine, just inconsistent layering.

### 24. [DOCUMENTATION] `README.md` is stale
Doesn't mention Contest module, Daily Challenge module, overall/cross-college leaderboard endpoints, or Socket.io. This `lms-docs/` set supersedes it.

---

## Suggested next steps (priority order)

1. **[BACKEND]** Fix or remove the broken `runSubmission` import (#1) — contest submission is unusable until then.
2. **[FRONTEND]** Fix the `localStorage` key in `useContestSocket.js` (#2) — one-line fix, unblocks live leaderboards immediately.
3. **[BACKEND]** Decide whether Redis should be enabled (uncomment `connectRedis()`, align auth headers) or removed entirely (#3).
4. **[BACKEND]** Fix the un-`new`'d `ObjectId` call in submission stats (#4).
5. **[BOTH]** Widen `ContestSubmission.language` enum to match Judge0's 10 languages, or explicitly restrict and document the subset (#5) — frontend's contest language picker should match whatever is decided.
6. **[BACKEND]** Reconcile the two leaderboard-score formulas to one source of truth (#7).
7. **[FRONTEND]** Implement the refresh-token flow, or explicitly accept 15-min forced re-logins as a product decision (#10).
8. **[BACKEND]** Add `submissionLimiter` + Joi validation to contest/daily/college-update routes for parity (#9, #12).
9. **[DOCUMENTATION]** Update `README.md` (#24) — largely superseded by this `lms-docs/` set already.

---

## Quick-reference: what NOT to build/rely on yet

- **Contest code submission** (`POST /api/contests/:id/submit/:problemOrder`) — non-functional, throws on every call (#1).
- **Live contest leaderboard updates** — broken until the frontend socket key fix ships (#2).
- **Admin submission stats filtered by college** (`?collegeId=...`) — 500s (#4). Unfiltered version works.
- **Contest language picker** — limit to `javascript, python, java, cpp, c` (#5).
- **Deleted contests** — expect a 404 on any reference to a deleted `contestId`, not a soft-deleted/inactive state (#6).
