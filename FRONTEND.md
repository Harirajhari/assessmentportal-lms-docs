# FRONTEND.md — CodeArena Frontend

> Source: Phase 1 analysis (`CodeArena-Frontend-Architecture.md`), read-only. No application code
> was modified in producing this document.
> Project: `coding-platform` (displayed as **CodeArena**), repo
> `assesmentportal-with-lms-frontend` — competitive-programming / LMS-style platform frontend,
> paired with a separate backend API (see `BACKEND.md`, `API_CONTRACT.md`).

---

## 1. Tech Stack & Dependencies

| Category | Library | Version | Purpose |
|---|---|---|---|
| Framework | React | ^18.2.0 | UI library |
| Build tool | Vite | ^5.1.3 | Dev server + bundler |
| Routing | react-router-dom | ^6.22.2 | Client-side routing |
| State | @reduxjs/toolkit | ^2.2.1 | Global state (slices + thunks) |
| State bindings | react-redux | ^9.1.0 | React ↔ Redux glue |
| HTTP | axios | ^1.6.7 | REST API calls |
| Realtime | socket.io-client | ^4.8.3 | Live contest leaderboard updates |
| Code editor | @monaco-editor/react | ^4.6.0 | In-browser code editing (VS Code engine) |
| Charts | recharts | ^2.12.2 | Admin/leaderboard visualizations |
| Icons | lucide-react | ^0.383.0 | Icon set |
| Styling | tailwindcss | ^3.4.1 | Utility-first CSS |
| CSS processing | postcss, autoprefixer | ^8.4.35 / ^10.4.17 | Tailwind pipeline |

No testing framework, linter config, TypeScript, or CI config is present. There is no
`src/setupTests` or `*.test.*` file — **zero automated tests currently exist.**

---

## 2. Project Structure

```
assesmentportal-with-lms-frontend/
├── index.html                  Vite entry HTML (loads Google Fonts, mounts #root)
├── vite.config.js              @ alias → src/, dev proxy /api → localhost:5000
├── tailwind.config.js          Custom color system (primary/success/warning/danger/surface), fonts, shadows, animations
├── postcss.config.js
├── package.json
├── .env / .env.example         VITE_API_URL=http://localhost:5000/api
└── src/
    ├── main.jsx                 ReactDOM root; wraps <App/> in Redux <Provider>
    ├── App.jsx                  All route definitions (react-router-dom v6)
    ├── index.css                Tailwind layers + shared component classes (.btn, .card, .badge, .input…)
    ├── app/
    │   └── store.js              configureStore() — combines 7 slices
    ├── features/                 Redux Toolkit slices (one folder per domain)
    │   ├── auth/authSlice.js
    │   ├── problems/problemSlice.js
    │   ├── submissions/submissionSlice.js
    │   ├── leaderboard/leaderboardSlice.js
    │   ├── ui/uiSlice.js
    │   ├── contest/contestSlice.js, useContestSocket.js, Usecountdown.js
    │   └── daily/dailySlice.js
    ├── services/                 Axios call wrappers, one file per REST resource
    │   ├── api.js                 Axios instance + interceptors (auth header, 401 handling)
    │   ├── authService.js
    │   ├── problemService.js
    │   ├── submissionService.js   (execution + submission history)
    │   ├── collegeService.js      (colleges + students + leaderboard)
    │   └── Contestservice.js      (note: capital "C" — see TODO.md #6)
    ├── hooks/
    │   └── index.js               useAuth(), useNotify(), useAppSelector
    ├── components/
    │   ├── layout/                AppLayout, Sidebar, Topbar (unused), ProtectedRoute, AdminRoute
    │   ├── editor/                CodeEditor (Monaco), CodingWorkspace, OutputPanel, ProblemPanel (unused duplicate)
    │   ├── ui/                    Shared primitives (Spinner, Modal, DataTable, PageHeader, badges…) + NotificationCenter
    │   └── widgets/                DailyStreakWidget
    ├── pages/                     One component per route (student-facing)
    │   ├── admin/                 6 admin CRUD/dashboard pages + contest/daily admin pages
    │   └── contest/                3 contest-related pages
    └── utils/
        ├── helpers.js              Formatting, verdict/difficulty maps, ALL_TAGS, debounce, cn()
        └── editorUtils.js          LANGUAGES, TEMPLATES, localStorage code persistence (loadCode/saveCode/clearCode)
```

