# DropCollect

> Paste one line of code. Start collecting emails instantly.

A zero-setup, embeddable email collection widget. Works on any website — Google Sites, Webflow, Framer, plain HTML. No API keys for your visitors, no backend to configure, no accounts required.

---

## How it works

1. Visitor goes to your DropCollect site → clicks **Generate**
2. They get a `<script>` snippet + a private dashboard URL
3. They paste the snippet anywhere — the widget renders immediately
4. Emails appear in their dashboard URL in real time

---

## Tech Stack

| Layer | Tech |
|-------|------|
| Frontend | Vanilla HTML/CSS/JS (no build step) |
| Backend | Vercel Serverless Functions (Node.js) |
| Database | Supabase (Postgres) |
| Widget | Self-contained IIFE script |
| Deploy | Vercel (one command) |

---

## Quick Deploy

### 1. Create a Supabase project

1. Go to [supabase.com](https://supabase.com) → New project (free tier is fine)
2. Open **SQL Editor** → paste contents of `supabase-schema.sql` → Run
3. Go to **Settings → API** → copy:
   - **Project URL** → `SUPABASE_URL`
   - **service_role** key → `SUPABASE_SERVICE_KEY` ⚠️ keep secret

### 2. Deploy to Vercel

```bash
# Install Vercel CLI
npm i -g vercel

# Clone and deploy
cd dropcollect
vercel

# Set environment variables
vercel env add SUPABASE_URL
vercel env add SUPABASE_SERVICE_KEY

# Redeploy with env vars
vercel --prod
```

Or deploy via the [Vercel dashboard](https://vercel.com/new) by connecting your GitHub repo — it auto-detects the config.

### 3. You're live

Visit your Vercel URL, generate a collector, paste the snippet anywhere.

---

## Project Structure

```
dropcollect/
├── index.html          # Landing page + generator wizard
├── dashboard.html      # Email dashboard (unique URL per collector)
├── public/
│   └── c.js            # The embeddable widget script
├── api/
│   ├── create.js       # POST /api/create — generate a new collector
│   ├── collect.js      # POST /api/collect — submit an email
│   ├── emails.js       # GET /api/emails?id= — dashboard data
│   └── config.js       # GET /api/config?id= — widget config (public)
├── supabase-schema.sql # Database setup
├── vercel.json         # Routing config
└── .env.example        # Environment variable template
```

---

## API Reference

### `POST /api/create`

Create a new collector.

**Body:**
```json
{
  "style": "minimal",          // "minimal" | "card"
  "headline": "Stay in the loop",
  "subheadline": "Get updates in your inbox",
  "buttonLabel": "Subscribe",
  "buttonColor": "#4dffcc",
  "tier": "free"               // "free" | "pro" | "pro_plus"
}
```

**Response:**
```json
{
  "id": "abc12345",
  "tier": "free",
  "emailCap": 200,
  "dashboardUrl": "https://yourdomain.com/dashboard.html?id=abc12345",
  "snippet": "<script src=\"https://yourdomain.com/c.js?id=abc12345\"></script>"
}
```

---

### `POST /api/collect`

Submit an email (called by the widget). CORS-open.

**Body:**
```json
{ "id": "abc12345", "email": "user@example.com" }
```

**Responses:**
- `201` — success
- `409` — already subscribed
- `429` — capacity reached
- `404` — collector not found

---

### `GET /api/emails?id=abc12345`

Get all emails for a collector (dashboard data).

**Response:**
```json
{
  "collector": { "id": "...", "tier": "free", "email_cap": 200, ... },
  "emails": [
    { "email": "user@example.com", "collected_at": "2025-01-15T10:30:00Z" }
  ],
  "count": 47
}
```

---

### `GET /api/config?id=abc12345`

Public widget config (called by c.js). CORS-open, cached 60s.

---

## Widget Styles

### Minimal
```
[ your@email.com _________ ] [ Subscribe ]
```
Single-line. Blends into any layout.

### Card
```
┌──────────────────────────────────────┐
│  Stay in the loop                    │
│  Get updates in your inbox           │
│  [ your@email.com _____ ] [Subscribe]│
└──────────────────────────────────────┘
```
Floating card with headline + subtext.

---

## Tier Configuration

| Feature | Free | Pro | Pro Plus |
|---------|------|-----|----------|
| Emails per collector | 200 | Unlimited | Unlimited |
| Collectors | Unlimited | Unlimited | Unlimited |
| Widget styles | Both | Both | Both |
| CSV export | ✓ | ✓ | ✓ |
| Custom domain | — | ✓ | ✓ |
| Email notifications | — | ✓ | ✓ |
| Team access | — | — | ✓ |
| Webhooks | — | — | ✓ |
| Price | Free | $7/mo | $19/mo |

To implement Pro/Pro Plus: add payment via Stripe, set `tier` in the collector row, and enforce via the serverless functions.

---

## Adding Pro/Pro Plus Payment (Roadmap)

1. Create a Stripe checkout session when user selects Pro/Pro Plus
2. On `checkout.session.completed` webhook, call `/api/create` with `tier: "pro"` or `tier: "pro_plus"`
3. The `email_cap` will be set to `null` (unlimited) automatically

---

## Security Notes

- Collector IDs are 8-char random alphanumeric (~2.8 trillion combinations). Brute force is impractical.
- Row-Level Security on Supabase tables — all access goes through the service key in serverless functions only
- IP addresses are stored for abuse detection but never returned to the dashboard
- No personally identifiable info beyond email address and submission timestamp
- CORS is open on `/api/collect` and `/api/config` because the widget runs on third-party sites

---

## Local Development

```bash
npm i -g vercel
cp .env.example .env.local
# Fill in SUPABASE_URL and SUPABASE_SERVICE_KEY

vercel dev   # starts at http://localhost:3000
```

---

## License

MIT
