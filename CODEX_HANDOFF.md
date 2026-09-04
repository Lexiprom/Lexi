# CODEX_HANDOFF.md — Lexi project state

## 1. Product
Lexi is a personal flashcard learning web app that may later become a public/commercial product.

Core concepts:
- folders and decks;
- spaced repetition;
- typed-answer mode;
- folder-specific review;
- progress categories;
- paused/resumable sessions;
- OCR import;
- manual card creation;
- duplicate detection;
- export/import;
- AI mnemonic associations in text or image form.

The app name is simply **Lexi**.

## 2. Important product requirements already decided

### New accounts
A brand-new account must start empty:
- 0 folders
- 0 decks
- 0 cards
- 0 progress

Old local data must not be copied to every new account.

### Existing local-data migration
Existing browser data should migrate only to the user's first real account on that browser/device, then be synced to the account.

Do not delete local data until migration/sync has been verified.

### Typed answers
If a stored answer contains alternatives separated by comma, `/`, `;`, `|`, bullets, etc., any correct individual alternative or any subset of correct alternatives should count as correct.
Adding an unrelated wrong alternative should make the answer wrong.
Small typo tolerance is allowed.

Example stored answer:
`випадок, подія, пригода`

Correct:
- `подія`
- `пригода`
- `подія, пригода`
- all three

Incorrect:
- correct alternatives plus an unrelated wrong answer

### Direction-separated learning state
Knowledge must be tracked separately for:
- A → B
- B → A

One direction must not repair or hide weakness in the other.

Intended behavior:
- first learning tests both directions at least once;
- do not show the same card twice in a row when the deck has at least 2 cards;
- first direction per card can be randomized;
- `Не знаю` creates recovery need for that same direction; it requires two later successful recalls in that direction;
- `Важко` requires one later successful recall in that direction and has milder interval reduction (~0.7);
- later review prioritization weights roughly:
  - `Знаю`: 1
  - `Важко`: 3
  - `Не знаю`: 5
  - recovery: strongest priority
- if both directions are equally strong, direction selection is 50/50;
- a card cannot be globally "good" while one direction failed or is untested;
- internal direction state should remain hidden from ordinary UI.

### Associations
Associations are chosen during study after revealing the answer, not in deck edit menus.

Formats:
- Text
  - AI
  - manual
- Image
  - AI
  - gallery/manual

Once selected, the format chooser disappears; `Змінити формат` can change it.

Text mnemonic style:
- short;
- concrete;
- vivid;
- absurd/funny when useful;
- connect word and meaning;
- exploit sound similarity/word chunks where natural;
- no leaked model reasoning.

Example concept:
`вербальний → словесний`: imagine a willow (`верба`) with words hanging instead of leaves.

AI images:
- should visualize the mnemonic idea;
- no text/captions/logos unless explicitly needed;
- safe/family-friendly;
- no celebrity/copyrighted-character likenesses.

## 3. Backend/account state already reached

Reported working state:
- Cloudflare Worker deployed as a new clean Worker after the previous Worker became corrupted.
- Public `workers.dev` route enabled.
- `/api/health` works.
- `AI`, `DB`, `ADMIN_EMAIL` were tested and returned `true`.
- Registration/login work.
- User successfully logged into the real account.
- D1 shows developer account:
  - `role = admin`
  - `plan = pro`
- Developer/admin is intended to have permanent Pro.
- Normal registrations get a 30-day Pro trial.
- Session lifetime currently intended as 30 days.
- Existing local decks/folders were recovered in the normal browser session.

Important runtime detail:
Cloudflare Workers PBKDF2 rejected 210,000 iterations:
`iteration counts above 100000 are not supported`.
The deployed code was manually changed to:
`iterations: 100000`

Do not blindly restore 210,000 in the current Worker runtime.

## 4. Intended Worker API

Expected endpoints:
- `GET /api/health`
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `POST /api/auth/logout`
- `DELETE /api/account`
- `GET /api/export`
- `GET /api/state`
- `POST /api/state`
- `POST /api/association`
- `POST /api/image`
- `POST /api/billing/checkout` — currently intentionally not configured

Expected D1 tables:
- `users`
- `sessions`
- `user_state`
- `usage_daily`
- `audit_log`

Reported auth design:
- PBKDF2-SHA256
- per-user salt
- random session token
- hash stored in D1
- 30-day session expiry

## 5. AI backend requirements

Cloudflare Workers AI only. Earlier OpenRouter/Gemini-external paths should not remain as production dependencies.

Intended models:
- text primary: `@cf/zai-org/glm-4.7-flash`
- text fallback: `@cf/google/gemma-4-26b-a4b-it`
- image: `@cf/black-forest-labs/flux-2-klein-4b`

Current intended Pro/trial daily caps:
- 40 text mnemonic generations/day
- 8 image generations/day

