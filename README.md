# Jabir — Perfumer's Atelier 🌿

Design fragrances note by note, grounded in real chemistry. Bilingual (English + Persian/RTL), with an AI composer, a scent‑map diagram, an education library, memberships, redeem codes, and an admin panel.

This repo is everything you need to publish Jabir as a web app (installable on iOS, Android and PC), give people real accounts, take $5/month subscriptions to your bank, and hand out access codes to users who can't pay online.

---

## What's in the box

```
index.html              The whole app (works on its own; freemium logic built in)
manifest.webmanifest    Makes it an installable app (PWA)
service-worker.js       Offline support + install
icons/                  App icons (replace with your own art anytime)
db/schema.sql           Supabase database: accounts, subscriptions, codes, lessons, security
api/gemini.js           Server proxy for the AI (keeps your key secret)
api/create-checkout.js  Starts a Stripe $5/mo subscription
api/stripe-webhook.js   Marks a user "paid" when they subscribe
.env.example            The secrets you'll set (never commit the real .env)
package.json, vercel.json
```

The app already runs **as‑is** with prototype accounts stored in the browser. The steps below upgrade it to real, secure, multi‑device accounts with payments.

---

## How the freemium model works

- **First 40 users are free forever.** The database assigns every signup a number; numbers 1–40 get `free_forever = true` (and full Pro access). This is enforced in `db/schema.sql` (the `handle_new_user` trigger).
- **Everyone after that is Free** and sees the AI composer locked and the scent‑map/diagrams blurred, with an "upgrade" prompt.
- **Pro = $5/month** (Stripe) *or* a **redeem code** you hand out. Pro unlocks the AI composer, the scent map & diagrams, and the full Education library.
- **You (admin)** sign in with `rezaghamighami@gmail.com` and get an Admin panel to generate and copy codes.

> The app currently flags admin by that email. Once the database is connected, admin is enforced server‑side (the `is_admin` column + RLS), which is the secure way.

---

## Step 1 — Put it online (free) and make it installable

1. Create a GitHub repo and push these files (see "Publishing to GitHub" below).
2. Import the repo into **Vercel** (free). It auto‑detects the static site + the `/api` functions and gives you a URL with HTTPS.
   - Static‑only alternative (no AI/Stripe functions): GitHub Pages or Netlify Drop.
3. Open the URL on a phone → browser menu → **Add to Home Screen**. Because of `manifest.webmanifest` + `service-worker.js`, it installs like a native app on **iOS, Android and desktop**. This is the fastest path to "on all platforms."
   - Want true App Store / Play Store listings later? Wrap the same URL with **PWABuilder** (pwabuilder.com) for Windows/Android, or **Capacitor** for iOS — no rewrite, it's the same app.

## Step 2 — Real accounts + database (free, secure) with Supabase

