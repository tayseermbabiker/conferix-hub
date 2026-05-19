# Consultation: Conferix Scraping Pipeline — Second Opinion Wanted

## Context

I run **Conferix** — three event curation sites (UAE, USA, Sudan) plus
**UAESurfer** (UAE travel ticker). Looking for a second opinion on
whether my scraping pipeline could be re-architected to **save money,
save time, fail less, produce better data, and scale to more sources**.

I already optimized API calls today (~83% reduction). Now I want to
know if the *architecture itself* is the best choice, or if I'm
sitting in a local maximum.

## Current architecture

```
GitHub Actions (cron)
  → Playwright scraper (Node.js, 10-21 sources per region)
  → POST to Netlify Function webhook
  → Airtable (Events table)
  → Static export to events.json (post-scrape commit)
  → Frontend reads events.json (no live Airtable hits)
```

**Stack:** GitHub Actions free tier · Playwright + Chromium · Airtable
free plan · Netlify (static + Functions) · Resend for weekly email
alerts.

**Cadence:**
- Conferix UAE: Mon + Thu @ 1AM UTC
- Conferix USA: Mon + Thu @ 1AM UTC
- Conferix Sudan: user-submitted, no scraper
- UAESurfer: daily @ 6AM UTC
- Weekly email alerts: Sat (USA/UAE), Sun (Sudan)

**Sources:**
- **UAE (10):** Eventbrite, DWTC, ADNEC, DIFC, ADGM, Meetup [disabled],
  Terrapinn, Informa, DMG, ExpoCity
- **USA (21):** Eventbrite, Meetup [disabled], Luma [disabled],
  eMedEvents, Pri-Med, AMS, Clio, StartupGrind, USChamber,
  AllConferenceAlert, LegalWeek, DigiMarCon, TerrapinnNA, Javits, FIA,
  Pionline, Milken, ACG, FamilyOffice, FintechWeekly, ANA

**Per-source pattern:** each scraper launches headless Chromium →
navigates to listing → extracts events (title, date, venue, industry,
image, registration URL) → returns validated array. Aggregator POSTs
to webhook (UAE batches per source; USA batches by count).

**Webhook (just optimized today):**
- Bulk dedup: 1 list call per batch (was 1 per event)
- Batched writes: 10 records per call
- UAE also runs a "deactivation sweep" — events that disappeared from
  a source get marked inactive

## Current pain points

### Cost
- Hit Airtable's 1,000 calls/workspace/mo free limit
- After today's webhook optimizations, projected ~1,200/mo combined
  (still slightly over)
- Next step: split into separate workspaces, or upgrade Team ($20/mo,
  100k calls)

### Reliability glitches
- Playwright `networkidle` times out on many sites → using
  `domcontentloaded + waitForTimeout` (brittle)
- DWTC requires cookie consent click before content loads
- ADNEC: occasional 403s, JS routing not real anchors
- ADGM: full SPA, no interceptable API
- Luma: moved from lu.ma → luma.com (broke once)
- Meetup datetime attrs have `[Asia/Dubai]` suffix to strip
- Eventbrite: city names sometimes in Arabic (دبي = Dubai); JSON-LD
  uses `itemListElement[].item` not direct `@type: Event`
- One bad source can hang the whole run (just added per-job timeout)
- A recent scraper ran for 6 hours before GitHub Actions killed it

### Data quality
- Industry categorization is keyword-mapped → brittle, often defaults
  to "General"
- No cross-source dedup — same conference appears with different IDs
  from Eventbrite + Meetup + organizer site
- Airtable rejects ISO timestamps (must strip time)
- Airtable silently rejects events with broken image URLs
- Some scrapers don't extract early_bird_deadline / venue_address

### Scaling
- Each new source = writing a new scraper class
- Some sources have official APIs (Eventbrite, Luma) but we still
  Playwright-scrape them
- Sequential per-source execution → run time grows linearly
- USA's 21 sources take ~30 min per run

## Already considered & ruled out
- **n8n**: tried, too rigid for custom site quirks
- **Apify / Bright Data**: too expensive for free events
- **RSS feeds**: most premium B2B conference sites don't publish them

## Questions

1. **Architecture**: is there a fundamentally better pipeline pattern
   for multi-source event aggregation? Queue-based? Cloudflare Workers
   + D1? Bun runtime? Self-hosted somewhere?

2. **Storage**: is Airtable the right system-of-record at this scale,
   or should I move to Supabase / Cloudflare D1 / Turso /
   SQLite+Litestream? Trade-offs?

3. **Scraping reliability**: best patterns for managing 30+ flaky
   site-by-site Playwright scrapers? Browserbase? Apify actors? Own
   infra with playwright-extra-stealth?

4. **LLM extraction**: is Claude-based event extraction (give it the
   raw HTML, ask for structured JSON) now viable vs. brittle selectors?
   What's the realistic cost per event? Which model/prompt pattern?

5. **Scaling to 50-100 sources**: how would you architect this so cost
   and maintenance don't grow linearly with source count?

6. **Anti-pattern check**: anything in this setup that's a known
   antipattern you'd refactor immediately?

## What I want back
- Pragmatic recommendation, not "rewrite in Rust"
- Concrete trade-offs (X saves $Y/mo but costs Z complexity)
- The 1-2 highest-leverage changes if I can only do a few things
- Honest call on whether to stick with current stack or migrate
