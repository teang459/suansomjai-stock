# Vercel Deployment Setup

The repo now ships with `vercel.json` (SPA rewrite + immutable cache for
`/assets/*`, no-cache for the PWA `sw.js`). HashRouter is gone — every
route is a real path now (`/settings`, `/pricing`, etc.), which means
Vercel's SPA fallback is required for direct navigation.

GitHub Pages is kept as a **live mirror**, not a retired path: its workflow
(`.github/workflows/deploy.yml`) still auto-deploys on every push to `master`
alongside Vercel. The Pages build sets `VITE_BASE_PATH=/chanthasy-stock/` so
it serves from the project sub-path, while Vercel builds at root (`/`).
Vercel is the primary origin; Pages is the backup/mirror at
`https://teang459.github.io/chanthasy-stock/`.

## 1. Link the repo

1. Sign in at https://vercel.com with the GitHub account that owns
   `teang459/chanthasy-stock`.
2. **Add New → Project → Import** `chanthasy-stock`.
3. Vercel auto-detects Vite. Confirm:
   - **Framework Preset:** Vite
   - **Build Command:** `npm run build` (already in `vercel.json`)
   - **Output Directory:** `dist`
4. Don't deploy yet — set env vars first.

## 2. Environment variables

Under **Project → Settings → Environment Variables**, add for both
**Production** and **Preview** environments:

| Name | Value |
| --- | --- |
| `VITE_SUPABASE_URL` | `https://kdsjqsfiunjhnajstwgi.supabase.co` |
| `VITE_SUPABASE_ANON_KEY` | (anon key from Supabase dashboard) |
| `VITE_SENTRY_DSN` | (optional — Sentry DSN; leave blank to disable) |

Vite inlines these at build time, so previews and production each get
their own bake.

## 3. First deploy

Click **Deploy**. Vercel:

* Runs `npm install` and `npm run build`.
* Picks up `vercel.json` for routing + headers.
* Returns a `*.vercel.app` URL once green.

Every PR from now on gets its own preview URL automatically.

## 4. Update Supabase / Stripe env

The Edge Functions read `APP_URL` to build redirect targets (password
reset email, Stripe checkout success / cancel, billing portal return).
Point it at the new origin:

```
APP_URL = https://<your-vercel-project>.vercel.app
```

Update under **Supabase → Project Settings → Edge Functions → Secrets**.

Also re-run the Stripe webhook test in Stripe Dashboard so it confirms
the endpoint is still reachable.

## 5. Custom domain (when ready)

When the domain (I1) is purchased:

1. **Vercel → Domains → Add** `chanthasystock.com` (or whatever).
2. Vercel shows the DNS records to add at the registrar.
3. After verification, set `APP_URL` to the custom domain in Supabase.
4. Update the Stripe webhook endpoint URL.
5. (Optional) Add the domain to Sentry's allowed domains.

## 6. Promoting the GitHub Pages mirror to primary

Pages already builds and deploys live on every push (see the intro), so no
code, router, or base-path change is required to fall back to it — the Pages
build serves from `/chanthasy-stock/` via `VITE_BASE_PATH`, and `BrowserRouter`
+ `public/404.html` handle SPA deep links there. To make Pages canonical:

1. Point users at `https://teang459.github.io/chanthasy-stock/` (USER_GUIDE
   already uses this URL).
2. Set `APP_URL` in Supabase → Edge Function secrets to the Pages URL so the
   password-reset and Stripe redirect targets resolve there.
3. Update the Stripe webhook endpoint if it references the Vercel origin.

> Before relying on Pages, sanity-check a direct deep link (e.g. open
> `…/chanthasy-stock/stock` in a fresh tab) — the sub-path + redirect flow is
> the part most likely to need attention.
