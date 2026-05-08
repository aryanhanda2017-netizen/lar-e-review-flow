# LAR-E — Project Context

**Product:** AI-powered B2B SaaS for Indian SMBs (luxury real estate, executive coaching, B2B services). Takes dormant lead CSVs → classifies (Tier A/B/C) → sends personalised WhatsApp → Claude agent qualifies → books appointments → notifies sales team when warm.

**One-liner:** "Goes back to people who were once interested in your business but never bought, and gets them to buy."

---

## Infrastructure

| Layer | Service | URL |
|-------|---------|-----|
| Backend (live) | Railway (FastAPI) | https://web-production-7f7ad.up.railway.app/ |
| Frontend (live) | Bolt (DEPRECATED) | https://lar-e-frontend-desig-f3lg.bolt.host |
| Frontend GitHub | `aryanhanda2017-netizen/LARE-FRONTEND` | branch: `main` |

**Railway start:** `alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port $PORT`

---

## Tech Stack

**Backend:** Python 3.12 · FastAPI async · PostgreSQL + SQLAlchemy 2.0 + asyncpg · Redis + arq · Alembic · Anthropic Claude (primary) · OpenAI (fallback) · Pydantic v2

**Frontend (NEW):** Vite + React 19 + TypeScript strict · TanStack Query v5 · Tailwind + shadcn/ui · Axios · React Router v6

**Frontend (OLD — DEPRECATED):** Next.js 14 / Bolt — do not touch

---

## Codebase Layout

```
LAR-E NEW/
├── CLAUDE.md                          ← Behavioral guidelines
├── PROJECT_MEMORY.md                  ← Full build history + API reference
├── .claude/
│   ├── CLAUDE.md                      ← This file
│   ├── CLAUDE.local.md                ← Local paths/session notes (gitignored)
│   └── rules/
│       ├── backend.md · frontend.md · smoke-tests.md · design.md
├── Code — Backend/
│   ├── lare-backend-documents/        ← PRIMARY — matches Railway. Edit this.
│   └── lare-backend-desktop/          ← STALE — do not edit
├── Product & Research/
├── Brand & Identity/
└── docs/                              ← Superpowers plans + design prototype
```

> ⚠️ Always confirm you're in `lare-backend-documents/` before editing backend files.

---

## Current Status (April 2026)

**Working ✅** Auth · Onboarding · Lead CSV upload + tiering · Campaign auto-approve + launch · Lead list/detail/transcript · Appointments list · Campaigns list · Dashboard summary · CORS on Railway

**Known Gaps 🟠** Campaign detail · Appointment detail · Conversations list · Dashboard activity components · Price range dropdown · Campaign pause/resume UI · Lead notes · Lead list auto-refresh

**Pre-MVP Blockers 🔴** WhatsApp BSP signup (WATI/AiSensy) · 3 WhatsApp templates to Meta · First pilot client

---

## Priority Queue

1. Execute frontend rebuild (`docs/superpowers/plans/2026-04-16-frontend-plan-*.md`)
2. Execute 7 backend fixes (`docs/superpowers/plans/2026-04-16-backend-fixes.md`)
3. Resume landing page
4. Sign WhatsApp BSP
