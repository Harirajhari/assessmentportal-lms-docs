# Database Schema — Assessment Portal

> Source of truth for data shapes. Every field below was read directly from the Mongoose model
> files in `assesmentportal-with-lms-backend-v1/models/` (verified 2026-08-16), not summarized
> from the analysis doc. Frontend should treat this file as authoritative for API payload shapes.

All models use `{ timestamps: true }` unless noted, so every document also has `createdAt`/`updatedAt`.

---

## 1. `User` (`models/User.js`)

Single collection for **both admins and students**, distinguished by `role`.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `name` | String | required, 2–100 chars, trimmed | |
| `email` | String | required, unique, lowercase, trimmed | regex-validated email format |
| `password` | String | required, min 6 chars, `select: false` | bcrypt-hashed (cost 12) via `pre('save')` hook, only when modified |
| `role` | String enum `admin`, `student` | default `student` | |
| `collegeId` | ObjectId → `College` | **required only if `role === 'student'`** (conditional function) | |
| `totalSolved` | Number | default 0, min 0 | |
| `streak` | Number | default 0, min 0 | **general** submission streak (any problem, any day) |
| `accuracy` | Number | default 0, min 0, max 100 | |
| `totalSubmissions` | Number | default 0 | |
| `lastSubmissionDate` | Date | — | |
| `currentStreak` | Number | default 0, min 0 | **daily-challenge** streak — separate from `streak` |
| `longestStreak` | Number | default 0, min 0 | |
| `lastChallengeDate` | String (`'YYYY-MM-DD'`) | default `null` | matches `DailyChallengeEntry.date` |
| `refreshToken` | String | `select: false` | single active refresh token per user |
| `isActive` | Boolean | default `true` | soft-delete/deactivation flag |
| `solvedProblems` | [ObjectId → `Problem`] | — | used to detect "first accepted" |

**Instance methods**
- `comparePassword(candidatePassword)` — bcrypt compare.
- `getLeaderboardScore()` — `totalSolved + floor(streak/7)*5 + floor(currentStreak/7)*10`. **Note**: this differs from the score actually used by `leaderboard.service.js` (see `TODO.md` / `BACKEND.md`) — this model method appears to be dead code (nothing in the controllers calls it).
- `updateAccuracy()` — increments `totalSubmissions`, recomputes `accuracy = (totalSolved/totalSubmissions)*100` (2 decimal places).
- `updateStreak()` — general submission streak: +1 if last submission was exactly 1 day ago, reset to 1 if a gap, no-op if same day.

**`toJSON` transform**: strips `password` and `refreshToken`.

**Indexes**: `collegeId` (asc), `totalSolved` (desc), `currentStreak` (desc, for streak-leaderboard queries).

---

## 2. `College` (`models/College.js`)

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `name` | String | required, unique, 2–150 chars, trimmed | |
| `code` | String | unique, uppercase, trimmed | **auto-generated** in `pre('save')` if not supplied: initials of `name` (max 8 chars) + `_` + base36 timestamp |
| `isActive` | Boolean | default `true` | |
| `studentCount` | Number | default 0 | **denormalized counter**, incremented/decremented manually by `student.controller.js` |

**Schema options**: `toJSON`/`toObject` include virtuals.

**Indexes**: text index on `name`.

---

## 3. `Problem` (`models/Problem.js`)

Sub-schemas: `testCaseSchema` (`{ input: String (custom validator: must be a string, allows empty), expectedOutput: String required, explanation: String }`, has its own `_id`), `exampleSchema` (`{ input, output required; explanation optional }`, no `_id`), `starterCodeSchema` (per-language default snippet strings for `javascript`/`python`/`java`/`cpp`/`c`, no `_id`).

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `title` | String | required, unique, max 200 chars, trimmed | |
| `slug` | String | unique, lowercase | **auto-generated** from `title` in `pre('save')` (lowercased, non-word chars stripped, spaces→hyphens) whenever `title` changes or `slug` is unset |
| `difficulty` | String enum `Easy`, `Medium`, `Hard` | required | |
| `tags` | [String] | trimmed, lowercased | |
| `description` | String | required | |
| `constraints` | String | required | |
| `examples` | [`exampleSchema`] | custom validator: min length 1 | |
| `testCases.sample` | [`testCaseSchema`] | custom validator: min length 1 | **publicly returned** |
| `testCases.hidden` | [`testCaseSchema`] | custom validator: min length 1, **`select: false`** | never returned unless a query explicitly does `.select('+testCases.hidden')` |
| `starterCode` | `starterCodeSchema` | — | |
| `isActive` | Boolean | default `true` | soft-delete flag |
| `acceptanceRate` | Number | default 0 | denormalized, recomputed by `updateAcceptanceRate()` |
| `totalSubmissions` | Number | default 0 | denormalized, updated by `execution.controller.js` |
| `totalAccepted` | Number | default 0 | denormalized, updated by `execution.controller.js` |
| `createdBy` | ObjectId → `User` | required | |
| `timeLimit` | Number (ms) | default 2000 | |
| `memoryLimit` | Number (MB) | default 256 | |

