# Niche City Bot

Automated TikTok draft posting system for [@dannybalentine.mp3](https://www.tiktok.com/@dannybalentine.mp3) — an indie pop/folk artist from the PNW.

Posts city-targeted slideshows using Mt. Hood photography + the *"to the person in [City]..."* format that has driven 60k+ views and 1000+ comments. Iterates on what works using real performance data.

---

## What It Does

1. Searches Unsplash for Mt. Hood / PNW mountain photos, scores them with Claude Vision against a reference "best performer," and adds only high-scoring photos to Supabase Storage
2. Maintains a curated pool of Mt. Hood photos in Supabase
2. Uses Claude AI to generate city-specific captions and hashtag sets
3. Renders text overlays onto photos (2–3 slides per post)
4. Stitches slides into a video (TikTok requires video format)
5. Uploads to TikTok as a **draft** via Content Posting API
6. Pulls 48hr performance stats back from TikTok Analytics API
7. Stores everything in Supabase and learns which cities, captions, and posting times perform best
8. Weights future posts toward winners automatically

---

## Post Format

**Slide 1**
> to the person in Tacoma, WA who listened to Heavyweight 33 times yesterday...

**Slide 2**
> I hope you're okay 🏔️

**Slide 3**
> Danny Balentine | Heavyweight on Spotify

**Caption (generated per city)**
> tacoma we see you 🌲 drop where you're listening from 👇 #HeavyweightSong #TacomaWA #IndieFolk #PNWMusic #DannyBalentine

---

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| App framework | Next.js | Web UI + API routes |
| Hosting | Vercel | Deployment + cron scheduling |
| Database | Supabase Postgres | Post history, city configs, analytics |
| Photo storage | Supabase Storage | Mt. Hood photo pool |
| Photo sourcing | Unsplash API | Search and pull Mt. Hood / PNW mountain photos |
| Photo scoring | Claude Vision API | Score new photos against reference before adding to pool |
| AI captions | Claude API (`claude-sonnet-4-20250514`) | City-specific caption + hashtag generation |
| Image rendering | Sharp | Text overlay onto photos |
| Video stitching | ffmpeg | Combine slides into TikTok-compatible video |
| TikTok posting | TikTok Content Posting API | Upload video to drafts |
| TikTok analytics | TikTok Analytics API | Pull performance data per post |
| Scheduling | Vercel Cron Jobs | Trigger posts on timezone-aware schedule |

---

## Architecture

```
Unsplash API (search Mt. Hood / PNW photos)
        │
        ▼
Claude Vision → score vs. reference photo
  └── above threshold → Supabase Storage (photos)
        │
        ▼
Post Generator (Next.js API route)
  ├── Claude API → city caption + hashtags
  ├── Sharp → text overlay on each slide
  └── ffmpeg → stitch slides into .mp4
        │
        ▼
TikTok Content Posting API
  └── drops to drafts (you review + publish)
        │
        ▼
Vercel Cron (48hrs later)
  └── TikTok Analytics API → pull stats
        │
        ▼
Supabase DB (store results)
        │
        ▼
Claude API → analyze performance
  └── update city weights for next batch
```

---

## Learning Loop

Every post is logged in Supabase with:
- City targeted
- Caption variant used
- Posting time
- Views, likes, comments, shares, watch time (pulled at 48hr)

Claude analyzes the accumulated data and surfaces:
- Which cities have Danny's audience
- Which caption styles (emotional hook vs. curiosity vs. call-to-action) drive comments
- Which posting times by timezone perform best
- Which Mt. Hood photo styles get more watch time

This feeds directly into the next batch — cities that perform get more posts, underperformers get rotated out.

---

## Pre-Release Strategy (Hiker)

The city targeting system is designed to warm up markets **before** a release:

1. Run Heavyweight posts across 15–20 cities for 4–6 weeks
2. Identify top 5–8 cities by engagement rate
3. At Hiker release: surge posts in proven markets + add release-specific caption variants
4. Use Analytics API data to inform Spotify pitching and playlist targeting

---

## Project Structure

```
heavyweight-bot/
├── app/
│   ├── page.tsx                  # Dashboard UI
│   ├── api/
│   │   ├── generate/route.ts     # Generate + queue a post
│   │   ├── photos/route.ts       # Unsplash search + Claude Vision scoring + Supabase ingest
│   │   ├── upload/route.ts       # Upload draft to TikTok
│   │   ├── analytics/route.ts    # Pull + store TikTok stats
│   │   └── insights/route.ts     # Claude analysis of performance data
│   └── components/
│       ├── CityManager.tsx       # Add/remove/weight cities
│       ├── PostQueue.tsx         # View queued drafts
│       ├── PhotoUpload.tsx       # Upload photos to Supabase
│       └── PerformanceDashboard.tsx
├── lib/
│   ├── claude.ts                 # Caption, insight, and photo scoring
│   ├── unsplash.ts               # Unsplash search + photo ingestion
│   ├── tiktok.ts                 # TikTok API client
│   ├── supabase.ts               # DB + storage client
│   ├── render.ts                 # Sharp image rendering
│   └── video.ts                  # ffmpeg slide stitching
├── cron/
│   ├── post.ts                   # Scheduled posting trigger
│   └── analytics.ts              # Scheduled analytics pull
├── config/
│   └── cities.ts                 # Default city list + hashtag sets
└── supabase/
    └── schema.sql                # DB schema
```

---

## Database Schema

```sql
-- Photos stored in Supabase Storage, referenced here
create table photos (
  id uuid primary key default gen_random_uuid(),
  storage_path text not null,
  uploaded_at timestamp default now()
);

-- City targeting config
create table cities (
  id uuid primary key default gen_random_uuid(),
  name text not null,         -- "Tacoma"
  state text not null,        -- "WA"
  timezone text not null,     -- "America/Los_Angeles"
  hashtags text[],            -- ["#TacomaWA", "#PNWMusic"]
  weight float default 1.0,   -- boosted by performance
  active boolean default true
);

-- Every post generated
create table posts (
  id uuid primary key default gen_random_uuid(),
  city_id uuid references cities(id),
  caption text,
  hashtags text[],
  video_path text,
  tiktok_post_id text,
  status text default 'draft', -- draft | published | failed
  created_at timestamp default now(),
  published_at timestamp
);

-- Performance data pulled from TikTok
create table analytics (
  id uuid primary key default gen_random_uuid(),
  post_id uuid references posts(id),
  views integer,
  likes integer,
  comments integer,
  shares integer,
  watch_time_avg float,
  pulled_at timestamp default now()
);
```

---

## Setup

### 1. Clone & Install

```bash
git clone https://github.com/yourusername/heavyweight-bot
cd heavyweight-bot
npm install
```

### 2. Environment Variables

Create a `.env.local` file:

```env
# Claude
ANTHROPIC_API_KEY=your_key_here

# Unsplash
UNSPLASH_ACCESS_KEY=your_access_key

# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_key

# TikTok
TIKTOK_CLIENT_KEY=your_client_key
TIKTOK_CLIENT_SECRET=your_client_secret
TIKTOK_ACCESS_TOKEN=your_access_token

```

### 3. TikTok API Setup

1. Go to [developers.tiktok.com](https://developers.tiktok.com)
2. Create an app and apply for **Content Posting API** + **Research API** access
3. Complete the OAuth flow to connect your @dannybalentine.mp3 account
4. Copy `client_key` and `client_secret` into `.env.local`

> **Note:** API approval takes 1–3 days. The app runs in dry-run mode until credentials are active — all posts preview locally without uploading.

### 4. Supabase Setup

```bash
# Run schema
npx supabase db push supabase/schema.sql
```

### 5. Deploy to Vercel

```bash
vercel deploy
```

Add all env variables in Vercel dashboard under **Settings → Environment Variables**.

---

## Usage

1. **Upload photos** — go to `/upload`, drag in Mt. Hood photos from your iPhone
2. **Manage cities** — go to `/cities`, add target cities with their hashtag sets
3. **Generate a batch** — hit "Generate Posts" to queue city variants
4. **Review drafts** — check your TikTok app drafts, hit publish on the ones you like
5. **Watch analytics roll in** — the cron job pulls stats at 48hr automatically
6. **Check insights** — `/insights` shows Claude's analysis of what's working

---

## Target Cities (Starter List)

Medium-sized cities with strong indie/folk streaming bases:

`Tacoma WA` · `Boise ID` · `Spokane WA` · `Eugene OR` · `Bellingham WA` · `Missoula MT` · `Fort Collins CO` · `Flagstaff AZ` · `Asheville NC` · `Burlington VT` · `Olympia WA` · `Bend OR` · `Chico CA` · `Duluth MN` · `Chattanooga TN`

---

## Roadmap

- [ ] TikTok Content Posting API integration
- [ ] Supabase photo upload UI
- [ ] Claude caption generator
- [ ] Sharp + ffmpeg slide renderer
- [ ] City manager UI
- [ ] Post queue dashboard
- [ ] TikTok Analytics pull cron
- [ ] Performance insights dashboard
- [ ] Hiker release campaign mode
- [ ] Unsplash photo ingestion + Claude Vision scoring pipeline

---

*Built for @dannybalentine.mp3 · Heavyweight out now on Spotify*