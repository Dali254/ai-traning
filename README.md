# AI Trainer Hub

AI evaluation marketplace. Members complete response-comparison, prompt-evaluation,
fact-checking and classification tasks and earn rewards paid to M-Pesa on an
admin-defined payout cycle.

Built for **one-click Vercel deploy**. No servers, no Docker, no cPanel.

> **Read [`PROJECT.md`](./PROJECT.md) before changing code.** It holds the architecture,
> the file map, the money rules and the progress checklist.

---

## Deploy to Vercel in 5 steps

### 1. Get a Postgres database (2 min)

Easiest is Neon's free tier:

1. Go to [neon.tech](https://neon.tech) → sign up → **Create project**
2. Copy the connection string. It looks like:
   `postgresql://user:pass@ep-xxx.aws.neon.tech/neondb?sslmode=require`

Vercel Postgres works identically — Storage tab → Create → Postgres.

### 2. Create the tables (1 min)

In Neon's dashboard, open **SQL Editor**, then:

- paste the whole contents of `db/schema.sql` → **Run**
- paste the whole contents of `db/seed.sql` → **Run**

Or from your machine, if you have `psql`:

```bash
psql "YOUR_CONNECTION_STRING" -f db/schema.sql
psql "YOUR_CONNECTION_STRING" -f db/seed.sql
```

### 3. Generate two secrets

```bash
node -e "console.log('SESSION_SECRET=' + require('crypto').randomBytes(32).toString('hex'))"
node -e "console.log('CRON_SECRET='    + require('crypto').randomBytes(32).toString('hex'))"
```

### 4. Push to GitHub, then import to Vercel

```bash
git init && git add -A && git commit -m "AI Trainer Hub"
git remote add origin git@github.com:YOUR_USER/ai-trainer-hub.git
git push -u origin main
```

On [vercel.com](https://vercel.com): **Add New → Project → Import** your repo.
Vercel detects Next.js automatically — leave every build setting alone.

Before clicking Deploy, add the environment variables:

| Variable | Value |
|---|---|
| `DATABASE_URL` | the Postgres string from step 1 |
| `SESSION_SECRET` | from step 3 |
| `CRON_SECRET` | from step 3 |
| `PESAFLUX_API_KEY` | LipaLink dashboard → Integration → API Key |
| `PESAFLUX_BUSINESS_ID` | the business ID you collect into (e.g. `366`) |
| `ANTHROPIC_API_KEY` | optional — enables admin task generation |
| `NEXT_PUBLIC_APP_URL` | your Vercel URL, e.g. `https://your-app.vercel.app` |

Click **Deploy**.

### 5. Point the M-Pesa webhook at your app

In the LipaLink dashboard, set your business's **webhook URL** to:

```
https://YOUR-APP.vercel.app/api/mpesa/webhook
```

That endpoint verifies the `X-Pesaflux-Signature` HMAC on every request and is the
only thing that can mark a payment successful.

**Done.** Register the first account, then promote it to admin:

```sql
UPDATE users SET role = 'admin' WHERE email = 'you@example.com';
```

---

## Local development

```bash
cp .env.example .env.local     # fill in DATABASE_URL and SESSION_SECRET
npm install
npm run dev                    # http://localhost:3000
```

The M-Pesa webhook can't reach localhost. To test payments locally, run
`npx untun@latest tunnel http://localhost:3000` (or ngrok) and use the public
URL as the webhook.

---

## Scheduled jobs

`vercel.json` registers two crons. Vercel runs them in **UTC**, so the schedules
are offset for EAT (UTC+3):

| Path | Cron (UTC) | EAT | What it does |
|---|---|---|---|
| `/api/cron/rotate` | `0 21 * * *` | 00:00 | Hands each active member tomorrow's task set |
| `/api/cron/payout` | `0 15 * * *` | 18:00 | Settles queued withdrawals if today is a payout day |

The payout cron runs daily but exits immediately unless today matches the
configured cycle (`payout_cycle`, `payout_day_of_month`, `payout_day_of_week`).
That way admin can switch daily/weekly/monthly without touching `vercel.json`.

Vercel's free Hobby plan allows 2 crons at daily granularity, which is exactly
what this uses.

---

## Security notes

- Passwords are bcrypt (cost 11). Plain text is never stored or logged.
- Sessions are signed JWTs in an `httpOnly`, `secure`, `sameSite=lax` cookie.
- All SQL uses bound parameters — no string interpolation anywhere.
- `wallet_transactions` is append-only, enforced by a Postgres trigger. Balance
  changes happen inside a transaction with `SELECT … FOR UPDATE` on the wallet.
- Payments are confirmed by the signed webhook or the provider's status API —
  never by a client request claiming success.
- Webhook replays are idempotent: `mpesa_transactions.transaction_id` is unique
  and state only advances from `pending`.
- Failed logins lock the account for a configured window.
- Admin wallet adjustments require a reason and write an `audit_logs` row.
- Cron routes require the `CRON_SECRET` bearer token.

**If you shared your PesaFlux API key anywhere, rotate it.** It is both the API
credential and the webhook HMAC secret, so a leak lets someone forge payment
confirmations.

---

## Project layout

See [`PROJECT.md`](./PROJECT.md) §4 for the full file map. Short version:

```
db/schema.sql      run once — all tables
db/seed.sql        run once — default settings, levels, models
src/lib/           db, session, settings, wallet ledger, rewards, rotation, mpesa
src/app/           pages (member + admin) and API routes
src/app/globals.css  the Neon Glass design tokens
```

Nothing that affects platform economics is hardcoded. Rewards, commissions,
bonuses, difficulty levels, withdrawal limits, fees, the payout cycle and the
task rotation rules all live in the `settings` table and are editable from the
admin dashboard.
