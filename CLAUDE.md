# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LTI is a full-stack recruitment/talent tracking system (ATS). It's a monorepo with a **backend** (Node.js/Express/TypeScript) and **frontend** (React/TypeScript/CRA). The database is PostgreSQL accessed via Prisma ORM, run locally through Docker Compose.

## Common Commands

### Database
```bash
docker-compose up -d                          # Start PostgreSQL
cd backend && npx prisma generate             # Generate Prisma Client
cd backend && npx prisma migrate dev          # Run migrations
cd backend && npx ts-node prisma/seed.ts      # Seed database
```

### Backend (runs on port 3010)
```bash
cd backend && npm install
cd backend && npm run dev                     # Dev server with hot reload
cd backend && npm run build                   # Compile TypeScript to dist/
cd backend && npm test                        # Run all Jest tests
cd backend && npx jest path/to/file.test.ts   # Run a single test file
```

### Frontend (runs on port 3000)
```bash
cd frontend && npm install
cd frontend && npm start                      # Dev server
cd frontend && npm run build                  # Production build
cd frontend && npm test                       # Jest unit tests
cd frontend && npm run cypress:open           # Cypress E2E (interactive)
cd frontend && npm run cypress:run            # Cypress E2E (headless)
```

## Architecture

### Backend — DDD / Clean Architecture layers

```
backend/src/
├── domain/models/          # Entities (Candidate, Position, Application, Interview, etc.)
├── application/
│   ├── services/           # Business logic (candidateService, positionService, fileUploadService)
│   └── validator.ts        # Input validation
├── presentation/
│   └── controllers/        # Express request handlers
└── routes/                 # Express route definitions
```

- PrismaClient is attached to `req.prisma` via middleware in `src/index.ts`, so controllers/services access it from the request object.
- Tests are colocated with source (`*.test.ts` next to the file they test).
- Prisma schema: `backend/prisma/schema.prisma`

### Frontend

```
frontend/src/
├── components/             # React components (AddCandidateForm, RecruiterDashboard, Positions, etc.)
├── services/               # API client layer
└── assets/                 # Static assets
```

- Uses React Bootstrap for UI and react-beautiful-dnd / react-dnd for drag-and-drop.
- Cypress E2E specs live in `frontend/cypress/integration/`.

### API Endpoints

- `POST /candidates` — Create candidate with education/experience/CV
- `GET /candidates/:id` — Get candidate details
- `PUT /candidates/:id` — Update candidate interview stage
- `POST /upload` — Upload files (PDF, DOCX)
- `GET /positions` — List visible positions
- `GET /positions/:id/candidates` — Candidates for a position
- `GET /positions/:id/interviewflow` — Interview flow for a position

## CI/CD Pipeline

The project goal is a GitHub Actions pipeline (`.github/workflows/`) that triggers on push to a branch with an open PR and:
1. Runs backend tests
2. Builds the backend
3. Deploys the backend to an AWS EC2 instance

Required GitHub Secrets: `EC2_HOST`, `EC2_SSH_KEY`.

## Environment

The root `.env` file provides `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_PORT`, and `DATABASE_URL` for PostgreSQL. The `docker-compose.yml` reads from this `.env` to configure the Postgres container.
