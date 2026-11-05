# From Open-Source Floor Planner to Commercial Product: Business & Architecture Plan

> Working title: **"RoomSay"** (placeholder) — an embeddable, AI-driven 3D home designer that interior-design and fit-out firms put on their own websites. Homeowners describe what they want, see it in 2D, 3D and walkthrough, and get an itemised quote at the firm's own prices.

---

## 0. Executive summary (the one-page answer)

**Verdict: the idea is viable, but not quite in the form you described.** Here are the three changes that matter most:

1. **Don't sell "design it yourself". Sell "qualified leads with a budget attached."** Interior designers make money from taste, trust and project management. A tool that suggests "you don't need us" gets resisted. A tool that turns an anonymous website visitor into *"Ayesha, 3-bed apartment, wants walnut floors and an island kitchen, ~$18k budget, here's her 3D design"* gets bought. The 3D designer is how you get the visitor in. **The lead and the quote are what you charge for.**
2. **Your best first customer is probably not a boutique interior designer.** Look at turnkey fit-out contractors, modular kitchen and wardrobe makers, and real-estate developers. They sell **repeatable floor plans** (the same 2BHK layout ×400 units) with **standard price lists**. That fits what this repo already is: a hard-coded single apartment with priced materials.
3. **The repo is a great demo engine, not a product yet.** It has a beautiful renderer and walkthrough. It does **not** have: a way to create or import new floor plans, an AI/natural-language layer, product SKUs, a pricing engine beyond flooring, multi-tenancy, lead capture or a backend. Those are the product. Expect roughly **4–6 months for a sellable v1** with a small team.

**Recommended path:** 90-day concierge pilot with 5–10 fit-out or design firms (you digitise their plans and price lists by hand). Prove that the embed creates more and better leads. Then build self-serve onboarding and the AI layer. Charge a SaaS subscription plus onboarding fees, and add per-lead or per-quote pricing later.

---

## 1. What's actually in this repo (technical due diligence)

| Area | What exists | Commercial gap |
|---|---|---|
| **Delivery** | One `index.html` (~185 KB). Vanilla JS, SVG 2D, Three.js r160 from the jsDelivr CDN. No build step. | Great for embedding. But there's no modularity, tests or versioning, and the CDN dependency is uncontrolled. |
| **Floor plan** | `ROOMS`, `WALLS`, `WINS`, `DOORS`, `SLIDES` are **hard-coded constants** for one Chinese apartment, in mm. | **Biggest gap.** No way to draw, upload or import a new plan. Every customer needs their own plans. |
| **Furniture** | `LIB`: about 70 generic items (bed, sofa, fridge…). Each is `[type, name, w, d, color]`. 3D models are built procedurally in `buildFurniture()`. | No SKUs, prices, brands, finishes or real product models. Generic look means you can't sell "this exact sofa". |
| **Pricing** | `MATS`: 8 floor materials with a ¥/m² price. Cost = area × price × 1.05 wastage. | Only flooring is priced. No furniture, labour, wall finishes, ceilings, electrical, plumbing, tax, margin or currency. |
| **Editing** | Drag, rotate, resize and snap furniture. Measure tool. Demolish non-load-bearing walls. Undo/redo. | Strong. Keep it. |
| **3D / walkthrough** | Orbit, top-down, first-person WASD plus touch joystick, opening doors, time-of-day sun, night lighting, 2D↔3D sync. | **This is your "wow".** It's demo-ready today. |
| **Persistence** | `localStorage` and JSON export/import. | No accounts, no server and no sharing link. The designer never sees what the customer built. |
| **i18n** | Chinese, Traditional Chinese, English. | Good base. Add Arabic/RTL, Urdu, Hindi and others depending on your market. |
| **AI** | None. | The "just say it" promise has to be built from scratch (see §6.3). |
| **License** | **MIT** (© 2026 wuyi). | Commercial use is allowed. Keep the copyright notice. **There's no code moat:** anyone can fork the same repo. Your moat has to be data, integrations and distribution. |

