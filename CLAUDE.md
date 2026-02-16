# CLAUDE.md — OTVP (RLS Scanner)

## What This Repo Does

Standalone Python tool that detects Supabase Row Level Security (RLS) misconfigurations. This was the starting point for the broader OTVP platform.

**Two modes:**
1. **Self-audit** (`supabase_rls_tester.py`) — Tests your own Supabase project for RLS gaps using the anon key. Read-only, safe for production.
2. **Recon** (`supabase.py`) — Crawls target websites to find exposed Supabase anon keys in JS bundles, then tests them for RLS vulnerabilities. **Only use on targets you own or have written authorization to test.**

## Why This Exists

Supabase exposes databases via PostgREST. Every table in the `public` schema is queryable through the REST API. RLS is the primary access control — if it's missing or misconfigured, data is exposed to anyone with the anon key (which is intentionally public).

This tool validates whether the gap between "schema is exposed" and "RLS is enforced" is properly closed.

## Part of a Larger System

This repo is the narrow, standalone version. The broader OTVP platform lives across sibling repos:
- **otvp-sdk** — 11-control agent framework (this repo only covers RLS)
- **otvp-app** — Next.js web interface for running agents and viewing results
- See `~/otvp-projects/CLAUDE.md` for the full picture

## Technical Details

- **Language:** Python
- **Virtual environment:** `otvp-env`
- **Key dependencies:** requests, beautifulsoup4 (for recon mode)
- **Output:** `results.txt` (local only, never commit this)

## Development Rules

- All scanning is **read-only** (GET requests only)
- No data leaves the local machine unless the operator exports it
- Never hardcode URLs, keys, or credentials
- `.gitignore` must include: `results.txt`, `.env`, `*.log`, `__pycache__/`
- Test only against projects you own

## Supabase Security Context

Key concepts for anyone working on this code:
- **Anon key** = public, grants the `anon` Postgres role, should be restricted by RLS
- **Service role key** = god-mode, bypasses RLS entirely, must NEVER be exposed client-side
- **PostgREST** = REST API auto-generated from DB schema
- **RLS** = Row Level Security, Postgres-native access control using policies
- **Auth model:** GoTrue issues JWTs → PostgREST enforces RLS via `auth.uid()` and `auth.role()`
