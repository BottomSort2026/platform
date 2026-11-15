# Plan: Wire services via docker compose (unfinished parts mocked)

Branch: `feat/wire-services-compose` (in `platform/` and `frontend/`)
Created: 2026-11-15

## Goal
`docker compose up` in `platform/` plus `pnpm dev` in `frontend/` gives a working demo:
gateway (8080) → mcp (8001) → engine (8000), chat answered through the Groq LLM, and the
frontend pointed at the gateway with the not-yet-finished backend routes falling back to
the built-in sample data instead of erroring.

## Context
- Compose file (`platform/docker-compose.yaml`) is committed and correct: env names, ports,
  healthchecks, `depends_on`, `host.docker.internal:host-gateway` all verified against the
  services' code. `docker compose config` passes.
- Reviewed gaps when the frontend points at the gateway (`frontend/src/api/client.ts:53-73`,
  `hasBackend` is all-or-nothing):
  - `GET /ml/model-info` → 404 (engine has no such route).
  - `POST /ml/predict` → 422 (frontend sends `{field: value}`; engine placeholder wants
    `{features: [4 floats]}`).
  - `GET /profiles` → 404 (default `PROFILES_MODE=static`, no `profiles.json` mounted;
    proxy mode also 404s — mcp has no `/profiles` route).
  - `POST /chat` works only if `LLM_URL` answers.
- LLM decision: Groq via `platform/.env` (OpenAI-compatible; mcp sends `Bearer` auth,
  `mcp/src/provider.ts:11-21`). `platform/.env` currently still contains only the commented
  example — the key must be pasted in.
- Frontend dev server proxies `/profiles`, `/chat`, `/ml` to `GATEWAY_URL`
  (`frontend/vite.config.ts`), so `VITE_API_URL=/` keeps the browser same-origin, no CORS.

## Approach
1. `platform/.env`: add Groq lines
   `LLM_URL=https://api.groq.com/openai/v1/chat/completions`,
   `LLM_MODEL=llama-3.3-70b-versatile`, `LLM_API_KEY=<user's key>` (gitignored).
2. `frontend/src/api/client.ts`: per-route mock fallback — `profiles`, `modelInfo`,
   `predict` fall back to the existing `mock.ts` when the gateway answers 404/422;
   `/chat` stays on the gateway. ~8 lines, no new files or dependencies.
3. `frontend/.env.local` (gitignored, from `frontend/.env.example`): `VITE_API_URL=/`,
   `GATEWAY_URL=http://localhost:8080`.
4. `platform/docker-compose.yaml` (optional, user to confirm): `restart: unless-stopped`
   on the three services.

## Files / Areas
- `platform/.env` (not committed — gitignored)
- `platform/docker-compose.yaml` (only if step 4 is wanted)
- `frontend/src/api/client.ts`
- `frontend/.env.local` (not committed — gitignored)
- No service code (mcp/engine/gateway) changes.

## Verification
- `cd platform && docker compose config` still passes.
- `cd platform && docker compose up --build`: all three containers healthy;
  `curl -s localhost:8080/health` → gateway ok;
  `curl -s -X POST localhost:8080/chat -H 'content-type: application/json' \
    -d '{"message":"how does CO2 suppress a heptane droplet flame"}'` → answer with citations.
- `cd frontend && pnpm dev`: Information and Simulator views show sample data (mock
  fallback), Chatbot answers through the stack, browser network shows same-origin :5173.

## Open Questions
- Confirm Groq model id (`llama-3.3-70b-versatile` proposed; user may prefer another).
- Step 4 (`restart: unless-stopped`): include or skip.