**Bottom line:** about 30% of a product is here: the hardest *visual* part. The other 70% is the business plumbing.

---

## 2. Critique of the original idea (the honest part)

### 2.1 What's strong
- **Real pain point.** Homeowners can't visualise drawings or mood boards. Designers lose deals on the first phone call because the client "can't picture it" or has no budget idea.
- **Instant price transparency is a real differentiator.** In most markets, renovation pricing is opaque and slow (site visit → wait 1–2 weeks → quote).
- **Embed/white-label model.** You ride on your customers' traffic and brands. You don't fight for consumers yourself.
- **Low marginal cost.** The rendering runs in the visitor's browser. Your server cost per session is close to zero, apart from AI calls.

### 2.2 Where the idea is weak or risky

| # | Risk | Why it matters | Mitigation |
|---|---|---|---|
| 1 | **Channel conflict:** "design it yourself" undercuts the designer's core value. | Designers won't put something on their site that commoditises them. | Position it as a **pre-design and qualification tool**: "Start your design, our designer will perfect it." The output is a *brief*, not a final design. |
| 2 | **Instant price = implied commitment.** | A wrong quote creates disputes and liability. Real costs depend on site conditions. | Show **ranges** ("$14.2k–$17.8k") and an "Estimate, final after site survey" disclaimer. Let the firm set the margin buffer. |
| 3 | **Floor-plan ingestion is the real bottleneck.** | No plan means no product. Homeowners don't have CAD files, only a photo of a brochure. | Phase 1: firm or developer uploads plans and you digitise them (a paid service). Phase 2: in-app wall drawing. Phase 3: AI image/PDF-to-plan. |
| 4 | **Catalog maintenance burden.** | Prices change monthly. A designer with 400 SKUs won't maintain them in yet another tool. | CSV/Google Sheet sync, Shopify/WooCommerce import, and "price groups" (Basic/Premium/Luxury) instead of per-SKU pricing for smaller firms. |
| 5 | **Generic 3D models look generic.** | "That's not the sofa I'll get" kills trust in the price. | Start with styles and tiers, not exact products. Later, let firms upload GLB models or photos (AI 3D generation is good enough for thumbnails and mid-detail). |
| 6 | **"AI understands everything" is overpromising.** | Natural language is ambiguous ("make it cozy"). Bad AI output makes the brand look bad. | AI produces **structured edits that the user can see and undo**, offers 2–3 options, and asks clarifying questions. Never silent magic. |
| 7 | **Crowded market.** | Planner 5D, RoomSketcher, Floorplanner, Homestyler, Coohom/Kujiale, Foyr, IKEA's planners, plus manufacturer configurators (Roomle, Threekit, etc.). | Don't compete on "3D planner". Compete on **"embeddable lead and quote engine for fit-out firms in [your region]"**: local prices, local language, WhatsApp handoff. |
| 8 | **Small boutique designers have low willingness to pay.** | Many are 1–5 person shops with little web traffic, so the embed delivers few leads. | Target firms that already **spend on ads** and have **repeatable products**. They feel the ROI. |
| 9 | **Zero code moat (MIT fork).** | A competitor can fork the same repo tomorrow. | Moat = tenant data (plans, catalogs, prices), integrations, AI quality tuned on real sessions, and local distribution and relationships. |

### 2.3 Reframed value proposition

> **For fit-out firms and interior designers who pay for leads:** "Put a 3D home designer on your website. Visitors design their space by talking to it, see it in 3D, and get an estimate at *your* prices. You get a qualified lead with their layout, choices and budget, delivered to WhatsApp/CRM in seconds."

The customer buys **more qualified leads, shorter sales cycles, fewer wasted site visits.** 3D is the feature, not the benefit.

---

## 3. Market & customer segmentation

Rank segments by **(repeatable plans) × (standard pricing) × (marketing spend) × (deal size)**:

