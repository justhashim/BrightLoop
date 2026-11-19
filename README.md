# BrightLoop

Subscription billing for a small business: plans, quotations, a subscription lifecycle, recurring invoices, discounts and payments. Built for the Odoo Hackathon 2026.

## What it does

- Products, plans (daily, weekly, monthly, yearly), customers and quotations.
- Subscriptions move through `draft`, `quotation`, `confirmed`, `active`, `paused` and `closed`. Every transition is checked on the server, with the subscription row locked for the duration.
- A billing run turns due subscriptions into invoices. A dry-run preview shows what a run would create and what it would skip, and writes nothing.
- Discounts (fixed or percentage) with minimum amount, minimum quantity and usage limits. Redemptions are tracked per invoice.
- Invoices with line items and tax, printable invoice template, payments allocated to invoices.
- Three roles: admin, staff, customer. Customers only see their own records.

## How it works

Backend: Fastify controllers call services, services call repositories, repositories run plain SQL through `pg`.

- **Billing is idempotent.** A partial unique index on `invoices (subscription_id, period_index)` (excluding voided invoices) means a subscription period can only be invoiced once, however many times the run is triggered. Periods are derived from the subscription's start date, in UTC, so month-end dates don't drift.
- **Money is integers.** Amounts are stored in paise. Tax is a basis-point rate from config.
- **Snapshots.** Quotations, subscriptions and invoices keep the names and prices agreed at the time.
- **Transactions and locks.** Payments, invoice lifecycle changes, discount quotas and subscription actions run in transactions with `SELECT ... FOR UPDATE`.
- Plans with auto-close are closed by an hourly in-process timer.

The schema (18 tables) is documented in [`DATABASE_STRUCTURE.md`](DATABASE_STRUCTURE.md). The backend serves Swagger UI at `/docs`.

## Stack

- **Backend:** Node.js, Fastify 5, TypeScript, PostgreSQL (`pg`), JWT, bcryptjs, Nodemailer
- **Frontend:** Next.js 16 (App Router), React 19, Tailwind CSS 4, shadcn/ui, TanStack Query, react-hook-form, zod
- pnpm workspace

## Run it locally

```bash
pnpm install
```

Frontend (runs on a localStorage mock adapter by default, no backend needed):

```bash
pnpm --filter frontend dev
```

The mock demo accounts are `admin@example.com` and `member@example.com` (see `frontend/AGENTS.md`). To talk to the real backend, set `NEXT_PUBLIC_USE_MOCKS=false` and `NEXT_PUBLIC_API_URL` in `frontend/.env.local`.

Backend (needs a PostgreSQL database; SSL is on, written for Neon):

```bash
cp backend/.env.example backend/.env
pnpm --filter backend migrate
ADMIN_EMAIL=... ADMIN_PASSWORD=... pnpm --filter backend seed:admin
pnpm --filter backend dev
```

Env vars needed: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `APP_URL`, `DEFAULT_TAX_RATE_BPS`, `DEFAULT_PAYMENT_TERMS_DAYS` (see `backend/.env.example`). Optional: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`. Without SMTP the password-reset link is logged.

## Notes

Hackathon project, built by three people: Hashim, [Alwin Johnson](https://github.com/Alwin-Johnson) (database schema, discount and invoice services) and [Harikrishnan Santhosh](https://github.com/HariCodesHere) (docs).

The frontend defaults to mock data and the backend listens on port 3000 (hardcoded), so wiring them together is still manual. There are no tests.