1. Create a project at **supabase.com** (free tier).
2. In **SQL Editor**, paste all of `db/schema.sql` and run it. This creates accounts, the first‑40‑free rule, subscriptions, redeem codes, lessons, and **Row‑Level Security** (the thing that stops one user reading another's data).
3. In **Authentication → Providers**, enable **Email** (turn on "magic link" for passwordless, or email+password). Sign up once with `rezaghamighami@gmail.com` — the trigger marks you admin automatically. **Set your password through this signup screen; never hard‑code it.**
4. In the app, replace the prototype storage with Supabase. The app was built so this is a small, contained change — everything goes through one `Store` object and one sign‑in step. You'll:
   - add the Supabase script tag and your `SUPABASE_URL` + `SUPABASE_ANON_KEY` (the anon key is safe in the browser **because** RLS protects the data);
   - swap the onboarding email step for `supabase.auth` sign‑in;
   - on sign‑in, load the user's rows into the app state; on change, upsert them;
   - read `is_paid` / `is_admin` from their `profiles` row instead of the local flags.

That gives you sign out → sign in on any device → your formulas are there.

## Step 3 — Track users & email them

Because every signup is a row in `profiles`, you already have your user list and who's paid:
```sql
select email, plan, is_paid, free_forever, created_at from profiles order by created_at;
select count(*) from profiles;                      -- how many users
select email from profiles where is_paid = true;    -- who paid
```
For **commercial email / updates**, export those emails (or connect via API) to an email tool like **Buttondown, Mailchimp, or Resend**, and send from there. Only email people who agreed; honour unsubscribe/deletion requests.

## Step 4 — Payments to your bank ($5/month)

1. Create a **Stripe** account and add your **payout bank account** (Stripe → Settings → Payouts). Stripe deposits subscription revenue to that bank on a schedule.
2. Create a **recurring Price** of **$5/month**; copy its `price_…` id into `STRIPE_PRICE_ID`.
3. Set env vars (`STRIPE_SECRET_KEY`, `STRIPE_PRICE_ID`, `APP_URL`) in Vercel.
4. Point the app's "Subscribe" button at `POST /api/create-checkout` (it returns a Stripe URL to redirect to). The included `startCheckout()` is the stub to replace.
5. In Stripe → **Webhooks**, add an endpoint to `https://your-domain/api/stripe-webhook` for `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`. Put the signing secret in `STRIPE_WEBHOOK_SECRET`. This webhook flips `is_paid` in your database — that's what unlocks Pro.

> **If you are in Iran (or anywhere Stripe/PayPal don't operate):** they will reject the account. Use a local gateway instead — e.g. **Zarinpal, IDPay, NextPay, or Pay.ir** — each has a simple "create payment → verify callback" API you'd call from a serverless function, then set `is_paid = true` on success. Until then (or always), your **redeem codes are the payment path**: sell/share codes through any channel you already use, and users type them in.

## Step 5 — Redeem codes (for users without a card)

- In the app, sign in as admin → **Account → Admin** → generate 10/40 codes → **Copy**. Hand them out.
- A user enters a code in **Account → Membership** and instantly becomes Pro.
- With Supabase connected, codes are validated server‑side by the `redeem_code()` function in `schema.sql` (so a code works on any device and can't be forged), and `generate_codes(n)` is admin‑only.

## Step 6 — Your own AI key (safely)

The "Describe a Feeling" composer uses Google Gemini.
1. Get a key at **aistudio.google.com/apikey** (free tier ≈ 500 requests/day on `gemini-2.5-flash`).
   - ⚠️ The string you shared earlier (`AQ.Ab8…`) is **not** an API key — Gemini keys start with `AIza`. Generate a proper one.
2. Put it in `GEMINI_KEY` (server env var) and let the app call **`/api/gemini`** (the proxy in this repo) instead of Google directly. That way your key is never visible to users and you can rate‑limit it.
   - For personal use only, the app also supports each user pasting *their own* key in Profile → AI.

---

## Publishing to GitHub (the commands)

```bash
cd jabir
git init
git add .
git commit -m "Jabir v1"
git branch -M main
git remote add origin https://github.com/<you>/jabir.git
git push -u origin main
```
Then "New Project" in Vercel → pick the repo → add the env vars from `.env.example` → Deploy. Every `git push` redeploys.

---

## UX/UI notes

The app is intentionally minimal — warm "atelier" palette, two clear fonts, soft animated transitions, a bottom tab bar on phones and a top rail on desktop, and gentle blur‑locks instead of hard walls. Keep new screens to the same pattern: one idea per panel, generous spacing, one primary action. Replace the icons in `icons/` with your own brand art (same sizes) whenever you like.

## A realistic order to ship in

1. **Today:** push to GitHub, deploy to Vercel, install it on your phone. Hand out a few redeem codes (admin panel).
2. **This week:** connect Supabase (Steps 2–3) so accounts sync and you can see/email users.
3. **When ready to charge:** wire Stripe (Step 4) or a local gateway; move the Gemini key to the server proxy (Step 6).

Everything is built so each step adds on without redoing the last. Happy blending. 🌸
