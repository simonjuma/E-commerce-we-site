# Afrishop — Kenyan E-Commerce Marketplace

A real, working full-stack application — not a mockup. Customer storefront,
seller dashboard, admin panel, REST API, real authentication, a real
database (SQLite locally / Postgres-Supabase in production), real image
uploads, and real inventory/order logic. Built with **zero required
external runtime dependencies for local dev** — only Node.js built-ins —
because this was developed in a sandbox with no package-registry access.
`pg` is used in production only, when you point it at Postgres.

```bash
npm run seed   # demo admin/seller/customer accounts + sample products
npm start      # http://localhost:3000
```

---

## 1. What changed in this production pass

This is a hardening pass on an already-working app. Summary of what was
audited and fixed:

| Area | Before | After |
|---|---|---|
| Database | SQLite only — breaks on serverless hosts (no persistent disk) | Dual adapter: SQLite locally, real Postgres (e.g. Supabase) in production via `DATABASE_URL` |
| Product images | Text `image_url` field only, no real upload | Real upload endpoint (`POST /api/seller/uploads`) — validates type/size, writes to disk, returns a servable URL. Wired into the seller dashboard's product form with a live preview. |
| SEO | No meta tags, no robots.txt, no sitemap | Meta description + Open Graph + Twitter card on every page, `robots.txt`, dynamic `sitemap.xml` generated from live products |
| CORS | Defaulted to `*` everywhere | Defaults to same-origin-only in production unless `CORS_ORIGIN` is explicitly set; still permissive locally for convenience |
| Rate limiting | One global limit for all routes | Auth routes (`/api/auth/*`) get a tight 8-req/min limit separate from the general 60-req/min API limit — resists credential stuffing without throttling normal browsing |
| Session secret | Silently fell back to an insecure default with a console warning | **Hard-fails to start** in `NODE_ENV=production` if `SESSION_SECRET` is unset — a warning nobody reads on a dashboard is not a safeguard |
| Cookies | No `Secure` flag | `Secure` flag added automatically when `NODE_ENV=production` |
| Response headers | None | `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` on every response |

## 2. Project structure

```
afrishop/
├── server.js               # HTTP server + all API routes (async, engine-agnostic)
├── db.js                   # Picks SQLite or Postgres based on DATABASE_URL
├── seed.js                 # Demo data: 1 admin, 1 seller+store, 1 customer, 6 products
├── lib/
│   ├── db-sqlite.js         # Local/dev engine (Node's built-in node:sqlite)
│   ├── db-postgres.js       # Production engine (pg + Supabase-compatible)
│   ├── schema-sqlite.js     # Schema, SQLite dialect
│   ├── schema-postgres.js   # Schema, Postgres dialect (same tables/relationships)
│   ├── auth.js               # scrypt password hashing + HMAC-signed sessions
│   └── http.js               # Body parsing, cookies, validation, image decode
├── migrations/
│   └── 001_init.sql         # Standalone SQL for the Supabase SQL editor / psql
├── data/afrishop.db          # SQLite file (local dev only, gitignored)
├── .env.example
└── public/                  # Frontend — plain HTML/CSS/JS, calls the API via fetch()
    ├── index.html, product.html, cart.html, checkout.html,
    │   order-confirmation.html, orders.html, account.html
    ├── seller/dashboard.html   # Store setup, product CRUD + image upload, orders, stats
    ├── admin/dashboard.html    # Platform stats, user/store suspend, all orders
    ├── uploads/                 # Seller-uploaded product images (gitignored)
    └── css/styles.css, js/api.js
```

---

## 3. Fully functional (tested against the running server, not just described)

- **Auth**: signup/login/logout, real `scrypt` password hashing, signed session
  cookies, wrong-password/duplicate-email/weak-password all rejected server-side.
- **Authorization**: verified — customer→seller routes get `403`,
  anonymous→admin routes get `401`, a seller editing another seller's product
  gets `404` (not leaked), a customer can't view another customer's order.
- **Product catalog**: search, category filter, price sort, real image
  display where uploaded.
- **Image upload**: real — a seller picks a file, it's read client-side as
  base64, POSTed to `/api/seller/uploads`, validated (PNG/JPEG/WebP, <5MB),
  written to `public/uploads/`, and the returned URL is attached to the
  product. Confirmed the file is actually retrievable afterward.
- **Cart & checkout**: add/update/remove, real stock check at checkout
  (rejects orders exceeding stock), real inventory decrement, Kenyan phone
  validation, multi-store order splitting.
- **Orders**: real status lifecycle, customer tracking view, seller status
  updates.
