# Jamena Farm: backend

A small Node.js server that hosts your website **and** powers its three forms.
**Zero npm dependencies**: it uses only what ships with Node 22.13+ (built-in SQLite, `fetch`, `http`).

## Run it

```bash
cp .env.example .env      # then fill in your Paystack keys + ADMIN_TOKEN
npm start                 # http://localhost:3000
npm test                  # 11 tests, Paystack is mocked (no network needed)
```

Requires **Node 22.13 or newer** (`node -v`). Your frontend lives in `public/` and is served as-is,
so `/api/...` calls in your JS work without any changes.

## What the frontend talks to

| Endpoint | Used by | What it does |
|---|---|---|
| `POST /api/signup` | `signup.js` | Validates and saves the delivery-list form |
| `POST /api/checkout/start` | `checkout.js` | Validates the order, **calculates the price on the server**, saves a `pending` order, returns `{publicKey, amountKobo, reference}` |
| `POST /api/checkout/verify` | `checkout.js` | Asks Paystack whether the reference was really paid for the right amount, then marks the order `paid` |
| `POST /api/paystack/webhook` | Paystack | Safety net: marks orders paid even if the customer closes the tab right after paying |
| `GET /api/admin/orders` | you | JSON list of orders (`?status=paid`, `?format=csv`, `?limit=`) |
| `GET /api/admin/signups` | you | JSON list of signups (`?format=csv`) |
| `GET /api/health` | hosting platform | Health check |

Admin calls need a header: `Authorization: Bearer <ADMIN_TOKEN>`. Example:

```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" "https://yourdomain.com/api/admin/orders?status=paid&format=csv" -o orders.csv
```

## Paystack setup (do this before going live)

1. Put your **test** keys in `.env` first and try a full purchase with Paystack's test cards.
2. In the Paystack dashboard → Settings → API Keys & Webhooks, set the **Webhook URL** to
   `https://yourdomain.com/api/paystack/webhook`.
3. Swap in your **live** keys when you're ready.

## Changing prices or products

Edit `PRODUCTS` in `src/config.js`. The price on the page (`data-price` in `products.html`) is only
for display. The server always charges its own price, so update both together. To add a product you
also need a matching card and `data-product` key in `products.html`.

## Deploying

Needs a host that runs Node 22.13+ (Render, Railway, Fly.io, a VPS). Set the env vars from
`.env.example` in the host's dashboard, set `TRUST_PROXY=1`, and **put `DB_PATH` on a persistent disk**.
The database is a single SQLite file, so back it up regularly. Serve over HTTPS.

## Layout

```
public/            your website (unchanged)
src/server.js      routes, static hosting, rate limiting
src/validate.js    input validation for signup + checkout
src/paystack.js    verify call, webhook signature check, reference generator
src/db.js          SQLite schema + queries
test/              API tests (mock Paystack)
data/              SQLite file is created here
```

## Security notes

- Prices, quantity limits and product keys are enforced server-side.
- Payments are confirmed with Paystack's secret key and the amount is compared; a mismatch is rejected.
- Webhooks are checked with Paystack's HMAC signature; secret keys never reach the browser.
- Rate limiting (20 requests/min/IP on public POST routes), 32 KB body cap, dotfile and path-traversal blocking.
- CSV exports neutralise spreadsheet formula injection.
- Not included: order confirmation emails/SMS. Currently new paid orders are printed to the server log and
  visible via the admin endpoints. Ask me if you'd like email notifications added.
