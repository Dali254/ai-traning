# AI Trainer Hub — Project Structure & Progress

**This file is the source of truth.** Read it first in any new session before writing code.
Update the Progress section as work lands. Never change decisions silently — add to the Decisions log.

---

## 1. What this is

A platform where members earn money completing AI evaluation tasks (response comparison,
prompt evaluation, fact checking, classification, etc.). Members are paid to M-Pesa on an
admin-defined payout cycle. Admin controls every economic value — nothing is hardcoded.

**Positioning (important):** this is AI *evaluation / human feedback / data annotation*.
Never claim members train or modify ChatGPT, Claude or Gemini directly.

---

## 2. Stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 15 (App Router) + TypeScript | Vercel's native target, zero-config deploy |
| Styling | Tailwind CSS v4 | Neon Glass tokens in `globals.css` |
| Database | Postgres (Neon / Vercel Postgres) | Serverless-friendly, works on Vercel free tier |
| DB client | `pg` Pool | Plain SQL, no ORM — schema stays readable |
| Auth | Own cookie sessions (`jose` JWT + `bcryptjs`) | No NextAuth complexity; one less thing to configure |
| Payments | LipaLink / PesaFlux API | STK push + signed webhook |
| Cron | `vercel.json` crons | Daily task rotation + payout runs |

**Stack change note:** the original brief specified PHP/MySQL on cPanel. The user chose Vercel
hosting, which cannot run PHP/MySQL. Ported to Next.js + Postgres. All functional requirements
(admin control, ledger accounting, M-Pesa, rotation, quality control) are preserved.

---

## 3. Design system — "Neon Glass"

Defined once as CSS custom properties in `src/app/globals.css`. Never hardcode these hex values
in components; use the tokens.

```
--bg            #06070d   page ground
--bg-elev       #0a0b13   sticky bars, nav
--surface       rgba(255,255,255,0.04)    glass fill
--border        rgba(255,255,255,0.085)   glass border
--hairline      inset 0 1px 0 rgba(255,255,255,0.055)  top highlight on every card
--text          #f4f6fb
--text-dim      #95a0b6
--text-faint    #747e95
--label         #646c86
--accent-1      #7C8CFF   indigo
--accent-1-lt   #8E9BFF   indigo on dark text
--accent-2      #00E0B4   teal (money, success)
--accent-2-dim  #2fd3ad
--warn          #ffc454
--danger        #f48fbe
--grad          linear-gradient(118deg, #7C8CFF, #00E0B4)
```

Rules: Inter 400–900. Money and counts get `font-variant-numeric: tabular-nums` and
`letter-spacing:-0.03em` (`.num`). Display type uses negative tracking (−0.03em to −0.042em).
Every glass card carries the `--hairline` inset. Icons are **inline stroke SVG only — never
emoji**. Touch targets ≥44px. Mobile screens fit one viewport; minimal scrolling.

Brand logos for AI models are **admin-uploaded** (`ai_models.logo_path`), with a neutral
geometric SVG mark as fallback. We never ship copies of the OpenAI/Anthropic/Google marks.

---

## 4. File map

```
ai-trainer-hub/
├── PROJECT.md                   ← this file
├── README.md                    setup + Vercel deploy steps
├── package.json
├── tsconfig.json  next.config.ts  postcss.config.mjs
├── vercel.json                  cron schedules
├── .env.example                 every env var, no secrets
├── db/
│   ├── schema.sql               full schema — run once
│   └── seed.sql                 default settings, levels, models, admin
└── src/
    ├── middleware.ts            route protection
    ├── app/
    │   ├── globals.css          Neon Glass tokens
    │   ├── layout.tsx
    │   ├── page.tsx             landing
    │   ├── register/page.tsx    4 fields
    │   ├── login/page.tsx
    │   ├── pay/page.tsx         M-Pesa membership STK
    │   ├── dashboard/page.tsx
    │   ├── tasks/page.tsx       today's allocation
    │   ├── tasks/[id]/page.tsx  active task
    │   ├── withdraw/page.tsx
    │   ├── leaderboard/page.tsx
    │   ├── admin/
    │   │   ├── page.tsx         stats
    │   │   ├── members/  tasks/  generate/  models/
    │   │   ├── economics/       rewards, commissions, bonuses
    │   │   ├── payouts/         cycle + withdrawal queue
    │   │   └── audit/
    │   └── api/
    │       ├── auth/{register,login,logout}/route.ts
    │       ├── mpesa/{stk,webhook,status}/route.ts
    │       ├── tasks/{today,submit}/route.ts
    │       ├── withdrawals/route.ts
    │       ├── admin/**/route.ts
    │       └── cron/{rotate,payout}/route.ts
    └── lib/
        ├── db.ts                pg Pool (singleton)
        ├── session.ts           sign/verify cookie, getUser, requireAdmin
        ├── settings.ts          getSetting/setSetting — nothing hardcoded
        ├── wallet.ts            append-only ledger
        ├── rewards.ts           centralized reward engine
        ├── rotation.ts          daily assignment, recycle window
        ├── mpesa.ts             PesaFlux client + HMAC verify
        ├── taskgen.ts           Claude API batch generation
        ├── audit.ts             admin action log
        └── fmt.ts               money/date formatting
```

