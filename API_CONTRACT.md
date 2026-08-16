# API Contract — Assessment Portal Backend

> Every route below was read directly from the route files (`routes/*.js`) and their controllers
> in `assesmentportal-with-lms-backend-v1` (verified 2026-08-16), not summarized from the analysis
> doc. Validation status was confirmed by checking whether each route wires `validate(schema)`
> from `middlewares/validate.js`.

## Response envelope (applies to all routes)

```json
// success
{ "success": true, "message": "...", "data": {...}, "meta": {...} }

// error (production)
{ "success": false, "message": "..." }

// error (development, NODE_ENV=development)
{ "success": false, "error": {...}, "message": "...", "stack": "..." }
```

`meta` (when present, on paginated list endpoints) has the shape:
```json
{ "currentPage": 1, "totalPages": 5, "totalItems": 97, "itemsPerPage": 20, "hasNextPage": true, "hasPrevPage": false }
```

All protected routes require header `Authorization: Bearer <accessToken>`.

Global error translation (`middlewares/errorHandler.js`): Mongoose `CastError` → 400, duplicate-key (`11000`) → 409, `ValidationError` → 400, `JsonWebTokenError`/`TokenExpiredError` → 401. Any other thrown error → 500 with a generic message in production (full error+stack in development).

---

## 1. Auth — `/api/auth` (`routes/auth.routes.js`)

| Method | Path | Auth | Rate limit | Validation |
|---|---|---|---|---|
| POST | `/login` | Public | `authLimiter` (10/15min) | `loginSchema`: `email` (valid email, required), `password` (min 6, required) |
| POST | `/refresh` | Public | — | `refreshTokenSchema`: `refreshToken` (required) |
| POST | `/logout` | Bearer | — | none |
| GET | `/me` | Bearer | — | none |

### `POST /api/auth/login`
- **Body**: `{ email, password }`
- **Success (200)**: `{ success: true, message: "Login successful", data: { accessToken, refreshToken, user: { id, name, email, role, collegeId, totalSolved, streak, accuracy } } }`
- **Errors**: `401` invalid email or password (also fires on inactive account, since the query filters `isActive: true`).

### `POST /api/auth/refresh`
- **Body**: `{ refreshToken }`
- **Success (200)**: `{ data: { accessToken } }` — **note: only a new access token is returned, not a new refresh token.**
- **Errors**: `401` invalid/expired refresh token; `401` refresh-token mismatch (doesn't match what's stored on the user doc — this rejects stolen-but-superseded tokens); `401` if account deactivated.

### `POST /api/auth/logout`
- **Auth**: Bearer.
- Clears `user.refreshToken` server-side.
- **Success (200)**: `{ message: "Logged out successfully" }`

### `GET /api/auth/me`
- **Auth**: Bearer.
- **Success (200)**: `{ data: <User doc with collegeId populated as {name, code}> }`

---

## 2. Colleges — `/api/colleges` (`routes/college.routes.js`)
All routes require `authenticate` (`router.use(authenticate)`).

