# BASEEVENTS SETUP - AGENT RUNBOOK

You are setting up BaseEvents in this project: a lightweight, self-hosted event analytics pipeline. It consists of a Supabase `events` table, a server-side ingestion endpoint, and a browser tracker script. Follow the steps in order. All file contents you need are inlined below at the point you need them. Do not skip steps.

There is no dashboard. The user queries their data by asking an AI agent (Cursor, Claude, Codex) to read the `events` table directly. Sample queries land in `BASEEVENTS-QUERIES.md` at the end of this setup.

> **Vercel location note:** For projects hosted on Vercel, do **not** use MaxMind or Cloudflare for city-level analytics. Vercel injects request geo headers on deployed apps, including `x-vercel-ip-country`, `x-vercel-ip-country-region`, `x-vercel-ip-city`, and `x-vercel-ip-timezone`. The tracker should post to a same-origin Next route handler such as `/api/analytics`, and that route should read Vercel's headers and write `country`, `region`, `city`, and `timezone` to Supabase. These headers are not present on localhost unless faked in tests.

> **Legacy note:** The Supabase Edge Function flow below still works for non-Vercel hosts, but it has no built-in geo data. If you deploy a vanilla Edge Function without Cloudflare in front, `country`, `region`, and `city` will all be null. For Vercel-hosted projects, prefer the Next route handler ingestion pattern in section 4a.

---

## 0. PREAMBLE - DO THIS FIRST

Before running any command, gather these inputs from the user by asking them directly:

1. **`SITE_SLUG`** - a short identifier for this prototype. Lowercase alphanumeric, underscore or dash only, max 32 chars (regex: `^[a-z0-9_-]{1,32}$`). Example: `hematic`, `pre`, `ember`. This is how events are tagged.
2. **`ALLOWED_ORIGIN`** - the production URL where the tracker will run, including scheme, no trailing slash. Example: `https://hematic.com`. Multiple origins are comma-separated. **This is required.** The endpoint will reject all requests if it is not set. There is no wildcard fallback.
3. **`SUPABASE_PROJECT_REF`** - the project ref from the Supabase dashboard URL (`https://supabase.com/dashboard/project/<THIS_IS_THE_REF>`). Only needed if the project is not already linked.

Do not proceed until you have `SITE_SLUG` and `ALLOWED_ORIGIN`. `SUPABASE_PROJECT_REF` is only required if `supabase/config.toml` does not already exist or is not linked.

If the user does not know their project ref, tell them to open the Supabase dashboard, click their project, and copy the string after `/project/` in the URL.

---

## 1. PREREQUISITES

Run these checks. If any fail, fix them before continuing.

```bash
# Check the Supabase CLI is installed
supabase --version
# If this fails, install it:
#   macOS:   brew install supabase/tap/supabase
#   Windows: scoop install supabase
#   Linux:   see https://github.com/supabase/cli#install-the-cli

# Check the user is logged in
supabase projects list
# If this fails with an auth error, run:
#   supabase login
```

If you need to install the CLI, prefer the user's platform-native package manager. Do not install it globally via npm unless nothing else is available.

---

## 2. INITIALIZE AND LINK THE PROJECT

Check whether Supabase is already initialized in this repo:

```bash
ls supabase/config.toml 2>/dev/null && echo "ALREADY_INITIALIZED" || echo "NEEDS_INIT"
```

**If `NEEDS_INIT`:**
```bash
supabase init
```

Check whether the project is linked:
```bash
cat supabase/.temp/project-ref 2>/dev/null || echo "NOT_LINKED"
```

**If `NOT_LINKED`:**
```bash
supabase link --project-ref SUPABASE_PROJECT_REF
```

