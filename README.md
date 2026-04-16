# Sentinel

**AI agent for monitoring interstate natural gas pipeline critical notices, scoring portfolio-level dollar impact, and drafting NAESB nomination adjustments before cycle cutoffs.**

Built by [Optilytic](#) for a one-week demo to Tenaska Marketing Ventures (TMV).

---

## Table of contents

1. [What this is](#what-this-is)
2. [Target demo outcome](#target-demo-outcome)
3. [System architecture](#system-architecture)
4. [Data model](#data-model)
5. [Data sources](#data-sources)
6. [Agent pipeline](#agent-pipeline)
7. [Prompt contracts](#prompt-contracts)
8. [API routes](#api-routes)
9. [Frontend layout](#frontend-layout)
10. [Mock portfolio spec](#mock-portfolio-spec)
11. [Tariff snippet corpus](#tariff-snippet-corpus)
12. [Day-by-day build plan](#day-by-day-build-plan)
13. [Repo layout](#repo-layout)
14. [Environment variables](#environment-variables)
15. [Local setup](#local-setup)
16. [Deployment](#deployment)
17. [Testing and evals](#testing-and-evals)
18. [Demo script hooks](#demo-script-hooks)
19. [Out of scope](#out-of-scope)

---

## What this is

Sentinel is a web app that:

1. **Ingests** critical notices from interstate natural gas pipeline Electronic Bulletin Boards (EBBs) in near-real time.
2. **Classifies** each notice using an LLM into structured JSON (notice type, severity, affected segments, effective window, cycle cutoff, tariff citations).
3. **Joins** classified notices against a configured gas-marketer portfolio (firm transport contracts, storage rights, MDQs at specific pipeline receipt/delivery points).
4. **Scores** portfolio-level dollar exposure per notice using tariff-derived penalty bands × MDQ × current basis/Henry Hub.
5. **Drafts** a recommended nomination adjustment (cycle, contract, delta volume, rationale) scoped to submit before the next NAESB cycle cutoff.
6. **Answers** natural-language questions ("what's my total OFO exposure tomorrow on SONAT?") via RAG over the notice corpus + tariff snippets + NOAA degree-day outlooks.

This is a **pitch-grade prototype**, not production software. It uses 100% public data so the demo requires no integration with the client's internal systems.

---

## Target demo outcome

When a user loads the app on pitch day, they see:

- **A live feed** of the last 7 days of critical notices across ~20 pipelines, newest first, auto-refreshing every 5 minutes.
- **Three pre-curated "flagship" scenarios** pinned to the top, demonstrating: (a) a real OFO that intersects the mock portfolio, (b) a real maintenance notice threatening a western-book contract, (c) a simulated cold-snap layering NOAA forecast data with a Northwest Pipeline constraint.
- **A selected notice's impact card** showing severity, affected contracts, dollar range, cycle cutoff countdown, and a draft nom adjustment.
- **A chat panel** grounded in the notice corpus + tariff snippets.

Performance target: sub-2s from selecting a notice to displaying the impact card. Sub-5s for chat responses.

---

## System architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Vercel (Next.js 15 App Router)               │
│                                                                      │
│  ┌──────────────┐  ┌──────────────────┐  ┌────────────────────────┐ │
│  │   Frontend   │  │  API Route        │  │  Background Jobs       │ │
│  │  (RSC + CSR) │  │  Handlers         │  │  (Vercel Cron)         │ │
│  └──────┬───────┘  └────────┬──────────┘  └───────────┬────────────┘ │
│         │                    │                         │              │
│         └────────────────────┼─────────────────────────┘              │
│                              │                                        │
│  ┌───────────────────────────┴─────────────────────────────────────┐ │
│  │              Server-side agent orchestration layer               │ │
│  │   (classification, impact scoring, nom drafting, chat/RAG)      │ │
│  └──────┬──────────────────────────────┬──────────────────────────┘ │
└─────────┼──────────────────────────────┼─────────────────────────────┘
          │                              │
          ▼                              ▼
  ┌──────────────────┐          ┌───────────────────────┐
  │ Anthropic API    │          │ Vercel Postgres       │
  │ Claude Sonnet    │          │  - notices            │
  │ 4.5              │          │  - classifications    │
  └──────────────────┘          │  - portfolio          │
                                │  - exposures          │
                                │  - tariff_snippets    │
                                │  - weather_snapshots  │
                                └───────────┬───────────┘
                                            │
                          ┌─────────────────┴──────────────────┐
                          ▼                                    ▼
                  ┌──────────────────┐                 ┌─────────────────┐
                  │ PipeRiv feed     │                 │ Direct scrapers │
                  │ (Playwright)     │                 │ (Playwright)    │
                  └──────────────────┘                 └─────────────────┘
                                                              │
                                                              ▼
                                          ┌────────────────────────────────────┐
                                          │ NAESB cycle deadlines (static)     │
                                          │ NOAA CPC 6-10 / 8-14 day outlooks  │
                                          │ EIA daily Henry Hub + basis        │
                                          └────────────────────────────────────┘
```

**Runtime constraints:**
- Vercel serverless functions cap at 60s on Hobby, 300s on Pro. Long classification runs MUST be chunked.
- Vercel Cron minimum granularity is 1 minute; we use every 15 minutes.
- Claude API rate limits: assume 50 requests/minute per tier; batch classification in parallel with `Promise.allSettled`.

---

## Data model

PostgreSQL schema (Vercel Postgres / Neon). Use Drizzle ORM.

```typescript
// db/schema.ts

export const pipelines = pgTable('pipelines', {
  id: text('id').primaryKey(),              // e.g. 'tgp', 'transco', 'ngpl'
  displayName: text('display_name').notNull(),
  operator: text('operator'),               // Kinder Morgan, Williams, etc.
  ebbUrl: text('ebb_url'),
  scraperType: text('scraper_type').notNull().$type<'piperiv' | 'direct' | 'hybrid'>(),
  scraperConfig: jsonb('scraper_config'),   // CSS selectors, auth cookies, etc.
  active: boolean('active').default(true),
});

export const notices = pgTable('notices', {
  id: text('id').primaryKey(),              // hash(pipeline_id + source_notice_id)
  pipelineId: text('pipeline_id').references(() => pipelines.id).notNull(),
  sourceNoticeId: text('source_notice_id'), // pipeline's own ID
  sourceUrl: text('source_url'),
  subject: text('subject').notNull(),
  bodyRaw: text('body_raw').notNull(),       // untruncated
  postedAt: timestamp('posted_at').notNull(),
  effectiveStart: timestamp('effective_start'),
  effectiveEnd: timestamp('effective_end'),
  ingestedAt: timestamp('ingested_at').defaultNow(),
  category: text('category'),               // critical | planned | non-critical
});

export const classifications = pgTable('classifications', {
  noticeId: text('notice_id').primaryKey().references(() => notices.id),
  noticeType: text('notice_type').$type<
    'OFO' | 'FM' | 'MAINTENANCE' | 'CAPACITY_CONSTRAINT' | 'UNDERPERFORMANCE' |
    'SCHEDULING' | 'RESTART' | 'EXTREME_CONDITIONS' | 'OTHER'
  >().notNull(),
  severity: integer('severity').notNull(),  // 1-5
  affectedSegments: jsonb('affected_segments').$type<string[]>(),
  affectedMeters: jsonb('affected_meters').$type<string[]>(),
  cycleCutoffRelative: text('cycle_cutoff_relative'), // 'TIMELY' | 'EVENING' | 'ID1' | 'ID2' | 'ID3' | 'NONE'
  tariffSectionCited: text('tariff_section_cited'),
  capacityReductionPct: real('capacity_reduction_pct'),
  summaryOneSentence: text('summary_one_sentence').notNull(),
  confidence: real('confidence').notNull(), // 0-1
  modelVersion: text('model_version').notNull(),
  classifiedAt: timestamp('classified_at').defaultNow(),
});

export const contracts = pgTable('contracts', {
  id: text('id').primaryKey(),
  contractName: text('contract_name').notNull(),
  contractType: text('contract_type').$type<'FT' | 'IT' | 'STORAGE' | 'PAL' | 'CAPACITY_RELEASE'>().notNull(),
  pipelineId: text('pipeline_id').references(() => pipelines.id).notNull(),
  receiptPoints: jsonb('receipt_points').$type<Array<{ meter: string; mdq: number }>>(),
  deliveryPoints: jsonb('delivery_points').$type<Array<{ meter: string; mdq: number }>>(),
  zone: text('zone'),
  segment: text('segment'),
  mdqDth: integer('mdq_dth'),
  rateSchedule: text('rate_schedule'),
  counterparty: text('counterparty'),
  notes: text('notes'),
});

export const exposures = pgTable('exposures', {
  id: text('id').primaryKey(),              // uuid
  noticeId: text('notice_id').references(() => notices.id).notNull(),
  contractId: text('contract_id').references(() => contracts.id).notNull(),
  matchReason: text('match_reason').notNull(), // 'segment' | 'meter' | 'zone' | 'pipeline_wide'
  dthAtRisk: integer('dth_at_risk').notNull(),
  dollarImpactLow: integer('dollar_impact_low'),
  dollarImpactHigh: integer('dollar_impact_high'),
  penaltyRateSource: text('penalty_rate_source'), // tariff section
  nextCycleCutoffUtc: timestamp('next_cycle_cutoff_utc'),
  recommendedActionDraft: text('recommended_action_draft'),
  rankKey: real('rank_key'),                // sort for "Top 10 Exposures"
  createdAt: timestamp('created_at').defaultNow(),
});

export const tariffSnippets = pgTable('tariff_snippets', {
  id: text('id').primaryKey(),
  pipelineId: text('pipeline_id').references(() => pipelines.id).notNull(),
  section: text('section').notNull(),       // e.g. 'GT&C §11.3'
  topic: text('topic').notNull(),           // 'OFO_PENALTY' | 'CASHOUT' | 'FM_DEFINITION' | 'IMBALANCE'
  text: text('text').notNull(),
  penaltyMin: real('penalty_min'),          // $/Dth
  penaltyMax: real('penalty_max'),
  sourceUrl: text('source_url'),
  embedding: vector('embedding', { dimensions: 1536 }), // pgvector
});

export const weatherSnapshots = pgTable('weather_snapshots', {
  id: text('id').primaryKey(),
  fetchedAt: timestamp('fetched_at').defaultNow(),
  region: text('region').notNull(),         // 'Northeast' | 'Midwest' | 'West' | 'South' | 'CONUS'
  outlookType: text('outlook_type').$type<'6-10' | '8-14' | 'MONTHLY'>().notNull(),
  temperatureSignal: text('temperature_signal').$type<'BELOW' | 'NEAR' | 'ABOVE'>(),
  hddAnomaly: real('hdd_anomaly'),
  rawPayload: jsonb('raw_payload'),
});
```

**Indexing:**
- `notices(pipeline_id, posted_at DESC)` for feed queries
- `exposures(rank_key DESC)` for top-10
- `tariff_snippets` uses pgvector HNSW index on `embedding`

---

## Data sources

### Primary: PipeRiv

- Feed URL: `https://www.piperiv.com/feed/critical-notices`
- No official public API documented — **scrape via Playwright** from the HTML feed page.
- Covers ~100+ pipelines including all top-tier interstates.
- Each notice card exposes: pipeline, subject, posted timestamp, and a detail URL.
- Respect robots.txt; throttle to 1 request/sec; identify via a descriptive User-Agent.

### Secondary: Direct pipeline EBB scrapers

Build Playwright scrapers only for pipelines where PipeRiv lag or coverage is insufficient. Priority order for MVP:

| Pipeline | Operator | Scraper target URL | Notes |
|---|---|---|---|
| Transco | Williams | `https://www.1line.williams.com/` | Requires navigating to informational postings; no auth for public notices |
| Tennessee Gas (TGP) | Kinder Morgan | `https://pipeline2.kindermorgan.com/Notices/Notices.aspx?type=C&code=TGP` | ASP.NET viewstate — handle session cookie |
| NGPL | Kinder Morgan | `https://pipeline2.kindermorgan.com/Notices/Notices.aspx?type=C&code=NGPL` | Same pattern as TGP |
| El Paso (EPNG) | Kinder Morgan | `https://pipeline2.kindermorgan.com/Notices/Notices.aspx?type=C&code=EPNG` | Same pattern |
| Northern Natural | Berkshire | `https://www.northernnaturalgas.com/infopostings/Pages/AtaGlance.aspx` | Straightforward HTML |
| NGTL | TC Energy | `https://www.tccustomerexpress.com/925.html` | TC Customer Express — static page |
| ANR | TC Energy | `https://ebb.anrpl.com/Notices/NoticesDL.asp?sPipelineCode=ANR&sSubCategory=Critical` | Classic ASP |
| Southern Star | Southern Star | `https://www.sscgp.com/` | Navigate to Informational Postings |

**Scraper contract:** each scraper module exports `async function scrape(): Promise<RawNotice[]>` returning the canonical shape:

```typescript
type RawNotice = {
  sourceNoticeId: string;
  subject: string;
  bodyRaw: string;
  postedAt: Date;
  effectiveStart?: Date;
  effectiveEnd?: Date;
  sourceUrl: string;
  category: 'critical' | 'planned' | 'non-critical';
};
```

### Tertiary: contextual data feeds

- **NAESB cycle deadlines** — static constants in `lib/naesb.ts`. Timely 1:00 p.m. CCT, Evening 6:00 p.m. CCT, ID1 10:00 a.m. CCT, ID2 2:30 p.m. CCT, ID3 7:00 p.m. CCT (per FERC Order 809).
- **NOAA Climate Prediction Center outlooks** — `https://www.cpc.ncep.noaa.gov/products/predictions/610day/` (6-10 day) and 8-14 day JSON equivalents. Fetch daily at 08:00 UTC via cron.
- **EIA daily Henry Hub + regional basis** — `https://api.eia.gov/v2/natural-gas/pri/fut/data/` (free API key required). Also pull daily.

---

## Agent pipeline

Four distinct agents, all Claude Sonnet 4.5, each with its own system prompt and schema-enforced output.

```
Raw notice HTML
    ▼
[1] Classifier Agent  ──────▶  Classification row (notice_type, severity, segments, cycle cutoff)
    ▼
[2] Matcher Agent  ─────────▶  For each contract in portfolio: match? (boolean + reason)
    ▼
[3] Impact Scorer Agent  ───▶  For each match: dth_at_risk, $ range, penalty source
    ▼
[4] Action Drafter Agent  ──▶  Cycle, contract, delta, rationale, two-sentence explanation
    ▼
Display in UI
```

**The chat panel is a separate fifth agent** using RAG over notices + tariff snippets + weather snapshots.

### Sequencing rules

- **[1] Classifier runs immediately on ingest** (within the cron job). Result cached forever for that notice hash.
- **[2] Matcher runs on classifier completion** — only against active contracts. Cheap: no LLM call needed for 90% of matches (direct string match on pipeline_id + segment). Use the LLM only when ambiguity exists (fuzzy segment names, multi-zone notices).
- **[3] Impact scorer** runs per match. Requires tariff RAG lookup.
- **[4] Action drafter** runs only for exposures where `rank_key > threshold`. Lazy-compute on user click to save tokens.

### Idempotency

Every agent must be idempotent keyed by `(notice_id, contract_id, model_version)`. Re-running an agent on the same input must upsert, not duplicate.

---

## Prompt contracts

All prompts live in `lib/prompts/` as TypeScript template functions. All outputs are JSON, validated with Zod.

### Classifier prompt

**System:**
```
You are a natural gas pipeline operations analyst. You read critical notices
posted to interstate pipeline Electronic Bulletin Boards (EBBs) and extract
structured data. You NEVER guess. If a field is not explicitly stated in the
notice, return null. You return valid JSON only, no prose.
```

**User (template):**
```
Pipeline: {pipelineDisplayName}
Subject: {subject}
Posted at: {postedAt ISO}
Notice body:
---
{bodyRaw}
---

Return a JSON object matching this schema:
{
  "notice_type": "OFO" | "FM" | "MAINTENANCE" | "CAPACITY_CONSTRAINT" |
                 "UNDERPERFORMANCE" | "SCHEDULING" | "RESTART" |
                 "EXTREME_CONDITIONS" | "OTHER",
  "severity": 1 | 2 | 3 | 4 | 5,
  "affected_segments": string[],          // zones, mainlines, specific segments named in notice
  "affected_meters": string[],            // meter numbers or receipt/delivery point names
  "effective_start": string | null,       // ISO timestamp
  "effective_end": string | null,
  "cycle_cutoff_relative": "TIMELY" | "EVENING" | "ID1" | "ID2" | "ID3" | "NONE",
  "tariff_section_cited": string | null,
  "capacity_reduction_pct": number | null, // 0-100
  "summary_one_sentence": string,          // for the feed card
  "confidence": number                     // 0-1, your self-assessed confidence
}

Severity guidance:
1 = Informational, no flow impact
2 = Minor restriction, narrow segment, easily rerouted
3 = Material restriction or Type 1/2 OFO, broader impact
4 = Major FM, Type 3 OFO, or restart affecting mainline
5 = Systemwide FM, extreme conditions declaration, multi-day event
```

**Validation:** parse with Zod; if parse fails, retry once with `{previous_output: "..."}` appended. If second parse fails, log and mark `confidence: 0`.

### Matcher prompt

Only invoked when deterministic matching is ambiguous. Given a notice classification and a single contract, return `{matches: bool, reason: string, confidence: number}`.

### Impact scorer prompt

**System:**
```
You are a natural gas contract and tariff analyst. You estimate the dollar
exposure of a gas marketing contract to a pipeline critical notice, using
only tariff-cited penalty rates and explicitly stated MDQs. You bound
estimates with a low and high range and cite your sources.
```

**User (template):**
```
Notice summary: {summary_one_sentence}
Notice type: {notice_type}
Severity: {severity}
Affected segments: {affected_segments}
Capacity reduction: {capacity_reduction_pct}%

Contract:
- Name: {contractName}
- Type: {contractType}
- MDQ: {mdqDth} Dth/d
- Receipt points: {receiptPoints}
- Delivery points: {deliveryPoints}

Tariff snippets (top 3 by vector similarity):
{tariff_snippets_formatted}

Current Henry Hub: ${henryHubPrice}/MMBtu
Basis at delivery zone: ${basisPrice}/MMBtu

Return JSON:
{
  "dth_at_risk": number,
  "dollar_impact_low": number,
  "dollar_impact_high": number,
  "penalty_rate_source": string,     // tariff section cited
  "reasoning_three_sentences": string
}
```

### Action drafter prompt

Given an exposure row, output:
```json
{
  "recommended_cycle": "ID1" | "ID2" | "ID3" | "EVENING",
  "cycle_cutoff_utc": "2026-04-16T19:00:00Z",
  "contract_to_modify": "string (contract name)",
  "delta_dth": number,
  "direction": "REDUCE" | "INCREASE" | "REROUTE",
  "rerouting_path": string | null,
  "rationale_two_sentences": string
}
```

### Chat RAG prompt

Retrieves top-k notices + tariff snippets + latest weather snapshot by embedding similarity, stuffs into context, answers user question. Hard rule: every factual claim must cite a retrieved source. Refuse to answer if retrieval returns nothing.

---

## API routes

All Next.js App Router route handlers under `app/api/`. All return JSON.

```
GET  /api/notices                          List notices, paginated, filtered by pipeline/date
GET  /api/notices/[id]                     Full notice + classification
GET  /api/notices/[id]/exposures           Exposures for this notice, ranked
POST /api/notices/[id]/draft-action        Invoke action drafter for a specific exposure
GET  /api/feed/stream                      Server-sent events for live feed updates
POST /api/chat                             Chat endpoint (streaming). Body: { messages: Message[] }
GET  /api/portfolio                        Return mock portfolio
GET  /api/exposures/top                    Top-10 ranked exposures
POST /api/cron/ingest                      Cron endpoint, triggered every 15 min
POST /api/cron/weather                     Cron endpoint, triggered daily at 08:00 UTC
POST /api/cron/prices                      Cron endpoint, triggered daily at 09:00 UTC
```

**Auth:** cron endpoints require `Authorization: Bearer ${CRON_SECRET}` header (Vercel Cron handles this automatically). No other auth on the demo build — the site is a pitch tool, not production.

**Error envelope:**
```typescript
type ApiError = {
  error: {
    code: 'VALIDATION' | 'UPSTREAM' | 'LLM' | 'INTERNAL';
    message: string;
    details?: unknown;
  };
};
```

---

## Frontend layout

```
┌───────────────────────────────────────────────────────────────────────────────┐
│  Sentinel — TMV Demo Instance                        ◉ Live • 47 notices today │
├────────────────────┬────────────────────────────┬─────────────────────────────┤
│   NOTICE FEED      │   SELECTED NOTICE           │   RECOMMENDED ACTION        │
│                    │                             │                             │
│   [Filter pill    ]│   SONAT — OFO Type 3        │   ID2 cutoff: 1h 47m         │
│   All / FM / OFO / │   East of Reform            │                             │
│   Maint. / Cap.    │   Posted 06:12 CT           │   REDUCE                    │
│                    │                             │   Contract: SONAT-N-FT-003  │
│   ● SONAT Type 3   │   Severity: 4 / 5           │   Delta: -8,000 Dth         │
│     OFO East of    │                             │                             │
│     Reform         │   AFFECTED CONTRACTS (2)    │   Rationale:                │
│                    │   ┌────────────────────┐    │   Type 3 OFO at $10/Dth     │
│   ● TGP 500 Leg    │   │ SONAT-N-FT-003     │    │   penalty rate on this      │
│     restriction    │   │ MDQ 25,000 Dth     │    │   segment; reducing nom     │
│                    │   │ $ 80K - $ 125K     │    │   below 17,000 Dth avoids   │
│   ● Transwestern   │   └────────────────────┘    │   penalty exposure while    │
│     underperform   │   ┌────────────────────┐    │   preserving deliveries to  │
│                    │   │ SONAT-N-FT-007     │    │   counterparty X.           │
│   ● EPNG SOC draft │   │ MDQ 10,000 Dth     │    │                             │
│                    │   │ $ 20K - $  40K     │    │   [Copy draft]  [Details]   │
│   ● NGTL NE Gas    │   └────────────────────┘    │                             │
│     constraint     │                             ├─────────────────────────────┤
│                    │   TARIFF CITATION           │   CHAT                      │
│   (older notices   │   GT&C §18.4 Operational    │                             │
│    truncated)      │   Flow Order — Type 3      │   > What's my total OFO      │
│                    │   penalties: $10.00/Dth     │     exposure tomorrow on    │
│                    │   for non-compliance.       │     SONAT if the cold front │
│                    │                             │     verifies?               │
│                    │                             │                             │
│                    │                             │   Based on NOAA 6-10 day    │
│                    │                             │   outlook (below normal,    │
│                    │                             │   southeast), three active  │
│                    │                             │   SONAT contracts would...  │
└────────────────────┴────────────────────────────┴─────────────────────────────┘
```

**Tech choices:**
- Next.js 15 App Router, React Server Components where possible.
- Shadcn/ui + Tailwind. Dark mode default (trading-desk aesthetic).
- Monospace font (JetBrains Mono) for timestamps, meter codes, dollar amounts.
- Feed uses Server-Sent Events (`/api/feed/stream`) for live updates.
- Chat uses the Vercel AI SDK `useChat` hook, streaming from `/api/chat`.

**Branding constraints:**
- Do NOT use Tenaska's logo or official colors directly. Use a neutral dark navy header.
- Banner text: "Sentinel — TMV Demo Instance" in monospace.
- Include an "illustrative" badge on the portfolio sidebar.

---

## Mock portfolio spec

Seed data in `db/seed/portfolio.json`. ~40 contracts, ~2.5 Bcf/d notional book. Designed to intersect the highest-frequency notice-posting pipelines. **Label explicitly as illustrative** in the UI.

Distribution targets:

| Pipeline | # Contracts | Total MDQ (Dth/d) | Rationale |
|---|---|---|---|
| Transco (Zones 3, 5, 6) | 6 | 450,000 | Eastern Seaboard demand pull |
| Tennessee Gas (500/800 Leg) | 5 | 300,000 | Appalachia → Northeast |
| TETCO | 3 | 180,000 | Northeast market zone |
| Columbia Gulf | 2 | 120,000 | Gulf → Midwest |
| Columbia Gas | 3 | 150,000 | MidAtlantic LDC |
| REX (East-to-West) | 3 | 200,000 | Appalachia takeaway |
| NGPL (TexOk, Amarillo) | 4 | 250,000 | Mid-Con |
| Northern Natural | 3 | 150,000 | Upper Midwest LDC |
| Southern Natural | 3 | 120,000 | Southeast LDC |
| ANR | 2 | 100,000 | Great Lakes |
| NGTL | 2 | 150,000 | Western Canada |
| El Paso | 2 | 120,000 | California deliveries |
| Kern River | 1 | 80,000 | Rockies → California |
| Northwest Pipeline | 1 | 50,000 | Pacific Northwest LDC (name "NW Natural" in notes field) |
| Storage — Central Valley | 1 | 40,000 WD | CA storage ratchet |
| Storage — Salt Plains | 1 | 30,000 WD | Mid-Con storage |

Each contract JSON shape:
```json
{
  "id": "TRANSCO-Z6-FT-001",
  "contractName": "Transco Zone 6 FT to Manhattan",
  "contractType": "FT",
  "pipelineId": "transco",
  "receiptPoints": [{ "meter": "9155201", "mdq": 75000 }],
  "deliveryPoints": [{ "meter": "9034212", "mdq": 75000 }],
  "zone": "Z6",
  "segment": "Station 210 Mainline",
  "mdqDth": 75000,
  "rateSchedule": "FT",
  "counterparty": "Illustrative LDC",
  "notes": "Winter premium zone; high exposure to Station 210 FM events"
}
```

---

## Tariff snippet corpus

Pre-built, not LLM-generated. Source text manually from the relevant GT&C sections of each pipeline's FERC tariff (publicly filed on `elibrary.ferc.gov`).

Minimum coverage for MVP:

| Pipeline | Topics required |
|---|---|
| Transco | OFO penalty structure, imbalance cashout, FM definition |
| TGP | OFO penalty, hourly imbalance tolerance |
| Southern Natural | Type 1/2/3 OFO penalties ($10/Dth Type 3 confirmed) |
| NGPL | OFO, operational imbalance |
| Northern Natural | System management, storage imbalance |
| El Paso | SOC declaration, allocation under constraint |
| NGTL | Daily balancing, imbalance treatment |
| Columbia Gas | OFO, maintenance allocation |

Each snippet stored with pgvector embedding (generated via OpenAI `text-embedding-3-small` or Anthropic equivalent). Target: ~80 snippets total. Retrieval k=5 in the impact scorer.

---

## Day-by-day build plan

### Monday — Spine

- [ ] Bootstrap Next.js 15 app, deploy empty shell to Vercel
- [ ] Provision Vercel Postgres, run Drizzle migrations for all tables
- [ ] Build PipeRiv scraper (Playwright); ingest last 7 days of notices
- [ ] Build Northern Natural scraper as second data source
- [ ] Seed `pipelines` table with top 20 pipelines
- [ ] Seed `portfolio.json` and load into `contracts` table
- [ ] Verify ~300+ notices ingested

**Done when:** `SELECT count(*) FROM notices` returns >300 and `/api/notices` endpoint returns feed JSON.

### Tuesday — Classifier

- [ ] Write classifier prompt (`lib/prompts/classifier.ts`)
- [ ] Write classifier agent (`lib/agents/classifier.ts`) with Zod validation + 1 retry
- [ ] Run classifier over the 300 seeded notices, store in `classifications`
- [ ] Hand-label 30 of them; compute precision on `notice_type`, `severity`, `cycle_cutoff_relative`
- [ ] If precision < 85% on severity, iterate prompt

**Done when:** eval harness shows ≥85% precision on `notice_type` and `severity`.

### Wednesday — Matcher + impact scorer

- [ ] Write deterministic matcher (string match on `pipeline_id` + `segment`)
- [ ] Write LLM matcher for ambiguous cases
- [ ] Build tariff snippet corpus (80 snippets, manual + FERC eLibrary)
- [ ] Embed snippets with pgvector HNSW index
- [ ] Write impact scorer agent with RAG retrieval
- [ ] Populate `exposures` table for all classified notices × contracts
- [ ] Verify ranked "Top 10 Exposures" query returns sensible results

**Done when:** for a hand-picked Type 3 OFO, the scorer produces a dollar range within an order of magnitude of (tariff penalty × contract MDQ).

### Thursday — Action drafter + chat

- [ ] Write action drafter prompt + agent
- [ ] Wire `/api/notices/[id]/draft-action` endpoint
- [ ] Build chat RAG endpoint (`/api/chat`) with streaming
- [ ] Fetch NOAA 6-10 and 8-14 day outlooks daily (cron)
- [ ] Fetch EIA Henry Hub + regional basis daily (cron)
- [ ] Integrate weather + prices into chat context

**Done when:** the chat can correctly answer "what's my biggest OFO exposure right now" using live data.

### Friday — Frontend + polish

- [ ] Build three-column layout
- [ ] Feed panel with filter pills, live SSE updates
- [ ] Impact card with tariff citations
- [ ] Action panel with copy-to-clipboard
- [ ] Chat panel (Vercel AI SDK)
- [ ] Dark mode, JetBrains Mono, "illustrative" badging
- [ ] Pin three flagship scenarios to feed top
- [ ] Pre-populate one "today's real OFO" card, one "real maintenance notice" card, one "simulated cold-snap" card

**Done when:** a non-developer can navigate the three columns and read a coherent impact card.

### Saturday — Resilience

- [ ] Record a 4-minute fallback walkthrough video
- [ ] Cache last 7 days of notices to local JSON for offline demo mode
- [ ] Add `DEMO_MODE=offline` env toggle that reads from cache instead of hitting Anthropic
- [ ] Full dry-run with all three co-founders
- [ ] Stress-test Claude rate limits by simulating 50 concurrent classifications

**Done when:** the demo works with Wi-Fi unplugged.

### Sunday — Morning-of rehearsal

- [ ] Pull the sharpest real notice posted that morning
- [ ] Reshuffle flagship pins if a better live event appeared
- [ ] Final read-through of the pitch narrative against the live feed

---

## Repo layout

```
sentinel/
├── README.md                               (this file)
├── package.json
├── next.config.mjs
├── tailwind.config.ts
├── drizzle.config.ts
├── .env.example
├── .env.local                              (gitignored)
├── vercel.json                             (cron config)
│
├── app/
│   ├── layout.tsx
│   ├── page.tsx                            (main dashboard)
│   ├── globals.css
│   └── api/
│       ├── notices/
│       │   ├── route.ts                    GET list
│       │   └── [id]/
│       │       ├── route.ts                GET detail
│       │       ├── exposures/route.ts      GET exposures for notice
│       │       └── draft-action/route.ts   POST draft action
│       ├── feed/stream/route.ts            SSE
│       ├── chat/route.ts                   Chat streaming
│       ├── portfolio/route.ts              GET portfolio
│       ├── exposures/top/route.ts          GET top exposures
│       └── cron/
│           ├── ingest/route.ts
│           ├── weather/route.ts
│           └── prices/route.ts
│
├── components/
│   ├── feed/
│   │   ├── FeedPanel.tsx
│   │   ├── NoticeCard.tsx
│   │   └── FilterPills.tsx
│   ├── impact/
│   │   ├── ImpactPanel.tsx
│   │   ├── ContractExposureCard.tsx
│   │   └── TariffCitation.tsx
│   ├── action/
│   │   ├── ActionPanel.tsx
│   │   ├── DraftCard.tsx
│   │   └── CycleCountdown.tsx
│   ├── chat/
│   │   └── ChatPanel.tsx
│   └── ui/                                 (shadcn components)
│
├── lib/
│   ├── agents/
│   │   ├── classifier.ts
│   │   ├── matcher.ts
│   │   ├── impactScorer.ts
│   │   ├── actionDrafter.ts
│   │   └── chatRag.ts
│   ├── prompts/
│   │   ├── classifier.ts
│   │   ├── matcher.ts
│   │   ├── impactScorer.ts
│   │   ├── actionDrafter.ts
│   │   └── chatSystem.ts
│   ├── scrapers/
│   │   ├── types.ts                        RawNotice type
│   │   ├── piperiv.ts
│   │   ├── tgp.ts
│   │   ├── ngpl.ts
│   │   ├── epng.ts
│   │   ├── northernNatural.ts
│   │   ├── ngtl.ts
│   │   └── transco.ts
│   ├── dataSources/
│   │   ├── noaa.ts                         CPC 6-10, 8-14 fetchers
│   │   └── eia.ts                          Henry Hub + basis
│   ├── naesb.ts                            Cycle deadline constants + utilities
│   ├── embedding.ts                        Embedding helpers (pgvector)
│   ├── anthropic.ts                        Claude client
│   └── db.ts                               Drizzle client
│
├── db/
│   ├── schema.ts
│   ├── migrations/
│   └── seed/
│       ├── pipelines.json
│       ├── portfolio.json
│       └── tariffSnippets.json
│
├── evals/
│   ├── classifierEval.ts
│   ├── impactScorerEval.ts
│   └── goldSet/
│       ├── notices-labeled.json            30 hand-labeled notices
│       └── exposures-labeled.json
│
└── scripts/
    ├── seed.ts
    ├── ingest-backfill.ts
    ├── embed-tariffs.ts
    └── offline-cache.ts
```

---

## Environment variables

```bash
# Anthropic
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-4-5

# Database (Vercel Postgres / Neon)
DATABASE_URL=postgres://...
DATABASE_URL_UNPOOLED=postgres://...

# Embeddings (OpenAI for text-embedding-3-small, cheapest option)
OPENAI_API_KEY=sk-...

# EIA
EIA_API_KEY=...

# Cron
CRON_SECRET=<random 32-byte hex>

# Feature flags
DEMO_MODE=live                              # or 'offline'
CLASSIFIER_VERSION=v1.0

# Observability
LOGTAIL_SOURCE_TOKEN=...                    # optional
```

---

## Local setup

```bash
# 1. Install
pnpm install

# 2. Start Playwright deps
pnpm exec playwright install chromium

# 3. Provision database
# Option A: Vercel Postgres via `vercel link` + `vercel env pull`
# Option B: Local Postgres via docker:
docker run --name sentinel-pg -e POSTGRES_PASSWORD=dev -p 5432:5432 -d pgvector/pgvector:pg16

# 4. Migrate and seed
pnpm db:migrate
pnpm db:seed

# 5. Embed tariff snippets (one-time, costs a few cents)
pnpm tsx scripts/embed-tariffs.ts

# 6. Run initial ingest
pnpm tsx scripts/ingest-backfill.ts --days 7

# 7. Dev server
pnpm dev
```

---

## Deployment

```bash
vercel link
vercel env pull .env.local
vercel --prod
```

**Vercel project settings required:**
- Framework preset: Next.js
- Node.js version: 20.x
- Postgres: provisioned from the Storage tab
- Cron jobs (`vercel.json`):
  ```json
  {
    "crons": [
      { "path": "/api/cron/ingest",  "schedule": "*/15 * * * *" },
      { "path": "/api/cron/weather", "schedule": "0 8 * * *" },
      { "path": "/api/cron/prices",  "schedule": "0 9 * * *" }
    ]
  }
  ```
- Function max duration: 300s (Pro plan required for classification backfills)
- Playwright: needs `@sparticuz/chromium` for the serverless runtime; use the `chromium-min` variant to stay under the 50MB function limit.

---

## Testing and evals

No production test suite — this is a prototype. But maintain two small eval harnesses:

**Classifier eval:** `evals/classifierEval.ts` runs the classifier over 30 hand-labeled notices and reports precision/recall on `notice_type` and `severity`. Acceptance gate: ≥85% precision on both.

**Impact scorer eval:** `evals/impactScorerEval.ts` runs the scorer over 10 hand-constructed (notice, contract) pairs with expected dollar ranges. Acceptance gate: predicted range must overlap the expected range for ≥8/10 cases.

Run before every demo: `pnpm eval`.

---

## Demo script hooks

Wire these into the UI so the pitch flows without manual stage-managing:

1. **"Flagship" pin mechanism** — admin-only query param `?pin=notice_id_1,notice_id_2,notice_id_3` forces three notices to the top of the feed in a fixed order.
2. **"Replay mode"** — `?replay=2026-02-22` reloads the feed as it looked on that date. Used to show the February 2026 SONAT Type 3 OFO as a "if this existed last winter" moment.
3. **"Cold snap" scenario** — `?scenario=cold-snap` injects a synthetic Northwest Pipeline constraint notice alongside real NOAA below-normal data to demonstrate the chat answering forward-looking questions. Clearly badged as synthetic.
4. **Timer freeze** — `?freezeCountdown=1h47m` stops the cycle countdown at a dramatic value for screenshots/video.

Every one of these hooks must be gated behind `NODE_ENV !== 'production'` OR a `?demoKey=` secret, so the live site never serves synthetic data by accident.

---

## Out of scope

Explicitly **not** building for this week:

- Authentication, user management, RBAC
- Multi-tenancy (the whole app is one "TMV demo instance")
- Writing to any external system (no actual nom submission to a pipeline)
- Integration with ION RightAngle, Allegro, Endur, or any ETRM
- SOC2, audit logs, data retention policies
- Mobile layout (demo is desktop-only)
- Real-time WebSocket streaming of notices (SSE is sufficient)
- Multi-language support
- Historical backtesting beyond the 7-day cached window
- Any feature requiring TMV's proprietary data

These belong in the 6-week paid discovery engagement that follows a successful pitch, not in the one-week demo build.

---

## License

Proprietary. All rights reserved, Optilytic Inc.
