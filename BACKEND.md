# Backend — Assessment Portal

> Verified against `assesmentportal-with-lms-backend-v1` on GitHub (2026-08-16). Repo structure, file
> names, and behavior described below were spot-checked directly against source (`server.js`,
> `services/judge0.service.js`, `controllers/contest.controller.js`, `controllers/submission.controller.js`,
> `utils/response.js`, `middlewares/auth.middleware.js`, `models/ContestSubmission.js`, `models/Submission.js`,
> `config/judge0.js`, `routes/*.js`).

## Tech stack & dependencies

| Concern | Library | Notes |
|---|---|---|
| HTTP framework | `express` 4.18 | |
| DB / ODM | `mongoose` 8.0 + MongoDB | source of truth for all entities |
| Cache / leaderboard store | `ioredis` 5.3 | sorted-set (`ZADD`/`ZREVRANGE`) leaderboard cache — **declared but not connected at runtime**, see "Known runtime gaps" below |
| Realtime | `socket.io` 4.8 | contest live-leaderboard push |
| Auth | `jsonwebtoken` 9.0, `bcryptjs` 2.4 | JWT access+refresh, bcrypt password hashing (cost 12) |
| Validation | `joi` 17.11 | request-body schema validation (partial coverage — see `TODO.md`) |
| Code execution | `axios` 1.6 → **Judge0 CE** (RapidAPI-style API) | polling-based async execution |
| Security middleware | `helmet`, `cors`, `express-rate-limit` | |
| Logging | `winston` 3.11 (+ `morgan` for HTTP access logs) | JSON logs, console in dev, file transport in prod |
| Env config | `dotenv` | |
| IDs | `uuid` (declared, not observed in use) | |
| Dev | `nodemon` | `npm run dev` |

Node.js >= 18 is required (per README). `package.json` name: `coding-platform-backend`.

## Project layout

```
assesmentportal-with-lms-backend-v1/
├── server.js                     # App bootstrap: middleware, routes, http+socket.io server
├── socket.js                     # Socket.io init: auth handshake, contest rooms, 30s status-sync loop
├── temp.js                       # Standalone dev script — scans local requires for broken paths (not wired into app)
├── config/
│   ├── db.js                     # Mongoose connection + reconnect logging
│   ├── redis.js                  # ioredis client factory + getRedis() singleton accessor
│   ├── logger.js                 # Winston logger (console + file-in-prod)
│   └── judge0.js                 # Judge0 language-id map, status-code map, polling constants
├── models/                       # Mongoose schemas — see DATABASE_SCHEMA.md
├── services/                     # Business logic / integration layer
│   ├── auth.service.js           # JWT sign/verify helpers
│   ├── judge0.service.js         # Judge0 HTTP client + polling + status classification
│   ├── timerHarness.js           # Wraps user code to measure pure algorithm runtime
│   ├── leaderboard.service.js    # Redis ZSET leaderboard (per-college + overall) w/ Mongo fallback
│   └── contest.service.js        # Contest scoring (ICPC penalty model), leaderboard, status sync
├── controllers/                  # Route handlers (thin-ish, call services/models directly)
│   ├── auth.controller.js
│   ├── college.controller.js
│   ├── student.controller.js
│   ├── problem.controller.js
│   ├── execution.controller.js
│   ├── submission.controller.js
│   ├── leaderboard.controller.js
│   ├── contest.controller.js
│   └── dailyChallenge.controller.js
├── routes/                       # Express routers, one per resource — see API_CONTRACT.md
├── middlewares/
│   ├── auth.middleware.js        # authenticate, authorize(...roles), collegeScope
│   ├── validate.js               # Joi-schema validation middleware factory
│   ├── rateLimiter.js            # apiLimiter, submissionLimiter, authLimiter
│   └── errorHandler.js           # Centralized Express error handler
├── utils/
│   ├── AppError.js               # Operational error class (statusCode, isOperational)
│   ├── asyncHandler.js           # try/catch wrapper for async route handlers
│   ├── response.js               # sendSuccess/sendError/buildPaginationMeta helpers
│   ├── validators.js             # All Joi schemas
│   └── seed.js                   # `npm run seed` — wipes & reseeds admin/colleges/students/problems
├── .env / .env.example
├── package.json
└── README.md                     # Pre-existing README — covers only the pre-contest/daily-challenge surface
```

