# Deployment & AI Architecture (VPS + white-label embed + AI agent)

This is a review of the proposed setup: a backend on our own VPS, an embed link that firms white-label on their websites, and an AI agent that builds homes from text, dimensions, photos and plans.

## 1. Verdict on the proposal

| Part of the idea | Verdict | Notes |
|---|---|---|
| Host the backend on our own VPS | ✅ Correct for the MVP | One VPS runs the first ~50–100 tenants comfortably. 3D rendering happens in the visitor's browser, not on the server. |
| One embed link per customer, white-labelled | ✅ Correct | Each firm gets `https://app.<ourdomain>/t/<tenant>` (iframe) or a `<script>` snippet. Branding, prices and plans load per tenant. |
| AI builds the home live in front of the user | ✅ Correct and achievable | The server streams AI **tool calls** to the browser, and each one applies instantly (wall appears, sofa drops in). |
| Power the AI with a **Claude Pro/Max subscription** | ❌ **Do not do this** | Consumer subscriptions are licensed for one person's own use, not as a backend for your customers' users. It breaches the terms, risks an account ban (which takes down every client at once), and has hard usage caps you can't scale. **Use the Claude API (Anthropic Console, API key, pay per token).** It's built for exactly this. |
| DeepSeek API as an alternative | ⚠️ Possible, with care | It's cheaper. But it adds data-residency and privacy questions for homeowners' photos and phone numbers, and its tool-calling and vision are less reliable for this task. If you use it, put it behind the same provider interface (§4) so you can switch models per tenant. |
| "Upload photos → exact house" | ⚠️ Set expectations | Photos give style, materials and a rough layout. **Accurate dimensions must come from the user or a floor-plan drawing.** A floor-plan image or PDF can be digitised with AI vision, but the user should confirm the result (§5). |

## 2. Topology (single VPS, MVP)

```
Internet
   │
   ▼
Caddy (TLS auto via Let's Encrypt, reverse proxy, per-IP rate limit)
   ├── app.<domain>      → web     (static embed bundle: today's index.html, modularised)
   ├── api.<domain>      → api     (Node/TypeScript: tenants, plans, catalog, quotes, leads)
   │                       └─ /ai/session (SSE/WebSocket stream of tool calls)
   ├── admin.<domain>    → admin   (firm dashboard: branding, price list, plans, leads)
   └── cdn.<domain>      → static files / uploads (or Cloudflare R2 later)

Docker Compose services on the VPS:
   web · api · worker (queue: vision/PDF/plan import, quote PDFs) · postgres · redis · caddy
Backups: nightly pg_dump + uploads → off-site object storage (R2/S3/Backblaze)
```

**Recommended VPS size to start:** 4 vCPU / 8 GB RAM / 80+ GB SSD, Ubuntu 24.04 LTS.

## 3. The AI agent loop (how "it builds before their eyes")

```
Browser (embed)                       API (VPS)                          LLM provider (Claude API)
──────────────                        ─────────                          ───────────────────────
user: "3-bed, 110 m², open kitchen,   → POST /ai/session/:id/message
       walnut floors" + photo.jpg        ├ load tenant catalog + price list
                                          ├ load current Plan JSON
                                          └ call model with TOOLS ──────→  model decides:
                                                                           create_room(...)
                                          ← stream tool_use blocks ←────  add_wall(...)
                                          ├ validate each call (geometry,  set_floor_material(...)
                                          │  load-bearing, catalog-only)   add_item(sku=...)
← SSE event: {op:'addWall', ...}  ←──────┤ apply to server-side Plan
  renderer animates it in 2D/3D          ├ recompute quote (pricing engine)
← SSE event: {quote: {...}}       ←──────┘ return tool_result to model → loop until done
```