| Segment | Fit | Why | Willingness to pay |
|---|---|---|---|
| **A. Real-estate developers / off-plan sales** | ★★★★★ | Fixed floor plans, many units. Buyers want to see "my apartment furnished" and buy fit-out packages. Matches the repo exactly. | High (marketing budgets per project) |
| **B. Turnkey fit-out / renovation contractors** | ★★★★★ | Standard packages (Basic/Premium/Luxury), price per m², heavy ad spend on Meta/Google. | Medium-high |
| **C. Modular kitchen & wardrobe makers** | ★★★★☆ | Fully parametric products with SKU pricing. They already sell by configuration. | Medium-high |
| **D. Furniture retailers** | ★★★☆☆ | Need exact product models, so the catalog work is heavy. Big competitors exist. | Medium |
| **E. Boutique interior designers** | ★★☆☆☆ | Bespoke work, low traffic, price-shy. Your original target. | Low-medium |

**Recommendation:** start with **B** (fast sales cycle, clear ROI). Use **A** as the high-value expansion, and serve **E** with a cheap self-serve tier later.

**Geography:** choose markets where turnkey fit-out packages are common, new apartment stock is high, and WhatsApp is the main sales channel. Examples are the GCC (UAE, Saudi Arabia), South Asia (India, Pakistan) and Southeast Asia. **Validate with 20 customer interviews before you commit.** Language and currency support (and RTL for Arabic) become requirements there.

**Sizing approach (do this with real local data, not with guesses):**
`# target firms in region × % reachable × ARPA × 12` = serviceable revenue.
Example to fill in: 3,000 fit-out firms × 10% × $150/mo × 12 ≈ $540k ARR from one region at modest penetration. Developer deals ($5k–$30k per project) sit on top of that.

---

## 4. Product definition

### 4.1 Three user types
1. **Homeowner (end user)**: on the firm's site. Picks a plan, describes wishes, edits, walks through, gets an estimate, requests a consultation.
2. **Firm admin (your paying customer)**: a dashboard to manage plans, catalog, prices, branding and leads.
3. **You (platform)**: tenants, billing, plan digitisation queue, analytics.

### 4.2 Core homeowner flow
```
Land on firm's page → choose plan (developer unit / template / "draw my own")
  → "Tell us what you want" (text or voice): "3-bed, open kitchen with island, walnut floors, mid budget"
  → AI applies the changes → 2D + 3D + walkthrough
  → Live itemised estimate (materials, furniture, labour), with Basic/Premium/Luxury toggle
  → "Get my detailed quote" → name + phone/WhatsApp (lead capture gate)
  → Shareable link + PDF brief → lands in the firm's CRM/WhatsApp
```

### 4.3 "Code, cost, height and whatever you can imagine" → the quote object
Each item in the design carries:
- **SKU / product code**, name, brand, finish, supplier
- **Dimensions** (W × D × H). The repo has W × D; height lives inside the 3D builders and must be moved into the data.
- **Unit price**, unit (each / m² / linear m), quantity (computed from geometry), wastage %, labour rate, lead time
- **Totals:** subtotal → firm margin → tax → total, shown as a range

Room-level items computed from geometry (the repo already computes room areas):
- Flooring m², wall paint m² (perimeter × ceiling height − openings), skirting linear m, false ceiling m², demolition m (from `state.demolished`), and electrical points per room type.

### 4.4 MVP scope (v1, sellable)
**In:** multi-plan support, admin-uploaded plans, catalog + price list via CSV, itemised BOQ estimate, tiers, lead capture, shareable link, PDF brief, embed snippet, branding, basic AI command bar, English + 1 local language, analytics.
**Out (v2+):** AI plan-from-photo, custom GLB uploads, payments/deposits, multi-floor, contractor scheduling, AR.

---

## 5. Business model & pricing (hypotheses to test)

