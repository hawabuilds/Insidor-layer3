# Go-live plan: putting this repo on insidor.app

Written 2026-10-10, before anything was changed. It follows `DEPLOY.md`'s order:
database, then the always-on services, then the apps, then the domain, then spending.

No secret, password or database URL appears in this file or anywhere in the repo. Where a
value is needed, this file names the variable and where it is pasted; you paste it.

---

## ★ Live now (2026-10-10): the web, with no backend

The plan changed to "web first, no backend". This is what is live:

| Address | Vercel project | What it serves |
|---|---|---|
| **https://insidor.app** and **https://www.insidor.app** | `insidor-mobile` (root `mobile/`) | The phone app, built with `VITE_SAMPLE_DATA=on`: sample data behind the permanent "Sample data" notice. Also at https://insidor-mobile.vercel.app |
| https://insidor-two.vercel.app | `insidor` (root `app/`) | The desktop app, same switch. No custom domain. |
| `insidor-layer3` | (old project) | Not touched. Its five cron jobs are still registered; turning them off is in its Settings → Cron Jobs. |

Both new projects: Node 24, Vite preset, install `pnpm install --frozen-lockfile`, build
`pnpm build`, output `dist`, and two env vars, `VITE_SAMPLE_DATA=on` and
`ENABLE_EXPERIMENTAL_COREPACK=1` (so the repo's pnpm 11 is used). No `VITE_READ_URL`, no
secret. Deploys are by CLI upload from the repo root (`vercel deploy . --project <name>`);
`.vercelignore` keeps every `.env` file out of the upload.

DNS at Namecheap (copied from Vercel): `A @ 216.150.1.1`, `A @ 216.150.16.1`,
`CNAME www 00ef20b6977e3de5.vercel-dns-016.com.`. Certificate: Let's Encrypt, issued and renewed by Vercel.

**When the backend exists:** set `VITE_READ_URL` on each project to the read service's URL,
REMOVE `VITE_SAMPLE_DATA` (the build fails if both are set), and redeploy.

**Rollback for the domain:** delete the three records above at Namecheap (the domain then
serves nothing, as it did before), or move `insidor.app` to another project in Vercel →
that project → Settings → Domains. Nothing else depends on it.

---

## 0. Read this first: what going live will and won't show

**After go-live the board will still show only the seeded sample stories, with the "Sample
data" notice on.** This is not a deployment problem. In the pipeline, GROUP decides which
posts belong together but does not yet write those memberships, nothing promotes a story,
and RANK is not wired (each throws `NotImplemented` in `services/runner/src/work/stages.ts`).
So no story built from real posts can exist yet, and the judge, which only looks at promoted
stories, will not be asked about live data.

What going live *does* give you: real posts ingested and tracked, real mints on the launches
rail, real market prices, the source indicator showing live sources, and every stage writing
its decisions — all on the production database, ready for the day GROUP's write-back and
promotion are built. That is the next piece of work if you want real narratives on the board.

---

## 1. What is live today (checked 2026-10-10, nothing changed)

| Thing | What I found | How |
|---|---|---|
| **insidor.app** | Registered at Namecheap (nameservers `pdns1/2.registrar-servers.com`). **Neither `insidor.app` nor `www.insidor.app` has a DNS record**, so the domain points at nothing and serves nothing. It is not attached to any Vercel project through DNS. | `dig`, `curl`, `whois` |
| **Old build on Railway** | Project `adventurous-kindness`, service `Insidor-layer3`. Its last five deployments (24 Jul – 4 Aug) all **FAILED**, so no worker from it is running there. | Railway CLI (logged in as you), read-only |
| **Old build on Vercel** | Unknown: the Vercel CLI is not installed here and I did not install it. The old repo's `vercel.json` declares **five cron jobs**: `/api/cron/ingest` every 2 min, `/api/cron/snapshot` every 2 min, `/api/cron/score` every 3 min, `/api/cron/cluster` every 3 min, `/api/cron/trends` every 45 min. Vercel runs crons whether or not a domain points at the project. | `hawabuilds/Insidor-layer3` on GitHub |

### ★ Check this yourself before Stage 3: is the old pipeline still spending?

`ingest` calls X and `score` calls Claude. If those crons still run with the same X key,
both pipelines would bill the same account. Check:

1. **Vercel dashboard** → the old Insidor project → **Settings → Cron Jobs**. If the five are
   listed and enabled, they are running. (Also: **Logs**, filtered to `/api/cron`, shows recent runs.)
2. **twitterapi.io dashboard** → usage for the last 24 hours. Any calls means something is still using the key.
3. **Anthropic Console** (platform.claude.com) → Usage → the key the old build used.

If anything is running, tell me before Stage 3. Pausing the old crons is your call (Stage 5).

---

## 2. What will run where

| Place | What | Why there |
|---|---|---|
| **Supabase** (a NEW project) | Postgres with pgvector: every table, role and migration (`store/migrations/0001`–`0038`) | Hosted Postgres. Your old Insidor Supabase project is not touched. |
| **Railway** (a NEW project) — always on | `services/read` (the API the apps read), `services/runner` (the pipeline loops and the judge), `services/chainwatch` (the mint websocket) | Each holds something open: an SSE live channel, a singleton advisory lock, a websocket. Serverless cannot. |
| **Railway** — cron jobs | `db:market` (every 5 min), `db:project` (every 5 min), `db:pairs` (every 5 min), `db:sources` (every 5 min), `db:label` (hourly) | Each runs once and exits. `DEPLOY.md` suggests Vercel Cron, but Vercel Cron calls an HTTP endpoint and these are shell commands, so Railway's cron services fit without new code. Railway cron runs at most every 5 minutes, so the board rebuilds every 5 minutes, not every minute as `DEPLOY.md` suggests; the live channel still pushes changes in between. `db:decide` is NOT scheduled: the runner already runs every stage, and `db:decide` takes the same lock, so it would fail while the runner is up. |
| **Vercel** — two projects | `app/` (desktop) and `mobile/` (phone), static builds | Static sites are what Vercel is for. |
| **Namecheap** | DNS for insidor.app | Stays where it is. Stage 5 adds the records Vercel asks for. |

---

## 3. Env vars for each place (names only, no values)

### Supabase
Nothing to set. Two things to do in the dashboard: **Database → Extensions → enable
`vector`**, and copy the **direct connection string on port 5432** (not the 6543 pooler).
Migrations create the roles; `pnpm db:migrate` creates the login user `insidor_app_user`.

### Railway — `read`
| Variable | What |
|---|---|
| `DATABASE_URL_APP` | `insidor_app_user` on port 5432. Never `postgres`. The service refuses to start if this equals `DATABASE_URL`. |
| `DATABASE_URL` | The owner connection, 5432 (read only so the service can refuse if the two match) |
| `PORT` | Railway sets this itself |
| `READ_ALLOWED_ORIGINS` | Exact origins, comma-separated: the two Vercel project URLs, then `https://insidor.app,https://m.insidor.app`. Matched exactly, so a per-deploy preview URL must be listed to work. Use each project's stable `*.vercel.app` alias for preview testing. |

### Railway — `runner`
| Variable | What |
|---|---|
| `DATABASE_URL` | The owner connection, 5432 (used for the `internal` role) |
| `HEALTH_PORT` | The health endpoint's port (Railway's health check points at it) |
| `SHUTDOWN_GRACE_MS`, `SINGLETON_LOCK_NAME` | As in `.env.example` (`20000`, `insidor.runner`) |
| `HEARTBEAT` | `off` to start |
| `SPEND` | `dry` to start |
| `DAILY_BUDGET_USD` | `1` to start (the policy ceiling is 3; the boot fails above it) |
| `DISCOVER` | `off` to start; `DISCOVER_TERMS` once it is `on` |
| `X_API_KEY` (`X_API_BASE` optional) | Stage 6 only |
| `JUDGE_MODEL`, `ANTHROPIC_API_KEY` | Stage 6 only. Both or neither. |
| `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT` | Optional: the free post source |