Rules:
- The model **never** writes free-form geometry or prices. It can only call typed tools (`createRoom`, `addWall`, `addOpening`, `addItem`, `moveItem`, `removeItem`, `setFloorMaterial`, `demolishWall`, `askUser`), validated by the same command bus the UI uses.
- **Prices always come from the tenant's price list** through the pricing engine, never from the model.
- **Model routing:** a fast, cheap model (e.g. `claude-haiku-5-5`) for single edits, and a stronger model (`claude-sonnet-5-5`) for full-home generation and vision/plan reading. Use prompt caching for the system prompt, tool schemas and the tenant catalog.
- Every session is logged (with consent). This becomes your evaluation and improvement dataset.

## 4. Provider abstraction

```ts
interface LlmProvider {
  run(opts: { system, messages, tools, images?, model }): AsyncIterable<ToolCall | Text>;
}
// implementations: AnthropicProvider (default), DeepSeekProvider (optional, per-tenant flag)
```
The tenant config chooses the provider and model, with a per-tenant monthly **AI budget cap**.

## 5. Inputs: what each one can realistically do

| User gives | Pipeline | Accuracy |
|---|---|---|
| Text + dimensions ("living 5×4 m, kitchen 3×2.5 m…") | Model → `createRoom`/`addWall` tools → auto-layout solver joins rooms | Good. The user can then drag walls to correct. |
| Floor-plan image / PDF / brochure | Worker: vision model extracts walls, doors and windows as JSON → user calibrates scale by marking one known dimension → confirm | Good on clean plans. Always needs a confirmation step. |
| Room photos | Vision model → style, materials, colours, approximate furniture → applied to rooms | Style: good. **Dimensions: unreliable.** Never quote from photo-only measurements. |
| DWG/DXF (designers) | Server-side DXF parser → walls/openings (v2) | Exact |

## 6. Abuse & cost protection (critical: the embed is public)
Anyone visiting any client website can spend your AI budget, so all of the following are needed:
- Per-tenant **and** per-visitor quotas (e.g. N AI turns per session, M sessions per IP/day) via Redis.
- CAPTCHA (Turnstile) before the first AI turn or the first upload.
- Lead gate: free tier of a few AI commands, full "build my whole house" after phone/email.
- Upload limits (size/type), image re-encoding, and stripping EXIF location data.
- Allowed-origin check: the embed only works on each tenant's registered domains.
- Hard monthly spend limit on the API key, with alerts.

## 7. White-labelling
- Tenant config: logo, colours, fonts, locale, currency, tax, margin, tiers, enabled AI features.
- Optional custom domain (`design.firmname.com` → CNAME to us; Caddy issues TLS automatically).
- "Powered by" badge removable on paid tiers.
- Firm sees all leads, sessions and quotes in the admin dashboard, and receives webhook/WhatsApp/email notifications.

## 8. Server access & security (how we deploy to your VPS)
- **Do not share passwords or root credentials in chat or in this repo.**
- On the VPS, create a `deploy` user with sudo, add **an SSH public key only**, disable password and root login, and enable `ufw` (22, 80, 443) and `fail2ban`.
- Secrets (`ANTHROPIC_API_KEY`, DB password, etc.) live in a `.env` on the server (chmod 600) or in a secrets manager, never in git.
- Deployment: GitHub Actions builds Docker images → pushes to GHCR → SSHes in with a deploy key and runs `docker compose pull && up -d`. Every deploy is reproducible and you can roll back.
- Monitoring: Uptime Kuma or BetterStack uptime checks, Sentry for errors, and a cost dashboard of AI tokens per tenant.

## 9. Build order
1. Extract `Plan` JSON and the command bus from `index.html` (prerequisite for everything).
2. API + Postgres: tenants, plans, catalog/price list, quote engine, leads.
3. Docker Compose + Caddy + CI deploy to the VPS.
4. AI session endpoint (streaming tool calls) with 6–8 tools, text-only.
5. Floor-plan image import with a calibration/confirm step.
6. Photo-to-style, admin dashboard, white-label custom domains.
