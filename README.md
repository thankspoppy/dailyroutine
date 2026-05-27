# Routine

A daily routine PWA. Tap-to-start countdowns per block, syncs across devices, sends a push when each block starts and ends.

**Stack:** Vite + vanilla JS · Supabase (auth + Postgres + Edge Function + pg_cron) · Vercel (static hosting) · Web Push.

---

## What it does

- Black terminal-style UI with green digital times
- Each block has a tap-to-start countdown timer (start / pause / resume / reset)
- "Right now" indicator shows which block you should be in
- State syncs in realtime across all your signed-in devices (start a timer on desktop, pause it on phone)
- Push notifications when blocks start and end — works even when the app is closed
- Installable as a real app on iOS, Android, Mac, Windows (PWA "Add to Home Screen")

---

## One-time setup

You'll do this once, in this order. Budget 30–45 minutes the first time.

### 1 · Clone & install

```bash
git clone <your-repo-url> routine
cd routine
npm install
cp .env.example .env.local        # fill in after step 4
```

> Recommend cloning to a **non-OneDrive** path (e.g. `~/dev/routine`). Node + OneDrive on Windows can be flaky.

### 2 · Push to GitHub

```bash
gh repo create routine --private --source=. --remote=origin --push
# or do it through the GitHub UI
```

### 3 · Create the Supabase project

1. Go to **supabase.com** → New project. Pick the EU (London) region.
2. In **SQL Editor**, paste the contents of `supabase/schema.sql` and run.
3. In **Authentication → Providers**, make sure **Email** is enabled.
4. In **Authentication → URL Configuration**, add your Vercel URL (and `http://localhost:5173`) to the **Site URL** and **Redirect URLs**.
5. From **Project Settings → API**, copy:
   - `Project URL` → `VITE_SUPABASE_URL`
   - `anon` `public` key → `VITE_SUPABASE_ANON_KEY`
   - `service_role` key → keep secret, used in step 5

### 4 · Generate VAPID keys (for push)

```bash
npx web-push generate-vapid-keys
```

Copy the **public** key → `VITE_VAPID_PUBLIC_KEY` in `.env.local` and Vercel.
Keep the **private** key — you'll paste it into Supabase secrets in step 5.

### 5 · Deploy the Edge Function + set secrets

Install the Supabase CLI: <https://supabase.com/docs/guides/cli>.

```bash
supabase login
supabase link --project-ref <your-project-ref>
supabase functions deploy notify --no-verify-jwt

supabase secrets set \
  VAPID_PUBLIC_KEY=<from step 4> \
  VAPID_PRIVATE_KEY=<from step 4> \
  VAPID_SUBJECT=mailto:matthias@theopium.studio
```

### 6 · Schedule pg_cron to call the function every minute

Open `supabase/cron.sql`, replace `<PROJECT-REF>` and `<ANON-KEY>`, paste into the SQL Editor and run.

Verify it scheduled: `select * from cron.job;`
Verify it's running: `select * from cron.job_run_details order by start_time desc limit 5;`

### 7 · Deploy to Vercel

1. Go to **vercel.com** → Add New Project → import the GitHub repo.
2. Framework preset: **Vite**. Build command and output stay at defaults.
3. **Environment Variables** — add the three from `.env.example`:
   - `VITE_SUPABASE_URL`
   - `VITE_SUPABASE_ANON_KEY`
   - `VITE_VAPID_PUBLIC_KEY`
4. Deploy.

### 8 · First sign-in + seed your schedule

1. Open your Vercel URL. Sign in with your email → click the magic link.
2. The UI will say "No schedule yet."
3. In Supabase SQL Editor: `select id, email from auth.users;` — copy your `id`.
4. Open `supabase/schema.sql`, uncomment the seed `insert into routine_blocks ...`, replace `YOUR-UUID-HERE` with your id, run it.
5. Refresh the app. Your routine appears.

### 9 · Install on phone + desktop

**iPhone:** open the Vercel URL in Safari → Share → **Add to Home Screen** → name it `ROUTINE`.

**Android:** open in Chrome → ⋮ menu → **Add to Home Screen** (or "Install app").

**Mac (Safari/Chrome/Edge):** open the URL → in Chrome/Edge there's an Install icon in the address bar; in Safari, File → "Add to Dock".

**Windows:** open in Chrome/Edge → Install icon in the address bar.

On first launch, the app will prompt for **push notification permission** — accept on every device you want pings on.

---

## Day-to-day

- **Generate PWA icons** (only when you change `public/icon.svg`):
  ```bash
  npm run generate-pwa-assets
  ```
- **Local dev:** `npm run dev` → <http://localhost:5173>
- **Push a UI change to prod:** `git push` — Vercel auto-deploys, PWAs auto-update on next open.
- **Change your schedule:** edit rows in the `routine_blocks` table in Supabase. Clients pick it up on next refresh. Edge function picks it up on next minute.
- **Disable push for a device:** open the app → top banner → "disable".
- **Disable push globally:** in Supabase, `delete from push_subscriptions where user_id = '<your-id>';` and unschedule cron with `select cron.unschedule('routine-notifier');`.

---

## File map

```
routine-app/
├── index.html
├── vite.config.js               PWA plugin config
├── pwa-assets.config.js         icon generation
├── public/icon.svg              source for all generated icons
├── src/
│   ├── main.js                  app entry, render, timer logic
│   ├── style.css                terminal aesthetic
│   ├── supabase.js              client singleton
│   ├── auth.js                  magic-link sign-in
│   ├── state.js                 load/save state + realtime sync
│   ├── notifications.js         push permission + subscribe
│   └── sw.js                    service worker (push handler)
└── supabase/
    ├── schema.sql               tables, RLS, realtime, seed
    ├── cron.sql                 pg_cron schedule
    └── functions/notify/index.ts   Deno edge function — sends pushes
```

---

## Notes

- **iOS push works only on PWAs added to home screen** (iOS 16.4+). It will not work in regular Safari tabs.
- **Realtime sync** uses Supabase's `postgres_changes` channel — works over websockets, included in free tier.
- **pg_cron** runs every minute; `notify` checks if the current London-time minute matches any block boundary and only sends if it does. So real push volume is ~16 pings/day, not 1,440.
- **Cost:** Supabase free tier + Vercel hobby = £0/month for personal use.
