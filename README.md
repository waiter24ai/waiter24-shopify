# Waiter24 — Shopify integration

Connects a Shopify store to a Waiter24 tenant: syncs the catalog into the
Waiter24 menu and embeds the AI chat widget on the storefront.

Unlike the WordPress plugin (a PHP file that runs inside the store), Shopify
does not allow merchant-hosted code. The app is therefore split in two:

| Part | Where it lives |
|------|----------------|
| OAuth, Admin API reads, webhook receivers, sync, embedded admin UI | **The Waiter24 platform** (server-side; part of the Waiter24 SaaS backend) |
| Deployable Shopify app config + storefront widget (theme app extension) | **This repository**, managed by the Shopify CLI |

## How it works

1. **Install (OAuth).** Merchant opens the install link → approves
   `read_products,read_inventory` → we exchange the code for a permanent Admin
   API token (stored encrypted in `shopify_stores`) and register webhooks.
2. **Link tenant.** In the embedded admin UI the merchant pastes their
   **import token** (Waiter24 dashboard → Site Integration → Menu auto-import).
   That links the Shopify store to the tenant and triggers the first sync.
3. **Sync.** `MenuSyncService` reads products via Admin GraphQL, maps them to the
   neutral import schema (`ProductMapper`), and calls `MenuImportService`
   directly. Mapping: `productType → category`, first collection → subcategory,
   variants → variations, `compareAtPrice > price` → sale price.
4. **Stay in sync.** Webhooks (`products/*`, `inventory_levels/update`) queue a
   debounced resync in real time; `shopify:resync` runs daily as a safety net;
   the merchant can also press **Sync now**.
5. **Widget.** The theme app extension adds a "Waiter24 Chat" app embed block the
   merchant toggles on in the theme editor; it loads `widget.js` with the public
   widget key.

## Shortcut: connect a single store without a Partner app

The OAuth flow above needs a Partner app and a deployed public app URL. To
connect one store (a test store, or a merchant you onboard by hand) you can skip
all of it — the merchant creates a **custom app inside their own store** and
hands you its Admin API token:

1. Shopify admin → **Settings → Apps and sales channels → Develop apps →
   Create an app**.
2. **Configure Admin API scopes** → tick `read_products` and `read_inventory` →
   **Save**.
3. **Install app** → reveal and copy the **Admin API access token** (`shpat_…`,
   shown once).
4. On the Waiter24 server:
   ```bash
   php artisan shopify:connect my-store.myshopify.com \
       --token=shpat_xxx --tenant=<tenant slug, id or import_token>
   ```
   The command validates the token against the Admin API, stores it encrypted,
   links the tenant, imports the catalog inline and prints the `<script>`
   snippet for the widget. `--queue` syncs via the queue instead, `--no-sync`
   only saves the credentials.
5. Widget: without the Partner app there is no app-embed block, so paste the
   printed `<script defer src=".../widget.js" data-key="...">` before `</body>`
   in **Online Store → Themes → Edit code → `layout/theme.liquid`**.

Trade-off: a custom app cannot register webhooks (those are created during
OAuth), so the catalog refreshes on the daily `shopify:resync` and on manual
`shopify:connect` / **Sync now** runs — not in real time.

## Requirements

- A Shopify **Partner account** and a store to install on (dev or live)
- The Waiter24 Laravel backend deployed and reachable at `SHOPIFY_APP_URL`
  (the OAuth, webhook and sync code lives there — **not** in this repo)
- **Shopify CLI 3.x** to deploy the app config + theme extension
- Admin API version **2025-01** (set in `shopify.app.toml` / `.env.example`)
- Storefront on an **Online Store 2.0** theme (for the app-embed widget block)

## Setup (developer, one-time)

1. Partner Dashboard → **Apps → Create app**. Note the **Client ID** and
   **Client secret**.
2. Put the credentials in the Waiter24 platform `.env` (see `.env.example` here):
   ```
   SHOPIFY_API_KEY=...
   SHOPIFY_API_SECRET=...
   SHOPIFY_APP_URL=https://waiter24.ai
   ```
3. In the app settings set:
   - **App URL**: `https://waiter24.ai/shopify/app`
   - **Allowed redirection URL**: `https://waiter24.ai/shopify/callback`
4. Put the Client ID into `shopify.app.toml` and run `shopify app deploy` from
   this folder to publish the config + theme extension.
5. `php artisan migrate` (creates `shopify_stores`).

## How a merchant installs it

1. Open the install link you send them (unlisted/custom distribution) and click
   **Install** → approve the requested data access.
2. The app opens inside Shopify admin. Paste the **import token** from the
   Waiter24 dashboard and press **Connect & sync**. The first menu sync starts.
3. Enable the widget: **Online Store → Themes → Customize → App embeds →
   Waiter24 Chat**, paste the **Widget Key**, **Save**. Optionally tick
   **Demo mode** to hide the widget from regular visitors — it then loads only
   on URLs carrying the `?waiter24_demo=1` parameter (e.g.
   `https://your-store.myshopify.com/?waiter24_demo=1`), which is remembered for
   the session and re-applied to in-chat links.
4. Done. The menu updates automatically from then on.

## Endpoints (Waiter24 platform)

| Route | Purpose |
|-------|---------|
| `GET /shopify/install` | Start OAuth |
| `GET /shopify/callback` | OAuth callback (HMAC + state verified) |
| `GET /shopify/app` | Embedded admin UI |
| `POST /shopify/app/connect` | Link tenant by import token |
| `POST /shopify/app/sync` | Manual "Sync now" |
| `POST /api/shopify/webhook` | All webhooks (HMAC-verified raw body) |

## Local testing

Shopify requires a **public HTTPS** app URL, so you can't point it straight at
`http://waiter.loc`. Expose your local Waiter24 instance through a tunnel
(`cloudflared tunnel --url http://waiter.loc`, ngrok, or `shopify app dev`) and
use the tunnel host everywhere a URL is configured:

- `shopify.app.toml` → `application_url` + the `auth.redirect_urls`
- `.env` → `SHOPIFY_APP_URL`
- the widget block's **Widget script URL** (theme editor) → `https://<tunnel>/widget.js`

Then `shopify app deploy` and install on a development store. Unlike the other
integrations, **this is not a Composer/standalone package** — the runtime lives
in the Waiter24 Laravel backend; this repo only ships the CLI app config + theme
extension.

## Follow-ups before App Store submission

- Replace the Blade embedded page with App Bridge + Polaris (session-token auth).
- Switch large-catalog full syncs to the Bulk Operations API.
- Optional app proxy so the widget key resolves by shop domain (no manual paste).
