# eBay Deal Tracker — Implementation Plan

## Context

Ethan wants a personal site that periodically scans eBay for items matching saved "wants" (item keywords, price min/max, warranty, shipping, buy-now vs auction, result count), tracks listings over time, and surfaces good deals. Bidding stays manual for now — the app only shows and tracks data.

The app lives at `D:/Projects/ebay-deals/` (final name TBD) and will mirror `D:/Projects/boopurnoes/`'s stack and visual identity so it feels like a sibling to the existing site. Backend lives on the self-hosted Supabase at `ssh ethan@supabase:~/supabase-project`, which already runs Postgres 15.8, Edge Functions, `pg_net`, and has `pg_cron` available.

## Approach

**Stack** (copied from boopurnoes):
- Vite 8 + React 19 + TypeScript 5.9, React Router 7
- Vanilla CSS with CSS variables — reuse the exact palette from `D:/Projects/boopurnoes/src/index.css` (`--bg: #0a0a0a`, `--text: #fafafa`, etc.)
- `@supabase/supabase-js` for data + realtime
- Express 5 server (SPA host + eBay OAuth token cache proxy)
- Vitest for unit tests
- Docker multi-stage build, port 3000

**Data flow:**
1. Postgres holds `watches` (user searches), `listings` (deduped by eBay item id), `price_snapshots` (time series), `deal_scores` (computed per snapshot).
2. `pg_cron` fires every 15 min → calls an Edge Function `scan-ebay` with each active watch.
3. Edge Function hits eBay Browse API `/item_summary/search` with the watch filters, upserts listings, writes a snapshot row per listing, computes a deal score.
4. Frontend subscribes to Postgres realtime on `listings` and `deal_scores` — UI updates live while the tab is open.
5. A single-page dashboard shows watches, their matching listings ranked by deal score, and a detail view with a price-history sparkline per listing.

**eBay access:**
- eBay Browse API (free tier, 5,000 calls/day — well under budget for ~50 watches @ 15-min refresh).
- App-level OAuth token (client credentials flow) cached in Postgres `ebay_token` table with `expires_at`. Edge Function refreshes on demand.
- No user OAuth needed since we're read-only.

**"Good deal" score (combined, 0–100):**
- 40% weight: `(user_max_price - current_price) / user_max_price` clamped to [0,1]
- 30% weight: `1 - (current_price / median_current_active_price_for_this_watch)` clamped
- 30% weight: `1 - (current_price / min_observed_price_for_this_listing)` clamped (only after ≥3 snapshots)
- Badge thresholds: 80+ "hot", 60–79 "good", 40–59 "ok", <40 hidden unless user toggles.

**Auth:**
- Single-user for now. Reuse boopurnoes' Supabase auth (`AuthContext.tsx`, PKCE, shared cookie on `.boopurno.es`). One user's rows only — RLS policy `auth.uid() = user_id` on every table.

**Notifications:** in-app only. Realtime badge in the header counts listings with score ≥ 80 unseen since last view. No email/Discord/push in v1.

## Files to create

```
D:/Projects/ebay-deals/
├── package.json                         # Copy boopurnoes scripts, swap name
├── vite.config.ts                       # Identical to boopurnoes
├── tsconfig.{json,app.json,node.json}   # Identical
├── eslint.config.js                     # Identical
├── Dockerfile, docker-compose.yml       # Adapt from boopurnoes
├── index.html
├── src/
│   ├── main.tsx, App.tsx
│   ├── index.css                        # Copy palette/layout from boopurnoes
│   ├── lib/
│   │   ├── supabase.ts                  # Copy from boopurnoes (swap VITE_ vars)
│   │   ├── AuthContext.tsx              # Copy from boopurnoes
│   │   ├── useWatches.ts                # CRUD + realtime on watches
│   │   ├── useListings.ts               # Realtime listings for a watch
│   │   └── dealScore.ts                 # Pure scoring fn (unit tested)
│   ├── pages/
│   │   ├── Home.tsx                     # Watch list + overall hot-deal feed
│   │   ├── WatchEditor.tsx              # Create/edit a watch
│   │   ├── WatchDetail.tsx              # Listings table for one watch
│   │   └── ListingDetail.tsx            # Price history sparkline + eBay link
│   └── components/
│       ├── WatchCard.tsx
│       ├── ListingRow.tsx
│       ├── DealBadge.tsx
│       └── PriceSparkline.tsx           # Minimal inline SVG, no chart lib
├── shared/
│   └── dealScore.ts                     # Shared scoring fn (FE + Edge Function)
├── server/
│   ├── index.ts                         # Adapt boopurnoes Express setup
│   └── tsconfig.json
└── supabase/
    ├── migrations/
    │   ├── 0001_init.sql                # tables + RLS + pg_cron + pg_net enable
    │   └── 0002_cron.sql                # schedule scan-ebay every 15 min
    └── functions/
        └── scan-ebay/
            └── index.ts                 # Deno edge function
```