| Method | Path | Role | Validation |
|---|---|---|---|
| GET | `/` | any authenticated | none (raw query destructure) |
| POST | `/` | admin | `createCollegeSchema`: `name` (2–150 chars, required), `code` (alphanum, max 8, optional) |
| GET | `/:id` | any authenticated | none |
| PUT | `/:id` | admin | **⚠ NOT VALIDATED** — raw `req.body` passed to `findByIdAndUpdate` (Mongoose `runValidators: true` still applies schema-level checks, but there's no request-shape whitelisting like on create) |
| DELETE | `/:id` | admin | none |
| GET | `/:id/students` | admin | none |

### `GET /api/colleges`
- **Query**: `page` (default 1), `limit` (default 20), `search` (uses Mongo text index on `name`)
- **Success (200)**: `{ data: College[], meta }` — only `isActive: true` colleges.

### `POST /api/colleges`
- **Body**: `{ name, code? }`
- **Success (201)**: `{ data: <College> }`
- **Errors**: `409` if a college with the same name already exists (case-insensitive regex check).

### `GET /api/colleges/:id`
- **Success (200)**: `{ data: <College> }`
- **Errors**: `404` not found.

### `PUT /api/colleges/:id`
- **Body**: any subset of College fields (unvalidated at the request layer).
- **Success (200)**: `{ message: "College updated", data: <College> }`
- **Errors**: `404` not found.

### `DELETE /api/colleges/:id`
- Soft-delete (`isActive = false`).
- **Success (200)**: `{ message: "College deactivated successfully" }`
- **Errors**: `404` not found.

### `GET /api/colleges/:id/students`
- **Query**: `page`, `limit`, `search` (matches name/email, case-insensitive regex)
- **Success (200)**: `{ data: { college, students: User[] (only name/email/totalSolved/streak/accuracy/createdAt) }, meta }`
- **Errors**: `404` college not found.

---

## 3. Students — `/api/students` (`routes/student.routes.js`)
All routes require `authenticate`.

| Method | Path | Role | Validation |
|---|---|---|---|
| POST | `/` | admin | `createStudentSchema`: `name` (2–100, required), `email` (required), `password` (min 6, required), `collegeId` (24-char hex, required) |
| GET | `/` | admin | none |
| GET | `/:id` | admin **or self** | none |
| PUT | `/:id` | admin | `updateStudentSchema`: `name`, `email`, `isActive` (all optional) |
| DELETE | `/:id` | admin | none |

### `POST /api/students`
- **Body**: `{ name, email, password, collegeId }`
- Validates college exists and is active; validates email uniqueness; increments `College.studentCount`.
- **Success (201)**: `{ data: { id, name, email, role, collegeId, createdAt } }`
- **Errors**: `404` college not found/inactive; `409` email already registered.

### `GET /api/students`
- **Query**: `page`, `limit`, `search` (name/email regex), `collegeId`
- **Success (200)**: `{ data: User[] (populated collegeId {name,code}, selected fields), meta }`

### `GET /api/students/:id`
- Students may only fetch their own profile (`req.userId === id`, checked in controller — admins bypass).
- **Success (200)**: `{ data: <User without refreshToken, collegeId populated> }`
- **Errors**: `403` if a student requests someone else's id; `404` not found or not a student.

### `PUT /api/students/:id`
- **Body**: any subset of `{ name, email, isActive }`
- **Success (200)**: `{ message: "Student updated", data: <User> }`
- **Errors**: `404` not found; `409` email already in use by another account.

### `DELETE /api/students/:id`
- Soft-deletes (`isActive=false`), removes the user from the Redis leaderboard (no-op today since Redis isn't connected — see `BACKEND.md`), decrements `College.studentCount`.
- **Success (200)**: `{ message: "Student deactivated successfully" }`
- **Errors**: `404` not found.

---

## 4. Problems — `/api/problems` (`routes/problem.routes.js`)
All routes require `authenticate`.

| Method | Path | Role | Validation |
|---|---|---|---|
| GET | `/` | any | none |
| POST | `/` | admin | `createProblemSchema` (full schema — see below) |
| GET | `/:id` | any | none |
| GET | `/:id/admin` | admin | none |
| PUT | `/:id` | admin | `updateProblemSchema` (same schema, all top-level fields optional) |
| DELETE | `/:id` | admin | none |

`createProblemSchema` fields: `title` (3–200, required), `difficulty` (`Easy`/`Medium`/`Hard`, required), `tags` (array, min 1, required), `description` (min 10, required), `constraints` (min 5, required), `examples` (array of `{input, output, explanation?}`, min 1, required), `testCases.sample`/`testCases.hidden` (array of `{input, expectedOutput, explanation?}`, min 1 each, both required), `starterCode` (optional per-language object), `timeLimit` (500–10000, optional), `memoryLimit` (16–512, optional).

### `GET /api/problems`
- **Query**: `page`, `limit`, `search` (title/tags), `difficulty`
- Always strips `testCases.hidden` and `testCases.sample[].explanation`.
- **Enriches each problem** with the requesting user's `isSolved`/`isAttempted`/`attempts` via a `Submission` aggregate grouped by `problemId`.
- **Success (200)**: `{ data: Problem[] (enriched), meta }`

### `POST /api/problems`
- **Body**: full problem shape per `createProblemSchema`.
- **Success (201)**: `{ data: <Problem, hidden test cases stripped> }`
- **Errors**: `409` duplicate title.

### `GET /api/problems/:id`
- Always strips `testCases.hidden`.
- **Success (200)**: `{ data: <Problem> }`
- **Errors**: `404` not found or inactive.

### `GET /api/problems/:id/admin`
- Includes `testCases.hidden` (explicit `.select('+testCases.hidden')`).
- **Success (200)**: `{ data: <Problem, full> }`
- **Errors**: `404` not found.

### `PUT /api/problems/:id`
- **Body**: any subset of problem fields per `updateProblemSchema`.
- **Success (200)**: `{ message: "Problem updated", data: <Problem, hidden stripped> }`
- **Errors**: `404` not found.

### `DELETE /api/problems/:id`
- Soft delete (`isActive=false`).
- **Success (200)**: `{ message: "Problem deleted successfully" }`
- **Errors**: `404` not found.

---

## 5. Code execution — `/api/execute`, `/api/submit` (`routes/execution.routes.js`, mounted at `/api`)
Both require `authenticate` + `submissionLimiter` (5/min per user, keyed on `req.userId` or IP).

| Method | Path | Validation |
|---|---|---|
| POST | `/api/execute` | `executeSchema`: `problemId` (24-char hex, required), `code` (1–65536 chars, required), `language` (one of the 10 supported, required) |
| POST | `/api/submit` | `submitSchema` — identical shape to `executeSchema` |

### `POST /api/execute`
- Runs code against the problem's **sample** test cases only. No DB write, no stats update.
- **Errors**: `404` problem not found; `400` if the problem has no sample test cases.
- **Success (200)**: `{ message: "Execution complete", data: { language, overallStatus, allPassed, passedCount, totalTestCases, avgRuntime, maxMemory, results: [{ passed, input, expectedOutput, actualOutput, stdout, stderr, compileOutput, statusDescription, runtime, memory }] } }`

### `POST /api/submit`
- Runs against **hidden** test cases. Creates a `Submission` (status `Pending` → final).
- On `Accepted`: increments `totalSolved` (first-solve only), sets `isFirstAccepted`, updates general streak + accuracy, calls `syncUserLeaderboard` (Redis write — silently no-ops today, see `BACKEND.md`).
- On any result: updates `Problem.totalSubmissions`/`totalAccepted`.
- Non-accepted results include only the *first* failing case in the response body.
- **Errors**: `404` problem not found; `500` if the problem has no hidden test cases configured; `500` "Execution failed: ..." if the Judge0 call itself throws (submission is marked `Internal Error` first).
- **Success (201)**: `{ message: "🎉 Accepted!..." | "Submission status: ...", data: { submissionId, status, language, passedCount, totalTestCases, runtime, memory, isFirstAccepted, compileOutput?, firstFailure? } }`

---

## 6. Submissions — `/api/submissions` (`routes/submission.routes.js`)
All routes require `authenticate`. **No Joi validation on any route in this file.**

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/` | any (scoped) | Admin sees all (filterable by `userId`/`collegeId`); student sees only own. Also filterable by `status`, `problemId`. |
| GET | `/stats` | admin | Aggregate counts grouped by `status` — **⚠ known bug, see below.** |
| GET | `/:id` | admin or owner | Full submission incl. code. |

### `GET /api/submissions`
- **Query**: `page`, `limit`, `status`, `problemId`, `userId`, `collegeId` (last two admin-only, ignored for students who are always scoped to their own `userId`)
- `code` field excluded from the list view.
- **Success (200)**: `{ data: Submission[] (userId/problemId/collegeId populated), meta }`

### `GET /api/submissions/stats`
- **Query**: `collegeId` (optional)
- **⚠ BUG (verified in source, `controllers/submission.controller.js` line 78)**: `require('mongoose').Types.ObjectId(collegeId)` is called **without `new`**. In Mongoose 8, `ObjectId` is a class and must be invoked with `new` — calling it as a plain function throws `TypeError: Class constructor ObjectId cannot be invoked without 'new'`. **`GET /api/submissions/stats?collegeId=...` will throw a 500 whenever `collegeId` is supplied.** Calling it *without* `collegeId` works fine (the match stage short-circuits to `{}`).
- **Success (200), when it doesn't hit the bug**: `{ data: { total, accepted, acceptanceRate, breakdown: [{status, count}] } }`

### `GET /api/submissions/:id`
- **Success (200)**: `{ data: <Submission, full, incl. code, populated userId/problemId/collegeId> }`
- **Errors**: `404` not found; `403` if a student requests someone else's submission.

---

## 7. Leaderboard — `/api/leaderboard` (`routes/leaderboard.routes.js`)
All routes require `authenticate`. No Joi validation on this router.

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/` | any | Student: own-college leaderboard + `myRank` + `myOverallRank`. Admin: summary of top-5 per college across **all** colleges. |
| GET | `/overall` | any | Cross-college leaderboard, paginated; students additionally get `myOverallRank`. |
| POST | `/overall/rebuild` | admin | Force-rebuild the global leaderboard cache from Mongo. |
| GET | `/:collegeId` | admin | Specific college's leaderboard. |
| POST | `/:collegeId/rebuild` | admin | Force-rebuild that college's leaderboard cache from Mongo. |

**Important caveat for all leaderboard endpoints**: since Redis is never connected at runtime (see `BACKEND.md`), every read here silently falls back to a MongoDB query, and both `rebuild` endpoints report success but are effectively no-ops (they rebuild a Redis cache that nothing subsequently reads from, since reads always fail over to Mongo before ever finding a populated cache).

### `GET /api/leaderboard`
- **Query**: `page` (default 1), `limit` (default 50)
- **Success (200), student**: `{ data: { <leaderboard entries + pagination fields from service>, myRank, myOverallRank } }`
- **Success (200), admin**: `{ message: "All college leaderboards", data: <top-5 per college> }`
- **Errors**: `400` if a student has no `collegeId` on their account.

### `GET /api/leaderboard/overall`
- **Query**: `page`, `limit`
- **Success (200)**: `{ data: { ...leaderboard, myOverallRank? } }` (`myOverallRank` present only for students)

### `POST /api/leaderboard/overall/rebuild`
- **Success (200)**: `{ message: "Overall leaderboard rebuilt successfully" }`

### `GET /api/leaderboard/:collegeId`
- **Success (200)**: `{ data: { college: {id, name, code}, ...leaderboard } }`
- **Errors**: `404` college not found.

### `POST /api/leaderboard/:collegeId/rebuild`
- **Success (200)**: `{ message: "Leaderboard rebuilt for <college name>" }`
- **Errors**: `404` college not found.

---

## 8. Contests — `/api/contests` (`routes/contest.routes.js`) — **undocumented in README**
All routes require `authenticate`. **No Joi validation on any contest route** — raw `req.body` used throughout. **No `submissionLimiter`** on the submit endpoint (only the blanket 200/min `apiLimiter` applies).

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/` | any (scoped) | List contests. |
| GET | `/:id` | any (scoped) | Single contest. |
| POST | `/:id/join` | any | Register for a contest. |
| GET | `/:id/leaderboard` | any | Computed contest leaderboard. |
| GET | `/:id/my-submissions` | any | Own `ContestSubmission` history for that contest. |
| POST | `/:id/submit/:problemOrder` | any | **Currently broken — see below.** |
| POST | `/` | admin | Create contest. |
| PUT | `/:id` | admin | Update contest (whitelisted fields). |
| DELETE | `/:id` | admin | **Hard delete.** |

### `GET /api/contests`
- **Query**: `status`, `page` (default 1), `limit` (default 20)
- Students only see contests with `status` in `[upcoming, live, frozen, ended]` (never `draft`) and where `scope='all'` OR their college is in `allowedColleges`.
- Response strips `problems.problemId` from the list view (problem identities not leaked before opening a contest).
- **Success (200)**: `{ data: Contest[] (with computed problemCount, durationMinutes), total, page, limit }` — **note: this endpoint does NOT use the standard `meta` pagination object; it returns flat `total`/`page`/`limit` fields instead**, inconsistent with every other paginated list endpoint in the app.

### `GET /api/contests/:id`
- Problem titles/details are replaced with `{ title: '— Hidden until start —' }` for students before `startTime`.
- If `sequentialUnlock`, computes `unlockedOrders` for the requesting student (unlocked up to and including the first unsolved problem).
- Includes `myStats` (the student's own `ContestParticipant` stats) if the requester is a student.
- **Success (200)**: `{ data: { ...contest, unlockedOrders, myStats, hasStarted, hasEnded, durationMinutes } }`
- **Errors**: `404` not found.

### `POST /api/contests/:id/join`
- **Success (200)**: `{ message: "Joined contest", data: <ContestParticipant> }` — idempotent (returns existing participant doc if already joined).
- **Errors**: `404` contest not found; `400` not started yet / already ended; `403` student's college not in `allowedColleges` for a `scope: 'college'` contest.

### `GET /api/contests/:id/leaderboard`
- **Success (200)**: `{ data: { contest: {id, title, status, isFrozen, endTime}, leaderboard } }`
- **Errors**: `404` contest not found.
- See `BACKEND.md` §"Contest scoring model" for how frozen-state leaderboards are computed.

### `GET /api/contests/:id/my-submissions`
- **Success (200)**: `{ data: ContestSubmission[] (problemId populated to title only) }`

### `POST /api/contests/:id/submit/:problemOrder` — **⚠ BROKEN**
- **Body**: `{ code, language, autoSubmitted? }`
- **Verified defect (source: `controllers/contest.controller.js` line 14 & 239, `services/judge0.service.js` line 198)**: the controller does `const { runSubmission } = require('../services/judge0.service')`, but `judge0.service.js`'s `module.exports` only contains `submitToJudge0, pollResult, runSingleTestCase, runMultipleTestCases` — there is no `runSubmission` export. `runSubmission` is `undefined`, and the call `await runSubmission({...})` throws `TypeError: runSubmission is not a function` every time this route is hit, **after** all the surrounding logic (contest lookup, sequential-unlock check, wrong-attempt count, already-solved check) has already run. **Contest submission is completely non-functional as currently wired.** This is the single most significant functional defect in the backend — see `TODO.md`.
- **Documented-as-designed behavior** (not currently reachable due to the bug above): would validate contest is live, check sequential-unlock, count wrong attempts, reject if already solved, run code via Judge0, create a `ContestSubmission`, and — on `Accepted` — update the participant's score and push a live Socket.io leaderboard update.

### `POST /api/contests` (admin)
- **Body**: `{ title, description?, startTime, endTime, problems: [{problemId, order, points}], scope?, allowedColleges?, freezeMinutes?, sequentialUnlock? }`
- Validates `endTime > startTime` and that every `problems[].problemId` exists and is active.
- Auto-sets `status` to `upcoming` (if `startTime` is in the future) or `live` (otherwise).
- **⚠ BUG (verified)**: calls `sendSuccess(res, { message: 'Contest created', data: contest }, 201)` — a third positional argument. `sendSuccess`'s actual signature is `(res, { statusCode, message, data, meta })`, so there is no third parameter; the `201` is silently discarded and the response defaults to **HTTP 200** instead of 201, inconsistent with every other "create" endpoint in the app (which correctly pass `statusCode: 201` inside the options object).
- **Response (200, not 201 — see bug above)**: `{ message: "Contest created", data: <Contest> }`
- **Errors**: `400` end time not after start time, or invalid problem IDs.

### `PUT /api/contests/:id` (admin)
- **Body**: any subset of `{title, description, startTime, endTime, problems, scope, allowedColleges, freezeMinutes, sequentialUnlock, status}` — whitelisted at the controller level, **not Joi-validated**.
- **Success (200)**: `{ message: "Contest updated", data: <Contest> }`
- **Errors**: `404` not found.

### `DELETE /api/contests/:id` (admin)
- **⚠ Hard delete** (`Contest.findByIdAndDelete`) — inconsistent with every other resource in the app (College/Problem/User are soft-deleted). Orphans any `ContestParticipant`/`ContestSubmission` history referencing this contest (no cascade, no FK constraint at the Mongo level).
- **Success (200)**: `{ message: "Contest deleted" }`
- **Errors**: `404` not found.

---

## 9. Daily Challenge — `/api/daily` (`routes/daily.routes.js`) — **undocumented in README**
Auth applied per-route (not `router.use()`, unlike other routers — functionally equivalent but an inconsistent pattern). **No Joi validation, no `submissionLimiter`** on `/submit` (Judge0-invoking route protected only by the blanket `apiLimiter`).

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/` | any | Today's challenge (IST date) + completion + streak. |
| POST | `/submit` | any | Submit code for today's problem. |
| GET | `/streak` | any | Current/longest streak + 30-day history. |
| GET | `/calendar` | any | Scheduled challenges for a month. |
| POST | `/schedule` | admin | Bulk-upsert `(date, problemId)` pairs. |
| DELETE | `/:date` | admin | Remove a scheduled day. |
| GET | `/admin/stats` | admin | Dashboard stats. |

### `GET /api/daily`
- Uses `todayStr()` — a **hardcoded IST (+5:30) offset**, not derived from `TZ` or the user's actual timezone. Every user worldwide sees the same "today."
- **Success (200)**: `{ data: { date, problem, completed, completedAt, currentStreak, longestStreak } }`
- **Errors**: `404` no challenge scheduled for today.

### `POST /api/daily/submit`
- **Body**: `{ code, language }` (manually checked for presence in the controller, not Joi)
- Judged against **sample + hidden** test cases combined (not hidden-only like `/api/submit`).
- Creates a `Submission`. On `Accepted`: creates a `DailyChallengeEntry` (unique per user/day — duplicate submit blocked with 400) and fully recalculates `currentStreak`/`longestStreak` from entry history.
- Awards a flat 10 "points" in the response — **not persisted anywhere**, purely informational in the API response.
- **Success (200)**: `{ data: { status, runtime, memory, testCasesPassed, totalTestCases, pointsAwarded, streak: {currentStreak, longestStreak} | null } }`
- **Errors**: `400` missing code/language; `404` no challenge scheduled today; `400` already completed today's challenge; `404` problem not found.

### `GET /api/daily/streak`
- **Success (200)**: `{ data: { currentStreak, longestStreak, lastChallengeDate, history: [{date, completedAt}] (last 30) } }`

### `GET /api/daily/calendar`
- **Query**: `month` (1–12, defaults to current), `year` (defaults to current)
- **⚠ BUG (verified, `controllers/dailyChallenge.controller.js` `getCalendar`)**: builds the upper date bound as `to = '${y}-${m}-31'` unconditionally — this is a **string comparison** against `'YYYY-MM-DD'` values, not real date arithmetic. It happens to work correctly today because MongoDB's lexicographic string comparison treats `'2026-02-29' < '2026-02-31'` as true even though Feb 29 doesn't exist as a real date — but it relies on that string-comparison coincidence rather than genuine date-range logic, and would break if `date` were ever changed to a real `Date` type.
- **Success (200)**: `{ data: [{ date, problem: {title, difficulty}, completed }] }` (`completed` only meaningful for students)

### `POST /api/daily/schedule` (admin)
- **Body**: `{ schedules: [{date, problemId}, ...] }`
- Validates all problem IDs exist and are active; bulk-upserts via `bulkWrite`.
- **Success (200)**: `{ message: "<n> day(s) scheduled" }`
- **Errors**: `400` missing/empty `schedules` array; `400` invalid problem IDs.

### `DELETE /api/daily/:date` (admin)
- **Success (200)**: `{ message: "Challenge removed" }`
- **Errors**: `404` nothing scheduled for that date.

### `GET /api/daily/admin/stats` (admin)
- **Success (200)**: `{ data: { totalScheduled, completedToday, topStreaks: User[] (top 10 by currentStreak) } }`

---

## 10. Health check

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/health` | Public | `{ status: "OK", timestamp, env }` — exempted from `apiLimiter` |

---

## Summary: validation coverage by route group

| Route group | Joi-validated? |
|---|---|
| Auth | ✅ (login, refresh) |
| Colleges | ✅ create only — **update is unvalidated** |
| Students | ✅ create + update |
| Problems | ✅ create + update |
| Execute/Submit | ✅ both |
| Submissions | ❌ none (read-only routes, low risk) |
| Leaderboard | ❌ none (read/admin-action routes, low risk) |
| Contests | ❌ **none at all**, including create/update which accept complex nested bodies |
| Daily Challenge | ❌ **none at all** — `/submit` does a manual presence check only |

## Summary: rate-limit coverage on Judge0-invoking routes

| Route | Rate limit |
|---|---|
| `POST /api/execute` | `submissionLimiter` (5/min/user) |
| `POST /api/submit` | `submissionLimiter` (5/min/user) |
| `POST /api/contests/:id/submit/:problemOrder` | ❌ only blanket `apiLimiter` (200/min/IP) — and currently broken regardless (see §8) |
| `POST /api/daily/submit` | ❌ only blanket `apiLimiter` (200/min/IP) |
