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
- **Multi-language support** — Python (10 lessons), C (2 lessons), Java (2 lessons) with language-specific syntax highlighting
- **Remote code execution** — user code runs server-side via Piston API (production) or Docker containers (development), not in the browser
- **Progress tracking** — lessons are automatically marked complete when all test cases pass, with per-course progress bars on the curriculum page
- **Code persistence** — user code is saved per lesson and restored on revisit, so learners never lose their work
- **Curriculum from database** — courses and lessons are served from PostgreSQL with ordering, not hardcoded in the frontend
- **JWT authentication** — signup/login with httpOnly cookie-based sessions
- **Dark theme** — clean dark palette with soft blue accents, easy on the eyes

## Screenshots

![Home Page](screenshots/1.png)
![Home Page - Logged In](screenshots/2.png)
![Curriculum Page](screenshots/3.png)
![Curriculum - Lesson List with Progress](screenshots/4.png)
![Lesson View - Submitting Code](screenshots/5.png)
![Lesson View - All Tests Passed](screenshots/6.png)
![Lesson View - Tests Failed](screenshots/7.png)

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
│   └── runtimes/                           # Docker images + scratch dirs for dev code execution
│       ├── python/
│       └── c/
│
├── client/                                  # Frontend (React + TypeScript)
│   ├── main.tsx                            # React entry point
│   ├── App.tsx                             # Router setup, AuthProvider wrapper
│   ├── index.css                           # Tailwind imports + custom scrollbar styles
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
│   │   └── useProgress.ts                  # Track/save progress per course
│   ├── components/
│   │   ├── ui/                             # Button, Input, Modal, LoadingSpinner
│   │   ├── layout/                         # Navbar, PageContainer
│   │   ├── code/                           # CodeEditor, CodeOutput, TestResults, CodeCell, RunButton
│   │   ├── markdown/                       # MarkdownRenderer (with CodeCell for runnable blocks)
│   │   └── progress/                       # ProgressBar, LessonStatusBadge
│   └── pages/
│       ├── HomePage.tsx                    # Landing page with auth-aware greeting
│       ├── AuthPage.tsx                    # Login/signup toggle form
│       ├── CurriculumPage.tsx              # Course grid with progress + lesson modal
│       └── LessonPage.tsx                  # Split-screen lesson view (content + editor)
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

| Language | Docker Image | Container |
|----------|-------------|-----------|
| Python | `python-runtime` (custom) | `python:latest` base |
| C | `gcc:12.2.0` | Compiles + runs with 5s timeout |
| Java | `openjdk:20` | Compiles + runs with 5s timeout |

The execution flow: write code to a temp file → run Docker container with volume mount → read output file → return result. All executions have a 5-second timeout to prevent infinite loops.

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
| Python | 10 | print, string variables, integer variables, f-strings, comments, if/else, if/elif/else, for loops, while loops, functions |
| C | 2 | variables, print |
| Java | 2 | variables, print |

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

- **Dual code execution strategy**: Development uses Docker for offline work and full control; production uses the free Piston API to avoid running Docker in hosted environments. The `code-execution.service` abstracts this behind a single `execute(code, language)` interface — the rest of the app doesn't know which backend is running.

- **Configuration split**: Environment loading and validation live in `server/config/env.ts` (with a small `required()` helper that throws if a critical variable is missing). The Prisma Client singleton lives in `server/config/db.ts`. `server/config/index.ts` re-exports both, so the rest of the codebase imports configuration from a single place.

- **File naming convention**: Backend files in `controllers/`, `services/`, `repositories/`, and `routes/` use the `<entity>.<role>.ts` convention (`auth.controller.ts`, `progress.service.ts`, `user.repository.ts`). It's instantly clear what layer a file belongs to when many tabs are open.

- **Vite proxy in development**: The Vite dev server proxies `/api` and `/auth` requests to the Express backend, eliminating CORS issues in development and allowing the frontend to use relative paths. In production, the Express server serves the built SPA directly, so no proxy is needed.

- **`concurrently` for dev workflow**: A single `npm run dev` command starts both frontend (Vite with HMR) and backend (`tsx watch` with auto-restart) in parallel. No need to manage two terminals manually during development.