**Instance methods**: `updateAcceptanceRate()` — sets `acceptanceRate = (totalAccepted/totalSubmissions)*100` (2 decimals) when `totalSubmissions > 0`.

**Schema options**: `toJSON` includes virtuals.

**Indexes**: text index on `title`, `tags` (asc), compound `difficulty + isActive`.

---

## 4. `Submission` (`models/Submission.js`)

One row per `/api/execute` or `/api/submit` call that reaches the "submit" path (execute does not persist).

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `userId` | ObjectId → `User` | required | |
| `collegeId` | ObjectId → `College` | required | |
| `problemId` | ObjectId → `Problem` | required | |
| `code` | String | required, max 65536 chars (64KB) | |
| `language` | String enum | required | `javascript, python, java, cpp, c, typescript, go, rust, ruby, csharp` (10 languages) |
| `status` | String enum | default `Pending` | `Accepted, Wrong Answer, Time Limit Exceeded, Runtime Error, Compilation Error, Memory Limit Exceeded, Pending, Internal Error` |
| `runtime` | Number (ms) | default `null` | |
| `memory` | Number (KB) | default `null` | |
| `stdout` | String | default `''` | |
| `stderr` | String | default `''` | |
| `compileOutput` | String | default `''` | |
| `testCasesPassed` | Number | default 0 | |
| `totalTestCases` | Number | default 0 | |
| `judgeToken` | String | `select: false` | Judge0 polling token |
| `isFirstAccepted` | Boolean | default `false` | |

**`toJSON` transform**: strips `judgeToken`.

**Indexes**: `userId + problemId`, `collegeId + createdAt desc`, `status`, `problemId + status`.

---

## 5. Contest module

### 5.1 `Contest` (`models/Contest.js`)

Embedded sub-schema `contestProblemSchema` (no `_id`): `{ problemId: ObjectId→Problem required, order: Number required, points: Number default 100 }`.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `title` | String | required, max 200 chars, trimmed | |
| `description` | String | default `''` | |
| `scope` | String enum `all`, `college` | default `all` | |
| `allowedColleges` | [ObjectId → `College`] | — | used only when `scope === 'college'` |
| `problems` | [`contestProblemSchema`] | custom validator: min length 1 | |
| `startTime` | Date | required | |
| `endTime` | Date | required | |
| `freezeMinutes` | Number | default 0 | leaderboard freeze window before end; `0` = never freeze |
| `status` | String enum `draft`, `upcoming`, `live`, `frozen`, `ended` | default `draft` | |
| `sequentialUnlock` | Boolean | default `false` | must solve problem N before N+1 is visible |
| `createdBy` | ObjectId → `User` | required | |
| `isActive` | Boolean | default `true` | **Note**: this flag exists but delete is actually hard, not soft — see below |

**Virtual `isFrozen`**: `true` if `freezeMinutes > 0` and current time is within `[endTime - freezeMinutes, endTime)`.

**Instance method `computeStatus()`**: derives live status from current time vs. `startTime`/`endTime`/`freezeMinutes` (`upcoming` → `live` → `frozen` (if applicable) → `ended`).

**Indexes**: `startTime`, `status`, `allowedColleges`.

**⚠ Deletion is a hard delete** (`Contest.findByIdAndDelete` in the controller) despite the schema having an `isActive` flag — every other resource (College, Problem, User) is soft-deleted. See `TODO.md`.

### 5.2 `ContestParticipant` (`models/ContestParticipant.js`)

One doc per `(contestId, userId)`.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `contestId` | ObjectId → `Contest` | required | |
| `userId` | ObjectId → `User` | required | |
| `collegeId` | ObjectId → `College` | required | |
| `totalPoints` | Number | default 0 | live score = sum of `pointsAwarded` for solved problems |
| `penaltyMinutes` | Number | default 0 | ICPC-style: wrong attempts × 20 min each + solve-time minutes |
| `solvedOrders` | [Number] | — | e.g. `[1, 3]` |
| `problemStats` | Mongoose `Map` of sub-schema, keyed by problem order (string) | default `{}` | sub-schema: `{ solved: Boolean default false, attempts: Number default 0, solveTimeSeconds: Number default null, pointsAwarded: Number default 0 }` |
| `joinedAt` | Date | default `Date.now` | |