### Railway — `chainwatch`
| Variable | What |
|---|---|
| `DATABASE_URL` | The owner connection, 5432 |
| `HEALTH_PORT`, `SHUTDOWN_GRACE_MS`, `SINGLETON_LOCK_NAME`, `HEARTBEAT` | As in `.env.example` (`8081`, `20000`, `insidor.chainwatch`, `off`) |
| `MINT_FEED_ID`, `MINT_FEED_CHAIN`, `MINT_FEED_TRANSPORT`, `MINT_STREAM_URL`, `MINT_STREAM_BUFFER_LIMIT`, `MINT_STREAM_STALE_MS`, `MINT_OBSERVATION_LAG_S`, `COVERAGE_TOLERANCE_MS`, `POLL_INTERVAL_MS`, `PAGE_LIMIT`, `BACKOFF_BASE_MS`, `BACKOFF_MAX_MS` | The values in `.env.example` (no secrets among them; the stream is free and keyless) |

### Railway — each cron job
| Variable | What |
|---|---|
| `DATABASE_URL` | The owner connection, 5432 |
| `DATABASE_URL_INTERNAL` | The same owner connection (as `DEPLOY.md` says for Supabase) |
| Reddit variables | Only on `db:sources`, if Reddit is configured, so the source indicator reads its state |