**Note:** README.md documents only the original surface (auth, colleges, students, problems, execute/submit, submissions, leaderboard). The **Contest** and **Daily Challenge** modules, and the **overall (cross-college) leaderboard**, exist in code but are undocumented in the README.

## Request lifecycle (`server.js`)

```
dotenv.config()
  → connectDB()                         # Mongo connects immediately
  → connectRedis()                      # ⚠ CALLED-OUT BUT COMMENTED OUT IN SOURCE — verified: `//connectRedis();`
  → helmet(), cors(), json/urlencoded parsers, morgan→winston
  → app.use('/api', apiLimiter)         # global rate limit (200 req / 60s / IP)
  → GET /health                         # liveness probe, skips rate limiter
  → mount routers (see API_CONTRACT.md)
  → 404 handler
  → errorHandler (global)
  → server.listen(PORT)
      → initSocket(server, app)         # Socket.io attached AFTER listen; io stored on app via app.set('io', io)
```

CORS origin = `process.env.CLIENT_URL` or `*`. Body size limit 10mb (large enough for source-code payloads; code itself is capped at 64KB by schema/model).

## Core business logic / workflows

### Judge0 code execution pipeline (`services/judge0.service.js` + `timerHarness.js`)

```
runMultipleTestCases(code, language, testCases[])
  for each test case (SEQUENTIAL, not parallel):
    wrapWithTimer(code, language)             # injects a timing harness, falls back
                                               # to the raw code if the language has no template
    submitToJudge0({source, stdin, expected_output, cpu_time_limit, memory_limit})
      → POST {JUDGE0_API_URL}/submissions?base64_encoded=true&wait=false
      → returns a `token`
    pollResult(token)
      → GET {JUDGE0_API_URL}/submissions/{token}?...  every 1s, up to 20 attempts (20s worst case)
      → stops polling once status is no longer IN_QUEUE/PROCESSING
    parse __exec_time__ marker out of stderr → "algorithm-only" runtime (excludes interpreter/JVM startup)
    strip the marker from stderr before returning to caller
    passed = (Judge0 status === Accepted) OR (stdout trims-equal to expectedOutput AND status === WrongAnswer)
       # the second clause is a safety net for cases where Judge0's own string diff is too strict
    if a case Compilation-Errors → stop early (no point running remaining cases)
  aggregate: allPassed, passedCount, avgRuntime (mean of case runtimes), maxMemory, overallStatus
    overallStatus priority: Accepted > Compilation Error > Time Limit Exceeded > Runtime Error > Internal Error > Wrong Answer