---

## 3. Application Bootstrap

`main.jsx` renders `<App/>` inside `<React.StrictMode>` and a Redux `<Provider store={store}>`.
`App.jsx` owns **all routing** — there is no nested router file; every route is declared directly
in one `<Routes>` tree using react-router-dom v6's element-based nesting.

### Route Map

| Path | Page | Guard | Layout |
|---|---|---|---|
| `/login` | `LoginPage` | none (public) | none |
| `/` | `HomePage` | `ProtectedRoute` | `AppLayout` |
| `/practice` | `PracticePage` | `ProtectedRoute` | `AppLayout` |
| `/problem/:id` | `ProblemSolvePage` | `ProtectedRoute` | `AppLayout` (sidebar hidden) |
| `/submissions` | `SubmissionsPage` | `ProtectedRoute` | `AppLayout` |
| `/leaderboard` | `LeaderboardPage` | `ProtectedRoute` | `AppLayout` |
| `/profile` | `ProfilePage` | `ProtectedRoute` | `AppLayout` |
| `/contests` | `ContestListPage` | `ProtectedRoute` | `AppLayout` |
| `/contests/:id` | `ContestArenaPage` | `ProtectedRoute` | `AppLayout` (sidebar hidden) |
| `/contests/:id/leaderboard` | `ContestLeaderboardPage` | `ProtectedRoute` | `AppLayout` |
| `/daily` | `DailyChallengePage` | `ProtectedRoute` | `AppLayout` (sidebar hidden) |
| `/admin` | `AdminDashboard` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/problems` | `AdminProblems` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/students` | `AdminStudents` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/colleges` | `AdminColleges` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/submissions` | `AdminSubmissions` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/leaderboard` | `AdminLeaderboard` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/contests` | `AdminContestListPage` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/contests/new`, `/admin/contests/:id/edit` | `AdminContestFormPage` | `AdminRoute` | `AppLayout isAdmin` |
| `/admin/daily` | `AdminDailySchedulePage` | `AdminRoute` | `AppLayout isAdmin` |
| `*` | redirect → `/` | — | — |

Guards (`components/layout/`):
- **`ProtectedRoute`** — reads `useAuth().isLoggedIn`; redirects to `/login` if false. Renders `<Outlet/>` otherwise.
- **`AdminRoute`** — redirects to `/login` if not logged in, or to `/` if logged in but `role !== 'admin'`.

`AppLayout` renders the `Sidebar` (unless the current path matches one of the "full-screen" regex
patterns for `/problem/:id`, `/contests/:id`, `/daily`) and an `<Outlet/>` for the page content.

> Frontend-side auth mechanics (token storage, interceptors, role model) have moved to
> `AUTHENTICATION.md`.

---

## 4. State Management (Redux Toolkit)

Store is assembled in `src/app/store.js` from 7 slices, all using `createSlice` +
`createAsyncThunk`. There is no RTK Query — all data fetching is done manually via thunks calling
the `services/*` Axios wrappers.

### Common conventions used across every slice
- Thunks return `res.data.data ?? res.data` — tolerant of both `{ success, data }` and raw-array
  backend responses.
- Errors are captured via `rejectWithValue(err.response?.data?.message || '<fallback text>')`.
- Every slice keeps its own `loading` / `error` booleans; there's no shared "request status" middleware.

| Slice | Key state | Thunks | Notes |
|---|---|---|---|
| `auth` | `user, token, role, loading, error` | `login`, `fetchMe`, `logoutUser` | Also exposes sync reducers `clearError`, `patchUser` |
| `problems` | `list, selected, total, pages, filters, page` | `fetchProblems`, `fetchProblem`, `createProblem`, `updateProblem`, `deleteProblem` | Client-side pagination/filter state kept in the slice, not the URL |
| `submissions` | `list, total, runResult, submitResult, running, submitting, consoleTab` | `runCode`, `submitCode`, `fetchSubmissions` | Normalizes verdict field across possible backend shapes (`verdict`/`overallStatus`/`status`) |
| `leaderboard` | `entries, myRank, overallEntries, myOverallRank` | `fetchLeaderboard`, `fetchCollegeLeaderboard`, `fetchOverallLeaderboard` | College-scoped vs. platform-wide leaderboards are separate state trees |
| `ui` | `sidebarOpen, notifications[], editorPrefs` | *(none — pure sync slice)* | Powers `NotificationCenter` toast system and Monaco editor prefs (font size, tab size, minimap) |
| `contest` | `list, current, leaderboard, lastResult, submitting` | `fetchContests`, `fetchContest`, `joinContest`, `submitContestSolution`, `fetchContestLeaderboard` | `setLiveLeaderboard` reducer is fed by the Socket.io listener (see §5) |
| `daily` | `today, streak, calendar, result` | `fetchDailyChallenge`, `submitDailyChallenge`, `fetchDailyStreak`, `fetchDailyCalendar` | Calls `services/api.js` directly rather than a dedicated `dailyService.js` file (inconsistent with other slices) |

`hooks/index.js` wraps common selector patterns:
- `useAuth()` → `{ user, token, role, loading, error, isAdmin, isStudent, isLoggedIn, logout }`
- `useNotify()` → `{ success, error, info, warn }` dispatching `notify()` into the `ui` slice
- `useAppSelector` → plain re-export of `useSelector`

---

## 5. Realtime (Socket.io)

`features/contest/useContestSocket.js` manages a **singleton** socket connection (module-level
`socketInstance`, lazily created via `getSocket(token)`), connecting to
`VITE_API_URL` with `/api` stripped off.

Flow inside `useContestSocket(contestId)`:
1. On mount, reads a JWT from `localStorage` (see `AUTHENTICATION.md` and `TODO.md` #3 for the
   known key mismatch) and calls `socket.connect()` if not already connected.
2. Emits `contest:join` with `{ contestId }`.
3. Listens for `leaderboard:update` events; when the payload's `contestId` matches, dispatches
   `setLiveLeaderboard(data)` into the `contest` slice.
4. On unmount, emits `contest:leave` and removes the `leaderboard:update` listener (but does **not**
   disconnect the underlying socket — it stays alive as a shared singleton across the app;
   `disconnectSocket()` exists but is never called anywhere).

`features/contest/Usecountdown.js` (`useCountdown`) is a generic countdown-timer hook (ticks every
second, formats `HH:MM:SS`, fires `onExpire` once, flags `isWarning` in the last 5 minutes) used by
the contest arena UI — it's independent of the socket and purely client-clock based.

---

## 6. API Integration Layer (`src/services/`)

All HTTP traffic funnels through a single Axios instance (`services/api.js`), `baseURL` from
`VITE_API_URL` (defaults to `http://localhost:5000/api`), 30s timeout, JSON content-type.

### Endpoint inventory (by service file)

**`authService.js`**
- `POST /auth/login`, `POST /auth/refresh` (unused by any thunk), `POST /auth/logout`, `GET /auth/me`

**`problemService.js`**
- `GET /problems` (paged/filtered), `GET /problems/:id`, `GET /problems/:id/admin` (with hidden test
  cases), `POST /problems`, `PUT /problems/:id`, `DELETE /problems/:id`

**`submissionService.js`**
- `executionService`: `POST /execute` (run against sample cases), `POST /submit` (full judge run)
- `submissionService`: `GET /submissions`, `GET /submissions/stats`, `GET /submissions/:id`

**`collegeService.js`** (bundles three resource groups in one file)
- `collegeService`: `GET/POST /colleges`, `GET/PUT/DELETE /colleges/:id`, `GET /colleges/:id/students`
- `studentService`: `GET/POST /students`, `GET/PUT/DELETE /students/:id`
- `leaderboardService`: `GET /leaderboard`, `GET /leaderboard/overall`, `GET /leaderboard/:collegeId`,
  `POST /leaderboard/:collegeId/rebuild`, `POST /leaderboard/overall/rebuild`

**`Contestservice.js`** (note casing — see `TODO.md` #6)
- `GET /contests`, `GET /contests/:id`, `POST /contests/:id/join`,
  `POST /contests/:id/submit/:problemOrder`, `GET /contests/:id/leaderboard`,
  `GET /contests/:id/my-submissions`, admin: `POST/PUT/DELETE /contests[/:id]`

**Daily challenge** (called directly via `api` from `dailySlice.js` and `AdminDailySchedulePage.jsx`
— no dedicated service file)
- `GET /daily`, `POST /daily/submit`, `GET /daily/streak`, `GET /daily/calendar`
- Admin: `POST /daily/schedule`, `DELETE /daily/:date`, plus `GET /problems` reused for the problem picker

### Expected backend response envelope
`README.md` documents the contract: `{ success: true, data: <payload> }`. Every thunk defensively
unwraps with `res.data.data ?? res.data`, so the frontend also tolerates a bare payload without the
envelope. **This should be cross-checked against `API_CONTRACT.md` once available** — see
`FRONTEND_BACKEND_FLOW.md`.

---

## 7. Component Architecture

### Layout (`components/layout/`)
- **`AppLayout`** — flex shell; conditionally renders `Sidebar` based on a hidden-sidebar route
  regex list; shifts `<main>`'s `margin-left` based on `ui.sidebarOpen` (240px open / 64px collapsed).
- **`Sidebar`** — collapsible nav; separate `STUDENT_NAV` / `ADMIN_NAV` arrays drive the menu items;
  shows a user avatar/name footer + logout button.
- **`Topbar`** — a fully-built breadcrumb/streak/avatar header component that is **not used anywhere**
  (see `TODO.md` #1).
- **`ProtectedRoute` / `AdminRoute`** — thin route guards described in §3.

### Editor (`components/editor/`) — the core "coding workspace" experience
- **`CodingWorkspace`** — the shared 2-pane (problem + editor/console) layout used by both practice
  and contest solve pages. Contains its own **inline** `ProblemPanel` function (left pane) and a
  collapsible **Output console** (`OutputPanel`) docked under the Monaco editor.
- **`CodeEditor`** — wraps `@monaco-editor/react`; toolbar has language selector, font-size
  +/- controls (persisted to `ui.editorPrefs` via Redux), copy button, and reset-to-template button
  (currently broken — see `TODO.md` #4).
- **`OutputPanel`** — tabbed "Test Cases" / "Output" console. Renders a verdict hero card (Accepted/
  Wrong Answer/etc. with color coding), runtime/memory stat blocks with a simulated "beats X%"
  percentile bar, per-test-case pass/fail rows, and raw stdout/stderr/compile-error blocks.
- **`ProblemPanel.jsx`** (standalone file) — a duplicate of the problem-description renderer embedded
  in `CodingWorkspace`. Not imported anywhere (see `TODO.md` #2).

### UI primitives (`components/ui/index.jsx`)
A small design-system file exporting: `Spinner`, `PageLoader`, `DiffBadge`, `VerdictBadge`,
`EmptyState`, `ErrorBanner`, `PageHeader`, `StatCard`, `Modal`, `ConfirmModal`, `DataTable`,
`Pagination`. These are composed throughout every admin page for consistent CRUD UIs (table + modal
form + confirm-delete pattern appears in `AdminProblems`, `AdminStudents`, `AdminColleges`, etc.).

### `NotificationCenter`
Fixed top-right toast stack, driven by `ui.notifications`; each toast self-dismisses via
`setTimeout` matching its `duration` (default 4000ms) and dispatches `dismissNotif`.

### Widgets
- **`DailyStreakWidget`** — dashboard card showing today's daily-challenge status, current/longest
  streak, and a 7-dot mini-calendar (only the "today" dot is actually wired up; the other 6 are
  static placeholders — see `TODO.md` #5).

### Pages
- **Student pages** (`pages/*.jsx`): `HomePage`, `PracticePage` (filterable/paginated problem list),
  `ProblemSolvePage` (thin wrapper around `CodingWorkspace` for `/problem/:id`), `SubmissionsPage`,
  `LeaderboardPage`, `ProfilePage`, `DailyChallengePage`.
- **Contest pages** (`pages/contest/`): `ContestListPage`, `ContestArenaPage` (uses
  `CodingWorkspace` + `useContestSocket` + `useCountdown`), `ContestLeaderboardPage`.
- **Admin pages** (`pages/admin/`): `AdminDashboard`, `AdminProblems`, `AdminStudents`,
  `AdminColleges`, `AdminSubmissions`, `AdminLeaderboard`, `AdminContestListPage`,
  `AdminContestFormPage`, `AdminDailySchedulePage`. These consistently follow a
  **table + modal-form + confirm-delete-modal** CRUD pattern built on the shared `ui/` primitives.

---

## 8. Styling System

- Tailwind is configured with a custom palette (`primary`, `success`, `warning`, `danger`, `surface`
  scales), two font families (Plus Jakarta Sans for UI, JetBrains Mono for code), custom shadows
  (`card`, `card-md`, `card-lg`, `blue-glow`), and three keyframe animations (`fade-in`, `slide-up`,
  `pulse-dot`).
- `src/index.css` defines a set of reusable component classes under `@layer components`
  (`.btn`/`.btn-primary`/`.btn-secondary`/`.btn-ghost`/`.btn-danger`/`.btn-success`, `.input`,
  `.select`, `.card`/`.card-md`, `.badge-*`, `.tag`, plus presumably `.table-base`, `.nav-link`,
  `.section-title` — used throughout but defined further down in the file than was inspected).
- Theme is white/light with blue (`primary-600 = #2563eb`) as the accent — documented in the README
  as intentionally easy to re-theme by editing `primary-*` in `tailwind.config.js`.
- The Monaco editor and its immediate toolbar use a **separate dark theme** (`slate-900`/`slate-950`,
  `vs-dark` Monaco theme) — the only part of the UI that deliberately breaks from the light theme.

---

## 9. Utilities

**`utils/helpers.js`**
- Date formatting: `fmtDate`, `fmtDateTime`, `timeAgo` (all locale `en-IN`)
- Metric formatting: `fmtRuntime` (ms→s), `fmtMemory` (KB→MB)
- `rankLabel(rank)` → medal emoji for top 3, else `#N`
- `VERDICT_MAP` / `getVerdict()` → verdict string → `{ label, cls }` for badge styling
- `DIFFICULTY_CLS` → difficulty string → Tailwind badge class
- `ALL_TAGS` → static list of ~24 DSA topic tags used by both problem filters and the admin problem form
- `debounce(fn, ms)`, `cn(...args)` (simple classnames joiner — not `clsx`, just `.filter(Boolean).join(' ')`)

**`utils/editorUtils.js`**
- `LANGUAGES` — 7 supported languages (python, javascript, java, cpp, c, go, rust) with Monaco
  language IDs
- `TEMPLATES` — per-language boilerplate stub code
- `loadCode(problemId, lang, starterCode)` — 3-tier priority: localStorage draft → problem's
  admin-defined starter code → generic template
- `saveCode(...)` — persists to `localStorage` under `code:<problemId>:<lang>`, but **clears** the
  entry if the code matches the starter/template (avoids storing no-op drafts)
- `clearCode(problemId)` — wipes all per-language drafts for a problem

---

## 10. Summary for Future Work

The codebase is a conventional **Vite + React 18 + Redux Toolkit** SPA with clean domain
separation (one slice + one service file per resource) and a consistent CRUD UI pattern on the
admin side. The riskiest areas for a "modify without breaking things" pass are:

- The **coding workspace** (`CodeEditor` + `CodingWorkspace` + `OutputPanel`), which is the most
  complex and most-reused piece of UI, and already has one confirmed runtime bug (`TODO.md` #4).
- The **contest socket integration**, which is silently broken auth-wise (`TODO.md` #3) and should
  be verified against how the backend's Socket.io middleware actually authenticates.
- The **`Contestservice.js` casing issue** (`TODO.md` #6), which is a landmine for any deployment to
  a case-sensitive filesystem/build environment.

See `AUTHENTICATION.md` for auth details, `TODO.md` for the full findings backlog, and
`FRONTEND_BACKEND_FLOW.md` for the per-route API contract cross-check.

This document reflects the state of the code as analyzed; no files under `src/` were changed while
producing it.
