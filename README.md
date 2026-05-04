# CyberStars

A free interactive coding education platform where users learn Python, C, and Java through structured lessons with embedded runnable code examples. Each lesson includes educational content on the left and a live code editor on the right, so learners can read, write, and execute code in the same view. Logged-in users get progress tracking and code persistence across sessions.

> **Note:** This is a portfolio project. This README intentionally includes detailed API endpoints, database schema, and architecture decisions that would normally live in internal documentation — the goal is to give reviewers a complete picture of the system without having to dig through the code.

## Table of contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Environment Variables](#environment-variables)
- [Project Structure](#project-structure)
- [API Endpoints](#api-endpoints)
- [Database Schema](#database-schema)
- [Code Execution](#code-execution)
- [Lesson Content](#lesson-content)
- [Architecture Decisions](#architecture-decisions)

## Features

- **Split-screen lesson view** — educational Markdown content on the left, live code editor (CodeMirror) on the right
- **Inline runnable code blocks** — code examples inside lesson text are interactive; click "Run Code" to execute them directly in the lesson
- **Test case validation** — each lesson has test cases that verify the user's code (like LeetCode). A lesson is marked complete only when all tests pass — there is no manual "complete" button
- **Multi-language support** — Python (18 lessons), C (16 lessons), Java (17 lessons) with language-specific syntax highlighting
- **Remote code execution** — user code runs server-side via Piston API (production) or Docker containers (development), not in the browser
- **Progress tracking** — lessons are automatically marked complete when all test cases pass, with per-course progress bars on the curriculum page
- **Gamification UI** — XP bar (15 XP per lesson, 300 XP per level), level badge, streak widget, achievement toasts on completion, and 4 unlockable badges (First Code / Speed Run / Bug Squasher / Perfect Score). All values are derived from real `UserLessonProgress` data — no separate gamification tables
- **Persistent course sidebar** — when viewing a lesson, a sidebar shows every lesson in the current course with a numbered marker (active = accent, completed = green ✓), a course progress bar, and the badges grid
- **Code persistence** — user code is saved per lesson and restored on revisit, so learners never lose their work
- **Curriculum from database** — courses and lessons are served from PostgreSQL with ordering, not hardcoded in the frontend
- **JWT authentication** — signup/login with httpOnly cookie-based sessions
- **Dark theme with design tokens** — palette built around an `--accent` purple (`#6C5CE7`), Space Grotesk for UI, JetBrains Mono for code; everything driven by CSS custom properties so theming/recoloring is a one-file change

## Screenshots

![Dashboard](screenshots/dashboard.png)
![Curriculum - Course Select](screenshots/select.png)
![Lesson View - All Tests Passed](screenshots/success.png)
![Lesson View - Tests Failed](screenshots/fail.png)

## Tech Stack

**Frontend:** React 19 + TypeScript + Vite 7 + Tailwind CSS 4 + React Router 7 + CodeMirror 4 + React Markdown

**Backend:** Node.js + Express 5 + TypeScript + Prisma 6 (PostgreSQL) + Zod 4 + tsx

**Code Execution:** Piston API (production) / Docker containers (development)

**Auth:** JWT (jsonwebtoken) + bcryptjs + httpOnly cookies

**Shared:** A top-level `shared/` folder holds DTO/contract types imported by both backend and frontend, so request/response shapes can never drift between the two sides.

## Prerequisites

- [Node.js](https://nodejs.org/) (v18+)
- [PostgreSQL](https://www.postgresql.org/) (v14+)
- [Docker](https://www.docker.com/) (development only — for local code execution)

### Installing prerequisites

**Ubuntu/Debian:**

```bash
# Node.js (via NodeSource)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# PostgreSQL
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql

# Docker (for local code execution)
sudo apt install -y docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER
```

**macOS (Homebrew):**

```bash
brew install node postgresql@14
brew services start postgresql@14
brew install --cask docker
```

### Setting up the database

```bash
sudo -u postgres psql
```

```sql
CREATE USER cyberstars WITH PASSWORD 'your_password';
CREATE DATABASE cyberstars OWNER cyberstars;
\q
```

## Setup

1. Clone the repo and install dependencies:

```bash
git clone https://github.com/your-username/cyberstars
cd cyberstars
npm install
```

`npm install` runs `prisma generate` automatically (via `postinstall`), producing the typed Prisma Client used by the backend.

2. Create and configure the `.env` file in the project root:

```env
DB_USER=cyberstars
DB_HOST=localhost
DB_NAME=cyberstars
DB_PASSWORD=your_password
DB_PORT=5432

DATABASE_URL=postgresql://cyberstars:your_password@localhost:5432/cyberstars

EXPRESS_PORT=5000
JWT_SECRET=your_secret_key

NODE_ENV=development
VITE_DEV_API_URL=http://localhost:5000
VITE_PROD_API_URL=
```

`DATABASE_URL` is what the Prisma CLI reads (`prisma migrate`, `prisma studio`, etc.). The runtime backend can construct it from the individual `DB_*` vars on its own, but the CLI requires the assembled URL.

3. Apply the database schema and seed the curriculum:

```bash
npm run db:deploy   # apply migrations to a fresh database
npm run db:seed     # seed courses + lessons
```

If the database already has the legacy SQL-migrated schema, mark the Prisma baseline as applied instead of running it:

```bash
npx prisma migrate resolve --applied 0_init
```

4. Start the development servers:

```bash
npm run dev
```

This starts both frontend (Vite on `http://localhost:5173`) and backend (`tsx watch` on `http://localhost:5000`) concurrently. Open `http://localhost:5173` in the browser. Hot reload is enabled for both — any file change is picked up automatically.

For production, build and start with:

```bash
npm run start
```

### Database scripts

| Script | What it does |
|--------|--------------|
| `npm run db:generate` | Regenerates the Prisma Client from `schema.prisma` |
| `npm run db:migrate` | `prisma migrate dev` — creates and applies a new migration in development |
| `npm run db:deploy` | `prisma migrate deploy` — applies pending migrations (used in production / fresh installs) |
| `npm run db:seed` | Seeds the curriculum + lessons via `prisma/seed.ts` |
| `npm run db:studio` | Opens Prisma Studio (DB browser at `http://localhost:5555`) |

## Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DB_USER` | PostgreSQL user | Yes | — |
| `DB_HOST` | PostgreSQL host | Yes | — |
| `DB_NAME` | Database name | Yes | — |
| `DB_PASSWORD` | Database password | Yes | — |
| `DB_PORT` | PostgreSQL port | No | `5432` |
| `DATABASE_URL` | Connection string used by the Prisma CLI | Yes (for `prisma` commands) | — |
| `EXPRESS_PORT` | Backend server port | No | `5000` |
| `JWT_SECRET` | Secret for signing JWT tokens | Yes | — |
| `NODE_ENV` | `development` or `production` | No | `development` |
| `VITE_PROD_API_URL` | Production API URL (for deployed frontend) | Production only | — |
| `CODE_RUN_DIR` | Host directory for per-run scratch dirs (dev Docker runner) | No | `os.tmpdir()/cyberstars-runs` |
| `CODE_RUN_MEMORY` | Per-container memory limit passed to `docker run --memory=` | No | `128m` |
| `CODE_RUN_PIDS` | Per-container PID limit passed to `docker run --pids-limit=` | No | `64` |

## Project Structure

```
cyberstars/
├── shared/                                 # Cross-cutting types (imported by both server and client)
│   ├── auth.ts                             # AuthenticatedUser, LoginPayload, SignupPayload, TokenPayload
│   ├── lesson.ts                           # LessonContent, LessonMeta, Course
│   ├── progress.ts                         # CourseProgress, LessonProgressItem
│   └── tests.ts                            # TestCase, TestResult, SubmitResult
│
├── prisma/                                 # Schema, migrations, seed
│   ├── schema.prisma                       # 5 models with camelCase fields + @map snake_case columns
│   ├── seed.ts                             # Seeds curriculum + lessons
│   └── migrations/
│       ├── migration_lock.toml
│       └── 0_init/
│           └── migration.sql               # Baseline schema (users, curriculum, lessons, progress, saved code)
│
├── server/                                   # Backend (Express + TypeScript)
│   ├── server.ts                           # Express entry point, route mounting, static SPA serving
│   ├── tsconfig.json                       # Backend TypeScript config (rootDir = repo root, includes shared/)
│   ├── config/
│   │   ├── env.ts                          # dotenv + required-var validation, typed config object
│   │   ├── db.ts                           # Prisma Client singleton (URL built from env or DATABASE_URL)
│   │   └── index.ts                        # Re-exports `config` and `prisma`
│   ├── schemas/                            # Zod request schemas (input validation)
│   │   ├── auth.schema.ts                  # signupSchema, loginSchema
│   │   ├── code.schema.ts                  # runCodeSchema, submitCodeSchema (language enum-restricted)
│   │   └── progress.schema.ts              # saveCodeSchema
│   ├── middleware/
│   │   ├── auth.ts                         # JWT verification (authenticateToken + optionalAuth)
│   │   ├── validate.ts                     # validateBody(schema) — Zod parser middleware
│   │   └── errorHandler.ts                 # AppError class + global error handler
│   ├── repositories/                       # Pure data access (Prisma Client only, no business logic)
│   │   ├── user.repository.ts              # findByEmail, findById, create
│   │   ├── curriculum.repository.ts        # getAllCourses, getLessonsByCourse, getAllLessons, getLessonCount
│   │   └── progress.repository.ts          # upsertProgress, upsertCode, getSavedCode, touchAccess
│   ├── services/                           # Business logic (no req/res, no SQL)
│   │   ├── auth.service.ts                 # signup, login, getUser (bcrypt + JWT)
│   │   ├── lesson.service.ts               # getLessonContent, getLessonCode, getCurriculum
│   │   ├── code-execution.service.ts       # execute (Piston API or Docker, supports stdin)
│   │   ├── test-runner.service.ts          # Run code against JSON test cases, compare output
│   │   └── progress.service.ts             # markComplete, saveCode, getSavedCode, getCourseProgress
│   ├── controllers/                        # Thin req/res layer
│   │   ├── auth.controller.ts              # signup, login, logout, me
│   │   ├── lesson.controller.ts            # getLesson, getLessonCode, getCurriculum
│   │   ├── code.controller.ts              # executeCode, submitCode
│   │   └── progress.controller.ts          # getCourseProgress, markComplete, saveCode, trackAccess
│   ├── routes/                             # Route wiring + per-route middleware (auth + validate)
│   │   ├── auth.routes.ts                  # /auth/*
│   │   ├── lesson.routes.ts                # /api/lessons/*, /api/lesson-code/*, /api/curriculum
│   │   ├── code.routes.ts                  # /api/run-code, /api/run-code/submit
│   │   └── progress.routes.ts              # /api/progress/* (all authenticated)
│   ├── types/
│   │   └── express.d.ts                    # Augments Express Request with `user` property
│   ├── lessons/                            # Markdown lesson content (read from filesystem)
│   │   ├── python/                         # 10 lessons (print, variables, loops, functions, etc.)
│   │   ├── c/                              # 2 lessons (variables, print)
│   │   └── java/                           # 2 lessons (variables, print)
│   └── runtimes/                           # Language registry — one file per supported language
│       ├── types.ts                        # LanguageRuntime interface (image, pistonVersion, innerCmd)
│       ├── registry.ts                     # getRuntime(lang) — single source of truth for supported langs
│       ├── python.ts                       # Python runtime config
│       ├── c.ts                            # C runtime config
│       └── java.ts                         # Java runtime config
│
├── client/                                  # Frontend (React + TypeScript)
│   ├── main.tsx                            # React entry point
│   ├── App.tsx                             # Router setup, AuthProvider wrapper
│   ├── index.css                           # Google Fonts, design tokens (CSS variables), lesson body markdown styles, scrollbars
│   ├── vite-env.d.ts                       # Vite type declarations
│   ├── types/
│   │   └── api.ts                          # ApiError + isApiError (frontend-only helper)
│   ├── services/
│   │   ├── apiClient.ts                    # Centralized fetch wrapper (credentials, error normalization)
│   │   ├── authService.ts                  # login, signup, logout, getMe
│   │   ├── lessonService.ts                # fetchLesson, fetchLessonCode, fetchCurriculum
│   │   ├── codeExecutionService.ts         # runCode, submitCode
│   │   └── progressService.ts              # getCourseProgress, markLessonComplete, saveCode
│   ├── context/
│   │   └── AuthContext.tsx                 # Global auth state (user, login, signup, logout)
│   ├── hooks/
│   │   ├── useLesson.ts                    # Fetches lesson content + saved code or template
│   │   ├── useCodeExecution.ts             # Runs code with loading/output state
│   │   ├── useProgress.ts                  # Track/save progress per course
│   │   └── useGamification.ts              # Derives XP, level, badges from real progress (no separate DB tables)
│   ├── components/
│   │   ├── ui/                             # Button (5 variants), Input, Modal, LoadingSpinner
│   │   ├── layout/                         # Topbar (logo + breadcrumb + streak + avatar), Sidebar (lesson nav + progress + badges)
│   │   ├── gamification/                   # XPBar, StreakWidget, Badge, AchievementToast
│   │   ├── code/                           # CodeEditor, CodeOutput, TestResults, CodeCell, RunButton
│   │   └── markdown/                       # MarkdownRenderer (with CodeCell for runnable blocks)
│   └── pages/
│       ├── HomePage.tsx                    # Hero landing — level/XP/badges if logged in, feature cards otherwise
│       ├── AuthPage.tsx                    # Login/signup card with logo + Topbar
│       ├── CurriculumPage.tsx              # Course cards with emoji icons + progress; click goes straight to lesson 1
│       └── LessonPage.tsx                  # Topbar + XPBar + Sidebar + content split + editor panel + Output
│
├── index.html                              # HTML entry point
├── package.json                            # Dependencies + scripts
├── tsconfig.json                           # Frontend TypeScript config (includes shared/)
├── vite.config.ts                          # Vite config (React, Tailwind, API proxy)
└── eslint.config.js                        # ESLint config (TS + React)
```

## API Endpoints

Authentication uses httpOnly JWT cookies. Protected endpoints require the `token` cookie set by login/signup. All write endpoints validate request bodies via Zod schemas in `server/schemas/` — invalid bodies return `400` with a list of issues.

### Auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/signup` | No | Register user, set JWT cookie. Body validated by `signupSchema` |
| POST | `/auth/login` | No | Login, set JWT cookie. Body validated by `loginSchema` |
| POST | `/auth/logout` | No | Clear JWT cookie |
| GET | `/auth/me` | Yes | Get current user (id, name, email) |

### Curriculum & Lessons

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/curriculum` | No | List all courses with their ordered lessons (from DB) |
| GET | `/api/lessons/:lang/:lesson` | No | Get lesson Markdown content |
| GET | `/api/lesson-code/:lang/:file` | No | Get starter code template for a lesson |

### Code Execution

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/run-code` | No | Execute code (`{ code, language }`) and return output. Validated by `runCodeSchema` |
| POST | `/api/run-code/submit` | Optional | Run code against lesson test cases (`{ code, language, courseKey, lessonSlug }`). Returns per-test results. Auto-marks lesson complete if all tests pass (when authenticated). Validated by `submitCodeSchema` |

### Progress (all authenticated)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/progress/:courseKey` | Yes | Get user's progress for a course (completed count, per-lesson status) |
| POST | `/api/progress/:courseKey/:lessonSlug/complete` | Yes | Mark a lesson as completed |
| GET | `/api/progress/:courseKey/:lessonSlug/code` | Yes | Get user's saved code for a lesson |
| PUT | `/api/progress/:courseKey/:lessonSlug/code` | Yes | Save user's code for a lesson. Validated by `saveCodeSchema` |
| POST | `/api/progress/:courseKey/:lessonSlug/access` | Yes | Track last access time for a lesson |

## Database Schema

The schema is defined in [`prisma/schema.prisma`](prisma/schema.prisma) — it is the single source of truth. Migrations live in `prisma/migrations/` and are applied with `prisma migrate deploy`.

### Models

- **User** (`users`) — account data (name, email, hashed password, createdAt). Email is unique.
- **Curriculum** (`curriculum`) — course definitions (key, title, description, sortOrder). `key` is unique.
- **Lesson** (`lessons`) — lesson metadata (courseKey, slug, title, sortOrder, hasCodeFile). Unique on `(courseKey, slug)`.
- **UserLessonProgress** (`user_lesson_progress`) — per-user lesson completion (completed, completedAt, lastAccessedAt). Unique on `(userId, courseKey, lessonSlug)`. Indexed on `userId` and `(userId, courseKey)`.
- **UserSavedCode** (`user_saved_code`) — per-user saved code per lesson (code, updatedAt). Unique on `(userId, courseKey, lessonSlug)`. Indexed on `userId`.

Field names use camelCase in TypeScript (`courseKey`, `sortOrder`, `completedAt`) and are mapped to snake_case columns (`course_key`, `sort_order`, `completed_at`) via Prisma's `@map`/`@@map`. Table names also use snake_case via `@@map`.

### Relationships

```
User (1) ──→ (N) UserLessonProgress    (ON DELETE CASCADE)
User (1) ──→ (N) UserSavedCode         (ON DELETE CASCADE)
Curriculum.key ←── Lesson.courseKey     (logical, not enforced as FK — kept simple)
```

Lesson content itself is stored as Markdown files on the filesystem (`server/lessons/:lang/:slug.md`), not in the database. The `Lesson` table stores metadata (ordering, titles) while the actual content is read from disk at request time.

## Code Execution

User code is never executed in the browser. It's sent to the backend, which delegates execution to an external runtime:

### Production: Piston API

The [Piston API](https://github.com/engineer-man/piston) is a free, open-source remote code execution engine. The backend sends code to `https://emkc.org/api/v2/piston/execute` with language-specific version pinning:

| Language | Piston Version | Compile Timeout | Run Timeout |
|----------|---------------|-----------------|-------------|
| Python | 3.10.0 | — | 5s |
| C | 10.2.0 (GCC) | 10s | 5s |
| Java | 15.0.2 | 10s | 5s |

### Development: Docker

In development, code runs in local Docker containers with volume-mounted temp directories:

| Language | Docker Image | Behavior |
|----------|-------------|----------|
| Python | `python:3.10-slim` | Runs with 5s timeout |
| C | `gcc:12.2.0` | Compiles with `-Wall`, then runs binary with 5s timeout |
| Java | `openjdk:20` | `javac` then runs `Main` with 5s timeout |

The execution flow: create a per-run temp dir under `os.tmpdir()/cyberstars-runs/<uuid>/` → write `<sourceFile>`, `stdin.txt`, empty `output.txt` → run the container with the dir mounted at `/work` → read `output.txt` → cleanup. All executions have a 5-second timeout to prevent infinite loops.

Containers are launched with sandboxing flags by default — `--network=none`, `--memory=128m`, `--pids-limit=64` — and Docker is invoked via `execFile` with an argument array (no shell interpolation). CPU is unbounded (the per-runtime `timeout 5` inside the inner command stops infinite loops). The limits can be tuned via the `CODE_RUN_*` env vars.

### Adding a new language

The runner is generic over the `LanguageRuntime` interface in `server/runtimes/types.ts`. Adding a language is two steps:

1. Create `server/runtimes/<lang>.ts` exporting an object with `name`, `image`, `pistonVersion`, `sourceFile`, and `innerCmd` (the shell command run inside the container; uses fixed paths under `/work/`).
2. Register it in `server/runtimes/registry.ts`.

No changes to the service, controllers, or routes are required.

## Lesson Content

Lessons are authored in Markdown and stored in `server/lessons/:language/`. Each lesson consists of:

- **`lesson-slug.md`** — the educational content (explanations, examples, inline code blocks)
- **`lesson-slug-code.md`** — the starter code template for the right-side editor
- **`lesson-slug-tests.json`** — test cases that validate the user's solution

Code blocks in the Markdown content that are tagged with a supported language (`` ```python ``, `` ```c ``, `` ```java ``) are rendered as interactive CodeCell components with their own editor and "Run Code" button, so learners can experiment with examples without leaving the lesson text.

### Test Cases

Each lesson's test file defines an array of test cases. Supported test modes:

| Mode | Description |
|------|-------------|
| `exact` | Output must match expected string exactly (trimmed) |
| `contains` | Output must contain the expected string |
| `any` | Any non-empty output passes |
| `line` | A specific line of the output must match (by line index) |

Tests can also use `overrides` to inject variable values into user code (for testing different inputs on the same logic), and `append` to add function calls after user code (for testing function definitions). The full test case shape lives in `shared/tests.ts` and is the same type used by both the test runner on the server and the result rendering on the client.

### Available lessons

| Language | Lessons | Topics |
|----------|---------|--------|
| Python | 18 | print, string variables, integer variables, f-strings, comments, if/else, if/elif/else, for loops, while loops, functions, input, operators, booleans, string methods, lists, looping over lists, break/continue, return values |
| C | 16 | print, variables (integers/floats), comments, if/else, if/else if/else, for loops, while loops, functions, input (scanf), operators, booleans, strings, arrays, looping over arrays, break/continue |
| Java | 17 | print, variables (numbers/strings), string concatenation, comments, if/else, if/else if/else, for loops, while loops, methods, input (Scanner), operators, booleans, string methods, arrays, looping over arrays, break/continue |

Adding a new lesson requires: (1) creating the `.md` file in `server/lessons/:lang/`, (2) optionally creating a `-code.md` starter template, (3) creating a `-tests.json` file with test cases, and (4) adding the lesson row to the database — either by extending `prisma/seed.ts` and running `npm run db:seed`, or by adding a fresh Prisma migration.

## Architecture Decisions

- **Controller → Service → Repository**: The backend follows a strict three-layer separation. Controllers handle HTTP concerns (req/res, cookies), services contain business logic (password hashing, token creation, progress aggregation), and repositories are the only place Prisma is touched. Each layer is testable and replaceable independently.

- **Prisma over hand-written SQL**: The data layer uses Prisma Client, with `prisma/schema.prisma` as the single source of truth for both the database structure and the TypeScript types in repositories. Migrations are versioned in `prisma/migrations/` and applied with `prisma migrate deploy`. Seed data lives in `prisma/seed.ts` (TypeScript), not SQL — it's idempotent (`upsert`) so it can run safely on top of an existing DB.

- **Zod-validated request bodies**: Every write endpoint (`POST` / `PUT`) is wrapped in a `validateBody(schema)` middleware that parses the body through a Zod schema from `server/schemas/`. Invalid input never reaches a controller — the middleware returns `400` with a list of issues. Schemas double as inferred TypeScript types, so the validated body is fully typed downstream.

- **Shared types between client and server**: A top-level `shared/` directory holds DTO/contract types (`auth`, `lesson`, `progress`, `tests`) imported by both `server/` and `client/`. The wire format is defined exactly once. If the server response shape changes, the frontend type-check fails immediately rather than silently drifting.

- **Cookie-based auth over Bearer tokens**: JWT tokens are stored in httpOnly cookies instead of localStorage. This prevents XSS from accessing tokens — the browser handles cookie attachment automatically via `credentials: "include"`, and the server never exposes the token to JavaScript.

- **Filesystem lessons, database metadata**: Lesson content lives in `.md` files (easy to author and version with git), while ordering and metadata live in PostgreSQL (easy to query and extend). This avoids putting large text blobs in the database while keeping the curriculum structure queryable.

- **Centralized API client**: All frontend API calls go through `apiClient.ts`, which handles base URL resolution, credentials, JSON parsing, and error normalization. No raw `fetch()` calls anywhere in the frontend — every service function is a one-liner that calls `api.get()` or `api.post()`.

- **AuthContext over per-page auth checks**: A single `AuthContext` provider wraps the entire app and checks `/auth/me` once on mount. Every page and component accesses auth state via `useAuth()` — no duplicate fetch calls, no prop drilling, and login/logout state updates propagate everywhere instantly.

- **Test-driven lesson completion**: Lessons are completed by passing all test cases, not by clicking a button. This ensures learners actually solve the exercise. Test cases are defined as JSON files on disk alongside lesson content, supporting exact match, contains, line-based checks, variable overrides (for testing different inputs), and code appending (for testing function definitions).

- **Design tokens via CSS variables, not Tailwind config**: Colors, fonts, radii, and shadows are declared once in `client/index.css` as `--accent`, `--bg`, `--bg2`, `--surface`, etc. Components reference them through Tailwind's arbitrary-value syntax (`bg-[var(--accent)]`, `text-[var(--text2)]`). Re-theming or building a light mode is a matter of overriding a handful of variables; no rebuild or component changes required. The values came from a Claude Design handoff spec — see `client/index.css` for the full palette.

- **Gamification derived, not stored**: XP, level, and badges are computed on the client from `UserLessonProgress` data already in the DB (`useGamification.ts` hook). 15 XP per completed lesson; level = floor(xp / 300) + 1; badges unlock at lesson-count thresholds and on full course completion. No new tables, no new endpoints, no risk of XP and progress drifting out of sync. Streak is currently a placeholder — adding it would require a `user_activity` table to track daily logins.

- **Persistent course-aware sidebar in lesson view**: The lesson layout is `Topbar + XPBar + Sidebar + content split + editor panel`. The sidebar lists every lesson in the *current course* with progress markers (numbered circle / green ✓), a course progress bar, and the badges grid — so the learner always knows where they are in the course and what's coming next, without leaving the lesson page.

- **User-resizable editor panel**: The right-side editor panel has a draggable splitter on its left edge — the user can drag it left/right to give more room to either the lesson text or the editor. The width is persisted to `localStorage` so it sticks across page loads. Constraints: minimum ~360px (so the editor stays usable), maximum ~70% of the viewport (so the lesson body doesn't disappear). The CodeMirror editor inside soft-wraps long lines — no horizontal scroll, code never falls off-screen.

- **Dual code execution strategy**: Development uses Docker for offline work and full control; production uses the free Piston API to avoid running Docker in hosted environments. The `code-execution.service` abstracts this behind a single `execute(code, language)` interface — the rest of the app doesn't know which backend is running.

- **Language runtime registry**: Per-language config (Docker image, Piston version, source filename, inner shell command) is isolated in `server/runtimes/<lang>.ts` files behind a `LanguageRuntime` interface. The execution service is fully generic — there is no `switch (language)` anywhere in the runner. Adding a new language is one file plus one line in `registry.ts`, with zero changes to the service, controllers, or routes.

- **Sandboxed code execution**: Docker is invoked via `execFile("docker", [...args])` with an argument array — no shell, no string interpolation, no injection surface. Containers run with `--network=none`, memory and PID limits, and the per-run scratch dir lives under `os.tmpdir()` instead of inside the source tree (so `tsx watch` doesn't reload on every execution). CPU is intentionally unbounded so user code can use the full machine when needed; infinite loops are stopped by the `timeout 5` wrapper inside each runtime's inner command. All limits are env-tunable via `CODE_RUN_*` variables.

- **Configuration split**: Environment loading and validation live in `server/config/env.ts` (with a small `required()` helper that throws if a critical variable is missing). The Prisma Client singleton lives in `server/config/db.ts`. `server/config/index.ts` re-exports both, so the rest of the codebase imports configuration from a single place.

- **File naming convention**: Backend files in `controllers/`, `services/`, `repositories/`, and `routes/` use the `<entity>.<role>.ts` convention (`auth.controller.ts`, `progress.service.ts`, `user.repository.ts`). It's instantly clear what layer a file belongs to when many tabs are open.

- **Vite proxy in development**: The Vite dev server proxies `/api` and `/auth` requests to the Express backend, eliminating CORS issues in development and allowing the frontend to use relative paths. In production, the Express server serves the built SPA directly, so no proxy is needed.

- **`concurrently` for dev workflow**: A single `npm run dev` command starts both frontend (Vite with HMR) and backend (`tsx watch` with auto-restart) in parallel. No need to manage two terminals manually during development.