| Tier | Target | Price (hypothesis) | Includes |
|---|---|---|---|
| **Starter** | Boutique designer | $49–79/mo | 3 plans, 100 catalog items, branding, 50 leads/mo, our watermark |
| **Pro** | Fit-out firm | $199–349/mo | 20 plans, unlimited catalog, tiers, CRM/WhatsApp integration, no watermark, AI commands |
| **Business** | Multi-branch firm / kitchen maker | $699–1,200/mo | Unlimited plans, multi-user, custom domain, API, priority support |
| **Developer project** | Real-estate project | $5k–30k per project + monthly hosting | All unit types digitised, custom furniture packages, sales-centre kiosk mode |

**Extra revenue lines:**
- **Onboarding / plan digitisation:** $50–150 per plan, or a $500–2,000 setup package. *This fees-for-service line also funds the concierge phase.*
- **Usage-based add-on (later):** a per qualified lead fee or AI-session overage.
- **Marketplace (long-term):** suppliers pay to list products in many firms' catalogs, with referral fees on orders.

**Unit economics sanity check (Pro tier, per tenant per month):**
- Hosting/storage/CDN: ~$2–5 (rendering happens client-side)
- AI: e.g. 1,000 sessions × ~5 commands × a small model's cost per call. Usually **low single-digit to tens of dollars**. Use a fast small model for edits and a larger one only for "design the whole flat".
- Target gross margin of **80%+** on subscription. Onboarding fees roughly break even.
- **Customer ROI pitch:** "If it closes **one** extra $10k fit-out per year, it paid for itself 3× over."

---

## 6. Target architecture (software architect view)

### 6.1 High-level
```
┌───────────── Firm's website ─────────────┐
│ <script src="cdn.roomsay.app/embed.js"   │
│   data-tenant="acme" data-plan="2bhk-a"> │
│ → <roomsay-designer> web component       │
│   (Shadow DOM / iframe)                  │
└───────────────┬──────────────────────────┘
                │ HTTPS (public tenant key, CORS allowlist)
┌───────────────▼──────────────────────────────────────────┐
│ API (Node/TypeScript)                                    │
│  ├ Tenant config (branding, locales, currency, tiers)    │
│  ├ Plans service (versioned plan JSON)                   │
│  ├ Catalog & pricing engine (BOQ, ranges, tax, margin)   │
│  ├ Designs service (save/share/fork homeowner designs)   │
│  ├ AI agent service (NL → validated design operations)   │
│  ├ Leads service → webhooks, WhatsApp, email, CRM        │
│  └ Render/PDF service (headless Chromium snapshots)      │
├──────────────────────────────────────────────────────────┤
│ Postgres (multi-tenant, row-level security) · S3/R2      │
│ (plans, GLBs, thumbnails, PDFs) · Redis (rate limits)    │
└──────────────────────────────────────────────────────────┘
       ▲
┌──────┴───────────────┐
│ Admin dashboard (web)│ plans editor · catalog CSV · leads · analytics · billing (Stripe)
└──────────────────────┘
```

### 6.2 Refactoring this repo (keep the magic, add structure)
Keep the zero-framework, small-bundle approach. It's an advantage for embedding. But modularise it:

```
packages/
  plan-schema/     # TypeScript types + JSON Schema: Plan, Room, Wall, Opening, Door, Item, Design
  core/            # pure state + commands (addItem, moveItem, demolishWall, setMaterial…), undo/redo, geometry (areas, perimeters)
  pricing/         # BOQ engine shared by client (live preview) and server (authoritative quote)
  renderer-2d/     # current SVG code (renderRooms, renderFurn, renderWalls…)
  renderer-3d/     # current Three.js code (buildArch, buildFurniture, walkthrough)
  embed/           # web component, theming, postMessage API, lead form
apps/
  api/  admin/  demo/   # demo = today's index.html, kept working as a regression test
```