### Vercel — `app` and `mobile`
| Variable | What |
|---|---|
| `VITE_READ_URL` | The Railway read service's public URL. Build-time: changing it needs a redeploy. |

Nothing else. No key of any kind goes into either app.

---

## 4. What each place costs per month

Prices read 2026-10-10 from each vendor's pricing page.

| Place | Plan | Monthly | Notes |
|---|---|---|---|
| **Supabase** | Pro | **$25** | Free allows 2 active projects and 500 MB. You already have the old project, the decision log grows every day, and a Free project is paused after a week without activity. Free works for a trial, not for production. |
| **Railway** | Hobby | **$5, likely $5–10** | $5 a month includes $5 of usage. Billed by use: $20 per vCPU-month, $10 per GB-month of memory. Three small Node services at roughly 0.1–0.2 GB each, mostly idle, plus short cron runs, is about $5–8 of usage. |
| **Vercel** | Pro | **$20** | Hobby is for personal, non-commercial use only, which a product isn't. Both apps fit in one Pro team. |
| **Vendors** (X, judge) | — | **$0 until Stage 6**, then at most `DAILY_BUDGET_USD` × 30 | At `DAILY_BUDGET_USD=1` that is at most **$30 a month**, all vendors together, enforced before every call and kept across restarts. X discovery alone is about $0.86 a day. The judge costs about $0.0006 per story on `claude-haiku-4-5`, and nothing until stories are promoted. |
| **Free sources** | — | $0 | DexScreener prices and the mint stream need no key and no card. |

**Total: about $50–55 a month fixed, plus at most $30 of vendor spend at the starting budget.**

---

## 5. How to roll back

Filled in for real in Stage 5, once the domain points somewhere. Until then there is nothing
to roll back: the domain serves nothing today, and Stages 2–4 touch no live surface.

- **Stages 2–4** (new Supabase, new Railway project, Vercel previews): delete the new
  projects. Nothing existing changes.
- **Stage 5** (domain): remove `insidor.app` / `m.insidor.app` from the new Vercel projects and
  delete the DNS records added at Namecheap. The old Vercel project stays deployed on its own
  `vercel.app` URL throughout, and can be given the domain back from its **Settings → Domains**.
- **Stage 6** (spending): set `SPEND=dry` (or `DISCOVER=off`) on the Railway runner. The
  change takes effect on restart; nothing is sent after it.

## 6. Not built yet

- **The earliest post using a sound.** When a promoted narrative's posts share a TikTok
  sound, its backfill should find the first post that used it. The TikTok adapter runs a
  hashtag, feed or account search only, and cannot search back to a cutoff, so this is not
  followed: the backfill logs `members share a sound or effect` with the story id and moves
  on. It needs a sound-search actor input, a price for it, and a cutoff the actor honours.
