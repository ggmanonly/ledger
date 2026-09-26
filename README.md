# LedgerX — accounts + PayPal billing setup

This wires the journal app to real accounts and real, server-verified paid
plans. Read the whole thing once before you start — the order matters.

## How it fits together

```
Browser (index.html)
  │  signs in / signs up
  ▼
Supabase Auth  ──────────────►  profiles table (Postgres, Row Level Security)
  │                                   ▲
  │  clicks "Subscribe"               │ updates plan/status
  ▼                                   │
PayPal Checkout (hosted by PayPal)    │
  │  subscription activated/cancelled/etc.
  ▼                                   │
PayPal Webhook  ───────────────────────
  (calls your Supabase Edge Function, which verifies the signature
   with PayPal's own API, then writes the plan using a service-role key
   the browser never sees)
```

The browser can *read* its own plan (to show/hide features) but can never
*write* it — only the Edge Function can, and only after PayPal confirms the
event is real. That's what makes this different from the old version, where
"Pro" was just a value sitting in `localStorage` that anyone could edit.

**Known limitation, on purpose:** trade/journal data itself still lives in
the browser's `localStorage`, not in the database. That keeps this a
one-evening project instead of a full data-migration. It means the "30 trade
cap" on the Base plan is a UI nudge, not a hard limit — a technically
determined user could bypass it locally. The *plan/subscription status*
itself, which is what people are actually paying for, is fully
server-enforced. If you later want hard limits on data too, the trades array
needs to move into a Supabase table with RLS — ask me and I'll build that.

---

## 1. Create the Supabase project

1. Go to supabase.com → New project. Note the **Project URL** and the
   **anon public key** (Project Settings → API) — you'll paste both into
   `index.html`.
2. SQL Editor → paste the contents of `schema.sql` → Run.
3. Authentication → Providers → make sure **Email** is enabled. For a first
   launch, under Authentication → Settings you can turn **off** "Confirm
   email" while you're testing (turn it back on before going live).

## 2. Create the PayPal app and billing plans

You're using PayPal, not Stripe, so subscriptions are set up as **Products →
Billing Plans** in the PayPal developer dashboard.

1. developer.paypal.com → Apps & Credentials → **Sandbox** tab first (always
   build and test in sandbox before touching live money).
2. Create an app → copy the **Client ID** and **Secret**.
3. Under the same dashboard, create a **Product** ("LedgerX subscription")
   and two **Billing Plans** under it: `Pro — $14.99/mo` and
   `Quant — $29.99/mo`. Copy each plan's **Plan ID** (looks like `P-xxxxx`).
4. Webhooks → Add Webhook → URL is your Edge Function URL (you'll have this
   after step 3 below) → subscribe to at least:
   `BILLING.SUBSCRIPTION.ACTIVATED`, `BILLING.SUBSCRIPTION.CANCELLED`,
   `BILLING.SUBSCRIPTION.EXPIRED`, `BILLING.SUBSCRIPTION.SUSPENDED`,
   `BILLING.SUBSCRIPTION.PAYMENT.FAILED`.
   Copy the **Webhook ID** it gives you.

## 3. Deploy the Edge Function

Requires the Supabase CLI (`npm install -g supabase`).

```bash
supabase login
supabase link --project-ref YOUR-PROJECT-REF
supabase functions deploy paypal-webhook --no-verify-jwt
```

Then set its secrets (all server-side, never shipped to the browser):

```bash
supabase secrets set \
  PAYPAL_CLIENT_ID=xxx \
  PAYPAL_SECRET=xxx \
  PAYPAL_WEBHOOK_ID=xxx \
  PAYPAL_ENV=sandbox \
  PAYPAL_PLAN_ID_PRO=P-xxxxx \
  PAYPAL_PLAN_ID_QUANT=P-xxxxx \
  SB_SERVICE_ROLE_KEY=xxx
```

`SB_SERVICE_ROLE_KEY` is the **service_role** key from Project Settings →
API — keep this out of git and out of the frontend entirely.

The deploy step prints your function's URL — go back to PayPal's webhook
config and paste it in there if you hadn't yet.

## 4. Fill in the frontend config

Open `index.html`, find the `CONFIG` object near the top of the `<script>`
tag, and fill in:

```js
var CONFIG={
  SUPABASE_URL:'https://your-project.supabase.co',
  SUPABASE_ANON_KEY:'your anon public key',
  PAYPAL_CLIENT_ID:'your sandbox client id',
  PAYPAL_ENV:'sandbox',
  PAYPAL_PLAN_ID_PRO:'P-xxxxx',
  PAYPAL_PLAN_ID_QUANT:'P-xxxxx'
};
```

## 5. Test end-to-end in sandbox

1. Host `index.html` anywhere (even just open it locally, or use
   `npx serve`) — no build step needed, it's one file.
2. Sign up with a test email, sign in.
3. Go to **Billing & plan**, click Subscribe on Pro, pay with a PayPal
   **sandbox buyer account** (developer.paypal.com → Sandbox → Accounts).
4. Within a few seconds the sidebar should show "Pro plan" — that means the
   webhook fired, was verified, and updated your row. Check
   Supabase → Table Editor → `profiles` to see it directly.
5. Cancel the subscription from the sandbox buyer's PayPal account and
   confirm the app drops back to "Base" within a few seconds.

If the plan never updates: check the Edge Function logs
(`supabase functions logs paypal-webhook`) — this will show you exactly
whether PayPal reached it, whether signature verification passed, and
whether the database write succeeded.

## 6. Go live

1. Repeat steps 2–3 on PayPal's **Live** tab (new client ID/secret, new
   plans, new webhook, new webhook ID — sandbox and live are separate).
2. `supabase secrets set PAYPAL_ENV=live PAYPAL_CLIENT_ID=... PAYPAL_SECRET=... PAYPAL_WEBHOOK_ID=... PAYPAL_PLAN_ID_PRO=... PAYPAL_PLAN_ID_QUANT=...`
3. Update `CONFIG` in `index.html` with the live client ID and
   `PAYPAL_ENV:'live'`.
4. Turn **on** "Confirm email" in Supabase Auth settings.
5. Deploy `index.html` to real hosting (Netlify, Vercel, Cloudflare Pages,
   GitHub Pages — any static host works, drag-and-drop is fine).
6. Do one real, small, real-money subscribe/cancel cycle yourself before
   telling anyone else it's live.

## Before you charge real people, also do this

- Add a **Terms of Service** and **Refund policy** page — PayPal will ask
  for these when you go live, and you'll want them regardless.
- Add a **Privacy Policy** — you're now collecting email addresses.
- Double check the in-app disclaimer (Settings → "Risk & data note" in the
  earlier build) says clearly that this is a journaling tool, not financial
  advice, and doesn't place trades.
- Decide what happens to a user's data if they cancel — this build keeps
  local data regardless of plan, which is a reasonable default, but say so
  somewhere a customer can find it.