- **Seller dashboard**: store setup, product CRUD with image upload, order
  management, revenue/stock stats — scoped to that seller's own store only.
- **Admin dashboard**: platform stats, user suspend/activate, store
  suspend/activate (confirmed: suspending a store immediately removes its
  products from the public catalog), full order list.
- **SEO**: every page has a title, meta description, and Open Graph tags;
  `robots.txt` correctly disallows private routes (cart/checkout/account/
  seller/admin) while allowing the catalog; `sitemap.xml` is generated live
  from active products.
- **Security**: auth-specific rate limiting confirmed (12 rapid login
  attempts → first 8 return `401`, the rest `429`); production mode
  confirmed to refuse to start without `SESSION_SECRET` and to warn if
  `CORS_ORIGIN` is unset.
- **UI states**: loading skeletons, empty states with CTAs, and error states
  with the real error message on every page.

## 4. What still requires an external account/credential

| Feature | What's built | What you add |
|---|---|---|
| Real Postgres/Supabase | Full adapter + migration file, structurally verified (fails with a clear, correct error when `pg` isn't installed — expected in this sandbox) | A Supabase project + `DATABASE_URL`, then `npm install` on your real host (installs `pg` automatically as an optional dependency) |
| M-Pesa payments | **Fully implemented**: real STK Push, status polling, and the Safaricom callback handler (`lib/payments/mpesa.js`) — verified against real Daraja-shaped success and failure payloads. Returns `501` and never fakes success until credentials are set. | Safaricom Daraja API credentials + a public HTTPS `MPESA_CALLBACK_URL` (Safaricom cannot call `localhost`) |
| PayPal payments | **Fully implemented**: Orders v2 API (`lib/payments/paypal.js`), redirect-based checkout, capture on return, webhook signature verification. Settles in USD (PayPal doesn't support KES) — converted from the order's KES total via `KES_TO_USD_RATE`. | PayPal REST app credentials (Client ID/Secret, Webhook ID) |
| Card payments | Structure only — `POST /api/webhooks/paystack` (returns `501` until configured) | Paystack or Flutterwave secret key |
| Email/SMS notifications | `notifications` table, trigger points identified in code | Email + SMS provider API keys |
| Social link-preview images | `product.html` sets Open Graph tags client-side via JS after fetching the product — works for crawlers that execute JS, but many social-preview bots (e.g. link unfurlers) don't. True per-product OG on first byte needs server-side rendering (e.g. a Next.js rewrite) — a real architecture change, not a config value | Optional — only worth it if social sharing previews matter to you |

---

## 5. Environment variables

Full list with comments in `.env.example`. Minimum for local dev: none — the
app runs with safe (dev-only) fallbacks and tells you in the console.
**Minimum for production**: `NODE_ENV=production`, `SESSION_SECRET` (the app
won't start without it), and `CORS_ORIGIN` if your frontend is served from a
different origin than the API.

---

## 6. Database setup

**Local (default, zero setup):** SQLite via Node's built-in `node:sqlite`.
`npm run seed` creates `data/afrishop.db` and applies the schema automatically.

**Production — Supabase (recommended):**
1. Create a project at supabase.com.
2. Project Settings → Database → Connection string → copy the URI (use the
   "connection pooling" string if available — better for serverless).
3. Set `DATABASE_URL` to that string in your hosting platform's environment
   variables.
4. Either let the app apply the schema automatically on first boot
   (`lib/db-postgres.js` runs `CREATE TABLE IF NOT EXISTS` for every table —
   safe, idempotent), **or** run `migrations/001_init.sql` yourself via the
   Supabase SQL editor first if you'd rather control that step explicitly.
5. `npm install` on the real host installs `pg` automatically (declared as
   an optional dependency in `package.json`) — nothing to do manually.

No table structure changes are needed between engines — same columns,
relationships, and the `store_id`/`customer_id` scoping used for
authorization carry over directly.

---

## 7. Deployment instructions

**Important hosting note:** this app is a single long-running Node process
holding an HTTP server (and, locally, a SQLite file on disk). That shape
does **not** fit Vercel or Netlify's serverless model — serverless functions
are stateless and don't keep a persistent server or local file between
requests. Once you're on Postgres (section 6), the *database* is no longer
the blocker, but you'd still need to rewrite `server.js`'s routes as
individual serverless functions to run on Vercel/Netlify — a real
architecture change, not a config flag.

**What I'd actually recommend** — Render, Railway, or Fly.io. These run a
persistent Node process (exactly what this app is), support custom domains
and free TLS, and are just as "real hosting" as Vercel/Netlify:

1. Push this project to a Git repository.
2. Create a new Web Service, point it at the repo.
3. Build command: `npm install` (installs `pg` if `DATABASE_URL` is set to
   Postgres; no-op otherwise). Start command: `npm start`.
4. Add the environment variables from `.env.example`.
5. Deploy. Run `npm run seed` once via the platform's shell/console if you
   want demo data (skip in a real launch — seed data is for evaluation only).

**VPS (Ubuntu/Debian) alternative:**
```bash
git clone <your-repo> && cd afrishop
cp .env.example .env   # fill in values, including SESSION_SECRET
npm install
npm install -g pm2
pm2 start server.js --name afrishop
pm2 save && pm2 startup
```
Put Nginx in front for TLS termination (see section 8).

**If Vercel/Netlify specifically matter to you**, the realistic path is:
migrate the Node routes in `server.js` into individual serverless functions
(Vercel: `/api/*.js` files; Netlify: `/netlify/functions/*.js`), keep the
Postgres adapter as-is (it already works statelessly per-request), and serve
`public/` as static assets — both platforms do this natively. That's a real
rewrite of the routing layer, not something safe to fake as "done" here;
I've kept `server.js`'s route handlers as small, self-contained async
functions specifically so that migration is mechanical when you're ready
for it.

---

## 8. Domain connection & HTTPS

1. **DNS**: on Render/Railway/Fly, use their "custom domain" flow — usually
   a CNAME to the URL they give you. On a VPS, add an A record to your
   server's IP.
2. **HTTPS**: automatic on Render/Railway/Fly once the domain is attached.
   On a VPS:
   ```bash
   sudo apt install nginx certbot python3-certbot-nginx
   sudo certbot --nginx -d yourdomain.co.ke
   ```
3. Set `PUBLIC_URL` and `CORS_ORIGIN` to `https://yourdomain.co.ke` in your
   environment variables (used for the sitemap and CORS respectively).

---

## 9. Testing performed in this pass

All run against the live server and confirmed working:
- ✅ Full async DB refactor — every route re-tested against SQLite, all
  previous flows (auth, cart, checkout, inventory locking, admin
  suspend/activate, cross-tenant authorization) still pass
- ✅ Postgres adapter selection — confirmed `DATABASE_URL=postgres://...`
  correctly routes to `lib/db-postgres.js` and fails with the intended,
  actionable error (no `pg` package in this offline sandbox — resolves on
  a real host's `npm install`)
- ✅ Image upload — real PNG uploaded, validated, stored, and re-fetched
  successfully at its returned URL; non-image payloads rejected with `400`
- ✅ Auth rate limiting — 12 rapid login attempts → 8× `401`, then `429`
- ✅ Production hard-fail — server refuses to start under
  `NODE_ENV=production` without `SESSION_SECRET`, starts normally once set
- ✅ `robots.txt` and `sitemap.xml` — served correctly, sitemap reflects
  live product data

---

## 10. Production-readiness checklist

- [x] Real authentication with hashed passwords
- [x] Role-based authorization enforced server-side on every route
- [x] Multi-tenant data isolation (verified)
- [x] Real image upload with type/size validation
- [x] Dual database engine — SQLite (dev) / Postgres-Supabase (production)
- [x] SEO: titles, meta descriptions, Open Graph, robots.txt, dynamic sitemap
- [x] CORS locked to same-origin by default in production
- [x] Auth-specific rate limiting (separate from general API limiting)
- [x] Hard-fail on missing session secret in production
- [x] Secure cookies in production
- [x] Baseline security headers on every response
- [x] No secrets in frontend code; no hardcoded localhost dependencies
- [ ] `DATABASE_URL` pointed at a real Supabase/Postgres instance (untestable from this sandbox — no network access)
- [ ] `SESSION_SECRET` set to a real random value on your host
- [ ] M-Pesa/Paystack credentials added and webhook signatures verified against provider docs (currently correctly return `501` rather than fake success)
- [ ] Email/SMS provider connected for order notifications
- [ ] Deployed to Render/Railway/Fly (or a VPS) — **not deployed from this environment**; see section 7 for why Vercel/Netlify need an additional routing rewrite
- [ ] Custom domain + HTTPS attached (section 8)
- [ ] Backups configured for the production database
- [ ] Error monitoring (e.g. Sentry) added — currently logs to console only
- [ ] Load-test checkout under concurrent stock-limited purchases before a real launch

**On the claim boundary**: nothing above is marked done unless it was
actually run and verified in this environment. Anything requiring a live
external network connection (a real Supabase instance, a real payment
provider, an actual public URL) is explicitly left unchecked — I can't
verify what I can't reach from here, and I won't claim otherwise.
