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
- **Multi-language support** — Python (10 lessons), C (2 lessons), Java (2 lessons) with language-specific syntax highlighting
- **Remote code execution** — user code runs server-side via Piston API (production) or Docker containers (development), not in the browser
- **Progress tracking** — logged-in users can mark lessons as complete, and progress is displayed per-course with visual progress bars
- **Code persistence** — user code is auto-saved per lesson and restored on revisit, so learners never lose their work
- **Curriculum from database** — courses and lessons are served from PostgreSQL with ordering, not hardcoded in the frontend
- **JWT authentication** — signup/login with httpOnly cookie-based sessions
- **Dark theme** — GTA Vice City-inspired color palette (muted rose + steel blue on warm purple backgrounds)

## Tech Stack

**Frontend:** React 19 + TypeScript + Vite 7 + Tailwind CSS 4 + React Router 7 + CodeMirror 4 + React Markdown

**Backend:** Node.js + Express 5 + TypeScript + PostgreSQL (pg driver) + tsx

**Code Execution:** Piston API (production) / Docker containers (development)

**Auth:** JWT (jsonwebtoken) + bcryptjs + httpOnly cookies

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

2. Create and configure the `.env` file in the project root:

```env
DB_USER=cyberstars
DB_HOST=localhost
DB_NAME=cyberstars
DB_PASSWORD=your_password
DB_PORT=5432

EXPRESS_PORT=5000
JWT_SECRET=your_secret_key

NODE_ENV=development
VITE_DEV_API_URL=http://localhost:5000
VITE_PROD_API_URL=
```

3. Run database migrations (creates tables and seeds curriculum data):

```bash
npm run migrate
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

## Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DB_USER` | PostgreSQL user | Yes | — |
| `DB_HOST` | PostgreSQL host | Yes | — |
| `DB_NAME` | Database name | Yes | — |
| `DB_PASSWORD` | Database password | Yes | — |
| `DB_PORT` | PostgreSQL port | No | `5432` |
| `EXPRESS_PORT` | Backend server port | No | `5000` |
| `JWT_SECRET` | Secret for signing JWT tokens | Yes | — |
| `NODE_ENV` | `development` or `production` | No | `development` |
| `VITE_PROD_API_URL` | Production API URL (for deployed frontend) | Production only | — |

## Project Structure