Replace `SUPABASE_PROJECT_REF` with the value the user gave you. If this fails asking for a database password, tell the user to paste their Supabase DB password (they set it when they created the project; they can reset it in Project Settings > Database if they've forgotten).

---

## 3. CREATE THE DATABASE SCHEMA

Create a new migration file. Use a timestamped filename to match Supabase conventions.

```bash
mkdir -p supabase/migrations
TIMESTAMP=$(date -u +"%Y%m%d%H%M%S")
MIGRATION_FILE="supabase/migrations/${TIMESTAMP}_baseevents_schema.sql"
echo "Creating $MIGRATION_FILE"
```

Write the following content to `$MIGRATION_FILE`:

```sql
-- BaseEvents schema

create extension if not exists "uuid-ossp";

create table if not exists public.events (
  id            uuid primary key default uuid_generate_v4(),
  site          text not null,
  event_name    text not null,
  visitor_id    text not null,
  user_id       text,
  session_id    text,
  path          text,
  referrer      text,
  utm_source    text,
  utm_medium    text,
  utm_campaign  text,
  utm_term      text,
  utm_content   text,
  country       text,
  region        text,
  city          text,
  timezone      text,
  device        text,
  browser       text,
  os            text,
  ip_hash       text,
  user_agent    text,
  props         jsonb not null default '{}'::jsonb,
  created_at    timestamptz not null default now(),

  constraint events_site_format        check (site ~ '^[a-z0-9_-]{1,32}$'),
  constraint events_event_name_length  check (char_length(event_name) between 1 and 100),
  constraint events_event_name_format  check (event_name ~ '^[$a-zA-Z0-9_.-]+$'),
  constraint events_visitor_length     check (char_length(visitor_id) <= 64),
  constraint events_user_length        check (user_id is null or char_length(user_id) <= 128),
  constraint events_session_length     check (session_id is null or char_length(session_id) <= 64),
  constraint events_path_length        check (path is null or char_length(path) <= 2000),
  constraint events_referrer_length    check (referrer is null or char_length(referrer) <= 2000),
  constraint events_user_agent_length  check (user_agent is null or char_length(user_agent) <= 500),
  constraint events_props_size         check (octet_length(props::text) <= 4096)
);

create table if not exists public.identities (
  site           text not null,
  visitor_id     text not null,
  user_id        text not null,
  user_props     jsonb not null default '{}'::jsonb,
  first_seen_at  timestamptz not null default now(),
  last_seen_at   timestamptz not null default now(),
  primary key (site, visitor_id, user_id),

  constraint identities_site_format  check (site ~ '^[a-z0-9_-]{1,32}$'),
  constraint identities_props_size   check (octet_length(user_props::text) <= 4096)
);

-- Core hot-path indexes
create index if not exists events_site_created_idx
  on public.events (site, created_at desc);
create index if not exists events_site_visitor_idx
  on public.events (site, visitor_id, created_at);
create index if not exists events_site_user_idx
  on public.events (site, user_id, created_at)
  where user_id is not null;
create index if not exists events_site_event_idx
  on public.events (site, event_name, created_at desc);
create index if not exists identities_user_idx
  on public.identities (site, user_id);

-- Optional: GIN index on props. Enables `props->>'plan' = 'pro'` queries
-- but adds write amplification. Uncomment if you query props frequently
-- and your event volume can absorb the write cost.
-- create index if not exists events_props_idx
--   on public.events using gin (props);

alter table public.events enable row level security;
alter table public.identities enable row level security;
-- No policies yet. Only the service role can read/write until you add policies
-- for an authenticated dashboard.
```

Push the migration to the remote database:

```bash
supabase db push
```

If `supabase db push` fails because of prior migration state conflicts (rare, but possible if the project has drifted), fall back to running the SQL directly via the Supabase dashboard SQL editor. Print the SQL above and tell the user to paste it into the SQL editor at `https://supabase.com/dashboard/project/<ref>/sql` and click Run. Then continue to step 4.

---

## 4. CREATE THE INGESTION ENDPOINT

Two paths. Pick one based on the host:

- **4a:** Vercel-hosted Next.js App Router (recommended). Same-origin route handler with full geo from Vercel headers.
- **4b:** Legacy/non-Vercel. Supabase Edge Function. No geo without Cloudflare.

### 4a. Vercel-hosted Next.js App Router

Create `app/api/analytics/route.ts` with the contents below. This is the preferred path because Vercel adds city-level request headers to deployed requests (`x-vercel-ip-country`, `x-vercel-ip-country-region`, `x-vercel-ip-city`, `x-vercel-ip-timezone`).

If `@supabase/supabase-js` is not already a dependency, install it:

```bash
npm install @supabase/supabase-js
```

Write the following to `app/api/analytics/route.ts`:

```typescript
// app/api/analytics/route.ts
//
// BaseEvents ingestion route handler for Vercel-hosted Next.js.
// Required env: SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY,
// IP_HASH_SALT, BASEEVENTS_ALLOWED_ORIGINS (comma-separated origins).

import { createClient } from '@supabase/supabase-js'

export const runtime = 'nodejs'
export const dynamic = 'force-dynamic'

// ----- limits -----
const MAX_BODY_BYTES = 8 * 1024
const MAX_PROPS_BYTES = 4 * 1024
const MAX_EVENT_NAME = 100
const MAX_PATH = 2000
const MAX_REFERRER = 2000
const MAX_VISITOR = 64
const MAX_USER = 128
const MAX_SESSION = 64
const MAX_UTM = 200
const MAX_TIMEZONE = 64
const MAX_USER_AGENT = 500
const IDENTIFY_BACKFILL_WINDOW_MIN = 60

// Per-visitor rate limit. 300/min is generous enough for chatty apps
// (canvas tools, form builders) while still catching infinite-loop bugs
// and abuse. Override with BASEEVENTS_RATE_LIMIT_PER_MINUTE if needed.
// Note: each event currently costs one extra Supabase count query for
// the rate check. See section 9 of SETUP-BASEEVENTS.md for token bucket
// alternatives at scale.
const RATE_LIMIT_PER_MINUTE = parseInt(
  process.env.BASEEVENTS_RATE_LIMIT_PER_MINUTE ?? '300',
  10,
)

const SITE_REGEX = /^[a-z0-9_-]{1,32}$/
const EVENT_NAME_REGEX = /^[$a-zA-Z0-9_.-]+$/

const ALLOWED_ORIGINS = (process.env.BASEEVENTS_ALLOWED_ORIGINS ?? '')
  .split(',')
  .map(s => s.trim())
  .filter(Boolean)

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { persistSession: false } },
)

const BOT_PATTERNS = [
  /bot/i, /crawler/i, /spider/i, /scraper/i,
  /facebookexternalhit/i, /slackbot/i, /twitterbot/i, /discordbot/i,
  /whatsapp/i, /telegrambot/i, /linkedinbot/i, /preview/i,
  /headless/i, /phantomjs/i, /puppeteer/i, /playwright/i,
  /lighthouse/i, /pingdom/i, /uptimerobot/i, /gtmetrix/i,
  /ahrefs/i, /semrush/i, /mj12/i, /dotbot/i,
]

// ----- helpers -----
function corsHeaders(origin: string | null) {
  const allowed = origin && ALLOWED_ORIGINS.includes(origin) ? origin : ''
  return {
    'Access-Control-Allow-Origin': allowed,
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'content-type',
    'Vary': 'Origin',
  }
}

function jsonResponse(body: unknown, status: number, cors: Record<string, string>) {
  return new Response(JSON.stringify(body), {
    status,
    headers: { ...cors, 'content-type': 'application/json' },
  })
}

function jsonError(code: string, status: number, cors: Record<string, string>) {
  return jsonResponse({ error: code }, status, cors)
}

function trunc(s: unknown, n: number): string | null {
  if (typeof s !== 'string') return null
  return s.length > n ? s.slice(0, n) : s
}

function stripQuery(url: string | null): string | null {
  if (!url) return null
  const cuts = [url.indexOf('?'), url.indexOf('#')].filter(n => n >= 0)
  return cuts.length ? url.slice(0, Math.min(...cuts)) : url
}

function originOnly(url: string | null): string | null {
  if (!url) return null
  try { return new URL(url).origin } catch { return null }
}

function parseUA(ua: string) {
  const device = /Mobi|Android|iPhone/i.test(ua) && !/iPad|Tablet/i.test(ua)
    ? 'mobile'
    : /iPad|Tablet/i.test(ua) ? 'tablet' : 'desktop'
  let browser = 'other'
  if (/Edg\//.test(ua)) browser = 'edge'
  else if (/Chrome\//.test(ua)) browser = 'chrome'
  else if (/Safari\//.test(ua)) browser = 'safari'
  else if (/Firefox\//.test(ua)) browser = 'firefox'
  let os = 'other'
  if (/Windows/.test(ua)) os = 'windows'
  else if (/Mac OS X/.test(ua)) os = 'macos'
  else if (/Android/.test(ua)) os = 'android'
  else if (/iPhone|iPad|iPod/.test(ua)) os = 'ios'
  else if (/Linux/.test(ua)) os = 'linux'
  return { device, browser, os }
}

async function hashIP(ip: string): Promise<string> {
  const salt = process.env.IP_HASH_SALT
  if (!salt) throw new Error('IP_HASH_SALT not set')
  const data = new TextEncoder().encode(ip + salt)
  const digest = await crypto.subtle.digest('SHA-256', data)
  return Array.from(new Uint8Array(digest))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('')
    .slice(0, 16)
}

// ----- handlers -----
export async function OPTIONS(req: Request) {
  return new Response(null, { headers: corsHeaders(req.headers.get('origin')) })
}

export async function POST(req: Request) {
  const origin = req.headers.get('origin')
  const cors = corsHeaders(origin)

  // Origin allowlist. No wildcard fallback. Misconfiguration fails closed.
  if (ALLOWED_ORIGINS.length === 0) {
    console.error('BASEEVENTS_ALLOWED_ORIGINS not set')
    return jsonError('not_configured', 500, cors)
  }
  if (!origin || !ALLOWED_ORIGINS.includes(origin)) {
    return jsonError('forbidden_origin', 403, cors)
  }

  // Honor DNT and GPC. Return 200 silently so the tracker does not retry.
  if (req.headers.get('dnt') === '1' || req.headers.get('sec-gpc') === '1') {
    return jsonResponse({ ok: true, filtered: 'dnt' }, 200, cors)
  }

  // Body size cap (cheap pre-read check).
  const lenHeader = req.headers.get('content-length')
  if (lenHeader && parseInt(lenHeader, 10) > MAX_BODY_BYTES) {
    return jsonError('body_too_large', 413, cors)
  }

  // Bot filter via UA.
  const ua = req.headers.get('user-agent') ?? ''
  if (BOT_PATTERNS.some(p => p.test(ua))) {
    return jsonResponse({ ok: true, filtered: 'bot' }, 200, cors)
  }

  // Parse body with hard size cap on actual bytes.
  let body: Record<string, unknown>
  try {
    const text = await req.text()
    if (text.length > MAX_BODY_BYTES) return jsonError('body_too_large', 413, cors)
    body = JSON.parse(text)
  } catch {
    return jsonError('invalid_json', 400, cors)
  }

  // Validate required fields and shapes.
  const site = trunc(body.site, 32)
  const event_name = trunc(body.event_name, MAX_EVENT_NAME)
  const visitor_id = trunc(body.visitor_id, MAX_VISITOR)
  if (!site || !event_name || !visitor_id) return jsonError('missing_fields', 400, cors)
  if (!SITE_REGEX.test(site)) return jsonError('invalid_site', 400, cors)
  if (!EVENT_NAME_REGEX.test(event_name)) return jsonError('invalid_event_name', 400, cors)

  // Cap props payload size.
  const props = (body.props && typeof body.props === 'object')
    ? body.props as Record<string, unknown>
    : {}
  try {
    if (JSON.stringify(props).length > MAX_PROPS_BYTES) {
      return jsonError('props_too_large', 400, cors)
    }
  } catch {
    return jsonError('invalid_props', 400, cors)
  }

  // Rate limit per visitor per minute. Uses events_site_visitor_idx.
  const minuteAgo = new Date(Date.now() - 60_000).toISOString()
  const { count: rateCount, error: rateErr } = await supabase
    .from('events')
    .select('id', { count: 'exact', head: true })
    .eq('site', site)
    .eq('visitor_id', visitor_id)
    .gte('created_at', minuteAgo)
  if (rateErr) {
    console.error('rate check failed:', rateErr)
    return jsonError('server_error', 500, cors)
  }
  if (rateCount !== null && rateCount >= RATE_LIMIT_PER_MINUTE) {
    return jsonError('rate_limited', 429, cors)
  }

  // Request-derived enrichment.
  const { device, browser, os } = parseUA(ua)
  const ip = (req.headers.get('x-forwarded-for') ?? '').split(',')[0].trim()
  const ip_hash = ip ? await hashIP(ip) : null
  const country = req.headers.get('x-vercel-ip-country') ?? null
  const region = req.headers.get('x-vercel-ip-country-region') ?? null
  const city = req.headers.get('x-vercel-ip-city') ?? null
  const timezone = trunc(body.timezone, MAX_TIMEZONE)
    ?? req.headers.get('x-vercel-ip-timezone')
    ?? null

  const row = {
    site,
    event_name,
    visitor_id,
    user_id: trunc(body.user_id, MAX_USER),
    session_id: trunc(body.session_id, MAX_SESSION),
    path: stripQuery(trunc(body.path, MAX_PATH)),
    referrer: originOnly(trunc(body.referrer, MAX_REFERRER)),
    utm_source: trunc(body.utm_source, MAX_UTM),
    utm_medium: trunc(body.utm_medium, MAX_UTM),
    utm_campaign: trunc(body.utm_campaign, MAX_UTM),
    utm_term: trunc(body.utm_term, MAX_UTM),
    utm_content: trunc(body.utm_content, MAX_UTM),
    country, region, city, timezone,
    device, browser, os, ip_hash,
    user_agent: ua.slice(0, MAX_USER_AGENT),
    props,
  }

  try {
    const { error: insertErr } = await supabase.from('events').insert(row)
    if (insertErr) {
      console.error('insert failed:', insertErr)
      return jsonError('insert_failed', 500, cors)
    }

    // Identify: upsert identity, backfill prior anonymous events within window.
    if (event_name === '$identify' && row.user_id) {
      const since = new Date(
        Date.now() - IDENTIFY_BACKFILL_WINDOW_MIN * 60 * 1000,
      ).toISOString()

      const { error: idErr } = await supabase.from('identities').upsert({
        site,
        visitor_id,
        user_id: row.user_id,
        user_props: props,
        last_seen_at: new Date().toISOString(),
      }, { onConflict: 'site,visitor_id,user_id' })
      if (idErr) console.error('identity upsert failed:', idErr)

      const { error: backfillErr } = await supabase
        .from('events')
        .update({ user_id: row.user_id })
        .eq('site', site)
        .eq('visitor_id', visitor_id)
        .is('user_id', null)
        .gte('created_at', since)
      if (backfillErr) console.error('backfill failed:', backfillErr)
    }

    return jsonResponse({ ok: true }, 200, cors)
  } catch (e) {
    console.error('ingest error:', e)
    return jsonError('server_error', 500, cors)
  }
}
```

Set the required environment variables in Vercel (Project Settings > Environment Variables):

- `SUPABASE_URL` (from Supabase project settings)
- `SUPABASE_SERVICE_ROLE_KEY` (Settings > API > `service_role` key, **server-side only, never expose to the browser**)
- `IP_HASH_SALT` (generate with `openssl rand -hex 32`)
- `BASEEVENTS_ALLOWED_ORIGINS` (set to `ALLOWED_ORIGIN` from step 0; comma-separated for multiple)

Optional:

- `BASEEVENTS_RATE_LIMIT_PER_MINUTE` (default `300`; per-visitor cap on events per minute)

Skip section 4b. Continue to section 5.

### 4b. Legacy/non-Vercel Supabase Edge Function

Use this path only when the app is **not** hosted on Vercel.

> **Geo warning:** A vanilla Supabase Edge Function does not have request geo headers. Without Cloudflare in front of it, `country`, `region`, and `city` will all be null in the database. This is a known limitation of the legacy path. If you need geo and you are not on Vercel, put Cloudflare in front of the function (Cloudflare adds `cf-ipcountry`).

```bash
mkdir -p supabase/functions/ingest
```

Write the following content to `supabase/functions/ingest/index.ts`:

```typescript
// Supabase Edge Function: ingest
// BaseEvents ingestion endpoint for non-Vercel hosts.

import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.45.0'

// ----- limits -----
const MAX_BODY_BYTES = 8 * 1024
const MAX_PROPS_BYTES = 4 * 1024
const MAX_EVENT_NAME = 100
const MAX_PATH = 2000
const MAX_REFERRER = 2000
const MAX_VISITOR = 64
const MAX_USER = 128
const MAX_SESSION = 64
const MAX_UTM = 200
const MAX_TIMEZONE = 64
const MAX_USER_AGENT = 500
const IDENTIFY_BACKFILL_WINDOW_MIN = 60

// Per-visitor rate limit. 300/min is generous enough for chatty apps
// while still catching abuse. Override with RATE_LIMIT_PER_MINUTE secret.
const RATE_LIMIT_PER_MINUTE = parseInt(
  Deno.env.get('RATE_LIMIT_PER_MINUTE') ?? '300',
  10,
)

const SITE_REGEX = /^[a-z0-9_-]{1,32}$/
const EVENT_NAME_REGEX = /^[$a-zA-Z0-9_.-]+$/

const ALLOWED_ORIGINS = (Deno.env.get('ALLOWED_ORIGINS') ?? '')
  .split(',')
  .map(s => s.trim())
  .filter(Boolean)

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
)

const BOT_PATTERNS = [
  /bot/i, /crawler/i, /spider/i, /scraper/i,
  /facebookexternalhit/i, /slackbot/i, /twitterbot/i, /discordbot/i,
  /whatsapp/i, /telegrambot/i, /linkedinbot/i, /preview/i,
  /headless/i, /phantomjs/i, /puppeteer/i, /playwright/i,
  /lighthouse/i, /pingdom/i, /uptimerobot/i, /gtmetrix/i,
  /ahrefs/i, /semrush/i, /mj12/i, /dotbot/i,
]

function corsHeaders(origin: string | null) {
  const allowed = origin && ALLOWED_ORIGINS.includes(origin) ? origin : ''
  return {
    'Access-Control-Allow-Origin': allowed,
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'content-type',
    'Vary': 'Origin',
  }
}
function jsonResponse(body: unknown, status: number, cors: Record<string, string>) {
  return new Response(JSON.stringify(body), {
    status, headers: { ...cors, 'content-type': 'application/json' },
  })
}
function jsonError(code: string, status: number, cors: Record<string, string>) {
  return jsonResponse({ error: code }, status, cors)
}
function trunc(s: unknown, n: number): string | null {
  if (typeof s !== 'string') return null
  return s.length > n ? s.slice(0, n) : s
}
function stripQuery(url: string | null): string | null {
  if (!url) return null
  const cuts = [url.indexOf('?'), url.indexOf('#')].filter(n => n >= 0)
  return cuts.length ? url.slice(0, Math.min(...cuts)) : url
}
function originOnly(url: string | null): string | null {
  if (!url) return null
  try { return new URL(url).origin } catch { return null }
}
function parseUA(ua: string) {
  const device = /Mobi|Android|iPhone/i.test(ua) && !/iPad|Tablet/i.test(ua)
    ? 'mobile'
    : /iPad|Tablet/i.test(ua) ? 'tablet' : 'desktop'
  let browser = 'other'
  if (/Edg\//.test(ua)) browser = 'edge'
  else if (/Chrome\//.test(ua)) browser = 'chrome'
  else if (/Safari\//.test(ua)) browser = 'safari'
  else if (/Firefox\//.test(ua)) browser = 'firefox'
  let os = 'other'
  if (/Windows/.test(ua)) os = 'windows'
  else if (/Mac OS X/.test(ua)) os = 'macos'
  else if (/Android/.test(ua)) os = 'android'
  else if (/iPhone|iPad|iPod/.test(ua)) os = 'ios'
  else if (/Linux/.test(ua)) os = 'linux'
  return { device, browser, os }
}
async function hashIP(ip: string): Promise<string> {
  const salt = Deno.env.get('IP_HASH_SALT')
  if (!salt) throw new Error('IP_HASH_SALT not set')
  const data = new TextEncoder().encode(ip + salt)
  const digest = await crypto.subtle.digest('SHA-256', data)
  return Array.from(new Uint8Array(digest))
    .map(b => b.toString(16).padStart(2, '0')).join('').slice(0, 16)
}

Deno.serve(async (req) => {
  const origin = req.headers.get('origin')
  const cors = corsHeaders(origin)

  if (req.method === 'OPTIONS') return new Response(null, { headers: cors })
  if (req.method !== 'POST') return jsonError('method_not_allowed', 405, cors)

  if (ALLOWED_ORIGINS.length === 0) {
    console.error('ALLOWED_ORIGINS not set')
    return jsonError('not_configured', 500, cors)
  }
  if (!origin || !ALLOWED_ORIGINS.includes(origin)) {
    return jsonError('forbidden_origin', 403, cors)
  }

  if (req.headers.get('dnt') === '1' || req.headers.get('sec-gpc') === '1') {
    return jsonResponse({ ok: true, filtered: 'dnt' }, 200, cors)
  }

  const lenHeader = req.headers.get('content-length')
  if (lenHeader && parseInt(lenHeader, 10) > MAX_BODY_BYTES) {
    return jsonError('body_too_large', 413, cors)
  }

  const ua = req.headers.get('user-agent') ?? ''
  if (BOT_PATTERNS.some(p => p.test(ua))) {
    return jsonResponse({ ok: true, filtered: 'bot' }, 200, cors)
  }

  let body: Record<string, unknown>
  try {
    const text = await req.text()
    if (text.length > MAX_BODY_BYTES) return jsonError('body_too_large', 413, cors)
    body = JSON.parse(text)
  } catch {
    return jsonError('invalid_json', 400, cors)
  }

  const site = trunc(body.site, 32)
  const event_name = trunc(body.event_name, MAX_EVENT_NAME)
  const visitor_id = trunc(body.visitor_id, MAX_VISITOR)
  if (!site || !event_name || !visitor_id) return jsonError('missing_fields', 400, cors)
  if (!SITE_REGEX.test(site)) return jsonError('invalid_site', 400, cors)
  if (!EVENT_NAME_REGEX.test(event_name)) return jsonError('invalid_event_name', 400, cors)

  const props = (body.props && typeof body.props === 'object')
    ? body.props as Record<string, unknown>
    : {}
  try {
    if (JSON.stringify(props).length > MAX_PROPS_BYTES) {
      return jsonError('props_too_large', 400, cors)
    }
  } catch {
    return jsonError('invalid_props', 400, cors)
  }

  const minuteAgo = new Date(Date.now() - 60_000).toISOString()
  const { count: rateCount, error: rateErr } = await supabase
    .from('events').select('id', { count: 'exact', head: true })
    .eq('site', site).eq('visitor_id', visitor_id)
    .gte('created_at', minuteAgo)
  if (rateErr) {
    console.error('rate check failed:', rateErr)
    return jsonError('server_error', 500, cors)
  }
  if (rateCount !== null && rateCount >= RATE_LIMIT_PER_MINUTE) {
    return jsonError('rate_limited', 429, cors)
  }

  const { device, browser, os } = parseUA(ua)
  const ip = (req.headers.get('x-forwarded-for') ?? '').split(',')[0].trim()
  const ip_hash = ip ? await hashIP(ip) : null

  // No native geo on this path. cf-ipcountry is read only if Cloudflare is in front.
  const country = req.headers.get('cf-ipcountry') ?? null

  const row = {
    site, event_name, visitor_id,
    user_id: trunc(body.user_id, MAX_USER),
    session_id: trunc(body.session_id, MAX_SESSION),
    path: stripQuery(trunc(body.path, MAX_PATH)),
    referrer: originOnly(trunc(body.referrer, MAX_REFERRER)),
    utm_source: trunc(body.utm_source, MAX_UTM),
    utm_medium: trunc(body.utm_medium, MAX_UTM),
    utm_campaign: trunc(body.utm_campaign, MAX_UTM),
    utm_term: trunc(body.utm_term, MAX_UTM),
    utm_content: trunc(body.utm_content, MAX_UTM),
    country,
    region: null,
    city: null,
    timezone: trunc(body.timezone, MAX_TIMEZONE),
    device, browser, os, ip_hash,
    user_agent: ua.slice(0, MAX_USER_AGENT),
    props,
  }

  try {
    const { error: insertErr } = await supabase.from('events').insert(row)
    if (insertErr) {
      console.error('insert failed:', insertErr)
      return jsonError('insert_failed', 500, cors)
    }

    if (event_name === '$identify' && row.user_id) {
      const since = new Date(
        Date.now() - IDENTIFY_BACKFILL_WINDOW_MIN * 60 * 1000,
      ).toISOString()
      const { error: idErr } = await supabase.from('identities').upsert({
        site, visitor_id,
        user_id: row.user_id,
        user_props: props,
        last_seen_at: new Date().toISOString(),
      }, { onConflict: 'site,visitor_id,user_id' })
      if (idErr) console.error('identity upsert failed:', idErr)

      const { error: backfillErr } = await supabase.from('events')
        .update({ user_id: row.user_id })
        .eq('site', site).eq('visitor_id', visitor_id).is('user_id', null)
        .gte('created_at', since)
      if (backfillErr) console.error('backfill failed:', backfillErr)
    }

    return jsonResponse({ ok: true }, 200, cors)
  } catch (e) {
    console.error('ingest error:', e)
    return jsonError('server_error', 500, cors)
  }
})
```

Set the required secrets:

```bash
supabase secrets set IP_HASH_SALT="$(openssl rand -hex 32)"
supabase secrets set ALLOWED_ORIGINS="ALLOWED_ORIGIN"
```

Replace `ALLOWED_ORIGIN` with the value from step 0. Multiple origins are comma-separated.

Optional: tune the per-visitor rate limit (default 300/min):

```bash
supabase secrets set RATE_LIMIT_PER_MINUTE=600
```

Deploy the function. `--no-verify-jwt` is required because the tracker runs in anonymous visitors' browsers and cannot send an auth header. The origin allowlist and rate limit are what protect the endpoint, not JWT:

```bash
supabase functions deploy ingest --no-verify-jwt
```

Capture the deployed endpoint URL:

```
https://<SUPABASE_PROJECT_REF>.supabase.co/functions/v1/ingest
```

Print this URL. The user will need it in the next step and you will use it to add the script tag.

---

## 5. ADD THE TRACKER SCRIPT

### 5a. Detect the framework

Check which framework this project uses. Run these in order and stop at the first match:

```bash
# Next.js App Router
test -f next.config.js -o -f next.config.mjs -o -f next.config.ts && test -d app && echo "NEXT_APP"

# Next.js Pages Router
test -f next.config.js -o -f next.config.mjs -o -f next.config.ts && test -d pages && echo "NEXT_PAGES"

# Vite (React, Vue, Svelte, etc.)
test -f vite.config.js -o -f vite.config.ts && echo "VITE"

# SvelteKit
test -f svelte.config.js && test -f src/app.html && echo "SVELTEKIT"

# Astro
test -f astro.config.mjs -o -f astro.config.ts && echo "ASTRO"

# Remix
test -f remix.config.js && echo "REMIX"

# Plain HTML
test -f index.html && echo "PLAIN_HTML"
```

If none match, ask the user what framework they use and which file is the root HTML template or layout.

### 5b. Place the tracker file

Determine the public assets directory based on the framework:

- `NEXT_APP` or `NEXT_PAGES`: `public/`
- `VITE`: `public/`
- `SVELTEKIT`: `static/`
- `ASTRO`: `public/`
- `REMIX`: `public/`
- `PLAIN_HTML`: same directory as `index.html`

The default tracker filename is `t.js`. Some adblock lists block scripts named `t.js` directly. If you want to maximize coverage, use a less obvious name (for example `q.js` or `m.js`) and update the script tag in 5c accordingly. The file content is identical.

Create the directory if it does not exist, then write the following content to `<public_dir>/t.js`:

```javascript
/*! BaseEvents tracker */
(() => {
  const script = document.currentScript || document.querySelector('script[data-site]')
  if (!script) return

  const SITE = script.getAttribute('data-site')
  const ENDPOINT = script.getAttribute('data-endpoint')
  if (!SITE || !ENDPOINT) {
    console.warn('[baseevents] missing data-site or data-endpoint')
    return
  }

  // Honor Do Not Track and Global Privacy Control. Skip the request entirely.
  const dnt = navigator.doNotTrack === '1'
    || window.doNotTrack === '1'
    || navigator.globalPrivacyControl === true
  if (dnt) return

  const SESSION_TIMEOUT_MS = 30 * 60 * 1000
  const K = {
    visitor: '_be_vid',
    user: '_be_uid',
    session: '_be_sid',
    sessionExp: '_be_sxp',
    utm: '_be_utm',
  }

  const lsGet = (k) => { try { return localStorage.getItem(k) } catch { return null } }
  const lsSet = (k, v) => { try { localStorage.setItem(k, v) } catch {} }
  const lsDel = (k) => { try { localStorage.removeItem(k) } catch {} }
  const ssGet = (k) => { try { return sessionStorage.getItem(k) } catch { return null } }
  const ssSet = (k, v) => { try { sessionStorage.setItem(k, v) } catch {} }
  const ssDel = (k) => { try { sessionStorage.removeItem(k) } catch {} }

  const getVisitorId = () => {
    let id = lsGet(K.visitor)
    if (!id) { id = 'v_' + crypto.randomUUID(); lsSet(K.visitor, id) }
    return id
  }
  const getSessionId = () => {
    const now = Date.now()
    const exp = parseInt(ssGet(K.sessionExp) || '0', 10)
    let id = ssGet(K.session)
    if (!id || now > exp) { id = 's_' + crypto.randomUUID(); ssSet(K.session, id) }
    ssSet(K.sessionExp, String(now + SESSION_TIMEOUT_MS))
    return id
  }
  const getUTMs = () => {
    const params = new URLSearchParams(location.search)
    const fresh = {}
    let hasNew = false
    for (const k of ['source', 'medium', 'campaign', 'term', 'content']) {
      const v = params.get('utm_' + k)
      if (v) { fresh['utm_' + k] = v; hasNew = true }
    }
    if (hasNew) { ssSet(K.utm, JSON.stringify(fresh)); return fresh }
    try { return JSON.parse(ssGet(K.utm) || '{}') } catch { return {} }
  }
  const getTimezone = () => {
    try { return Intl.DateTimeFormat().resolvedOptions().timeZone || null } catch { return null }
  }
  const referrerOrigin = () => {
    if (!document.referrer) return null
    try { return new URL(document.referrer).origin } catch { return null }
  }

  const send = (payload) => {
    const body = JSON.stringify(payload)
    if (navigator.sendBeacon) {
      try {
        if (navigator.sendBeacon(ENDPOINT, new Blob([body], { type: 'application/json' }))) return
      } catch {}
    }
    try {
      fetch(ENDPOINT, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body,
        keepalive: true,
        mode: 'cors',
      }).catch(() => {})
    } catch {}
  }

  const buildPayload = (eventName, props) => ({
    site: SITE,
    event_name: eventName,
    visitor_id: getVisitorId(),
    user_id: lsGet(K.user) || null,
    session_id: getSessionId(),
    path: location.pathname,
    referrer: referrerOrigin(),
    timezone: getTimezone(),
    props: props || {},
    ...getUTMs(),
  })

  const track = (eventName, props) => {
    if (eventName) send(buildPayload(eventName, props))
  }
  const identify = (userId, userProps) => {
    if (!userId) return
    lsSet(K.user, String(userId))
    send(buildPayload('$identify', userProps || {}))
  }
  const reset = () => {
    lsDel(K.user); lsDel(K.visitor)
    ssDel(K.session); ssDel(K.sessionExp); ssDel(K.utm)
  }

  let lastPath = ''
  const pageview = () => {
    const p = location.pathname
    if (p === lastPath) return
    lastPath = p
    track('$pageview')
  }
  const origPush = history.pushState
  history.pushState = function () { origPush.apply(this, arguments); pageview() }
  const origReplace = history.replaceState
  history.replaceState = function () { origReplace.apply(this, arguments); pageview() }
  window.addEventListener('popstate', pageview)

  window.track = track
  window.identify = identify
  window.trackReset = reset

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', pageview)
  } else {
    pageview()
  }
})()
```

### 5c. Add the script tag

Use these exact values:
- `SITE_SLUG` = the value the user gave you in step 0
- `ENDPOINT_URL` = `/api/analytics` for Vercel-hosted Next.js App Router projects
- `ENDPOINT_URL` = `https://<SUPABASE_PROJECT_REF>.supabase.co/functions/v1/ingest` for legacy/non-Vercel direct-to-Supabase ingestion

If the host site uses a Content Security Policy, ensure the script source and the endpoint origin are in `script-src` and `connect-src` respectively. For Vercel-hosted same-origin setups this is usually a non-issue.

Optional hardening: add a Subresource Integrity (SRI) hash to the script tag so a tampered tracker fails to load:

```bash
# Compute the SRI hash for t.js
openssl dgst -sha384 -binary public/t.js | openssl base64 -A
```

Then prefix the result with `sha384-` and use it as the `integrity` attribute. Example: `integrity="sha384-XXXXX..." crossorigin="anonymous"`. Skip this for now if you expect to iterate on `t.js`; rebuild the hash whenever the file changes.

**For `NEXT_APP` (Next.js App Router):**

Edit `app/layout.tsx` (or `.jsx`). Import `Script` from `next/script` at the top, then add the component inside the `<body>` tag:

```tsx
import Script from 'next/script'

// ... inside the layout component's return:
<body>
  {children}
  <Script
    src="/t.js"
    strategy="afterInteractive"
    data-site="SITE_SLUG"
    data-endpoint="ENDPOINT_URL"
  />
</body>
```

**For `NEXT_PAGES` (Next.js Pages Router):**

Edit `pages/_app.tsx` (or `.jsx`). Wrap the existing return with a fragment and add `<Script>`:

```tsx
import Script from 'next/script'

export default function MyApp({ Component, pageProps }) {
  return (
    <>
      <Component {...pageProps} />
      <Script
        src="/t.js"
        strategy="afterInteractive"
        data-site="SITE_SLUG"
        data-endpoint="ENDPOINT_URL"
      />
    </>
  )
}
```

**For `VITE` (React, Vue, Svelte, or plain):**

Edit `index.html` at the project root. Add the script tag just before the closing `</body>` tag:

```html
<script async src="/t.js" data-site="SITE_SLUG" data-endpoint="ENDPOINT_URL"></script>
```

**For `SVELTEKIT`:**

Edit `src/app.html`. Add the script tag just before `%sveltekit.body%` at the end:

```html
<script async src="/t.js" data-site="SITE_SLUG" data-endpoint="ENDPOINT_URL"></script>
```

**For `ASTRO`:**

Edit the base layout (usually `src/layouts/Layout.astro` or whatever is used by all pages). Add the script tag before the closing `</body>`:

```html
<script is:inline async src="/t.js" data-site="SITE_SLUG" data-endpoint="ENDPOINT_URL"></script>
```

**For `REMIX`:**

Edit `app/root.tsx`. Add the script tag before the closing `</body>` in the Document component:

```tsx
<script async src="/t.js" data-site="SITE_SLUG" data-endpoint="ENDPOINT_URL"></script>
```

**For `PLAIN_HTML`:**

Edit `index.html` (and any other top-level HTML files). Add the script tag just before `</body>`:

```html
<script async src="/t.js" data-site="SITE_SLUG" data-endpoint="ENDPOINT_URL"></script>
```

Replace both `SITE_SLUG` and `ENDPOINT_URL` with the actual values. Do not leave the placeholders in the code.

---

## 6. VERIFY

Run a test request against the deployed endpoint. Use the actual `ALLOWED_ORIGIN` value from step 0 (the request will be rejected without a valid Origin header).

For Vercel-hosted Next.js projects, test the local route with fake Vercel geo headers and a valid origin:

```bash
curl -X POST "http://localhost:3000/api/analytics" \
  -H "Content-Type: application/json" \
  -H "Origin: ALLOWED_ORIGIN" \
  -H "x-vercel-ip-country: AU" \
  -H "x-vercel-ip-country-region: NSW" \
  -H "x-vercel-ip-city: Sydney" \
  -H "x-vercel-ip-timezone: Australia/Sydney" \
  -d '{"site":"SITE_SLUG","event_name":"setup_test","visitor_id":"v_setup_test"}'
```

For legacy/non-Vercel Supabase Edge Function ingestion:

```bash
curl -X POST "https://<SUPABASE_PROJECT_REF>.supabase.co/functions/v1/ingest" \
  -H "Content-Type: application/json" \
  -H "Origin: ALLOWED_ORIGIN" \
  -d '{"site":"SITE_SLUG","event_name":"setup_test","visitor_id":"v_setup_test"}'
```

Expected response: `{"ok":true}`.

Common error responses and what they mean:

- `{"error":"forbidden_origin"}` (403): the `Origin` header doesn't match `BASEEVENTS_ALLOWED_ORIGINS` / `ALLOWED_ORIGINS`. Check the env var.
- `{"error":"not_configured"}` (500): the allowlist env var is missing entirely.
- `{"error":"invalid_site"}` (400): site slug doesn't match `^[a-z0-9_-]{1,32}$`.
- `{"error":"rate_limited"}` (429): more than 300 events for that visitor in the last minute (default; configurable via `BASEEVENTS_RATE_LIMIT_PER_MINUTE`). Wait or use a different `visitor_id` for the test.

If you see `{"ok":true}`, verify the row landed in the database. Tell the user to run this in the Supabase SQL editor:

```sql
select * from public.events where visitor_id = 'v_setup_test';
```

They should see one row with `event_name = 'setup_test'` and `site = <their slug>`.

If they do, delete the test row and confirm setup is complete:

```sql
delete from public.events where visitor_id = 'v_setup_test';
```

If the curl returns a 5xx error, check:
- For Vercel route ingestion: `SUPABASE_SERVICE_ROLE_KEY`, `IP_HASH_SALT`, and `BASEEVENTS_ALLOWED_ORIGINS` are all set in the Vercel project.
- For Edge Function: `supabase secrets list` shows `IP_HASH_SALT` and `ALLOWED_ORIGINS`. The function deployed (rerun `supabase functions deploy ingest --no-verify-jwt`).
- The schema migration ran (check `supabase/migrations/` for the file and re-run `supabase db push`).

---

## 7. TELL THE USER IT IS DONE

Print a summary like this (fill in real values):

```
BaseEvents pipeline live.

Site slug:       SITE_SLUG
Allowed origin:  ALLOWED_ORIGIN
Endpoint:        /api/analytics
Tracker:         /t.js (hosted from this app)
Location:        Vercel request headers provide country, region, city, and timezone on deployed Vercel apps.

Privacy posture:
  - Raw IP is hashed with a server-side salt before insert.
  - Path query strings are stripped before storage.
  - Referrer is reduced to origin only before storage.
  - DNT and Sec-GPC signals are honored. Affected visitors are not recorded.

Pageviews are tracked automatically, including SPA route changes.

For custom events, call these from anywhere in your app code:

  window.track('signup_completed', { plan: 'pro' })
  window.identify('user_abc123', { email: 'a@b.com' })
  window.trackReset()   // on logout

Note: anything you pass into props is stored as-is. Do not put PII in
props unless you mean to.

Events live in the `events` table in this project's Supabase database.
Starter queries are in BASEEVENTS-QUERIES.md (see below).

To query your data, ask your agent. Examples:
  "What were my top pages last week?"
  "How many unique visitors did I get yesterday?"
  "Where did signups drop off in the funnel?"

The agent reads the events table and writes the SQL for you.

Next time you want to ship BaseEvents to another prototype: give your
agent this same SETUP-BASEEVENTS.md file and a new site slug.
```

---

## 8. CREATE A QUERY REFERENCE FILE FOR THE USER

Write the following to `BASEEVENTS-QUERIES.md` in the project root, so the user has a reference they can paste into the Supabase SQL editor.

````markdown
# BaseEvents queries

Paste any of these into the Supabase SQL editor
(`https://supabase.com/dashboard/project/<ref>/sql`).

## Daily traffic (last 30 days)
```sql
select
  date_trunc('day', created_at) as day,
  count(*) filter (where event_name = '$pageview') as pageviews,
  count(distinct visitor_id) as unique_visitors,
  count(distinct session_id) as sessions
from public.events
where created_at > now() - interval '30 days'
group by day
order by day desc;
```

## Top pages (last 7 days)
```sql
select path, count(*) as views, count(distinct visitor_id) as uniques
from public.events
where event_name = '$pageview' and created_at > now() - interval '7 days'
group by path
order by views desc
limit 50;
```

## Traffic sources
```sql
select
  coalesce(utm_source, '(direct)') as source,
  coalesce(utm_medium, '(none)') as medium,
  coalesce(utm_campaign, '(none)') as campaign,
  count(distinct visitor_id) as visitors
from public.events
where created_at > now() - interval '30 days'
group by source, medium, campaign
order by visitors desc;
```

## Funnel: pageview to signup
Replace `signup_completed` with whatever event you fire on conversion.
```sql
with f as (
  select visitor_id,
         bool_or(event_name = '$pageview') as viewed,
         bool_or(event_name = 'signup_completed') as signed_up
  from public.events
  where created_at > now() - interval '30 days'
  group by visitor_id
)
select count(*) filter (where viewed) as visited,
       count(*) filter (where signed_up) as converted,
       round(100.0 * count(*) filter (where signed_up) /
             nullif(count(*) filter (where viewed), 0), 2) as pct
from f;
```

## Full journey for a specific user
Works pre- and post-identify because $identify backfills prior events
within the configured window (default 60 minutes).
```sql
select created_at, event_name, path, props
from public.events
where user_id = 'user_abc123'
order by created_at;
```

## Country breakdown
On Vercel, country comes from request headers automatically. On the
legacy Edge Function path, country is null unless Cloudflare is in
front of the function.
```sql
select coalesce(country, 'unknown') as country,
       count(distinct visitor_id) as visitors
from public.events
where created_at > now() - interval '7 days'
group by country
order by visitors desc;
```

## Retention: cleaning up old events
The events table grows forever by default. For prototypes this is fine
for months. When you want to trim, this deletes events older than 90 days:
```sql
delete from public.events where created_at < now() - interval '90 days';
```
For higher-volume installs, partition by `created_at` monthly so you can
drop entire partitions instead of issuing deletes.
````

---

## 9. PRODUCTION CONSIDERATIONS

These do not block the initial install but are worth flagging to the user before they ship to traffic:

**Subdomains.** `localStorage` is per-origin. If the site spans `app.example.com` and `www.example.com`, the same human gets a different `visitor_id` on each. To unify, either consolidate to a single domain, set up a shared cookie at `.example.com` and adapt the tracker, or accept the split.

**Adblockers.** Default tracker filename `/t.js` is on some block lists. If meaningful traffic is technical users, rename to a less obvious path (`/q.js`, `/m.js`, anything site-specific). The script content does not change.

**Retention.** The `events` table grows forever. Add a periodic cleanup (Supabase scheduled function or pg_cron) once volume warrants. Sample query is in `BASEEVENTS-QUERIES.md`.

**Partitioning.** Past a few million rows, partition `events` by `created_at` (monthly). Cleanup becomes a `drop partition` instead of a `delete` and queries get faster.

**GDPR consent.** If you serve EU traffic and your legal posture requires consent before tracking, gate the script tag behind a consent banner. The tracker honors DNT and GPC out of the box but does not implement IAB TCF or any specific consent flow.

**PII in props.** Anything passed to `track(name, props)` or `identify(userId, props)` is stored as-is in JSONB. The endpoint enforces a 4KB cap and rejects anything larger. It does not inspect contents. If you put `{email, phone, dob}` in props, those fields land in the database in plaintext. That is the user's call.

**Rate limiting and ingestion latency.** Default cap is 300 events per visitor per minute. This is generous enough for most apps including chatty interactive ones, but each event currently costs one extra Supabase count query for the rate check, doubling ingestion latency and DB query budget per event. If your app fires events at high rates, or you're optimizing free-tier query budget, replace the per-event count check with an in-memory token bucket using `@vercel/kv`, Upstash Redis, or similar. The current check is correct and cheap to reason about; optimize when the volume justifies it. Tune via `BASEEVENTS_RATE_LIMIT_PER_MINUTE` (Vercel) or `RATE_LIMIT_PER_MINUTE` secret (Edge).

**IP hash rotation.** The IP hash uses a fixed salt, so the same IP produces the same hash forever. This is intentional for deduplication but means the hash is a stable pseudo-identifier. For stronger anonymization, rotate `IP_HASH_SALT` daily; the tradeoff is losing IP-based stitching across the rotation boundary.

**Service role key blast radius.** The route handler and Edge Function use `SUPABASE_SERVICE_ROLE_KEY` which bypasses RLS. The hardening above (origin allowlist, rate limit, input validation, sanitized errors) is what keeps this safe. If you fork this code, do not relax any of those checks without a replacement.

---

## CHECKLIST FOR THE AGENT

Before declaring done, confirm all of these:

- [ ] Got `SITE_SLUG` and `ALLOWED_ORIGIN` from the user
- [ ] Supabase CLI installed and authenticated
- [ ] Project initialized and linked (`supabase/config.toml` exists, linked to a ref)
- [ ] Migration file created in `supabase/migrations/` with BaseEvents schema and CHECK constraints
- [ ] `supabase db push` ran successfully (or user confirmed they pasted SQL into dashboard)
- [ ] For Vercel path: `app/api/analytics/route.ts` exists with correct content
- [ ] For Vercel path: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `IP_HASH_SALT`, `BASEEVENTS_ALLOWED_ORIGINS` set in Vercel env
- [ ] For Edge path: `supabase/functions/ingest/index.ts` exists with correct content
- [ ] For Edge path: `IP_HASH_SALT` and `ALLOWED_ORIGINS` secrets set
- [ ] For Edge path: `supabase functions deploy ingest --no-verify-jwt` ran successfully
- [ ] `t.js` placed in the correct public directory (or the renamed equivalent)
- [ ] Script tag added to the correct layout/HTML file with real `SITE_SLUG` and real endpoint URL
- [ ] curl test with valid Origin returned `{"ok":true}`
- [ ] `BASEEVENTS-QUERIES.md` written to project root
- [ ] Printed summary to user, including the production considerations from section 9

## CONSTRAINTS FOR THE AGENT

- Do not invent file paths. Detect the framework before placing files.
- Do not hardcode the Supabase project ref in the tracker. It comes from the endpoint URL in the script tag only.
- Do not commit `IP_HASH_SALT` or `SUPABASE_SERVICE_ROLE_KEY` anywhere. They live only in environment variables (Vercel) or Supabase secrets (Edge Function).
- Do not relax the `ALLOWED_ORIGINS` check or default it to `*`. The endpoint must fail closed when misconfigured.
- Do not echo internal error messages to the response body. Use the documented error codes only.
- Do not skip the input validation regexes (`SITE_REGEX`, `EVENT_NAME_REGEX`) or the size caps. They are load-bearing.
- Do not add read policies to RLS. Leave that decision to the user when they build a dashboard.
- Do not install optional dependencies beyond `@supabase/supabase-js`. This setup adds zero other npm packages.
- Do not skip the verification curl. If it fails, debug before declaring done.
