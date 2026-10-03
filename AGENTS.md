# Base44 Dev Notes — Toxic-MD

## What this is
Toxic-MD is a WhatsApp Multi-Device bot (Node.js + Baileys) with a small Express
web server. The web server is the preview entry point; the WhatsApp connection
is a separate background process that needs a session credential.

## Running it
```
docker compose -f docker-compose.base44.yml up -d
```
- Web/health: http://localhost:3000 (`/` serves `public/index.html`, `/health` returns JSON).
- Healthcheck uses `curl` against `/health` (curl is installed in `Dockerfile.base44`).

## Architecture / startup order
1. `npm install` (no lockfile in repo — uses `npm install`, not `npm ci`).
2. `node patch-baileys.js` pre-applies two Baileys source patches.
3. A `node -e` snippet pre-applies the **usync.js timeout patch**. This is critical:
   `index.js` has an autopatch block at the very top that patches Baileys files and,
   if it patches `usync.js`, calls `process.exit(0)` + respawns. If `node index.js`
   is PID 1 that exit kills the container. Pre-applying all patches means the
   autopatch finds nothing to do and never restarts.
4. `exec node keep-alive.js` — the project's own supervisor that spawns `index.js`
   and respawns it on crash or on its `fs.watchFile` self-restart (triggered when
   `index.js` itself is edited). This keeps PID 1 alive across restarts.

## Database
The DB layer (`database/config.js`) prefers PostgreSQL when `DATABASE_URL` is set,
else falls back to a JSON file (`whatsasena.json`, written in the repo root and
persisted via the bind mount). The `Pool` config **hardcodes `ssl: { rejectUnauthorized: false }`**,
so it only works against an SSL-enabled hosted Postgres; a local non-SSL Postgres
always fails and falls back to JSON. We intentionally leave `DATABASE_URL` unset
and use the JSON backend — no code changes needed, data persists across restarts.

## SESSION (WhatsApp session) — the only external credential
- `SESSION` is a base64-JSON WhatsApp session ID from the pairing site
  (https://toxicx.tech/pairing). It is **NOT required to boot**: without it the web
  page still loads and the process stays healthy; the bot just logs
  "No session detected" and does not connect to WhatsApp.
- Provide it via the Base44 secrets dashboard; it reaches the container through
  `/run/base44/app.env` (wired as the last `env_file` in compose).
- API keys for AI/media integrations are baked into the obfuscated `keys.js`, not
  env vars, so no other credentials are needed to boot.

## Editing
- `public/` is served via `express.static` from the bind-mounted source — edits to
  `public/index.html` appear on the next request (no rebuild).
- Other source edits: `index.js` watches itself and self-restarts via `keep-alive.js`;
  for changes elsewhere, run `docker compose -f docker-compose.base44.yml restart bot`.
- After backend/compose changes, call `reload_preview` so the preview refreshes.
