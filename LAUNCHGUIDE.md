# WanderFare — Launch Guide

Everything is already built and tested. This guide takes the code from the
`wanderfare.bundle` file (sent to you in chat) to a live site with real data
flowing. Total time: ~10 minutes, most of it waiting for the first build.

**You need:** git installed, the `wanderfare.bundle` file downloaded (use the
NEWER one — 8 commits), and your GitHub + Vercel logins. On Windows, run the
commands in Git Bash; on Mac/Linux, any terminal.

---

## Step 1 — Push the code to GitHub (3 min)

Open a terminal in the folder where you want the project to live:

```bash
# 1. Clone your (currently empty) repo
git clone https://github.com/andrewrath/WanderUpon.git
cd WanderUpon

# 2. Pull the code out of the bundle
#    (adjust the path to wherever you downloaded the file)
git fetch ~/Downloads/wanderfare.bundle claude/wanderfare-handoff-h1kcvk:claude/wanderfare-handoff-h1kcvk

# 3. Switch to it and push — both the working branch AND main
git checkout claude/wanderfare-handoff-h1kcvk
git push -u origin claude/wanderfare-handoff-h1kcvk
git push origin claude/wanderfare-handoff-h1kcvk:main
```

✅ **Check:** refresh github.com/andrewrath/WanderUpon — you should see
~50 files (README.md, SPEC.md, HANDOFF.md, src/, scripts/) on `main`.

> Windows note: replace `~/Downloads/wanderfare.bundle` with the real path,
> e.g. `/c/Users/you/Downloads/wanderfare.bundle` in Git Bash.

---

## Step 2 — Connect Vercel and deploy (3 min)

1. Go to **vercel.com/new**.
2. Under "Import Git Repository", pick **andrewrath/WanderUpon**.
   - If it's not listed, click "Adjust GitHub App Permissions" and grant
     Vercel access to the repo.
3. Vercel will want to create a project. Name it **wanderfare** — it will
   detect the existing project with that name; using it is fine (its one
   failed placeholder deployment gets superseded). Framework auto-detects
   as Next.js. Leave build settings alone.
4. Before clicking Deploy, open **Environment Variables** and add:

   | Name | Value |
   |---|---|
   | `CRON_SECRET` | any long random string (e.g. run `openssl rand -hex 24`) |

   That's the only required one. (Supabase URL + publishable key ship in the
   repo's `.env.production` — they're public-by-design client values.)
5. Click **Deploy** and wait for the build to go green (~2–3 min; the build
   fetches fresh airport data from OurAirports as a prebuild step — that's
   normal).

✅ **Check:** open the deployment URL. You should see the dark
"✈ DEPARTURES" screen. Type `DEN` in the airport box, pick Denver, hit
**"Show me somewhere"** — ~10 destination cards should stream in.

---

## Step 3 — Let the real data flood in (0 min — automatic)

Nothing to do. Supabase `pg_cron` jobs (already scheduled) call your app's
refresh endpoint **hourly**, pulling for every one of ~3,200 airports:

- descriptions from Wikipedia
- things-to-do from Wikivoyage
- photos + author credits from Wikimedia Commons
- popularity from Wikimedia pageviews (drives Match vs ✨ Wildcard)

The full catalog fills over the first day and refreshes daily after that.

**Want it to start right now instead of at the top of the hour?**
Get the job token: Supabase dashboard → project `wanderfare` → Table Editor
→ `job_secrets` → copy the `token` value of the `refresh` row. Then:

```bash
curl "https://YOUR-APP.vercel.app/api/jobs/refresh?task=all" \
  -H "Authorization: Bearer PASTE_TOKEN_HERE"
```

(Replace YOUR-APP with your actual Vercel domain.) The first run takes a few
minutes — it loads all airports, then starts on content.

✅ **Check:** Supabase → Table Editor → `airports` should show ~3,200 rows;
`destinations` rows flip from `pending` to `ok` as content lands. In the
app, new searches start showing real Wikimedia photos instead of
placeholders.

---

## Step 4 — Turn on real fares (5 min, whenever you're ready)

Cards say **"~ est."** until a fare provider is connected (no fare API has a
keyless tier).

1. Sign up free at **travelpayouts.com** → get your **API token** (and your
   **marker** if you want affiliate booking links).
2. Vercel → project **wanderfare** → Settings → Environment Variables → add:
   - `TRAVELPAYOUTS_TOKEN` = your token
   - `TRAVELPAYOUTS_MARKER` = your marker (optional)
3. Redeploy (Deployments → ⋯ → Redeploy).

Cards now show real market fares labeled **"~ est. from recent fares"**,
cached 6h in Supabase. Optional later upgrade: Amadeus Self-Service keys
(`AMADEUS_CLIENT_ID`/`AMADEUS_CLIENT_SECRET`) enable live "Live ✓" quotes.

---

## Step 5 — Optional hardening & niceties

- **Tighten database security:** Supabase dashboard → Project Settings →
  API keys → copy the `service_role` key → add it in Vercel as
  `SUPABASE_SERVICE_ROLE_KEY` → redeploy → then run the SQL in
  `scripts/tighten-rls.sql` (Supabase → SQL Editor). This removes the
  bootstrap "anon can write" policies.
- **Supabase MCP for local Claude Code:** the repo ships `.mcp.json`. On
  your machine, inside the repo: run `claude`, then `/mcp`, select
  `supabase`, and authenticate. Optional: `npx skills add supabase/agent-skills`.
- **Custom domain:** Vercel → Settings → Domains. ("WanderFare" is still a
  placeholder name — pick your real one whenever.)

---

## If something goes wrong

| Symptom | Fix |
|---|---|
| Vercel build fails at "prebuild" | OurAirports fetch hiccup — just redeploy. |
| Cards show placeholder photos | Content job hasn't reached those airports yet; run the Step 3 curl or wait an hour. |
| `/api/jobs/refresh` returns 401 | Wrong token — must be the `job_secrets` value or your `CRON_SECRET`. |
| Searches return nothing | Check Supabase `destinations` has `status = 'ok'` rows; if the DB is empty the app falls back to 60 curated destinations, so truly empty results mean a deploy/env issue — check Vercel runtime logs. |

## What's next on the roadmap (say the word in a Claude session)

Phase 3: Unsplash photo upgrade + Claude-written editorial prose + live fare
verification. Phase 4: MapLibre map view with animated arcs, saved-flights
sync, mobile bottom sheet. Phase 5: Playwright e2e + hardening. The full
context lives in `HANDOFF.md` and `SPEC.md` in the repo.