```

**Timer harness** (`timerHarness.js`): per-language source-rewriting templates for `java`, `python`, `javascript`, `cpp`, `c`, `go`, `rust`. Wraps the user's code so the *algorithm's own* execution time is printed to stderr as `__exec_time__:<ms>`, isolating it from interpreter/JVM/compiler startup overhead. If wrapping throws (e.g. code doesn't match the expected `class Main`/`func main()` shape the regex looks for), it silently falls back to the unwrapped code and the raw Judge0 wall-clock time is used instead.

Supported languages (10, for practice-mode execute/submit — **verified in `config/judge0.js`**): `javascript, python, java, cpp, c, typescript, go, rust, ruby, csharp`.

### Submit-and-grade flow (`execution.controller.js: submit`)

1. Fetch problem **with** hidden test cases.
2. Create `Submission` doc as `Pending`.
3. Run all hidden cases through Judge0 (sequential).
4. Update the submission with final status/runtime/memory/first-case stdout-stderr.
5. Increment `Problem.totalSubmissions` (+`totalAccepted` if accepted).
6. If accepted:
   - Re-fetch a **fresh** `User` doc (avoids stale in-memory `req.user`).
   - First-solve check via `solvedProblems` array membership → only then increments `totalSolved`/marks `isFirstAccepted`.
   - Always calls `updateStreak()` (general streak) and `updateAccuracy()` regardless of first-solve.
   - Syncs the Redis leaderboard (`syncUserLeaderboard`) — see note below on why this doesn't actually hit Redis.
7. If not accepted: still calls `updateAccuracy()` (counts the attempt against accuracy).
8. Response hides all failing-case detail except the first failure (keeps other hidden tests secret even on failure).

### Leaderboard model (`services/leaderboard.service.js`)

- **Storage (as designed)**: Redis sorted sets, key `leaderboard:{collegeId}` per college and `leaderboard:overall` for the cross-college board. Member = `userId` string, score = computed number.
- **Score formula (as actually implemented here)**: `totalSolved + floor(streak/7) * 5` — this **omits** the daily-challenge streak bonus that `User.getLeaderboardScore()` (a model method) includes. The two code paths disagree — see `TODO.md`.
- **Cache-aside pattern**: every read checks `EXISTS`; on miss, rebuilds the full ZSET from Mongo (`buildCollegeLeaderboard`/`buildOverallLeaderboard`) with a 300s TTL (`EXPIRE`) set on every write, i.e. leaderboards self-expire and lazily rebuild.
- **Write path**: `syncUserLeaderboard(user)` is called after every accepted submission; it upserts into both the per-college and (only if already cached) the overall ZSET.
- **Fallback (what actually runs today)**: every read function wraps Redis calls in try/catch and falls back to an equivalent MongoDB sort/skip/limit query (`getCollegeLeaderboardFromDB`/`getOverallLeaderboardFromDB`) if Redis throws. Since Redis is never connected at boot (`connectRedis()` is commented out in `server.js` — verified), **this fallback path is what runs 100% of the time in production today.**
- Admin "get all leaderboards" (`GET /api/leaderboard` as admin) fetches top-5 per college via `Promise.all` — O(colleges) round-trips, fine at small scale.

### Contest scoring model (`services/contest.service.js`) — ICPC-style

- **Points**: awarded once, only on a problem's *first* `Accepted` submission by that participant (`stats.solved` guard prevents double-award on later accepted re-submits).
- **Penalty**: `penaltyMinutes += floor(solveTimeSeconds/60) + (wrongAttemptsBefore * 20)`. Wrong attempts before the accepted one are counted from `ContestSubmission` history (`status !== 'Accepted'` count).
- **Ranking**: sort by `totalPoints DESC`, tie-break `penaltyMinutes ASC`.
- **Freeze window**: if `contest.status === 'frozen'`, the leaderboard is recomputed **on the fly** using only `ContestSubmission`s with `createdAt < freezeAt` — it re-derives frozen state from raw submission history every call rather than reading a snapshot. Correct, but "frozen" reads are heavier than "live" reads (which read the pre-aggregated `ContestParticipant` doc).
- **Status transitions**: `syncContestStatuses()` runs every 30s from `socket.js`'s `setInterval` (not a real cron — depends on the Node process staying up) and re-derives `draft→upcoming→live→frozen→ended` per-contest via `contest.computeStatus()`, persisting on change.
- **Sequential unlock**: enforced both for *visibility* (`getContest`, computes `unlockedOrders`) and for *submission* (`submitToContest` blocks submitting to problem N unless N-1 is in the participant's `solvedOrders`).
- **Live leaderboard push**: after every accepted contest submission, `emitLeaderboard(io, contestId)` broadcasts to the `contest:{id}` Socket.io room. Clients also get an immediate snapshot on `contest:join`.
- **Important**: contest submission is currently non-functional at the code level — see `TODO.md` and `API_CONTRACT.md` for `POST /api/contests/:id/submit/:problemOrder`.

### Daily Challenge streak logic (`dailyChallenge.controller.js`)

- Date bucketing uses a **hardcoded IST offset** (`+5.5h`) baked into `todayStr()`, not derived from the college's/user's actual timezone or a `TZ` env var — every user worldwide sees the same "today" in IST.
- On each `Accepted` submit, `recalcStreak()` **fully rescans** all of the user's `DailyChallengeEntry` history (sorted desc) rather than incrementally updating a counter — O(n) per submit but always self-correcting/consistent, and immune to drift from missed-update bugs.
- `currentStreak`: walks back from today, day-by-day, counting a `1` for every consecutive completed day; stops at the first gap.
- `longestStreak`: separately scans the whole (chronological) list for the longest consecutive run, and is stored as `max(longest-run-found, currentStreak)`.
- This streak state is **independent of** the general `User.streak` field used by the practice-mode leaderboard — a user can have a 10-day daily-challenge streak and a 0-day general streak simultaneously.

### Socket.io realtime layer (`socket.js`)

- Auth middleware on the `io` server verifies the JWT access token from `socket.handshake.auth.token` before allowing connection; unauthenticated sockets are rejected at handshake.
- Room-based fanout: clients `contest:join`/`contest:leave` a `contest:{contestId}` room; server pushes `leaderboard:update` events to the room.
- A **process-level `setInterval` (30s)** drives contest status transitions — a single-process, in-memory scheduler with no persistence/locking. If horizontally scaled to multiple Node processes, each instance runs its own independent 30s loop (redundant but not harmful, since `computeStatus()` is idempotent — more a wasted-work concern than a correctness bug).

## Cross-cutting patterns & conventions

- **Controller shape**: every handler wrapped in `asyncHandler(fn)` so thrown/rejected errors flow to Express's `next()` and the central `errorHandler` — no manual try/catch boilerplate in controllers.
- **Errors**: `AppError(message, statusCode)` for all "operational" (expected) errors; the global `errorHandler` translates Mongoose `CastError`/`ValidationError`/duplicate-key (`11000`) and JWT errors into the same 40x envelope in production, and dumps full stack+error object in development (`NODE_ENV=development`).
- **Responses**: `sendSuccess(res, {statusCode, message, data, meta})` standardizes the envelope; `buildPaginationMeta(page, limit, total)` standardizes pagination metadata (`currentPage/totalPages/totalItems/itemsPerPage/hasNextPage/hasPrevPage`). Note: page/limit query params arrive as strings and are `Number(...)`-coerced ad hoc in most controllers rather than centrally in Joi (Joi *does* coerce when `validate()` middleware is applied, but several list routes — colleges, students(update), contests, daily — don't run their query through `validate(paginationSchema)` at all).
- **Soft delete** is the norm for College/Student/Problem (`isActive=false`); **Contest is the outlier** (hard `findByIdAndDelete` — verified in `contest.controller.js`).
- **Hidden-data protection**: Mongoose `select:false` is used consistently for `password`, `refreshToken`, `judgeToken`, and `testCases.hidden` — a deliberate, systematic pattern to keep these out of default query results without needing per-controller `.select()` calls everywhere (contrasted with `toJSON` transforms, used for the same purpose on `User`/`Submission`/`ContestSubmission`).
- **Validation coverage is inconsistent**: Joi (`utils/validators.js` + `middlewares/validate.js`) is applied to auth, college-create, student-create/update, problem-create/update, execute/submit — but **not** to college-update, contest-*, or daily-* routes, which trust `req.body` directly.
- **Rate limiting is inconsistent**: `submissionLimiter` (5/min) protects `/api/execute` and `/api/submit`, but the functionally-equivalent `/api/contests/:id/submit/:problemOrder` and `/api/daily/submit` (both of which also invoke Judge0) have no per-user submission throttle beyond the blanket 200/min `apiLimiter`.
- **Logging**: Winston JSON-structured logs; `morgan('combined')` piped into Winston for HTTP access logs; `logger.debug` used liberally in hot paths (polling, socket events) and only surfaces in non-production (`level: 'debug'` unless `NODE_ENV=production`, which raises the floor to `'warn'`).
- **Pagination**: uniform `page`/`limit` skip/limit pattern across all list endpoints, default `page=1, limit=20` (contests default `limit=20`; leaderboards default `limit=50`).
- **`temp.js`** at the project root is a standalone, ad-hoc dev utility (not required by `server.js`, not an npm script) that walks the source tree and flags `require('./relative/path')` statements that don't resolve to a real file — useful as a lint check but currently dead weight in the shipped repo.

## Known runtime gaps (see `TODO.md` for full list)

- **Redis is never connected** — `connectRedis()` is commented out in `server.js`. The app works, but purely off MongoDB fallback queries; Redis provides zero benefit today.
- **Contest submission is broken** — `contest.controller.js` imports `runSubmission` from `services/judge0.service.js`, which does not export a function by that name (verified: the module's `module.exports` only includes `submitToJudge0, pollResult, runSingleTestCase, runMultipleTestCases`). `POST /api/contests/:id/submit/:problemOrder` throws `runSubmission is not a function` at call time.

## Environment Variables (`.env.example`)

| Variable | Purpose | Default in example |
|---|---|---|
| `PORT` | HTTP port | 5000 |
| `NODE_ENV` | `development`/`production` — gates error verbosity, log level, log file transport | development |
| `MONGO_URI` | Mongo connection string | `mongodb://localhost:27017/coding_platform` |
| `JWT_SECRET` / `JWT_REFRESH_SECRET` | Signing secrets — **must differ** | placeholders |
| `JWT_EXPIRES_IN` / `JWT_REFRESH_EXPIRES_IN` | Token lifetimes | 15m / 7d |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` (unused at runtime — Redis is never connected, see above) |
| `JUDGE0_API_URL` / `JUDGE0_API_KEY` / `JUDGE0_HOST` | Judge0 CE | `JUDGE0_API_KEY`/`JUDGE0_HOST` are declared but not actually read anywhere in `judge0.service.js` — only `JUDGE0_API_URL` is used, and no RapidAPI auth headers (`X-RapidAPI-Key`/`X-RapidAPI-Host`) are ever sent; `getJudge0Headers()` only sets `Content-Type`. This means the code currently assumes a **self-hosted / non-RapidAPI-proxied** Judge0 instance at `JUDGE0_API_URL`, not the RapidAPI product described in the README/.env comments |
| `RATE_LIMIT_WINDOW_MS` / `RATE_LIMIT_MAX_SUBMISSIONS` | Submission limiter tuning | 60000 / 5 |

An actual `.env` file (with real values) is present in the project root but was not read into this document; it is excluded from version control via `.gitignore`.

## Setup / operational notes

```bash
npm install
cp .env.example .env        # then fill in real secrets/URIs
npm run seed                 # wipes Users/Colleges/Problems, creates 1 admin, 2 colleges, 4 students, 3 problems
npm run dev                  # nodemon, or `npm start` for production
```
Health check: `GET /health` → `{ status, timestamp, env }`, exempted from the global API rate limiter.

Seeded credentials (from `utils/seed.js`): `admin@platform.com` / `Admin@123`; students `arjun@iitm.ac.in`, `priya@iitm.ac.in`, `vikram@nitt.ac.in`, `sneha@nitt.ac.in`, all `Student@123`. **Note**: the seed script does not create any Contest or DailyChallenge documents, so those modules will start empty even after seeding.
