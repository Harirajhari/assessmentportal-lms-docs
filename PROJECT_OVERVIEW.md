# Project Overview — Assessment Portal (Coding Platform + LMS)

> Phase 2 documentation. Split from `ARCHITECTURE.md` (Phase 1 backend read-through) and
> cross-checked directly against `assesmentportal-with-lms-backend-v1` (main branch).
> This file describes the whole application, not just the backend.

## What this project is

The Assessment Portal is a **multi-tenant (college-scoped) online judge / coding-platform
application**. Each participating college's students and admins share one deployment, with
data isolated per college.

The product combines:

- A **LeetCode-style problem bank** with code execution and judging, powered by Judge0.
- **Per-college and global (cross-college) leaderboards.**
- A **contest module**: ICPC-style scoring (points + penalty minutes), a leaderboard freeze
  window before contest end, optional sequential problem unlock, and a live Socket.io
  leaderboard.
- A **daily-challenge module**: a "Problem of the Day" feature with streak tracking and a
  monthly calendar view.
- **JWT-based authentication** with role-based access control (`admin` / `student`) and
  college-scoped data access.

## System shape

- One Express.js REST API + one Socket.io namespace for realtime contest updates.
- **MongoDB** is the source of truth for every entity (users, colleges, problems,
  submissions, contests, daily challenges).
- **Redis** is present in the dependency graph as an intended leaderboard cache, but is
  **currently not connected at runtime** — see `TODO.md` item 1 and `BACKEND.md` for detail.
  The app still functions because every Redis-backed read path falls back to MongoDB.
- Code execution/judging is delegated to an external **Judge0** instance over HTTP.

## Roles

Two roles exist: `admin` and `student`. There is no intermediate `instructor`/`TA` role,
despite the "LMS" naming — admins manage colleges, students, problems, and contests;
students consume the platform.

## Where to look next

| Topic | File |
|---|---|
| Tech stack, project layout, backend patterns, env vars | `BACKEND.md` |
| Every Mongoose model, field, and relationship | `DATABASE_SCHEMA.md` |
| Every API route, request/response shape, auth/validation status | `API_CONTRACT.md` |
| JWT flow, roles, college-scoping | `AUTHENTICATION.md` |
| Known defects and suggested next steps | `TODO.md` |

## Source note

This documentation set describes the **actual, existing system as read from the code** —
nothing here has been "cleaned up" or idealized. Where something is broken or incomplete in
the real backend (e.g. contest submission is currently non-functional, Redis is never
connected), it is documented as broken. See `TODO.md` for the full defect list.