## Schema (`supabase/migrations/0001_init.sql`)

```sql
create extension if not exists pg_cron;
create extension if not exists pg_net;

create table watches (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  query text not null,
  price_min numeric, price_max numeric,
  require_warranty boolean default false,
  shipping_free_only boolean default false,
  mode text check (mode in ('buy_now','auction','both')) default 'both',
  max_results int default 25,
  active boolean default true,
  created_at timestamptz default now()
);

create table listings (
  ebay_item_id text primary key,
  watch_id uuid not null references watches(id) on delete cascade,
  title text, url text, image_url text,
  seller_username text, seller_feedback_pct numeric,
  condition text, buying_options text[],
  auction_end_at timestamptz,
  current_price numeric, shipping_cost numeric, currency text,
  last_seen_at timestamptz default now(),
  first_seen_at timestamptz default now()
);

create table price_snapshots (
  id bigserial primary key,
  ebay_item_id text references listings(ebay_item_id) on delete cascade,
  price numeric, bid_count int,
  captured_at timestamptz default now()
);

create table deal_scores (
  ebay_item_id text primary key references listings(ebay_item_id) on delete cascade,
  score int, computed_at timestamptz default now()
);

create table ebay_token (
  id int primary key default 1,
  access_token text, expires_at timestamptz
);

-- RLS
alter table watches enable row level security;
alter table listings enable row level security;
alter table price_snapshots enable row level security;
alter table deal_scores enable row level security;

create policy "own watches" on watches for all using (auth.uid() = user_id);
create policy "own listings" on listings for all
  using (exists (select 1 from watches w where w.id = listings.watch_id and w.user_id = auth.uid()));
-- analogous policies for snapshots, scores
```

## Reusable code from boopurnoes

- `D:/Projects/boopurnoes/src/lib/supabase.ts` → copy wholesale, change env var names
- `D:/Projects/boopurnoes/src/lib/AuthContext.tsx` → copy wholesale
- `D:/Projects/boopurnoes/src/index.css` → copy `:root` palette, layout primitives (`.back`, `.login`, fluid `clamp()` padding), dropdown keyframes
- `D:/Projects/boopurnoes/server/index.ts` → copy Express skeleton, drop deck-solve/uma-proxy routes
- `D:/Projects/boopurnoes/vite.config.ts`, eslint config, tsconfigs, Dockerfile → copy

## Build sequence

1. Scaffold project from boopurnoes copy; strip domain code; verify `npm run dev` boots.
2. Write migrations (`0001_init.sql`), apply via `docker exec supabase-db psql ... -f`; verify tables + RLS.
3. Enable `pg_cron`, write `0002_cron.sql` that schedules 15-min call to edge function via `pg_net.http_post`.
4. Write `scan-ebay` edge function; test manually with curl against one watch row.
5. Wire `shared/dealScore.ts` with unit tests (Vitest) — test the three-factor formula against fixtures.
6. Build `Home` + `WatchEditor` + `useWatches` hook; get watch CRUD working end-to-end.
7. Build `WatchDetail` + `useListings` realtime subscription; confirm live updates when the cron runs.
8. Add `PriceSparkline` + `ListingDetail`.
9. Dockerize; deploy alongside boopurnoes.

## Verification

- **Unit**: `npm test` — `shared/dealScore.ts` fixtures cover hot/good/ok/hidden thresholds.
- **Integration**: create a watch for a common item (e.g. "steelbook blu-ray"), manually invoke edge function via `curl`, verify listings + snapshots appear in Studio.
- **Cron**: wait 15 min after applying `0002_cron.sql`, check `cron.job_run_details` for a successful run.
- **Realtime**: open the Vite dev server, watch the `WatchDetail` page, manually insert a `listings` row in Studio, confirm UI updates without refresh.
- **End-to-end**: edit a watch's `price_max` down, confirm deal scores recompute on next cron tick.
- **Rate limit**: confirm daily call count stays under 5,000 — with 50 watches × 96 runs/day = 4,800, sits just under; add a per-run cap as safety.

## Open items (not blockers)

- Final project name (placeholder: `ebay-deals`).
- Domain / subdomain on `boopurno.es`.
- Whether to expand to multi-user later (RLS is already set up for it).
