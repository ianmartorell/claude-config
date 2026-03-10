# Lubo Project Memory

## What is Lubo
- **Routine-first personal knowledge system** — "love your routines"
- Automates recurring workflows: daily focus, weekly reviews, backlog pruning
- Name: Irish *lúb* (loop) + Slavic *lubo* (beloved) = "beloved routines"
- Status: Ideation → PoC (March 2026)

## Product Docs (Notion)
- Main page: `31c179c60963818dbb46db250d93b2f1`
- PoC Spec: `31c179c609638105ba8df292291cf27f`
- Investment Memo: `31c179c60963813e82bdf9cb7e649b8f`
- Competitor Landscape: `31e179c6096381fb8bf3e110edf80ce3`
- Dead Startups Research: `31e179c60963817cb1d3cba71d4e6d2d`

## Architecture Decisions
- **Frontend:** Next.js (App Router), React, Tailwind, Zustand
- **Backend:** Supabase (Postgres + RLS + Auth) + Next.js API routes (replaces Edge Functions)
- **AI:** Claude API with streaming (Sonnet)
- **Integrations:** Google Calendar + Linear (v1 task integration)
- **Auth:** Supabase magic link (passwordless)
- **Data model:** Typed graph (projects, tasks, notes, meetings, sources) with JSONB metadata
- **Single project** (not monorepo) — everything in one Next.js app + `supabase/` dir

## v1 Validation Target
- Daily Routine: triage yesterday → Big 3 → inbox sweep
- Success: 10 consecutive completions, <5 min each, faster than manual
- Weekly Review and Backlog Pruning are planned but NOT v1

## Key Design Choices
- Routines are first-class objects (trigger, scope, steps, adaptation)
- **Chat-driven routine UX:** Split panel — chat (left) navigates the routine, routine panel (right) shows live state
- **Dual interaction:** User can click buttons on task cards OR type in chat; both update same state
- AI uses **Claude tool use** (function calling) to update routine state during chat
- Adaptation layer tracks user behavior and auto-adjusts (rule-based in v1)
- External integrations are read-primary; write-back only during routine execution
- **Visual style:** Warm / friendly — soft colors, rounded corners, gentle gradients
- **Graph model:** Hybrid — single `nodes` table with promoted columns + JSONB `data`

## Deployment
- **App:** https://lubo.vercel.app (Vercel, auto-deploys from `main`)
- **Database:** Supabase project `iejvipsxbjxbemxoalje`
- **NEVER deploy to prod manually** (`vercel --prod`) — let Vercel auto-deploy on merge to main. Use preview deployments from PR branches for testing.

## Repo State
- Path: `/Users/kayleigh/dev/Lubo` (same as `/Users/kayleigh/Dev/Lubo` — macOS case-insensitive)
- Git repo, main branch tracks origin/main
