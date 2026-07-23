# RkruterAI — AI Interviewer

RkruterAI is an AI-powered voice interviewing platform that automates the
first-round technical screen. A candidate submits their GitHub profile and is
interviewed live by a realtime voice agent that asks questions grounded in
their actual public work — repositories, languages, and project activity. The
whole conversation is transcribed in real time, and once the interview ends the
system automatically evaluates the transcript to produce a score and written
feedback.

### How it works

1. **Profile intake** — The candidate enters their GitHub URL. The backend
   scrapes their public profile and repositories to build context for the
   interview.
2. **Live voice interview** — A realtime voice agent (OpenAI Realtime API over
   WebRTC) conducts a spoken interview. The candidate's speech is streamed to
   Deepgram for live transcription, and every exchange is saved.
3. **Automated evaluation** — When the session ends, the transcript is scored by
   Google Gemini, which returns a numeric score and qualitative feedback.
4. **Results** — The candidate (or recruiter) can retrieve the score, feedback,
   and full transcript for the session.

The goal is to give recruiters a scalable, consistent way to run structured
first-round interviews without scheduling human interviewer time.

## Tech stack

| Layer     | Tech                                                                 |
| --------- | ------------------------------------------------------------------- |
| Runtime   | [Bun](https://bun.sh) (not Node.js)                                  |
| Monorepo  | [Turborepo](https://turbo.build)                                    |
| Backend   | Express 5, Prisma 7 (PostgreSQL), OpenAI Realtime API, Google Gemini |
| Frontend  | React 19, React Router 7, Tailwind CSS 4, Radix UI, Deepgram        |

## Repository layout

```
.
├── apps/
│   ├── backend/     # Express API — interview flow, scoring, WebRTC sideband
│   └── frontend/    # React SPA served by Bun.serve()
├── packages/
│   ├── ui/                  # Shared React components
│   ├── eslint-config/       # Shared ESLint config
│   └── typescript-config/   # Shared tsconfig bases
└── turbo.json
```

## Prerequisites

- [**Bun**](https://bun.sh) `v1.1+` — `curl -fsSL https://bun.sh/install | bash`
  (Windows: `powershell -c "irm bun.sh/install.ps1 | iex"`)
- A **PostgreSQL** database (local or hosted, e.g. Neon/Supabase)
- API keys for:
  - **OpenAI** — Realtime API access (`gpt-realtime`)
  - **Google Gemini** — used to score the interview
  - **Deepgram** — live speech-to-text on the frontend
  - An **HTTPS proxy** URL — used by the GitHub scraper

## Setup

### 1. Clone & install

```bash
git clone https://github.com/Priyabartkumar/RkruterAI.git
cd RkruterAI
bun install
```

`bun install` at the repo root installs dependencies for all workspaces.

### 2. Configure environment variables

Create `apps/backend/.env` (see `apps/backend/.env.example`):

```bash
# PostgreSQL connection string
DATABASE_URL="postgresql://user:password@host:5432/dbname"

# OpenAI key with Realtime API access
OPENAI_KEY="sk-..."

# Google Gemini API key (interview scoring)
GEMINI_API_KEY="..."

# HTTPS proxy used by the GitHub scraper
PROXY_URL="http://user:pass@proxy-host:port"
```

Bun loads `.env` automatically — no `dotenv` import required.

> **Frontend Deepgram key:** the Deepgram token is currently read inline in
> `apps/frontend/src/components/Interview.tsx` (~line 83). Paste your Deepgram
> API key there. _Note: shipping a prod key to the browser is insecure — the
> code has a TODO to switch to ephemeral keys before any real deployment._

### 3. Set up the database

From `apps/backend`, apply the Prisma migrations and generate the client:

```bash
cd apps/backend
bunx prisma migrate deploy    # apply existing migrations
bunx prisma generate          # generate the Prisma client into ./generated
```

During development you can use `bunx prisma migrate dev` instead to create new
migrations. Inspect data with `bunx prisma studio`.

## Running the app

Open two terminals.

**Backend** (Express API on `http://localhost:3001`):

```bash
cd apps/backend
bun --hot index.ts
```

**Frontend** (React SPA on `http://localhost:3000`):

```bash
cd apps/frontend
bun run dev
```

Then open **http://localhost:3000**. The frontend talks to the backend at
`http://localhost:3001` (configured in `apps/frontend/src/lib/config.ts`).

## API overview

| Method | Route                                       | Purpose                                        |
| ------ | ------------------------------------------- | ---------------------------------------------- |
| `POST` | `/api/v1/pre-interview`                      | Scrape a GitHub profile, create an interview   |
| `POST` | `/api/v1/session/:interviewId`               | Establish the OpenAI Realtime WebRTC session   |
| `POST` | `/api/v1/session/user/response/:interviewId` | Persist a user message to the transcript       |
| `GET`  | `/api/v1/result/:interviewId`                | Fetch (and lazily compute) the score & feedback |

## Building for production

```bash
cd apps/frontend
bun run build     # outputs to apps/frontend/dist
```

The backend runs directly with Bun in production:

```bash
cd apps/backend
NODE_ENV=production bun index.ts
```

## Notes

- This project uses **Bun**, not Node.js. Use `bun`/`bunx` instead of
  `node`/`npx`. See each app's `CLAUDE.md` for conventions.
- Secrets live in `.env` files, which are gitignored — never commit real keys.