```
cyberstars/
├── front/                                 # Frontend (React + TypeScript)
│   ├── main.tsx                           # React entry point
│   ├── App.tsx                            # Router setup, AuthProvider wrapper
│   ├── index.css                          # Tailwind imports + custom scrollbar styles
│   ├── vite-env.d.ts                      # Vite type declarations
│   ├── types/
│   │   ├── auth.ts                        # User, LoginPayload, SignupPayload
│   │   ├── curriculum.ts                  # Course, LessonMeta, LessonContent
│   │   ├── progress.ts                    # LessonProgress, CourseProgress
│   │   └── api.ts                         # ApiError, isApiError helper
│   ├── services/
│   │   ├── apiClient.ts                   # Centralized fetch wrapper (credentials, error handling)
│   │   ├── authService.ts                 # login, signup, logout, getMe
│   │   ├── lessonService.ts               # fetchLesson, fetchLessonCode, fetchCurriculum
│   │   ├── codeExecutionService.ts        # runCode
│   │   └── progressService.ts             # getCourseProgress, markComplete, saveCode
│   ├── context/
│   │   └── AuthContext.tsx                 # Global auth state (user, login, signup, logout)
│   ├── hooks/
│   │   ├── useLesson.ts                   # Fetches lesson content + saved code or template
│   │   ├── useCodeExecution.ts            # Runs code with loading/output state
│   │   └── useProgress.ts                 # Track/save progress per course
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx                 # 4 variants: primary, outline, danger, ghost
│   │   │   ├── Input.tsx                  # Styled input with error state
│   │   │   ├── Modal.tsx                  # Backdrop + centered card overlay
│   │   │   └── LoadingSpinner.tsx         # Animated spinner
│   │   ├── layout/
│   │   │   ├── Navbar.tsx                 # Top navigation bar with CyberStars logo
│   │   │   └── PageContainer.tsx          # Consistent page background wrapper
│   │   ├── code/
│   │   │   ├── CodeEditor.tsx             # CodeMirror wrapper (language detection, oneDark theme)
│   │   │   ├── CodeOutput.tsx             # Output display panel with auto-scroll
│   │   │   ├── CodeCell.tsx               # Self-contained inline code block (editor + run + output)
│   │   │   └── RunButton.tsx              # Run button with loading state
│   │   ├── markdown/
│   │   │   └── MarkdownRenderer.tsx       # ReactMarkdown with CodeCell for runnable code blocks
│   │   └── progress/
│   │       ├── ProgressBar.tsx            # Visual completion bar per course
│   │       └── LessonStatusBadge.tsx      # Completed/incomplete indicator
│   └── pages/
│       ├── HomePage.tsx                   # Landing page with auth-aware greeting
│       ├── AuthPage.tsx                   # Login/signup toggle form
│       ├── CurriculumPage.tsx             # Course grid with progress + lesson modal
│       └── LessonPage.tsx                 # Split-screen lesson view (content + editor)
│
├── back/                                  # Backend (Express + TypeScript)
│   ├── server.ts                          # Express entry point, route mounting, static serving
│   ├── config.ts                          # Centralized environment config
│   ├── tsconfig.json                      # Backend TypeScript config
│   ├── types/
│   │   ├── auth.ts                        # User, TokenPayload, AuthenticatedUser
│   │   ├── lesson.ts                      # LessonContent, LessonRow, CurriculumRow
│   │   ├── progress.ts                    # UserLessonProgress, UserSavedCode, CourseProgressSummary
│   │   └── express.d.ts                   # Extends Express Request with user property
│   ├── middleware/
│   │   ├── authToken.ts                   # JWT verification + optional auth middleware
│   │   └── errorHandler.ts                # AppError class + global error handler
│   ├── repositories/                      # Pure SQL queries, typed results
│   │   ├── userRepository.ts              # findByEmail, findById, create
│   │   ├── progressRepository.ts          # upsertProgress, upsertCode, getSavedCode, touchAccess
│   │   └── curriculumRepository.ts        # getAllCourses, getLessonsByCourse, getLessonCount
│   ├── services/                          # Business logic
│   │   ├── authService.ts                 # signup, login, getUser (bcrypt + JWT)
│   │   ├── lessonService.ts               # getLessonContent, getLessonCode, getCurriculum
│   │   ├── codeExecutionService.ts        # execute (Piston API or Docker)
│   │   └── progressService.ts             # markComplete, saveCode, getSavedCode, getCourseProgress
│   ├── controllers/                       # Request handlers (thin req/res layer)
│   │   ├── authController.ts              # signup, login, logout, me
│   │   ├── lessonController.ts            # getLesson, getLessonCode, getCurriculum
│   │   ├── codeController.ts              # executeCode
│   │   └── progressController.ts          # getCourseProgress, markComplete, saveCode, trackAccess
│   ├── routes/                            # Route wiring to controllers
│   │   ├── authRoutes.ts                  # /auth/*
│   │   ├── lessonRoutes.ts                # /api/lessons/*, /api/lesson-code/*, /api/curriculum
│   │   ├── codeRoutes.ts                  # /api/run-code
│   │   └── progressRoutes.ts              # /api/progress/* (all authenticated)
│   ├── db/
│   │   ├── db.ts                          # PostgreSQL connection pool
│   │   └── migrations/
│   │       ├── 001_create_users.sql       # Users table
│   │       ├── 002_create_lessons.sql     # Lessons table (course_key, slug, sort_order)
│   │       ├── 003_create_curriculum.sql  # Curriculum table (key, title, description)
│   │       ├── 004_create_user_progress.sql # Lesson completion tracking
│   │       ├── 005_create_user_code.sql   # Saved user code per lesson
│   │       ├── 006_seed_curriculum.sql    # Seeds courses + lessons from markdown files
│   │       └── migrate.ts                 # Migration runner
│   └── lessons/                           # Markdown lesson content (read from filesystem)
│       ├── python/                        # 10 lessons (print, variables, loops, functions, etc.)
│       ├── c/                             # 2 lessons (variables, print)
│       └── java/                          # 2 lessons (variables, print)
│
├── index.html                             # HTML entry point
├── package.json                           # Dependencies + scripts
├── tsconfig.json                          # Frontend TypeScript config
├── vite.config.ts                         # Vite config (React, Tailwind, API proxy)
└── eslint.config.js                       # ESLint config (TS + React)
```

## API Endpoints

Authentication uses httpOnly JWT cookies. Protected endpoints require the `token` cookie set by login/signup.

### Auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/signup` | No | Register user, set JWT cookie |
| POST | `/auth/login` | No | Login, set JWT cookie |
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
| POST | `/api/run-code` | No | Execute code (`{ code, language }`) and return output |