Concrete steps:
1. **Pull the hard-coded constants out into a `Plan` JSON document.** `ROOMS/WALLS/WINS/DOORS/SLIDES` become `plan.rooms`, `plan.walls`, etc., loaded by URL. This one change unlocks multi-plan.
2. **Turn every mutation into a named command.** `mutate(fn)` becomes `dispatch({op:'moveItem', id, x, y})`. The UI and the AI use the same command bus, which gives you undo, validation, analytics and AI-safety for free.
3. **Move furniture to a catalog model.** `LIB` entries become `{sku, type, name, w, d, h, priceRef, model:'procedural:sofa' | 'glb:url'}`. Keep the procedural builders as the fallback renderer.
4. **Self-host Three.js** and pin the version (no runtime jsDelivr dependency for customers). Lazy-load the 3D bundle as it does today.
5. **Add Vite + TypeScript + Vitest + Playwright** visual tests. The current code is dense one-liners, so tests come before refactors.
6. **Add a plan editor:** a wall-drawing tool built on the existing snap and measure code, plus a "trace over uploaded image" mode at a known scale.

### 6.3 The AI layer ("just say it")
**Principle: the LLM never writes geometry freely. It calls typed tools that map to the command bus.**

```
User: "Make the kitchen open to the dining room and add an island, walnut floors everywhere except bathrooms"
  → Context sent: compact plan summary (rooms, walls with ids, load-bearing flags, items), catalog subset, budget tier
  → LLM tool calls:
      demolishWall({wallId:"w17"})          // validated: not load-bearing
      addItem({sku:"KIT-ISL-180", room:"kitchen", placement:"center"})
      setFloorMaterial({rooms:["*"], except:["mbath","gbath"], material:"walnut"})
  → Core validates (collisions, clearances, load-bearing rules) → applies → diff preview
  → User sees: "3 changes · +$2,340 · Undo"
```

- **Placement solver:** the LLM picks *intent* ("island in the centre of the kitchen, 1 m clearance"). A deterministic layout function picks exact coordinates. The repo's `snapMove`/`pushOut` logic is a good start.
- **Clarifying questions** when the request is ambiguous. Offer **2–3 variants** for broad requests ("design my living room").
- **Model choice:** a fast, cheap model for single edits, a stronger model for whole-home generation. Cache the plan context with prompt caching to cut cost.
- **Voice:** browser speech-to-text first. Server STT later for local languages.
- **Guardrails:** load-bearing walls locked, budget caps, the firm-defined allowed catalog only, per-tenant rate limits, and full logging. Session logs become your training and evaluation data, which is part of the moat.

### 6.4 Embedding
- **Option A (default): iframe** `https://app.roomsay.app/t/acme/plan/2bhk-a`. Best isolation, no CSS clashes, works on WordPress/Wix/Squarespace/Shopify.
- **Option B: web component** via `<script>` for deeper integration. Shadow DOM for style isolation.
- **JS API via `postMessage`:** `onLead`, `onQuoteChange`, `setPlan`, `setTier` for firms that want it in their own funnel.
- **WordPress plugin + Shopify app.** These are distribution channels, not only integrations.
- **Security:** a public tenant key plus a domain allowlist, rate limiting, CAPTCHA on lead submit, and a server-side authoritative quote (client totals are only a preview).

### 6.5 Non-functionals
- **Performance:** the 3D bundle under ~1 MB gz, lazy-loaded. Mobile-first (the repo already has touch walkthrough). Target <3 s to interactive 2D on 4G.
- **Privacy:** GDPR/local data-protection for leads, consent checkbox, data export and deletion per tenant.
- **Multi-tenancy:** Postgres row-level security on `tenant_id`, per-tenant asset prefixes.
- **Observability:** product analytics events (plan_opened, ai_command, tier_switched, lead_submitted) feed the firm's dashboard. That dashboard is how you prove ROI and reduce churn.

---

## 7. Go-to-market

### Phase 0: Validate (weeks 1–4, near-zero code)
- Interview **20 firms** (fit-out contractors, kitchen makers, 2–3 developers). Ask: how do leads come in, what's the cost per lead, close rate, time to first quote?
- Make a **demo video** with the current repo, re-skinned in English with local currency. It's impressive already.
- **Pre-sell:** get 5 firms to pay a setup fee (e.g. $300–500) for a pilot. Paying pilots are the only real validation.

