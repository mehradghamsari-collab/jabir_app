# Jabir — Launch Guide (do these in order)

A plain, do-this-then-that checklist. You can stop after Step 2 and already have a real, installable app; the later steps add accounts, AI and money.

---

## STEP 0 — What you need (free to start)
- A **GitHub** account (code hosting)
- A **Vercel** account (free hosting + the small server functions)
- A **Supabase** account (free database + login)
- A **Google AI Studio** key for Gemini (free tier)
- A **Stripe** account *or* a local gateway (Zarinpal/IDPay) for money
- The `jabir-github-repo.zip` contents, unzipped into a folder called `jabir`

---

## STEP 1 — Put the code on GitHub
1. Install Git (git-scm.com). Open a terminal in the `jabir` folder.
2. Run:
   ```bash
   git init
   git add .
   git commit -m "Jabir v1"
   git branch -M main
   git remote add origin https://github.com/<your-username>/jabir.git
   git push -u origin main
   ```
   (Create the empty `jabir` repo on github.com first to get that URL.)

## STEP 2 — Deploy + make it installable (iOS / Android / PC)
1. On **vercel.com** → **Add New → Project** → import your `jabir` repo → **Deploy**. You'll get a URL like `jabir.vercel.app` with HTTPS.
2. Open that URL on your **iPhone in Safari** → tap **Share** → **Add to Home Screen**. It now opens full‑screen like a native app (this works because of `manifest.webmanifest` + `service-worker.js`).
   - Android: Chrome shows an **Install app** prompt automatically.
   - PC: Chrome/Edge show an **install icon** in the address bar.
3. (Optional, later) For real **App Store / Play Store** listings, go to **pwabuilder.com**, paste your URL, and it packages the same app for stores. iOS store submission needs a paid Apple Developer account ($99/yr); "Add to Home Screen" needs nothing.

✅ At this point Jabir is live and installable, redeem codes work, and the first 40 sign‑ups (once Step 3 is done) are free.

## STEP 3 — Database + real login (Supabase)
1. **supabase.com → New project.** Wait for it to finish.
2. **SQL Editor → New query →** paste **all** of `db/schema.sql` → **Run**. (Creates accounts, the first‑40‑free rule, subscriptions, codes, lessons, and security.)
3. **Authentication → Providers → Email →** enable it (turn on "Confirm email" off for testing, or magic links for passwordless).
4. **Sign up once with `rezaghamighami@gmail.com`** in your app's login screen — the database automatically makes you **admin**. Choose your password here; it is never stored in the code.
5. **Project Settings → API:** copy `Project URL` and `anon public` key.
6. Tell the app to use Supabase: in `index.html`, add before the closing `</body>`:
   ```html
   <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
   <script>const sb = supabase.createClient("PROJECT_URL","ANON_KEY");</script>
   ```
   Then swap the prototype `Store` (browser) calls for `sb.from(...).select/insert/upsert`, and the onboarding email step for `sb.auth.signInWithOtp(...)`. The app is structured so this is one contained change (one storage object, one sign‑in step). Read `is_paid` / `is_admin` from the user's `profiles` row.

## STEP 4 — See your users & email them
In Supabase **Table editor / SQL**:
```sql
select count(*) from profiles;                    -- total users
select email, plan, is_paid, created_at from profiles order by created_at;
select email from profiles where is_paid = true;  -- who paid
```
Export those emails to **Mailchimp / Buttondown / Resend** for updates and marketing. Only email people who opted in.

## STEP 5 — Turn on the AI (Gemini)
1. **aistudio.google.com/apikey → Create API key** (starts with `AIza…`).
   - The string you had before (`AQ.Ab8…`) was **not** an API key; generate this proper one.
2. In **Vercel → your project → Settings → Environment Variables**, add `GEMINI_KEY = AIza…`.
3. Point the app's AI call at your own server proxy `/api/gemini` (file already in the repo) so the key stays secret. Redeploy.

## STEP 6 — Money to your bank ($5/month)
**If Stripe is available to you:**
1. **stripe.com →** create account → add your **payout bank account** (Settings → Payouts). Stripe deposits there automatically.
2. Create a **Product** with a **recurring $5/month price**; copy the `price_…` id.
3. Vercel env vars: `STRIPE_SECRET_KEY`, `STRIPE_PRICE_ID`, `APP_URL`, and later `STRIPE_WEBHOOK_SECRET`.
4. Make the "Subscribe" button call `/api/create-checkout` (in the repo) and redirect to the returned URL.
5. **Stripe → Developers → Webhooks → Add endpoint:** `https://your-domain/api/stripe-webhook`, events `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`. Copy the signing secret into `STRIPE_WEBHOOK_SECRET`. This is what flips a user to **Pro** in the database.

**If you're in Iran (Stripe/PayPal won't work):** use **Zarinpal, IDPay, NextPay, or Pay.ir**. Each has a "request payment → user pays → verify callback" API. Create a serverless function like `/api/pay-start` and `/api/pay-verify`; on a verified payment, set `profiles.is_paid = true`. Meanwhile your **redeem codes** are the no‑gateway way to sell access.

## STEP 7 — Redeem codes (works with or without a gateway)
1. Sign in as admin → **Account → Admin → +40** → **Copy**.
2. Sell/share codes however you like; a user pastes one in **Account → Membership** and becomes Pro instantly.
3. With Supabase connected, codes are validated server‑side (`redeem_code()` in `schema.sql`) so they work on any device and can't be faked.

## STEP 8 — Add lessons to the Education tab
Insert rows into the `lessons` table (Supabase): set `youtube_id` for videos or `body_en`/`body_fa` for articles, and `is_free` for previews. Members (Pro) see them all.

---

### Quick reference — every secret you'll set in Vercel
```
GEMINI_KEY, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE,
STRIPE_SECRET_KEY, STRIPE_PRICE_ID, STRIPE_WEBHOOK_SECRET, APP_URL
```
Never put these in `index.html` or commit them to GitHub. `SUPABASE_ANON_KEY` is the only one that's safe in the browser (Row‑Level Security protects the data).
