# Contributing to Ágora Global

Thank you for your interest in contributing to Ágora Global. This document explains how to set up the local environment, the coding conventions, and the pull request process.

Before writing any code, read [ARCHITECTURE.md](ARCHITECTURE.md) — it covers the system design, the database schema, the AI agent model, and the technical decisions that shape every contribution.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Setup](#local-setup)
3. [Project Structure](#project-structure)
4. [Coding Conventions](#coding-conventions)
5. [Running Tests](#running-tests)
6. [Opening an Issue](#opening-an-issue)
7. [Pull Request Process](#pull-request-process)
8. [Upstream-First Policy](#upstream-first-policy)

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Node.js | 18+ | Next.js frontend |
| Python | 3.11+ | Statistical agents |
| Docker + Docker Compose | latest | Local container for Python agents |
| Supabase CLI | latest | Migrations and local DB |
| Git | any | Version control |

Install the Supabase CLI:

```bash
npm install -g supabase
```

---

## Local Setup

```bash
# 1. Clone the repo
git clone https://github.com/CabPiz/agora-global.git
cd agora-global

# 2. Install Node.js dependencies
npm install

# 3. Install Python dependencies (inside agents/)
cd agents
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cd ..

# 4. Copy the environment template and fill in the values
cp .env.example .env.local

# 5. Start the local Supabase instance
supabase start

# 6. Apply all migrations
supabase db push

# 7. Start the Next.js dev server
npm run dev

# 8. (separate terminal) Start the Python agents container
docker compose up agents
```

The app will be at `http://localhost:3000`.

### Environment Variables

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key (server-side only) |
| `ANTHROPIC_API_KEY` | Anthropic API key for the Citizen Report Writer agent |
| `MUNICIPALITY_IBGE_CODE` | IBGE code for the pilot municipality (default: `3550308` = São Paulo-SP) |
| `ATIPICITY_THRESHOLD` | Score above which a contract enters the review queue (default: `0.40`) |
| `INNGEST_SIGNING_KEY` | Inngest signing key |
| `INNGEST_EVENT_KEY` | Inngest event key |

---

## Project Structure

```
agora-global/
├── app/              # Next.js 15 App Router (TypeScript)
├── lib/
│   ├── ai/           # Claude agent loop, Zod schemas, observability
│   └── supabase/     # Supabase server client
├── inngest/          # Async functions (cron + event-driven)
├── messages/         # next-intl translation files (pt-BR, en)
├── supabase/
│   └── migrations/   # PostgreSQL migrations
└── agents/           # Python statistical agents (Docker container)
    ├── anomaly/      # Benford, Isolation Forest, CNPJ risk, Z-score
    ├── ingestor.py   # PNCP → OCDS normalizer
    └── orchestrator.py
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full explanation.

---

## Coding Conventions

### TypeScript (Next.js)

- **Linter:** ESLint (`npm run lint`)
- **Formatter:** Prettier (`npm run format`)
- No `any` types — use proper types or `unknown`.
- All Supabase server-side calls use the service role client from `lib/supabase/server.ts`.
- All Claude API calls go through `tracedLLMCall` in `lib/ai/observe.ts` — never call the API directly.
- AI-generated text that reaches the UI must be validated by the `AnomalyReportSchema` Zod schema. The schema enforces the score-not-accusation principle at the type level.

### Python (agents/)

- **Linter + formatter:** `ruff` (`ruff check .` and `ruff format .`)
- Python 3.11+ type hints on all public functions.
- All database writes go through `db.py` — no direct Supabase client calls in analysis modules.
- Each analysis module (`benford.py`, `isolation_forest.py`, etc.) must return a list of `AuditFlag` dataclasses with `flag_code`, `score_weight`, and `evidence_data`. It must not write to the database itself.

### Commit messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(ingestor): add PNCP pagination support
fix(benford): correct digit frequency calculation for values < 10
docs(architecture): add V2 pgvector schema
chore(deps): bump pandas to 2.2.2
```

---

## Running Tests

```bash
# TypeScript unit tests
npm test

# TypeScript E2E tests (Playwright)
npm run test:e2e

# Python tests
cd agents && pytest
```

All tests must pass before opening a PR. The CI pipeline runs the full suite automatically.

---

## Opening an Issue

Before opening an issue, search existing ones to avoid duplicates.

Use the appropriate label:
- `bug` — something is broken
- `feature` — new functionality
- `area: ingestion` / `area: analysis` / `area: hitl` / `area: ui` — area of the system
- `area: governance` — governance and community

For bugs, include: steps to reproduce, expected behavior, actual behavior, and your environment.

For features, include: what problem it solves and a brief description of your proposed solution.

---

## Pull Request Process

1. **Open an issue first.** Every PR should be linked to an issue. Coordinate before investing time in implementation.

2. **Fork the repo** and create a branch with the pattern `feat/short-description` or `fix/short-description`.

3. **Write tests** for new behavior. PRs without tests for new logic will not be merged.

4. **Run the full test suite** before pushing. Fix any failures locally — don't push a broken suite.

5. **Open the PR** against `main` and fill in the PR template. Link the issue with `Closes #N`.

6. **Two approvals required** for merge (or one approval from a maintainer during the early project phase).

7. **Squash and merge** is the default strategy for keeping history clean.

### PR Checklist

- [ ] Linked to an open issue
- [ ] Tests added or updated
- [ ] `npm run lint` passes (TypeScript) / `ruff check .` passes (Python)
- [ ] All CI checks green
- [ ] `ARCHITECTURE.md` updated if the PR changes the schema, the folder structure, or a technical decision

---

## Upstream-First Policy

Ágora Global is the upstream core. National implementations (e.g., `agora-brasil`) are downstream forks.

**If you are contributing to a national fork:** any improvement that is not country-specific must be submitted to this repository first via PR. It can only flow to the fork after it is merged upstream. This prevents silent divergence and keeps the core applicable to all countries.

**If you are unsure** whether your change belongs in the core or a national fork: open an issue here and ask. The rule of thumb is: if the change works the same way for any country's open contracting data, it belongs in the core.

---

## Questions

Open a [Discussion](https://github.com/CabPiz/agora-global/discussions) for questions that don't fit an issue — architecture questions, onboarding help, or general feedback.

---

*Ágora Global · CONTRIBUTING.md · AGPL-3.0*
