# AGENTS.md — Lexi

## Mission
Continue development of Lexi, a flashcard/spaced-repetition web app.

The repository is the source of truth for code. This file and `CODEX_HANDOFF.md` describe product intent and known project history. If they conflict with the current code, inspect the code first and report the mismatch before changing behavior.

## Working rules
1. Do not rewrite the app from scratch unless explicitly requested.
2. Preserve existing user data formats and migration paths.
3. Before edits:
   - inspect the repository;
   - identify the current frontend/backend/PWA files;
   - run available syntax checks/tests;
   - note any mismatch with `CODEX_HANDOFF.md`.
4. After edits:
   - run syntax/tests again;
   - summarize changed files and behavior;
   - call out any manual Cloudflare/GitHub action still required.
5. Never claim a feature is implemented without verifying the actual code.
6. Do not expose API keys, secrets, admin email, session tokens, or private data in commits/logs.
7. Keep Block A (infrastructure) separate from Block B (product/UI). Finish Block A first.
8. Current user edits/deploys mostly from a phone. Prefer changes that can be committed directly. If manual editing is unavoidable, give exact file name, search string, and approximate/current line number.
9. Keep manual code snippets compact when possible, but prefer correctness/readability inside the repository.
10. Do not silently change product requirements.

## Validation
For JavaScript:
- run `node --check` on standalone JS files when applicable;
- for inline scripts in HTML, extract/check them or use the project's existing validation method;
- verify service-worker syntax separately.

For backend changes:
- preserve Cloudflare Workers ES-module format;
- verify API routes and CORS behavior;
- preserve D1 compatibility/migrations;
- do not assume bindings exist: document required bindings.

## Architecture
Current intended architecture:
- Frontend: GitHub Pages
- Backend: Cloudflare Worker
- AI: Cloudflare Workers AI
- Database: Cloudflare D1
- PWA: manifest + service worker
- Accounts/sync: Worker API + D1

Expected Worker bindings/variables:
- `AI` — Workers AI binding
- `DB` — D1 database binding
- `ADMIN_EMAIL` — developer/admin email
- `ALLOWED_ORIGIN` — production frontend origin when CORS is locked down

Do not hardcode secrets.

## Product boundary
### Block A — finish before Block B
Accounts, security, sync, AI backend, quotas, email verification/reset, anti-abuse, offline/online sync, account export/delete, PWA/cache reliability, monitoring/errors, backups/recovery, deployment/security checks.

### Block B — later
Learning UX, visual design, Free/Pro product design, study algorithm refinements, associations UI, statistics, polish.

Do not spend time redesigning UI while Block A is unfinished unless a UI change is required to test infrastructure.