### Phase 1: Concierge pilot (months 1–3)
- *You* digitise their 2–5 most common plans and their price sheet by hand.
- Ship the embed with an itemised estimate and lead capture (WhatsApp/email). AI can be a simple command bar.
- **Measure:** visitor→engaged rate, engaged→lead rate, lead→site-visit, the firm's close rate vs. their normal leads. **This data is your sales deck.**

### Phase 2: Productise (months 3–6)
- Self-serve admin: plan editor, CSV catalog, branding, Stripe billing.
- Full AI command layer, PDF brief, shareable links, CRM webhooks.
- Launch the WordPress plugin. Pursue partnerships with web agencies that build sites for interior firms (rev-share 20–30%).

### Phase 3: Scale (months 6–18)
- Developer project package (sales-centre kiosk, all unit types, upsell fit-out packages).
- AI plan-from-image, GLB uploads, supplier marketplace.
- A second region or language.

**Acquisition channels (by expected efficiency):** founder-led sales and referrals → agency partners → content ("before/after: how Firm X got 3× more qualified leads") → LinkedIn/Instagram ads aimed at firms → trade shows.

---

## 8. KPIs and targets (first 12 months, illustrative)

| Metric | Month 3 | Month 6 | Month 12 |
|---|---|---|---|
| Paying tenants | 5 (pilots) | 25 | 80–120 |
| MRR | ~$1k | ~$6k | $20–35k |
| Embed → lead conversion | measure | ≥ 3% | ≥ 5% |
| Lead → site visit (firm-reported) | measure | ≥ 25% | ≥ 30% |
| Logo churn (monthly) | — | < 5% | < 3% |
| Gross margin | — | > 75% | > 80% |

The north-star metric: **qualified leads delivered per tenant per month.**

---

## 9. Team & budget (lean)
- **Founder (you):** sales, customer interviews, plan digitisation in the pilot phase.
- **1 senior full-stack/3D engineer** (Three.js + TypeScript + Node). This is the critical hire, or you as founder if technical.
- **1 part-time designer/3D artist:** catalog models, UI polish.
- **Later:** customer-success person for onboarding, and a sales hire per region.
- **6-month burn (lean, region-dependent):** roughly $40k–120k. Pilot setup fees plus early MRR offset part of it.

---

## 10. Key risks & kill criteria

| Risk | Early signal | Response |
|---|---|---|
| Firms like the demo but won't pay | < 2 of 20 interviewed firms pay for a pilot | Pivot segment (developers or kitchen makers) or pivot to a service (3D sales visuals) |
| Embed gets traffic but no leads | < 1% lead conversion after UX iterations | Move the lead gate earlier, offer a "free designer review", and A/B test |
| Onboarding cost per tenant too high | > 10 hours per tenant after month 4 | Prioritise the plan editor and AI plan import, and raise setup fees |
| AI quality complaints | High undo rate after AI commands | Narrow the AI to safer operations, add variants and clarifying questions |
| Competitor clones the fork | — | Speed, local integrations (WhatsApp, local CRMs), and per-region price data |

---

## 11. Next 30 days: concrete checklist
- [ ] Rebrand and translate the demo with local currency. Record a 60-second walkthrough video.
- [ ] Book 20 discovery calls, aiming for 5 paid pilots.
- [ ] Engineering spike: extract `Plan` JSON from `index.html` constants and load a plan by URL (≈1 week).
- [ ] Engineering spike: generalise `MATS` into a catalog/BOQ engine covering furniture + paint + skirting + labour (≈1 week).
- [ ] Engineering spike: a minimal lead form plus webhook to email/WhatsApp (≈2–3 days).
- [ ] Prototype the AI command bar with 5 tools (`addItem`, `moveItem`, `removeItem`, `setFloorMaterial`, `demolishWall`) (≈1 week).
- [ ] Legal: keep the MIT notice in the bundle, and draft simple ToS plus an estimate disclaimer.
