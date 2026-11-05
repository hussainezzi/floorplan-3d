---
name: floorplan-ai-build
description: Build and deploy the white-label AI 3D home designer (floorplan-3d) on this shared VPS. Adds a backend with a DeepSeek AI agent that builds designs live, Deepgram voice input/voiceover, per-client branding/prices and lead capture, then puts it behind the EXISTING reverse proxy without disturbing any other container. Follow steps 1-100 in order.
argument-hint: "[start | resume <step-number> | status]"
disable-model-invocation: true
---

# Floorplan AI: build & deploy on a shared VPS (steps 1–100)

You are Claude Code running **on the production VPS**. The server already runs **other people's/other projects' containers** and a **reverse proxy** that serves live traffic. Your job is to add our product next to them **without breaking anything**.

Argument: `$ARGUMENTS`. `start` = begin at step 1. `resume N` = re-read the ground rules, run the step-1–12 recon again (read-only), then continue at step N. `status` = run steps 92–95 only.

Keep a running log at `/opt/floorplan/BUILD_LOG.md`: for every step, record the step number, what you did, and the result. On `resume`, read it first.

---

## GROUND RULES (read before every phase; they override any step below)

**Shared-server safety**
- R1. **Never stop, restart, recreate, remove, rename or `exec` into a container you did not create.** Our containers are only those in compose project `floorplan` (label `com.docker.compose.project=floorplan`).
- R2. **Forbidden commands:** `docker system prune`, `docker volume prune`, `docker network prune`, `docker image prune -a`, `docker rm`/`stop`/`kill` on non-floorplan containers, `docker compose down` in any directory other than `/opt/floorplan`, `apt upgrade`/`dist-upgrade`, `reboot`, `shutdown`, changes to SSH config, `ufw`/`iptables` changes, editing `/etc/docker/daemon.json`, restarting the Docker daemon.
- R3. **Never bind ports 80 or 443.** The existing reverse proxy owns them. Our containers either publish **nothing** (proxy reaches us over a Docker network) or bind **only `127.0.0.1:<free port>`**.
- R4. **Reverse-proxy changes:** back up any file before editing it (`cp file file.bak.$(date +%s)`). Only **add** our own site/route; never edit or delete other sites. Always validate before reloading (`nginx -t`, `caddy validate`, etc.), and **reload gracefully, never restart.** If validation fails, restore the backup immediately. If the proxy is managed by a UI (e.g. Nginx Proxy Manager), **don't touch its files or DB**. Give the user exact click-by-click instructions instead.
- R5. **Ask the user before:** installing any system package, editing any file outside `/opt/floorplan`, any reverse-proxy change, using a domain name, or anything you're unsure about. Show the exact command or diff first.
- R6. **Resource caps:** every container we create gets memory and CPU limits. If free RAM is under 1.5 GB or free disk under 5 GB, stop and report before building.
- R7. **Unique names:** compose project `floorplan`, network `floorplan_net`, container names prefixed `floorplan-`. Before creating anything, check the name isn't already used.

**Secrets**
- S1. **Never print secret values** (no `echo $KEY`, no `cat .env`, no `env`, no `docker inspect` output with env, no secrets in logs, commits or chat). To check that a key exists, use `[ -n "$DEEPSEEK_API_KEY" ] && echo set || echo missing`. To show `.env`, mask it: `sed -E 's/=.+/=****/' .env`.
- S2. Secrets live only in `/opt/floorplan/.env` (`chmod 600`). The DeepSeek and Deepgram keys are **server-side only**. They must never reach the browser, the frontend bundle, or git.
- S3. Never ask the user to paste a secret into chat. If you can't find a key, ask *where* it's stored (file path or env var name), not *what* it is.

**Engineering**
- E1. Work in small verified increments. After each phase, run its check before moving on.
- E2. Don't rewrite the existing `index.html` renderer. Add to it (an API bridge, a chat panel, tenant config). The 2D/3D/walkthrough must keep working exactly as before.
- E3. The AI may only change the design through **validated tool calls**. Prices always come from the tenant price list, never from the model.
- E4. Commit to git locally on branch `vps/ai-mvp` after each phase. **Don't push** unless the user confirms that push credentials exist and says to push.

---

## PHASE 1: Reconnaissance (read-only) · steps 1–12