---

## 5. Money rules (do not violate)

1. **Ledger is append-only.** `wallet_transactions` rows are never UPDATEd or DELETEd.
   Every row records `balance_before` and `balance_after`. Corrections are new reversing rows.
2. **All balance math is server-side**, inside a DB transaction with `SELECT ... FOR UPDATE`
   on the wallet row.
3. **Payments are only successful when the provider says so** — the signed webhook or a
   status-API check. Never on a button click.
4. **Webhooks are idempotent**: unique index on `mpesa_transactions.transaction_id`;
   a repeat delivery must not double-credit.
5. **Every reward passes through `lib/rewards.ts`** so the calculation is identical everywhere.
6. Admin wallet adjustments require a reason and write an `audit_logs` row.

---

## 6. Payout cycle

Admin picks `daily | weekly | monthly` plus the day/time (`payout_settings` in `settings`).
Withdrawal requests are queued (`status='pending'`), then a cron run settles them together.
Members see the next payout date everywhere money appears.

## 7. Task rotation

- Admin generates a large pool via Claude API (`lib/taskgen.ts`), held for review by default.
- `task_assignments` gives each member N tasks/day (admin-set, default 2).
- A member never receives the same task twice inside the recycle window (default 30 days).
- Daily reset at the configured EAT time via `/api/cron/rotate`.
- **Open question:** generated tasks need a gold answer for accuracy scoring. Current plan is
  audit tasks — a fraction of each batch carries a human-verified answer, and member accuracy
  is scored only on those. Needs user sign-off.

---

## 8. Decisions log

| Date | Decision |
|---|---|
| 2026-11-02 | Vercel hosting → Next.js + Postgres instead of PHP/MySQL on cPanel |
| 2026-11-02 | Neon Glass chosen from 3 landing concepts (A) |
| 2026-11-02 | Model logos are admin-uploaded, never shipped; neutral SVG fallback |
| 2026-11-02 | Payout cycle configurable daily/weekly/monthly; monthly-on-30th is the default |
| 2026-11-02 | Registration cut to 4 fields; no confirm-password, no referral input (auto-detected) |
| 2026-11-02 | Tasks are a daily allocation from a recycling pool, not a browsable marketplace |
| 2026-11-02 | PesaFlux API key lives only in `PESAFLUX_API_KEY`; it is also the webhook HMAC secret |

---

## 9. Progress

### Phase 1 — Foundation ✅ COMPLETE
- [x] Scaffold (package.json, configs, globals.css tokens, vercel.json)
- [x] `db/schema.sql` + `db/seed.sql`
- [x] `lib/db.ts`, `lib/session.ts`, `lib/settings.ts`, `lib/fmt.ts`
- [x] `lib/wallet.ts` ledger, `lib/rewards.ts` engine
- [x] `lib/rotation.ts` daily assignment engine
- [x] `lib/mpesa.ts` + webhook/stk/status routes
- [x] `components/Icon.tsx` (SVG set, ModelMark fallbacks, Logo)
- [x] `middleware.ts` route protection
- [x] README with Vercel deploy steps
- [x] **Production build verified green**

### Phase 2 — Member flow  ← RESUME HERE
- [x] Landing page
- [x] Register (4 fields) + login + logout
- [ ] `/pay` — M-Pesa membership page (STK trigger + status polling UI)
      API routes are done; only the page/UI is left.
- [ ] `/dashboard` — balance card w/ next payout, quick actions, performance, level
- [ ] `/tasks` — today's allocation (use `getTodaysTasks`, `todaysPotential`,
      `poolStats`, `nextResetAt` from `lib/rotation.ts`)
- [ ] `/tasks/[id]` — active task UI per task_type + `POST /api/tasks/submit`
      (submit must: score vs expected_answer, call `computeReward`, `payReward`,
      update accuracy/points/streak, all in ONE transaction)
- [ ] `/withdraw` — request form, min/fee/net, payout-cycle notice
- [ ] `/leaderboard` — metric from `leaderboard_metric` setting, respect `hide_name`
- [ ] `/profile`

### Phase 3 — Admin
- [ ] Admin shell + stats
- [ ] Economics (rewards, commissions, bonuses, difficulty levels)
- [ ] Payout schedule + withdrawal queue
- [ ] Task generation (Claude API) + review + delete all
- [ ] AI models + logo upload
- [ ] Members management
- [ ] Audit log

### Phase 4 — Hardening
- [ ] Rate limiting on auth + submit
- [ ] Quality control scoring + audit tasks
- [ ] Cron: rotate, payout
- [ ] Notifications