**Indexes**: unique compound `(contestId, userId)`; compound `(contestId, totalPoints desc, penaltyMinutes asc)` for leaderboard sort.

### 5.3 `ContestSubmission` (`models/ContestSubmission.js`)

One doc per contest submission attempt.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `contestId` | ObjectId → `Contest` | required | |
| `userId` | ObjectId → `User` | required | |
| `collegeId` | ObjectId → `College` | required | |
| `problemId` | ObjectId → `Problem` | required | |
| `problemOrder` | Number | required | which problem (1, 2, 3...) within the contest |
| `code` | String | required, max 65536 chars | |
| `language` | String enum | required | **`javascript, python, java, cpp, c` — only 5 languages**, narrower than `Submission.language`'s 10. A submission in `typescript`/`go`/`rust`/`ruby`/`csharp` fails Mongoose validation here — see `TODO.md`. |
| `status` | String enum | default `Pending` | same 8-value enum as `Submission.status` |
| `pointsAwarded` | Number | default 0 | only set when `Accepted` |
| `wrongAttemptsBefore` | Number | default 0 | penalty input |
| `solveTimeSeconds` | Number | default `null` | time from contest start; tiebreaker input |
| `runtime` | Number | default `null` | |
| `memory` | Number | default `null` | |
| `testCasesPassed` | Number | default 0 | |
| `totalTestCases` | Number | default 0 | |
| `autoSubmitted` | Boolean | default `false` | true if submitted automatically on time expiry |
| `judgeToken` | String | `select: false` | |

**`toJSON` transform**: strips `judgeToken`.

**Indexes**: `(contestId, userId)`, `(contestId, problemId)`, `(contestId, userId, problemId)`, `(contestId, status)`.

---

## 6. Daily Challenge module

### 6.1 `DailyChallenge` (`models/DailyChallenge.model.js`)

One doc per calendar day.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `date` | String (`'YYYY-MM-DD'`) | required, unique | timezone-naive string key |
| `problemId` | ObjectId → `Problem` | required | |
| `scheduledBy` | ObjectId → `User` | required | admin who scheduled it |

**Indexes**: `date`.

### 6.2 `DailyChallengeEntry` (`models/DailyChallengeEntry.model.js`)

One doc per `(user, day)` they *completed*.

| Field | Type | Required / Default | Notes |
|---|---|---|---|
| `userId` | ObjectId → `User` | required | |
| `date` | String (`'YYYY-MM-DD'`) | required | |
| `problemId` | ObjectId → `Problem` | required | |
| `submissionId` | ObjectId → `Submission` | required | |
| `completedAt` | Date | default `Date.now` | |

**Indexes**: unique compound `(userId, date)` — one completion per user per day; `date` (for calendar/admin queries).

---

## 7. Entity-relationship summary

```
College 1───* User (role=student)
User 1───* Submission ───* Problem
User 1───* ContestParticipant ───1 Contest
User 1───* ContestSubmission ───1 Contest, ───1 Problem
Contest 1───* (embedded) problems[] ───ref── Problem
DailyChallenge 1───1 Problem   (per calendar date)
User 1───* DailyChallengeEntry ───1 DailyChallenge's Problem, ───1 Submission
```

## 8. Cross-model gotchas frontend should know about

- **Hidden fields**: `User.password`, `User.refreshToken`, `Problem.testCases.hidden`, `Submission.judgeToken`, `ContestSubmission.judgeToken` are all `select: false` — none of these will ever appear in a normal API response.
- **Two disagreeing leaderboard-score formulas** exist (`User.getLeaderboardScore()` vs. `leaderboard.service.js: calculateScore()`) — see `BACKEND.md` and `TODO.md`. The service function's formula (no daily-streak bonus) is what actually powers the leaderboard endpoints.
- **`ContestSubmission.language` enum (5 languages) is narrower than `Submission.language` (10 languages)** — a contest UI that lets a student pick from all 10 languages will hit a server-side validation error for the extra 5 inside a contest.
- **Contest is hard-deleted**; a deleted contest's `ContestParticipant`/`ContestSubmission` documents are orphaned (no cascade), so historical data referencing a deleted `contestId` will not resolve.