1. Print the date, `hostname`, `whoami`, `id`, and whether passwordless sudo works (`sudo -n true && echo ok`). Note it in BUILD_LOG.
2. OS and resources: `cat /etc/os-release | head -3; nproc; free -h; df -h / /var/lib/docker 2>/dev/null`. Apply R6.
3. Docker: `docker version --format '{{.Server.Version}}'; docker compose version`. If Docker or Compose v2 is missing, **stop and ask** the user (don't install it on a shared box without permission).
4. List every running container: `docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Label "com.docker.compose.project"}}'`. Save it to BUILD_LOG as the **"do-not-touch list"**.
5. List networks: `docker network ls`. Note any that look like proxy networks (`proxy`, `traefik`, `web`, `nginx-proxy`, `npm_default`, `caddy`, …).
6. Find what owns ports 80/443: `sudo ss -tlnp | grep -E ':(80|443)\s'` and `docker ps --filter publish=80 --filter publish=443 --format '{{.Names}} {{.Image}}'`.
7. Identify the **reverse proxy type** (one of):
   - (a) **Traefik** container: image `traefik`. Read its labels/command for `providers.docker`, `exposedbydefault`, entrypoint names (`web`, `websecure`), and certresolver name. Run `docker inspect <traefik> --format '{{json .Config.Cmd}} {{json .Config.Labels}}'`, filtering out anything secret.
   - (b) **nginx-proxy / acme-companion**: image `nginxproxy/nginx-proxy` or `jwilder/nginx-proxy`.
   - (c) **Nginx Proxy Manager**: image `jc21/nginx-proxy-manager`.
   - (d) **Caddy** container or host service: image `caddy`, or `systemctl is-active caddy`. Find its Caddyfile (`docker inspect` mounts, or `/etc/caddy/Caddyfile`) and check for an `import` directive.
   - (e) **Host nginx**: `systemctl is-active nginx`. Look at `/etc/nginx/sites-enabled/` and `/etc/nginx/conf.d/`, and check whether certbot is used (`ls /etc/letsencrypt/live`).
   - (f) Something else or unclear: **stop and ask**.
8. Record which Docker network the proxy is on (for a–d) and how TLS certs are issued (Let's Encrypt resolver name, acme-companion, certbot, Caddy auto).
9. Find free localhost ports for possible use: check that `3710`, `3711` (or the next free ones) aren't used: `ss -tln | grep -E ':(3710|3711)\s' || echo free`.
10. Locate the API keys **without printing them**. Check, in order: the current shell env (`[ -n "$DEEPSEEK_API_KEY" ]`, `[ -n "$DEEPGRAM_API_KEY" ]`), `~/.bashrc`, `~/.profile`, `/etc/environment`, and `.env` files of the user's other projects via `grep -l 'DEEPSEEK_API_KEY\|DEEPGRAM_API_KEY' <file>` (filenames only). If not found or ambiguous, ask the user where they are (S3).
11. Check tools: `git`, `curl`, `jq`, `node --version` (host Node isn't required; we build inside Docker), and outbound HTTPS: `curl -sS -o /dev/null -w '%{http_code}\n' https://api.deepseek.com` and `https://api.deepgram.com`.
12. Write a **recon summary** to BUILD_LOG and show it to the user: resources, the do-not-touch list, the proxy type and network, the TLS method, the chosen ports, whether keys were found (yes/no only), and any blockers.

## PHASE 2: Agree the plan · steps 13–16

13. Ask the user (one message): the **domain/subdomain** for the app (e.g. `design.example.com`); confirmation that its DNS A record points to this server (check with `dig +short <domain>` vs `curl -4 -s ifconfig.me`); the repo URL/branch (default `https://github.com/hussainezzi/floorplan-3d.git`, `master`); the default currency (e.g. `USD`, `AED`, `PKR`); and the first client (tenant) slug and display name.
14. Present the integration method for the detected proxy type (from Phase 9 below) and the exact changes you will make, with **no execution yet**. Get explicit "yes".
15. Confirm the scope of this MVP with the user:
    - **In:** AI chat + voice that edits the existing demo apartment live (add/move/remove furniture, change floors, remove non-load-bearing walls), tenant branding and prices, a live estimate, lead capture, a Deepgram voiceover of the AI's replies, and image upload stored with the lead.
    - **Phase 2 (later):** building new floor plans from dimensions or photos.
16. Record the agreed answers in BUILD_LOG.

## PHASE 3: Workspace & secrets · steps 17–25

17. Create the workspace: `sudo mkdir -p /opt/floorplan && sudo chown "$USER":"$USER" /opt/floorplan`. If it already exists and isn't ours (no BUILD_LOG), **stop and ask**.
18. `git clone --branch <branch> <repo> /opt/floorplan/src` (or `git -C /opt/floorplan/src pull --ff-only` if it already exists). Then `git -C /opt/floorplan/src checkout -B vps/ai-mvp`.
19. Create `/opt/floorplan/data/uploads` and `/opt/floorplan/backups`.
20. Create `/opt/floorplan/.env` with `umask 077`. Copy the keys in **without printing them**, e.g. `printf 'DEEPSEEK_API_KEY=%s\n' "$DEEPSEEK_API_KEY" >> .env` (or extract the line from the source file with `grep '^DEEPSEEK_API_KEY=' <file> >> .env`). Do the same for `DEEPGRAM_API_KEY`.
21. Append the generated values: `POSTGRES_PASSWORD=$(openssl rand -hex 24)`, `ADMIN_TOKEN=$(openssl rand -hex 32)`, `SESSION_SECRET=$(openssl rand -hex 32)`, plus `POSTGRES_USER=floorplan`, `POSTGRES_DB=floorplan`, `APP_DOMAIN=<domain>`, `DEFAULT_CURRENCY=<cur>`, `DEEPSEEK_MODEL=deepseek-chat`, `DEEPSEEK_BASE_URL=https://api.deepseek.com`, `DEEPGRAM_STT_MODEL=nova-3`, `DEEPGRAM_TTS_MODEL=aura-2-thalia-en`, `AI_MAX_TURNS_PER_SESSION=30`, `AI_MAX_SESSIONS_PER_IP_PER_DAY=20`, `TENANT_DAILY_AI_CAP=500`.
22. `chmod 600 /opt/floorplan/.env`, and show it **masked** (S1).
23. Make sure `.env`, `data/`, `backups/` are never committed: the workspace root isn't the git repo, but also add `.env` to `src/.gitignore` if it's missing.
24. **Smoke-test DeepSeek** (key read from .env inside a subshell, never echoed): send a tiny request to `$DEEPSEEK_BASE_URL/chat/completions` with model `$DEEPSEEK_MODEL`, one user message `"reply OK"`, and **one dummy tool** to confirm function calling works. Print only the HTTP status and whether `tool_calls` came back. Also call `GET $DEEPSEEK_BASE_URL/models` and list the model ids. Note whether any model accepts **image input**. If none does, image upload in the MVP is stored with the lead for the human designer, not analysed by AI. Record this.
25. **Smoke-test Deepgram:** call TTS (`POST https://api.deepgram.com/v1/speak?model=$DEEPGRAM_TTS_MODEL`, header `Authorization: Token <key>`, JSON `{"text":"Hello"}`) and save the result to `/tmp/fp-tts-test.mp3`. Then call STT (`POST https://api.deepgram.com/v1/listen?model=$DEEPGRAM_STT_MODEL&smart_format=true`) on that file. Print only the status codes and the transcript text. If a model name is rejected, list the valid ones from Deepgram's error or docs, update `.env`, and retry. Delete the temp file.

## PHASE 4: Backend scaffold · steps 26–40

Create `src/apps/api/`: **Node 22 + TypeScript + Fastify**. Keep dependencies minimal: `fastify`, `@fastify/static`, `@fastify/multipart`, `@fastify/cors`, `@fastify/rate-limit`, `pg`, `zod`, and dev deps `typescript`, `tsx`, `vitest`. Use plain `fetch` for DeepSeek and Deepgram (no vendor SDKs needed).

26. `package.json` with scripts `dev`, `build` (tsc), `start` (node dist/server.js), `test` (vitest), `migrate`, and `tenant:create`.
27. `tsconfig.json`: strict, ES2022, module NodeNext, outDir `dist`.
28. `src/config.ts`: read env with zod and fail fast on missing vars. Never log values.
29. `src/db.ts`: a `pg` Pool from `POSTGRES_*`, host `floorplan-db`.
30. `migrations/001_init.sql`:
    - `tenants(id uuid pk, slug text unique, name text, branding jsonb, prices jsonb, currency text, allowed_origins text[], ai_enabled bool default true, created_at timestamptz default now())`
    - `sessions(id uuid pk, tenant_id uuid fk, ip_hash text, turns int default 0, created_at)`
    - `messages(id bigserial pk, session_id uuid fk, role text, content jsonb, created_at)`
    - `leads(id uuid pk, tenant_id uuid fk, session_id uuid, name text, phone text, email text, notes text, design jsonb, quote jsonb, uploads text[], consent bool, created_at)`
    - `ai_usage(day date, tenant_id uuid, calls int, primary key(day, tenant_id))`
31. `src/migrate.ts`: apply the `migrations/*.sql` files in order, tracked in a `schema_migrations` table. It must be idempotent.
32. `src/scripts/tenant-create.ts`: a CLI taking `--slug --name --currency --origin https://client.com [--logo URL --primary #hex]`. It inserts a tenant with **default prices** copied from the repo's `MATS` (floor prices per m²) plus a `furniture` price map keyed by LIB `type` (seed with reasonable placeholder prices and mark them `"placeholder": true`). Labour defaults: `{"demolition_per_m": 0, "install_pct": 0.1}`, `margin_pct: 0`, `tax_pct: 0`.
33. `src/server.ts`: Fastify with `trustProxy: true` (we're behind the reverse proxy), a JSON body limit of 1 MB, `@fastify/rate-limit` (global 120/min per IP), and CORS limited to the tenant's `allowed_origins` for `/api/*`.
34. Route `GET /healthz` returns `{ok:true, db:<bool>}`.
35. Route `GET /api/tenant/:slug` returns the public config only: `name, branding, currency, prices` (never internal fields).
36. Static hosting: serve `src/index.html` (the frontend) at `/` and `/t/:slug`, so the same page loads for every tenant. The page reads the slug from the URL. Set the header `Content-Security-Policy: frame-ancestors <tenant allowed_origins>` for `/t/:slug` (use `*` only for `/` demo), and remove `X-Frame-Options`.
37. Route `POST /api/session` with body `{tenant}` creates a session and returns `{sessionId}`. Hash the IP with `SESSION_SECRET` (sha256) and never store the raw IP. Enforce `AI_MAX_SESSIONS_PER_IP_PER_DAY`.
38. Route `POST /api/leads` (zod-validated, honeypot field, 5/min per IP) stores the lead, **recomputes the quote server-side** from the submitted design + tenant prices (step 52), and returns `{ok, leadId}`. If the tenant has `branding.webhook_url`, POST the lead JSON to it (5 s timeout, fire-and-forget, log only success or failure).
39. Route `POST /api/uploads` (multipart, max 3 files, 10 MB each, images/PDF only): check magic bytes, store under `/data/uploads/<tenant>/<uuid>.<ext>`, and return ids. The ids are attached to the lead. Never serve uploads publicly; add an admin-only `GET /admin/uploads/:id` with `Authorization: Bearer ADMIN_TOKEN`.
40. Route `GET /admin/leads?tenant=slug` (Bearer `ADMIN_TOKEN`) lists leads as JSON. Write `vitest` tests for config parsing, the lead validation, and the admin auth. Run `npm test`. **Check:** all tests pass. Commit: "api scaffold".

## PHASE 5: AI agent (DeepSeek, streaming, validated tools) · steps 41–55

41. `src/ai/provider.ts`: an interface `LlmProvider.chat({system, messages, tools}) → {text, toolCalls}`, with a `DeepSeekProvider` that calls `${DEEPSEEK_BASE_URL}/chat/completions` (OpenAI-compatible format) with a 60 s timeout and 1 retry on 429/5xx with backoff. Keep the interface generic so another provider (e.g. the Claude API) can be added later via env without touching the agent.
42. `src/ai/tools.ts` defines **exactly these tools** (JSON Schema). Coordinates are in **mm** in plan space, like the frontend:
    - `add_item({type, name?, room_id, x?, y?, w?, d?, rot?})`: `type` must be one of the LIB types sent by the client.
    - `move_item({id, x, y, rot?})`
    - `remove_item({id})`
    - `set_floor_material({room_ids: string[], material})`: material must be a key of the tenant prices' floor materials.
    - `demolish_wall({wall_id})` and `restore_wall({wall_id})`: only walls of kind `'n'` (non-load-bearing).
    - `ask_user({question, options?: string[]})`: use this when the request is ambiguous.
    - `finish({summary})`
43. `src/ai/context.ts`: build a compact context from the client's snapshot (sent with each message):
    - rooms: id, name, bbox and area in m²
    - walls: id, kind, length
    - items: id, type, room, x, y, w, d, rot
    - the LIB catalog: type, name, default w×d
    - the tenant's floor materials with prices
    - the currency

    Keep it under ~6k tokens. Round numbers and drop colours.
44. `src/ai/validate.ts`: validate every tool call against the snapshot.
    - The type, room, wall and material must exist.
    - The wall must be kind `'n'`.
    - The item must lie inside the room's bbox (clamp, and if it still doesn't fit, return an error).
    - Sizes must be between 200 and 6000 mm, and rotation must be a multiple of 90 unless specified.

    Invalid calls are **not** sent to the browser. The error goes back to the model as the tool result so it can correct itself.
45. `src/ai/system-prompt.md`: the agent role is a friendly interior design assistant working for `{tenant.name}`. It must:
    - change the design **only via tools**
    - place items with sensible clearances (walkways ≥ 800 mm, beds against walls, TV opposite the sofa)
    - never invent prices
    - keep replies to 1–3 short sentences in the user's language
    - use `ask_user` when unsure
    - call `finish` at the end of each request
    - refuse unrelated tasks politely
46. `src/ai/agent.ts`: the loop. Send the context + history + user message, receive tool calls, validate, **stream each valid op to the client immediately**, return results to the model, and repeat until `finish`/`ask_user`/text-only, or at most 12 tool rounds. Persist messages.
47. Route `POST /api/ai/:sessionId/message` with body `{text, snapshot, lang}` responds as **Server-Sent Events**: events `op` (one validated operation), `say` (assistant text), `ask` (question + options), `done`, `error`. Set `X-Accel-Buffering: no` and flush after each event so nginx-type proxies don't buffer.
48. Limits for each call: session turns ≤ `AI_MAX_TURNS_PER_SESSION`, tenant daily calls ≤ `TENANT_DAILY_AI_CAP` (increment `ai_usage`), text ≤ 1,000 chars, snapshot ≤ 200 KB. When a limit is hit, send a friendly `say`: "Let our designer take it from here", followed by an `ask` that opens the lead form.
49. Prompt-injection hygiene: the user text is data, the tool schema is fixed, and the server never executes anything except the validated design ops above. The model never sees secrets or other tenants' data.
50. Logging: log per AI call the tenant, latency, token usage (from the response `usage`), op count and error class. **Never log** API keys, raw IPs or full lead PII.
51. Tests (vitest, with a mocked provider): a valid `add_item` passes; a load-bearing `demolish_wall` is rejected; an unknown material is rejected; the loop stops at the 12-round cap; the SSE output format is correct.
52. `src/pricing.ts`: one pricing function used for both the preview and the lead.
    - Floors: Σ room area × floor price × 1.05 wastage.
    - Furniture: Σ price per item type.
    - Demolition: Σ demolished wall length × `demolition_per_m`.
    - Then add install %, margin % and tax %.
    - Return line items and a **range** (total ×0.9 – ×1.15), with `placeholder` flags carried through.

    Add unit tests for this.
53. Expose `POST /api/quote` with body `{tenant, design}` returning the quote JSON, rate-limited.
54. Run `npm test` and `npm run build`. **Check:** green.
55. Commit: "ai agent + pricing".

## PHASE 6: Voice (Deepgram) · steps 56–62

56. `src/voice/deepgram.ts`: `transcribe(buffer, mime, lang)` calls `/v1/listen` with `$DEEPGRAM_STT_MODEL`, `smart_format=true`, and `language` when known. `speak(text)` calls `/v1/speak?model=$DEEPGRAM_TTS_MODEL` and returns the audio stream.
57. Route `POST /api/voice/stt` (multipart audio ≤ 2 MB / ≤ 60 s, webm/ogg/mp4/wav, per-session rate limit 20/hour) returns `{text}`. The browser never sees the Deepgram key.
58. Route `POST /api/voice/tts` with body `{sessionId, text ≤ 400 chars}` streams `audio/mpeg`. Only allow text that the agent produced in that session (check the last assistant `say` messages) so the endpoint can't be abused as a free TTS service.
59. Add a per-tenant toggle `branding.voice_enabled` (default true) and a voice choice `branding.tts_voice` (falling back to env).
60. Tests: mock `fetch`; check the size limits and the "only agent text" rule.
61. Manual test from the server: `curl` STT with a short generated wav (or the TTS output from step 25, if regenerated) and print only the transcript.
62. Commit: "deepgram voice".

## PHASE 7: Frontend integration (`src/index.html`) · steps 63–75

The page has a classic `<script>` (main app: `state`, `mutate`, `addItem`, `toggleWall`, `renderAll`, `ROOMS`, `WALLS`, `LIB`, `MATS`) followed by a `<script type="module">` (Three.js). Top-level `const`/`let` aren't on `window`, so add a bridge **at the end of the classic script**.

63. Add `window.FP` at the end of the classic `<script>`:
    - `snapshot()`: rooms with id/name/bbox/area, walls `'w'+i` with kind `WALLS[i][4]` and length, items from `state.furniture`, demolished walls, the LIB types, and floor materials.
    - `apply(op)`: `add_item` builds an `F(...)` item and pushes it via `mutate`, using `pushOut`; `move_item`, `remove_item`, `set_floor_material` (`state.rooms[id].mat`), `demolish_wall`/`restore_wall` (`state.demolished`). Each op is one `mutate` call so the user can **undo** AI changes. Call `renderAll()`; the 3D view syncs automatically if open.
    - `setPrices(prices)`
    - `onChange(cb)`
64. Tenant bootstrap: if the URL matches `/t/<slug>`, fetch `/api/tenant/<slug>`. Apply branding (logo, primary colour via CSS variables, page title) and overwrite `MATS[*].price` from tenant prices. Show the currency symbol instead of the hard-coded `¥`: replace those literal `¥` occurrences with a `money()` helper using `Intl.NumberFormat(lang, {style:'currency', currency})`.
65. Use a **per-tenant localStorage key**: change `STORE` to include the slug, so designs from different clients don't mix.
66. Default the UI language to **English** when on `/t/*`, unless the tenant sets `branding.lang`. Keep the language toggle.
67. Add a collapsible **"Design with AI" chat panel** (bottom-right; full-width sheet on mobile, consistent with the existing `narrow` breakpoints) with: a message list, a text input, a **mic button**, a **speaker toggle**, an **upload button** (photos/plan, attached to the lead), and quick-start chips such as "Make the living room cozy", "Add a home office in the kids room", "Walnut floors everywhere except bathrooms".
68. Sending: call `POST /api/session` once, then `fetch` the SSE endpoint with `{text, snapshot: FP.snapshot(), lang}` and read the stream with `ReadableStream`. On `op`, call `FP.apply(op)` **with ~250 ms spacing** and a brief highlight of the changed item, so the user watches it being built. On `say`, append the message and, if the speaker is on, play `/api/voice/tts`. On `ask`, show the options as buttons. On `error`, show a gentle message.
69. Mic: use `MediaRecorder` (webm/opus, or mp4 on Safari) for push-to-talk with a 60 s max, POST to `/api/voice/stt`, put the transcript in the input and send it. Hide the mic if `MediaRecorder` is missing or the tenant disabled voice.
70. Live **estimate bar**: on every change, POST `/api/quote` (debounced 600 ms) and show the range (e.g. "Estimated $14,200 – $18,100") with a "See breakdown" drawer and the disclaimer "Estimate only. Final price after site survey."
71. **Lead form** ("Get my detailed quote"): name, phone/WhatsApp, email (optional), notes, a consent checkbox, and a hidden honeypot. It submits the design (`state`), the quote and the upload ids. On success, show "Thanks, {tenant.name} will contact you" and offer **Export PNG** (existing feature).
72. Keep the existing behaviour intact: 2D/3D toggle, walkthrough, undo/redo, export/import, and the shortcuts. Make sure the chat input **doesn't trigger global keyboard shortcuts** (stop propagation of keydown when focus is inside the panel).
73. Pin Three.js: keep the jsDelivr import map for now, but note in BUILD_LOG that self-hosting `three@0.160.0` is a follow-up.
74. Test in headless Chromium if available (`npx playwright` inside a temporary Docker container or the host). Load `/t/<slug>`, check there are no console errors, open the chat, send "add a plant to the living room" against the real API, and assert the furniture count increased. If no browser is available, test the endpoints with curl and ask the user to try it in their browser.
75. Commit: "frontend AI panel, voice, tenant config, estimate, leads".

## PHASE 8: Containers · steps 76–83

76. `src/apps/api/Dockerfile`: multi-stage (`node:22-alpine` build → runtime), `npm ci`, `npm run build`, `USER node`, copy `src/index.html` into the image (or mount it read-only), `EXPOSE 3000`, and `HEALTHCHECK` with wget against `/healthz`.
77. `/opt/floorplan/docker-compose.yml` (project name `floorplan`):
    - `app`: build `./src/apps/api`, `container_name: floorplan-app`, `env_file: .env`, `restart: unless-stopped`, `mem_limit: 512m`, `cpus: 1.0`, volume `./data/uploads:/data/uploads`, `depends_on` db healthy, networks `floorplan_net` (+ the proxy network if the method needs it; see step 85). Ports: **none**, or `127.0.0.1:3710:3000` for host-proxy setups (types d-host and e).
    - `db`: `postgres:16-alpine`, `container_name: floorplan-db`, `mem_limit: 512m`, volume `floorplan_pg:/var/lib/postgresql/data`, healthcheck `pg_isready`, network `floorplan_net` only, **no ports**.
    - The proxy network (if used) is declared as `external: true` with its exact existing name. **Never create or modify it.**
78. Add log rotation to both services: `logging: {driver: json-file, options: {max-size: "10m", max-file: "3"}}`.
79. `cd /opt/floorplan && docker compose -p floorplan config --quiet` (validates). Don't print the full config, because it contains env values.
80. `docker compose -p floorplan build`, then `docker compose -p floorplan up -d db` and wait until it's healthy.
81. Run migrations: `docker compose -p floorplan run --rm app node dist/migrate.js`. Then create the first tenant with `tenant:create` using the answers from step 13.
82. `docker compose -p floorplan up -d app`. **Check:** `docker compose -p floorplan ps` shows both services healthy. Re-run step 4's listing and **diff it against the do-not-touch list**: every other container must still be running, with the same status/uptime as before.
83. Internal check: `docker compose -p floorplan exec app wget -qO- http://localhost:3000/healthz` returns `{"ok":true,"db":true}`.

## PHASE 9: Reverse-proxy integration (only the method matching step 7) · steps 84–91

84. Re-confirm the method with the user, show the exact change (labels, file diff or UI steps), and wait for "yes".
85. Apply the change for the detected proxy type. In every case, add only our own route; never edit anything that belongs to other sites.
    - **(a) Traefik:** attach `app` to the Traefik network (external). Add labels to `app` only:
      - `traefik.enable=true`
      - `traefik.docker.network=<proxy_net>`
      - `traefik.http.routers.floorplan.rule=Host(\`<domain>\`)`
      - `traefik.http.routers.floorplan.entrypoints=<websecure-name>`
      - `traefik.http.routers.floorplan.tls.certresolver=<resolver-name>`
      - `traefik.http.services.floorplan.loadbalancer.server.port=3000`

      Use the entrypoint and resolver names discovered in step 7. No Traefik restart is needed; it picks up labels live.
    - **(b) nginx-proxy:** attach `app` to its network and set env `VIRTUAL_HOST=<domain>`, `VIRTUAL_PORT=3000`, `LETSENCRYPT_HOST=<domain>`. Add `LETSENCRYPT_EMAIL` only if the existing containers use it.
    - **(c) Nginx Proxy Manager:** attach `app` to the NPM network. Then give the user the UI steps: Hosts → Proxy Hosts → Add: domain `<domain>`, scheme `http`, forward host `floorplan-app`, port `3000`, ✔ Websockets, ✔ Block exploits. SSL tab: request a new certificate, ✔ Force SSL, ✔ HTTP/2. Advanced: `proxy_buffering off;`. Wait for them to confirm.
    - **(d) Caddy:** back up the Caddyfile. If it uses `import`, add a new file for our site; otherwise append a block:
      ```
      <domain> {
          encode zstd gzip
          reverse_proxy floorplan-app:3000 {
              flush_interval -1
          }
      }
      ```
      Use `127.0.0.1:3710` instead of `floorplan-app:3000` for host Caddy, and attach `app` to Caddy's network for container Caddy. Validate (`caddy validate --config <file>`, or via `docker exec <caddy> caddy validate ...`), then reload (`caddy reload` / `docker exec <caddy> caddy reload --config /etc/caddy/Caddyfile`).
    - **(e) Host nginx:** create `/etc/nginx/sites-available/floorplan.conf` (or `conf.d/floorplan.conf`, matching the existing style) with `server_name <domain>;` and `location / { proxy_pass http://127.0.0.1:3710; proxy_http_version 1.1; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; proxy_buffering off; proxy_read_timeout 120s; client_max_body_size 12m; }`. Symlink it if `sites-enabled` is used. Run `sudo nginx -t`, then `sudo systemctl reload nginx`. For TLS: `sudo certbot --nginx -d <domain>` (**only** our domain; ask first). If certbot is missing, ask before installing it.
86. If you used a docker-network method, run `docker compose -p floorplan up -d app` again to apply the network/labels. This recreates **only** `floorplan-app`.
87. Make sure uploads aren't blocked: the proxy body size must be ≥ 12 MB (nginx `client_max_body_size`; NPM advanced config; Traefik and Caddy are fine by default).
88. Make sure SSE isn't buffered: run `curl -N https://<domain>/api/ai/...` against a test session and confirm events arrive progressively, not all at the end. If they're buffered, add the proxy's no-buffering setting (`proxy_buffering off` / `flush_interval -1`).
89. External checks: `curl -sSI https://<domain>/healthz` returns 200 with a valid TLS cert (`curl -vI` shows the issuer), and `curl -sS https://<domain>/t/<slug> | grep -c '<title>'` returns ≥ 1.
90. **Verify that nothing else broke:** for every other domain the proxy serves (from its config/labels), run `curl -sS -o /dev/null -w '%{http_code} %{url_effective}\n' https://<other-domain>` and compare with a run done **before** step 85. (Run that "before" pass now if you didn't, and do it before step 85 on future runs.) If anything regressed, **roll back step 85 immediately** (restore the `.bak` / remove our labels or network) and report.
91. Record in BUILD_LOG exactly what was changed in the proxy and how to undo it.

## PHASE 10: Operations, verification & handover · steps 92–100

92. Backups: create `/opt/floorplan/scripts/backup.sh`. It runs `docker compose -p floorplan exec -T db pg_dump -U floorplan floorplan | gzip > backups/db-<ts>.sql.gz`, tars `data/uploads`, and deletes files older than 14 days. Add a **user** crontab line at 03:17 (don't edit system cron files). Run it once and verify the file exists. Suggest off-site copies (rclone to B2/R2/S3) to the user.
93. Health status: `docker compose -p floorplan ps`, plus `curl https://<domain>/healthz`, disk usage of `/opt/floorplan`, size of the latest backup, and today's `ai_usage` per tenant (`SELECT` through `docker compose exec db psql`).
94. Cost watch: show today's AI call count and estimated token usage from the logs. Remind the user to set a **spending limit/alert in the DeepSeek and Deepgram dashboards**.
95. Security review: confirm that `.env` is mode 600; no ports of ours are published except possibly `127.0.0.1:3710`; `db` has no published ports; the admin routes reject requests without a token; CORS rejects origins that aren't allowed; and uploads aren't publicly reachable (`curl` an upload path without a token should return 401/404).
96. Run a full end-to-end test as a homeowner from the public URL: open `/t/<slug>`; type "Make the living room cozy and change bedroom floors to walnut"; watch the ops apply; check that the estimate updates; try voice; submit a test lead marked `TEST`. Then check `GET /admin/leads` (with the token) and confirm the lead and its quote are there. Delete the test lead.
97. Write `/opt/floorplan/README-OPS.md` covering: how to create a tenant (`docker compose -p floorplan run --rm app node dist/scripts/tenant-create.js --slug ... --origin ...`), update tenant prices (SQL `UPDATE tenants SET prices = ...` or a JSON file import), view leads, update the code (`git pull` → `build` → `up -d app`, with a backup first), roll back (`git checkout <sha>` → rebuild), and undo the proxy integration (from step 91).
98. Commit everything in `src` on `vps/ai-mvp` with a clear message. Ask the user whether to push (E4). If yes and credentials exist, push the branch, and tell them to open a PR.
99. Print the **embed snippet** for each tenant:
    ```html
    <iframe src="https://<domain>/t/<slug>"
            style="width:100%;height:85vh;border:0;border-radius:12px"
            allow="microphone; fullscreen" loading="lazy"
            title="Design your home with AI"></iframe>
    ```
    Explain that the client's domain must be in that tenant's `allowed_origins` (set via `tenant:create --origin`), and that `allow="microphone"` is required for voice inside the iframe.
100. Final report to the user:
    - what was built
    - the URLs
    - what changed on the server (our containers, the proxy route) and how to undo it
    - confirmation that all pre-existing containers and sites are still healthy (from steps 82 and 90)
    - known limitations: images aren't AI-analysed if DeepSeek has no vision model; it only works on the demo apartment until Phase 2 (custom plans from dimensions); prices are placeholders until the client fills them in
    - suggested next steps: a tenant admin dashboard, Plan-JSON refactor for custom floor plans, self-hosted Three.js, off-site backups

    Update BUILD_LOG with "DONE".
