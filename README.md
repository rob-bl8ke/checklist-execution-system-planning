# Checklist Execution System

A single-user **Runbook + Todo manager** for engineers performing repeatable operational tasks. Define step-by-step templates once, then start a run and execute steps in order — with variable substitution, markdown instructions, and a "Today" dashboard showing your active runs and outstanding todos.

---

## Repositories

| Repo | Description |
|---|---|
| [checklist-execution-system-planning](https://github.com/rob-bl8ke/checklist-execution-system-planning) | Architecture, spec, planning docs, Docker Compose |
| [checklist-execution-system-api](https://github.com/rob-bl8ke/checklist-execution-system-api) | NestJS REST API (TypeScript + SQLite) |
| [checklist-execution-system-ui](https://github.com/rob-bl8ke/checklist-execution-system-ui) | Angular 19 SPA frontend |

---

## Architecture Overview

```
┌─────────────────────┐        HTTP/JSON       ┌─────────────────────┐
│   Angular 19 SPA    │  ─────────────────────► │   NestJS REST API   │
│   (port 80 / 4200)  │   /api/*               │   (port 3000)       │
│                     │                         │                     │
│  - Templates        │                         │  - TypeORM + SQLite │
│  - Runs             │                         │  - Migrations       │
│  - Todos            │                         │  - Swagger /api/docs│
│  - Today dashboard  │                         │                     │
└─────────────────────┘                         └──────────┬──────────┘
                                                           │
                                                    checklist.db
                                                   (SQLite on disk)
```

**Key design decisions:**
- Single-user, local-first — no auth
- Steps are **copied** from templates into runs at start time; editing a template doesn't affect active runs
- Variables in step instructions use `{{variable}}` syntax, substituted at run creation
- Instance status: `IN_PROGRESS` → `COMPLETED` (auto when all steps done) or `ABANDONED`

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/) v2+
- For local dev: Node.js 22 (`nvm use` from either app repo)

---

## Running via Docker Compose

Clone all three repos into the same parent directory:

```bash
git clone https://github.com/rob-bl8ke/checklist-execution-system-planning
git clone https://github.com/rob-bl8ke/checklist-execution-system-api
git clone https://github.com/rob-bl8ke/checklist-execution-system-ui
```

From the **parent directory** (where `docker-compose.yml` lives):

```bash
# Build images and start all services
docker compose up --build

# Stop services
docker compose down

# Stop and remove the database volume (full reset)
docker compose down -v
```

Services:
- **UI** → `http://localhost:80`
- **API** → `http://localhost:3000/api`
- **Swagger** → `http://localhost:3000/api/docs`

The SQLite database is persisted in the `db-data` Docker volume at `/app/data/checklist.db`. Migrations run automatically on API startup.

---

## Local Development

For a faster dev loop, run the backend and frontend separately. See each repo's README:

- [Backend setup](https://github.com/rob-bl8ke/checklist-execution-system-api#readme)
- [Frontend setup](https://github.com/rob-bl8ke/checklist-execution-system-ui#readme)

---

## Planning Docs

- [`docs/full-spec.md`](docs/full-spec.md) — Original product specification
- [`docs/spec-plan.md`](docs/spec-plan.md) — Phased implementation plan with task breakdown