### Progress (all authenticated)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/progress/:courseKey` | Yes | Get user's progress for a course (completed count, per-lesson status) |
| POST | `/api/progress/:courseKey/:lessonSlug/complete` | Yes | Mark a lesson as completed |
| GET | `/api/progress/:courseKey/:lessonSlug/code` | Yes | Get user's saved code for a lesson |
| PUT | `/api/progress/:courseKey/:lessonSlug/code` | Yes | Save user's code for a lesson |
| POST | `/api/progress/:courseKey/:lessonSlug/access` | Yes | Track last access time for a lesson |

## Database Schema

6 tables across 2 migration phases. The first phase (001–003) defines the content structure, the second (004–005) adds user progress tracking. Migration 006 seeds the curriculum data.

### Tables

- **users** — account data (name, email, hashed password, created_at)
- **curriculum** — course definitions (key, title, description, sort_order). Unique on `key`
- **lessons** — lesson metadata (course_key, slug, title, sort_order, has_code_file). Unique on `(course_key, slug)`
- **user_lesson_progress** — per-user lesson completion tracking (completed flag, completed_at, last_accessed_at). Unique on `(user_id, course_key, lesson_slug)`. Indexed on `user_id` and `(user_id, course_key)`
- **user_saved_code** — per-user saved code for each lesson (code text, updated_at). Unique on `(user_id, course_key, lesson_slug)`. Indexed on `user_id`

### Relationships

```
users (1) ──→ (N) user_lesson_progress   (ON DELETE CASCADE)
users (1) ──→ (N) user_saved_code        (ON DELETE CASCADE)
curriculum.key ←── lessons.course_key     (logical, not FK — kept simple)
```

Lesson content itself is stored as Markdown files on the filesystem (`back/lessons/:lang/:slug.md`), not in the database. The `lessons` table stores metadata (ordering, titles) while the actual content is read from disk at request time.

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

Lessons are authored in Markdown and stored in `back/lessons/:language/`. Each lesson consists of:

- **`lesson-slug.md`** — the educational content (explanations, examples, inline code blocks)
- **`lesson-slug-code.md`** — the starter code template for the right-side editor

Code blocks in the Markdown content that are tagged with a supported language (`` ```python ``, `` ```c ``, `` ```java ``) are rendered as interactive CodeCell components with their own editor and "Run Code" button, so learners can experiment with examples without leaving the lesson text.

### Available lessons

| Language | Lessons | Topics |
|----------|---------|--------|
| Python | 10 | print, string variables, integer variables, f-strings, comments, if/else, if/elif/else, for loops, while loops, functions |
| C | 2 | variables, print |
| Java | 2 | variables, print |

Adding a new lesson requires: (1) creating the `.md` file in `back/lessons/:lang/`, (2) optionally creating a `-code.md` starter template, and (3) adding a row to the `lessons` table via a new migration or direct insert.

## Architecture Decisions

- **Controller → Service → Repository**: The backend follows a strict three-layer separation. Controllers handle HTTP concerns (req/res, cookies), services contain business logic (password hashing, token creation, progress aggregation), and repositories are pure SQL. This keeps each layer testable and replaceable independently.

- **Cookie-based auth over Bearer tokens**: JWT tokens are stored in httpOnly cookies instead of localStorage. This prevents XSS from accessing tokens — the browser handles cookie attachment automatically via `credentials: "include"`, and the server never exposes the token to JavaScript.

- **Filesystem lessons, database metadata**: Lesson content lives in `.md` files (easy to author and version with git), while ordering and metadata live in PostgreSQL (easy to query and extend). This avoids putting large text blobs in the database while keeping the curriculum structure queryable.

- **Centralized API client**: All frontend API calls go through `apiClient.ts`, which handles base URL resolution, credentials, JSON parsing, and error normalization. No raw `fetch()` calls anywhere in the frontend — every service function is a one-liner that calls `api.get()` or `api.post()`.

- **AuthContext over per-page auth checks**: A single `AuthContext` provider wraps the entire app and checks `/auth/me` once on mount. Every page and component accesses auth state via `useAuth()` — no duplicate fetch calls, no prop drilling, and login/logout state updates propagate everywhere instantly.

- **Dual code execution strategy**: Development uses Docker for offline work and full control; production uses the free Piston API to avoid running Docker in hosted environments. The `codeExecutionService` abstracts this behind a single `execute(code, language)` interface — the rest of the app doesn't know which backend is running.

- **Vite proxy in development**: The Vite dev server proxies `/api` and `/auth` requests to the Express backend, eliminating CORS issues in development and allowing the frontend to use relative paths. In production, the Express server serves the built SPA directly, so no proxy is needed.

- **`concurrently` for dev workflow**: A single `npm run dev` command starts both frontend (Vite with HMR) and backend (`tsx watch` with auto-restart) in parallel. No need to manage two terminals manually during development.