AI endpoints must enforce Pro/trial on the server, not only in the frontend.

## 6. Sync work — CURRENT NEXT TASK

This is where development stopped.

The Worker `POST /api/state` was manually replaced with a revision-aware version.

Intended protocol:
- frontend sends state with `base_revision`;
- Worker compares it with the current server revision;
- mismatch returns HTTP 409 with:
  - `error: "STATE_CONFLICT"`
  - current server `state`
  - current `revision`
  - current `updated_at`
- successful write increments revision;
- frontend must not blindly overwrite a newer server state.

The frontend version prepared in ChatGPT was intended to:
- use revision-based sync;
- handle conflicts;
- sync after returning online;
- avoid resurrecting deleted decks/folders;
- sync paused/incomplete study state;
- sync OCR-created decks.

IMPORTANT: verify the current repository before assuming all of this frontend logic is actually present. The user was in the middle of replacing `index.html`.

### Required sync test plan
Do not mark sync complete until these pass:

1. Existing data survives frontend upgrade.
2. Normal browser session syncs successfully.
3. Login in a second session/device loads server data.
4. Create a test deck on session A → sync → appears on B.
5. Edit on B → sync → appears correctly on A.
6. Delete on one session → deletion does not resurrect after the other syncs.
7. Offline edit → reconnect → sync succeeds.
8. Concurrent edits on A and B cause deterministic conflict handling, not silent data loss.
9. Paused study session/progress survives device switch.
10. OCR-created deck syncs.
11. Logout/login does not lose server data.

Prefer automated tests for the merge/conflict logic where feasible.

## 7. Block A remaining work

Finish in this order unless code inspection suggests a dependency change:

### A1. Sync
- finish/verify revision protocol;
- multi-device conflict resolution;
- offline → online sync;
- deletion/tombstone handling;
- migration from local-only data;
- tests.

### A2. AI integration
- test text mnemonic generation;
- test image generation;
- verify 40/8 quotas;
- verify admin Pro;
- verify trial Pro;
- verify expired/free user receives `PRO_REQUIRED`;
- graceful error messages.

### A3. Account security
- email verification;
- forgot-password/reset flow;
- transactional email provider integration;
- login/register rate limiting;
- anti-bot/abuse controls;
- session cleanup/revocation;
- review password storage under Cloudflare runtime constraints.

Do not fake email verification/reset if no email provider/domain exists.

### A4. CORS/security
Current development may allow `Access-Control-Allow-Origin: *`.
Before public release:
- set production `ALLOWED_ORIGIN`;
- allow only the Lexi frontend origin;
- verify OPTIONS/preflight;
- audit auth endpoints.

### A5. Account management
- export UI;
- delete-account UI;
- logout;
- expired-session handling;
- verify backend deletion really removes user data.

### A6. PWA/cache
Files expected:
- `manifest.webmanifest`
- `sw.js`

Earlier service worker used network-first caching and a versioned cache.
Fix/update logic so users do not remain stuck on an old `index.html`.
Test install/update/offline behavior.

### A7. Reliability
- useful frontend error messages instead of generic "Не вдалося підключитися";
- Worker logs/observability;
- quota monitoring;
- D1 backup/export/recovery plan;
- failure-mode testing.

### A8. Legal/public-release foundation
Before public launch:
- Privacy Policy
- Terms
- data deletion policy

### A9. Billing
Server route may exist but real payment integration is intentionally unfinished.
Do not implement a payment provider until Free/Pro product boundaries and account/legal ownership are decided.

## 8. Block B — DO NOT START YET
Later:
- design/UI;
- learning-flow polish;
- Free vs Pro feature package;
- statistics;
- mnemonic UI polish;
- visual redesign;
- further algorithm tuning.

The user explicitly wants infrastructure finished first so it does not have to be repeatedly revisited.

## 9. Historical deployment issues to avoid repeating
- Old Worker editor became corrupted/empty.
- Bindings appeared to disappear.
- A clean Worker was created instead.
- `workers.dev` initially showed "No URLs enabled"; route had to be enabled.
- Registration initially failed because PBKDF2 requested 210,000 iterations; Cloudflare runtime accepted max 100,000.
- Normal browser initially used stale backend configuration/local state while incognito worked.
- Do not clear normal browser site data while legacy/local data migration is still relevant.
- PWA/service-worker caching can make an old frontend look current.

## 10. First Codex task
1. Inspect the repository completely.
2. Identify:
   - current `index.html`
   - Worker source if present
   - `sw.js`
   - `manifest.webmanifest`
   - schema/migration files
3. Run syntax/tests.
4. Compare actual implementation with this handoff.
5. Report mismatches.
6. Continue only Block A, starting with sync.
7. Make the smallest safe set of changes required to complete the sync test plan.
8. Do not redesign the UI.
