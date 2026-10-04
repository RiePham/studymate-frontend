# StudyMate — Frontend

> The web app for StudyMate, an AI-powered study workspace. Students organize their course materials in one place and turn them into searchable knowledge, AI explanations with sources, flashcards, quizzes, and measurable progress.

**Current phase:** 1 of 7: Foundation (login, register, protected routes, dashboard)

StudyMate is split into two repositories:

| Repo | What it contains |
| --- | --- |
| **studymate-frontend** (this repo) | Next.js app |
| [studymate-backend](https://github.com/RiePham/studymate-backend) | NestJS API, database, migrations, Docker Compose |

**The backend repo is the source of truth for the API contract.** The [API contract](#api-contract) section below is a copy; if they disagree, the backend README wins, and this copy should be updated.

---

## Table of contents

1. [Why StudyMate](#why-studymate)
2. [Architecture](#architecture)
3. [Tech stack](#tech-stack)
4. [Repository structure](#repository-structure)
5. [Getting started](#getting-started)
6. [Environment variables](#environment-variables)
7. [Common commands](#common-commands)
8. [Pages](#pages)
9. [How auth works on the frontend](#how-auth-works-on-the-frontend)
10. [API contract](#api-contract)
11. [Conventions](#conventions)
12. [Git workflow](#git-workflow)
13. [Testing and verification](#testing-and-verification)
14. [Troubleshooting](#troubleshooting)
15. [Roadmap](#roadmap)

---

## Why StudyMate

**The problem:** study materials are scattered across the LMS, cloud storage, and chat apps. Students spend more time finding material and hand-making flashcards and practice tests than actually studying.

**The product:** a unified workspace where a student creates a course, uploads its materials, and the system turns them into something they can search, ask questions about, and practice with.

**Guiding principle:**

> Use AI where language understanding helps. Use deterministic software where correctness matters.

**Core user flow:** register / log in → create a course → upload documents → system processes them → ask the AI Tutor, generate flashcards / quizzes → review → track progress.

**MVP:** authentication · courses · upload + storage · PDF text extraction · basic document search · RAG AI Tutor · flashcards · simple quizzes · basic dashboard

**Later:** spaced repetition (SM-2) · advanced analytics · study sessions · study groups · calendar and notifications · adaptive quizzes · mobile · OCR

---

## Architecture

```
Browser
  │
  ▼
┌──────────────── this repo ────────────────┐
│ Next.js frontend  (localhost:3000)        │
│   UI, routing, loading / empty / error    │
│   all requests go through src/lib/api.ts  │
└────────────────────┬──────────────────────┘
                     │  credentials: 'include' (auth travels as an httpOnly cookie)
                     ▼
studymate-backend: NestJS API  (localhost:4000/api)
  auth, authorization, validation, the only layer that touches data
  │
  ├──▶ PostgreSQL + pgvector
  └──▶ Redis → worker → storage / embeddings / LLM   [Phase 3+]
```

The frontend's job is **display and routing**. It contains no important business rules, and it is never where security is enforced. Every permission check happens in the backend.

---

## Tech stack

| Layer | Technology | Phase |
| --- | --- | --- |
| Language | TypeScript | 1 |
| Framework | Next.js (App Router) | 1 |
| Auth | httpOnly cookie set by the backend; `middleware.ts` for redirects | 1 |
| Deployment | Vercel | 7 |

---

## Repository structure

```
studymate-frontend/
├── src/
│   ├── app/              # App Router
│   │   ├── (public)/     # route group: /login, /register
│   │   └── (protected)/  # route group: /dashboard and other logged-in pages
│   ├── components/       # reusable UI components
│   ├── lib/api.ts        # the single gateway for all backend calls
│   └── middleware.ts     # redirects to /login when there's no auth cookie
├── .env.local            # NOT committed
├── .env.example          # committed: template for others
├── .gitignore
├── CLAUDE.md             # instructions for Claude Code
└── README.md
```

---

## Getting started

### 1. Prerequisites

- **Node.js (LTS)**, **Git**, and an editor
- **The backend running** on `localhost:4000`. Follow the setup in [studymate-backend](https://github.com/RiePham/studymate-backend) first, then check <http://localhost:4000/api/health> responds.

### 2. Clone, configure, run

```bash
git clone https://github.com/RiePham/studymate-frontend.git
cd studymate-frontend
cp .env.example .env.local
npm install
npm run dev
```

Open <http://localhost:3000>, register an account, log in, and create a course.

---

## Environment variables

### `.env.local`

| Variable | Example | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_API_URL` | `http://localhost:4000/api` | Base URL used by `src/lib/api.ts`. |

Anything prefixed with `NEXT_PUBLIC_` is bundled into the browser code, so anyone can read it. **Never put secrets in frontend env variables.** Restart `npm run dev` after changing `.env.local`.

When you add a new variable: add it to `.env.example` and to this table.

---

## Common commands

| Command | What it does |
| --- | --- |
| `npm run dev` | Run the dev server at localhost:3000 |
| `npm run build` | Production build (catches type errors) |
| `npm run lint` | Lint the code |

---

## Pages

| Route | Access | What it does |
| --- | --- | --- |
| `/login` | public | Form → `POST /auth/login` → redirect to `/dashboard`. Shows the `401` error message on failure. |
| `/register` | public | Form → `POST /auth/register` → redirect to `/login`. Shows `400` (invalid data) and `409` (email taken) errors. |
| `/dashboard` | protected | Lists the user's courses (`GET /courses`) and has a "Create course" form (`POST /courses`). The list updates right after creating. |

Every page handles **loading, empty, and error** states.

---

## How auth works on the frontend

1. On login, the **backend** sets an `access_token` cookie with the `HttpOnly` flag. JavaScript can't read it, which protects it from XSS. The browser sends it automatically.
2. `src/lib/api.ts` sends every request with `credentials: 'include'`. **This is the most important line in the file.** Without it, the browser won't send the cookie to a different port.
3. `middleware.ts` checks whether the cookie exists and redirects to `/login` if it doesn't.
4. **The frontend blocks, the backend decides.** The middleware only makes the experience smoother. The real check is the backend's `JwtAuthGuard`, which returns `401` for a missing or invalid token. The UI should handle a `401` by sending the user to `/login`.

---

## API contract

Copied from the backend README. Base URL: `NEXT_PUBLIC_API_URL` (`http://localhost:4000/api` locally).

| Method | Endpoint | Access | Behavior |
| --- | --- | --- | --- |
| `GET` | `/health` | public | Liveness check |
| `POST` | `/auth/register` | public | `400` invalid body, `409` email already registered |
| `POST` | `/auth/login` | public | Sets the cookie. `401` for wrong email or password (same message) |
| `POST` | `/auth/logout` | auth | Clears the cookie |
| `GET` | `/auth/me` | auth | Current user |
| `GET` | `/courses` | auth | The current user's courses |
| `POST` | `/courses` | auth | Create a course. Never send `userId`; the backend takes it from the token |
| `PATCH` | `/courses/:id` | auth | Update. `404` if not found or not yours |
| `DELETE` | `/courses/:id` | auth | Delete, returns `204`. `404` if not found or not yours |

| Code | How the UI should react |
| --- | --- |
| `400` | Show validation errors on the form |
| `401` | On login: show "invalid email or password". Elsewhere: redirect to `/login` |
| `404` | Show a "not found" state |
| `409` | Show the conflict message (e.g. "email already registered") |

---

## Conventions

These rules apply to everyone working on the codebase, human or AI assistant.

- **Components never call `fetch` directly.** All requests go through `src/lib/api.ts`.
- **No security or business rules in the frontend.** Hiding a button is UX, not protection. The backend enforces everything.
- **Every screen handles loading, empty, and error states.**
- **URL structure mirrors the data,** e.g. `/courses/[courseId]/documents`.
- **Reusable component tree:** split the UI into small components; keep server state (data from the API) separate from client state (form inputs, open/closed menus).
- **Forms:** disable the submit button while a request is in flight (`disabled={loading}`) to prevent double submits; show errors with `role="alert"` for screen readers.
- **Never commit `.env.local`.** Never put secrets in `NEXT_PUBLIC_` variables.

### Later phases

- **Document status (Phase 3):** after upload, poll `GET /documents/:id` until the status is `READY` or `FAILED`. `FAILED` shows the reason and a Retry button.
- **AI output (Phase 4–5):** AI Tutor answers always display their sources (document + page). AI-generated flashcards and quizzes are shown as editable drafts.
- **File downloads:** request a signed URL from the backend; never build storage URLs on the frontend.

---

## Git workflow

- **`main` always works.** Nobody pushes directly to it.
- **One branch per Jira ticket**, named with the ticket key: `SM-14-register-page`.
- **Small commits** with clear messages that include the ticket key: `SM-14 show 409 error on register`.
- **One pull request per ticket**, describing what changed and how to test it. The mentor reviews before merge.
- **Tickets that touch both repos** get one branch and one PR in each repo, using the same ticket key so they're easy to match up. If the page needs a new endpoint, the backend PR merges first.
- Run `git status` before every `git add .` to make sure `.env.local` isn't staged.

---

## Testing and verification

Before calling a feature "done", walk through it in the browser with the backend running.

- [ ] Register a new user → redirected to `/login`
- [ ] Register the same email again → `409` error shown on the page
- [ ] Submit the register form with invalid data → `400` errors shown
- [ ] Log in with a wrong password → error shown, no redirect
- [ ] Log in → in DevTools (Application → Cookies), `access_token` has the **HttpOnly** flag
- [ ] Visit `/dashboard` while logged out → redirected to `/login`
- [ ] Dashboard shows a loading state, then the courses (or an empty state for a new user)
- [ ] Create a course → it appears in the list immediately
- [ ] Double-clicking submit doesn't send two requests

---

## Troubleshooting

When something breaks, **open the Network tab in DevTools before guessing.** Check the request URL, the status code, and the response body.

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Failed to fetch` / connection refused | Backend isn't running | Start the backend; check <http://localhost:4000/api/health> |
| CORS error in the console | Backend CORS config | Backend needs `credentials: true` and `FRONTEND_URL=http://localhost:3000` |
| Login works but the next request returns `401` | Cookie isn't being sent | Make sure the request goes through `api.ts` with `credentials: 'include'` |
| `NEXT_PUBLIC_API_URL` is `undefined` | Env file missing or not reloaded | Create `.env.local`, then restart `npm run dev` |
| Redirect loop between `/login` and `/dashboard` | Middleware and backend disagree about the session | Check the cookie name in `middleware.ts` matches `access_token`, and that it hasn't expired |

---

## Roadmap

| Phase | Name | Frontend scope |
| --- | --- | --- |
| **1** | **Foundation** ← current | Login, register, protected routes, dashboard calling the real API |
| 2 | Course system | Full course CRUD UI |
| 3 | Documents | Upload UI, processing status with polling, Retry on `FAILED` |
| 4 | RAG | AI Tutor chat with cited sources |
| 5 | Study features | Flashcard and quiz screens |
| 6 | Analytics | Progress and weak-topic dashboard |
| 7 | Production quality | Testing, error handling, deployment to Vercel |

### Phase 1 checklist

- [ ] Next.js (App Router + TypeScript) project with `.gitignore` from the first commit
- [ ] `src/lib/api.ts` with `credentials: 'include'`
- [ ] Login page with error handling and loading state
- [ ] Register page showing `400` and `409` errors
- [ ] `middleware.ts` redirecting logged-out users to `/login`
- [ ] Dashboard listing the user's courses, with loading / empty / error states
- [ ] Create-course form that updates the list immediately
- [ ] A README that lets anyone clone and run the project
