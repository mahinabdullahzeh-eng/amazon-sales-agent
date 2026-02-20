# Amazon Sales Intelligence Agent

AI-powered Amazon product analysis and listing optimization platform. The system combines web scraping, multi-agent AI pipelines, and structured data processing to research products, analyze keywords, score opportunities, and generate SEO-optimized listing content.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Four-Agent Pipeline](#the-four-agent-pipeline)
   - [Agent 1 – Research Agent](#agent-1--research-agent)
   - [Agent 2 – Keyword Categorization Agent](#agent-2--keyword-categorization-agent)
   - [Agent 3 – Scoring Agent (with Sub-Agents)](#agent-3--scoring-agent-with-sub-agents)
   - [Agent 4 – SEO Optimization Agent](#agent-4--seo-optimization-agent)
3. [Amazon Scraping System](#amazon-scraping-system)
4. [Anti-Blocking System](#anti-blocking-system)
5. [Keyword Processing Utilities](#keyword-processing-utilities)
6. [Job Management System](#job-management-system)
7. [API Endpoints](#api-endpoints)
8. [Frontend Application](#frontend-application)
9. [External APIs and Services](#external-apis-and-services)
10. [Configuration & Environment Variables](#configuration--environment-variables)
11. [Technology Stack](#technology-stack)
12. [Deployment](#deployment)

---

## Architecture Overview

```
CSV Files (Helium 10 Cerebro export)
        │
        ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         FastAPI Backend                                     │
│                                                                              │
│  POST /api/v1/amazon-sales-intelligence  (sync)                             │
│  POST /api/v1/start-analysis             (async background job)             │
│  GET  /api/v1/job-status/{job_id}                                           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  4-AGENT AI PIPELINE                                 │   │
│  │                                                                       │   │
│  │  [1] ResearchAgent   → scrape + market position + relevancy scores  │   │
│  │          ↓                                                            │   │
│  │  [2] KeywordAgent    → categorize keywords (6 categories)            │   │
│  │          ↓                                                            │   │
│  │  [3] ScoringRunner   → intent + metrics + broad volume + variants    │   │
│  │     ├─ IntentScoringSubagent                                         │   │
│  │     ├─ MetricsSubagent (deterministic)                               │   │
│  │     ├─ BroadVolumeAgent                                              │   │
│  │     ├─ KeywordVariantAgent                                           │   │
│  │     ├─ RootRelevanceAgent                                            │   │
│  │     └─ OpportunitySubagent                                           │   │
│  │          ↓                                                            │   │
│  │  [4] SEOOptimizationAgent → optimized title + bullets + backend kws │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Storage: Redis (Upstash) or local file system                              │
└────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│   Next.js Frontend      │
│   (KeywordAI Dashboard) │
└─────────────────────────┘
```

---

## The Four-Agent Pipeline

All agents are built on the **OpenAI Agents SDK** (`openai-agents>=0.2.9`) and use the `gpt-5-mini-2025-08-07` model with minimal reasoning effort for cost efficiency.

### Agent 1 – Research Agent

**File:** `backend/app/local_agents/research/`

**Purpose:** Scrapes an Amazon product listing and produces structured market intelligence.

**What it does:**
1. Scrapes the target product page using multiple scrapers (Scrapy, Playwright, SERP API fallback).
2. Scrapes competitor products listed in the uploaded CSV files (revenue and design competitors).
3. Computes a **base relevancy score (0–10)** for each keyword from the CSV files:
   - Formula: counts how many competitor ASINs rank in the top 10 for each keyword.
   - Score = `(top_10_count / total_asins) × 20`, capped at 10.
4. Applies **literal relevance** adjustment: boosts score for keywords whose tokens are found in the product title.
5. Applies **competitor title relevance** adjustment: boosts score when many competitors use the keyword in their titles.
6. Applies a **dynamic relevancy threshold** (3–9 depending on dataset size) to filter low-quality keywords before AI processing.
7. Sends scraped data + slim CSV context to the `ResearchAgent` AI model for structured analysis.

**Output schema (`ResearchOutput`):**
- `content_sources` – extraction quality of Title, Images, A+ Content, Reviews, Q&A
- `market_position` – budget/premium tier with price and unit data
- `main_keyword` – chosen main keyword with candidates and rationale
- `current_listing` – existing title, bullet points, backend keywords

**Also returns (for downstream agents):**
- `scraped_product` – full structured scraped data
- `base_relevancy_scores` – keyword → score (0–10), filtered by dynamic threshold
- `full_base_relevancy_scores` – complete unfiltered scores
- `adjusted_relevancy_scores` – scores after literal and competitor adjustments
- `keyword_root_analysis` / `priority_roots` – root extraction results
- `ai_keyword_root_analysis` – AI-powered root analysis (via `RootExtractionAgent` subagent)
- `competitor_scrapes` – scraped competitor data (price, rating, review count)

**Runner:** `backend/app/local_agents/research/runner.py` → `ResearchRunner.run_research()`

---

### Agent 2 – Keyword Categorization Agent

**File:** `backend/app/local_agents/keyword/`

**Purpose:** Classifies every keyword into exactly one of six categories.

**Six keyword categories:**

| Category | Description |
|---|---|
| `Relevant` | Core keywords directly describing the product in its base form |
| `Design-Specific` | Keywords describing attributes/variations (material, size, color, packaging) of the same product form |
| `Irrelevant` | Keywords describing a different product form or with no connection |
| `Branded` | Keywords containing any brand name (own brand or competitors) |
| `Spanish` | Keywords in Spanish or other non-English languages |
| `Outlier` | Extremely broad, high-volume generic terms showing wide product variety |

**How it works:**
1. Extracts product context: title, brand, base product form (slices/powder/whole/liquid/capsules/etc.) from the scraped product.
2. Filters out zero-relevancy keywords.
3. Splits keywords into **batches of 75** to prevent JSON truncation.
4. For each batch, builds a prompt with full product context, brand detection rules, and base relevancy scores.
5. Calls `KeywordAgent` (AI model) which applies a sequential categorization algorithm: Brand → Language → Product Form → Contextual Connection → Outlier → Attributes → Final.
6. Merges batch results and normalizes field names.

**Output:** `KeywordAnalysisResult` with:
- `items[]` – each keyword with `phrase`, `category`, `relevancy_score`
- `stats` – per-category counts and examples
- `product_context` – extracted product metadata

**Runner:** `backend/app/local_agents/keyword/runner.py` → `KeywordRunner.run_keyword_categorization()`

---

### Agent 3 – Scoring Agent (with Sub-Agents)

**File:** `backend/app/local_agents/scoring/`

**Purpose:** Enriches categorized keywords with metrics, intent scores, broad volume data, variant analysis, and opportunity flags.

**Sub-agents and components:**

#### IntentScoringSubagent
**File:** `backend/app/local_agents/scoring/subagents/intent_agent.py`

Scores buyer intent for each keyword on a **0–3 scale**:
- `0` – Irrelevant
- `1` – One relevant aspect
- `2` – Two relevant aspects  
- `3` – Three relevant aspects (strong transactional intent)

Uses `gpt-5-mini-2025-08-07` with the scraped product + base relevancy scores as context.

#### MetricsSubagent (deterministic)
**File:** `backend/app/local_agents/scoring/subagents/metrics_agent.py`

Extracts Helium 10 metrics **directly from the uploaded CSVs** without AI — deterministic lookup by keyword phrase:
- `search_volume` – from "Search Volume" column
- `title_density` – from "Title Density" column  
- `cpr` – from "CPR" column
- `competition` – Competing Products, Ranking Competitors, Competitor Rank, Competitor Performance Score

#### BroadVolumeAgent
**File:** `backend/app/local_agents/scoring/subagents/broad_volume_agent.py`

Computes **aggregate search volume per keyword root**:
- Identifies the root token for each keyword (main noun, excluding stopwords/brands).
- Sums search volume across all keywords sharing that root.
- Only counts `Relevant` and `Design-Specific` keywords in the sum.
- Returns `broad_search_volume_by_root` map.

#### KeywordVariantAgent  
**File:** `backend/app/local_agents/scoring/subagents/keyword_variant_agent.py`

Handles **singular/plural and article variants** (e.g., "strawberry freeze dried" vs "strawberries freeze dried" vs "the strawberry freeze dried"):
- Detects variant groups using AI.
- Selects the optimal variant based on search volume and grammatical structure.
- Returns `variant_groups` and `optimized_keywords` with deduplication.

#### RootRelevanceAgent
**File:** `backend/app/local_agents/scoring/subagents/root_relevance_agent.py`

Filters which **keyword roots** should be included in broad volume calculations by assessing AI-driven relevance. Replaces programmatic filtering with intelligent root relevance assessment.

#### OpportunitySubagent
**File:** `backend/app/local_agents/scoring/subagents/opportunity_agent.py`

Applies **zero-title-density opportunity rules**:
- If `title_density == 0` and keyword is relevant: flag as opportunity.
- If `title_density` is low and volume is decent: flag as opportunity.
- Returns `opportunity_decision` and `opportunity_reason` per keyword.

**Runner:** `backend/app/local_agents/scoring/runner.py` → `ScoringRunner.run_scoring()`

---

### Agent 4 – SEO Optimization Agent

**File:** `backend/app/local_agents/seo/`

**Purpose:** Analyzes the current listing's SEO state and generates fully optimized title, bullet points, and backend keywords.

**What it does:**

1. **Current SEO Analysis** (deterministic):
   - Calculates keyword coverage percentage (how many relevant keywords appear in title, bullets, backend).
   - Measures character efficiency (how well available character space is used).
   - Identifies root keyword coverage gaps.
   - Identifies missing high-intent and high-volume keywords.

2. **Keyword Validator** (`SEOKeywordValidator`):
   - Prevents AI hallucination — validates that every keyword in the AI output actually exists in the research dataset.
   - Runs post-generation validation and correction.

3. **AI Optimization** (via `SEOOptimizationAgent`):
   - Generates an optimized **title** (≤200 chars): brand name first, then keywords in descending volume order, includes 2–3 Design-Specific keywords from the highest-volume root.
   - Generates **5 bullet points**: first 2 use top-5 highest volume keywords, bullets 3–5 use medium-high volume terms.
   - Generates **backend keywords**: terms not already in title/bullets, includes synonyms, alternate spellings, and misspellings.

4. **SEO Keyword Filter** (`seo_keyword_filter.py`): Post-processes AI output to validate and correct keyword inclusion.

5. **Amazon Compliance Agent** (`subagents/amazon_compliance_agent.py`): Checks SEO content against Amazon's listing guidelines.

6. **Competitor Title Analysis Agent** (`subagents/competitor_title_analysis_agent.py`): Analyzes patterns in competitor titles to inform optimization strategy.

**Output schema (`SEOAnalysisResult`):**
- `current_seo` – title/bullet/backend analysis with coverage metrics
- `optimized_seo` – improved title, bullets, backend keywords with keyword lists and character counts
- `comparison` – before/after metrics: coverage %, intent score, volume capture
- `improvement_rationale` – explanation of optimization decisions

**Runner:** `backend/app/local_agents/seo/runner.py` → `SEORunner.run_seo_analysis()`

---

## Amazon Scraping System

**Directory:** `backend/app/services/amazon/`

The system uses multiple scraping strategies with automatic fallback:

### Scrapy Scraper
**File:** `scraper.py`, `search_scraper.py`

Primary scraper using **Scrapy** framework. Extracts specific HTML element IDs from Amazon product pages:

| Element ID | Content |
|---|---|
| `#productTitle` | Product title |
| `#productOverview_feature_div` | Brand, size, and key-value features |
| `#feature-bullets` | Bullet point features |
| `#productDescription` | Product description |
| `#prodDetails` | Technical specifications |
| `#detailBullets_feature_div` | Detail bullets |
| `#aplus` | A+ (Enhanced Brand Content) modules |

Also extracts: product images (via `data-a-dynamic-image` JSON, `srcset` attributes), customer reviews, Q&A pairs.

### Playwright Scraper
**File:** `playwright_scraper.py`

Headless-browser scraper using **Playwright** for JavaScript-rendered content. Used as a fallback when Scrapy is blocked or when content requires JavaScript execution.

### SERP API Scraper
**File:** `serp_api_scraper.py`

Integration with **SerpAPI** (`serpapi.com`) using the `amazon_product` engine. Provides reliable, fast scraping without IP blocking risk. Requires a SerpAPI key (`SERPAPI_API_KEY` env var). Returns normalized product data matching the pipeline's internal format.

### Standalone Scrapers
**Files:** `standalone_scraper.py`, `standalone_search_scraper.py`

Lightweight scrapers using **requests + BeautifulSoup** and **CloudScraper** for fast, self-contained scraping without the full Scrapy framework overhead.

### Multi-Marketplace Support
**File:** `country_handler.py`

Supports multiple Amazon marketplaces by mapping marketplace codes to base URLs:

| Code | Marketplace |
|---|---|
| `US` | amazon.com |
| `UK` / `GB` | amazon.co.uk |
| `DE` | amazon.de |
| `FR` | amazon.fr |
| `IT` | amazon.it |
| `ES` | amazon.es |
| `CA` | amazon.ca |
| `JP` | amazon.co.jp |
| `AU` | amazon.com.au |
| `IN` | amazon.in |
| `MX` | amazon.com.mx |
| `BR` | amazon.com.br |

---

## Anti-Blocking System

**Directory:** `backend/app/services/amazon/anti_blocking/`

A comprehensive anti-detection system to avoid Amazon's bot-blocking measures.

### User Agent Rotation
**File:** `user_agents.py`

Maintains a pool of real browser user-agent strings (Chrome, Firefox, Safari on Windows, Mac, Linux) and randomly selects one per request.

### Header Randomization
**File:** `headers.py`

Generates realistic HTTP request headers per request:
- `Accept-Language` rotation (US English variants)
- `Accept-Encoding` combinations
- `Cache-Control` variations
- Amazon session-specific headers (`x-requested-with`, `x-amzn-RequestId`)
- `Referer` and `Origin` headers

### Proxy Rotation
**File:** `proxy_manager.py`

Supports multiple proxy providers configured via environment variables:
- Direct proxy list (`SCRAPER_PROXY`, `SCRAPER_PROXY_LIST`)
- **Bright Data** (Luminati): `BRIGHT_DATA_HOST`, `BRIGHT_DATA_USER`, `BRIGHT_DATA_PASS`
- **Smartproxy**: `SMARTPROXY_HOST`, `SMARTPROXY_USER`, `SMARTPROXY_PASS`
- **Oxylabs**: `OXYLABS_HOST`, `OXYLABS_USER`, `OXYLABS_PASS`

Rotates proxies every 30 seconds by default.

### Scrapy Middlewares
**File:** `middlewares.py`

Custom Scrapy downloader middlewares:
- `RotateUserAgentMiddleware` – rotates user agent on every request
- `RotateHeadersMiddleware` – generates fresh realistic headers per request
- `ProxyRotationMiddleware` – injects rotating proxy configuration
- `CustomRetryMiddleware` – retries on 503/429 with exponential backoff, marks proxies as failed

### CloudScraper Integration
**File:** `anti_blocking.py`

Uses the **CloudScraper** library to bypass Cloudflare and JavaScript challenge pages. Falls back to standard requests when CloudScraper is unavailable.

---

## Keyword Processing Utilities

**Directory:** `backend/app/services/keyword_processing/`

### Root Extraction
**File:** `root_extraction.py`

Deterministic root extraction for keywords:
- Identifies priority root terms (most meaningful nouns) for a keyword list.
- Used for grouping keywords and computing broad search volume.
- `get_priority_roots_for_search()` returns top roots ranked by aggregate search volume.

### Batch Processor
**File:** `batch_processor.py`

`optimize_keyword_processing_for_agents()` combines root extraction across both revenue and design CSV keyword lists, deduplicates, and returns an optimized analysis for AI agent consumption.

### Intent Sorting
**File:** `sort.py`, `intent.py`

Sorts keywords by intent score and relevancy score for prioritized analysis.

### AI Root Extraction Subagent
**File:** `backend/app/local_agents/keyword/subagents/root_extraction_agent.py`

AI-powered extraction of root terms using the OpenAI Agents SDK. Provides richer root analysis than the deterministic approach by understanding product context.

### AI Intent Classification Subagent
**File:** `backend/app/local_agents/keyword/subagents/intent_classification_agent.py`

AI subagent for classifying keyword intent using product context and base relevancy scores.

### Keyword Deduplication (in ResearchRunner)

Before AI agents process keywords, the pipeline:
1. Merges keywords from both revenue and design CSVs.
2. Removes exact duplicates (case-insensitive).
3. When a keyword appears in both CSVs, keeps the **highest relevancy score** across all sources.
4. Applies a dynamic relevancy threshold filter before sending to AI agents.

---

## Job Management System

**File:** `backend/app/services/job_manager.py`

For long-running analyses (processing thousands of keywords can take minutes), the system provides an **asynchronous background job** system.

### Storage Backends (automatic selection):
1. **Redis (Upstash)** – primary storage for production deployments
   - Configures via `UPSTASH_REDIS_URL` and `UPSTASH_REDIS_TOKEN`
   - Jobs expire after `JOB_TTL_HOURS` (default: 24 hours)
2. **File system** – fallback for local development (stored in `jobs/` directory)

### Job Lifecycle:
1. `JobManager.create_job()` → returns unique UUID job ID
2. Status: `pending` → `processing` (with progress 0–100%) → `complete` / `failed`
3. `JobManager.update_status()` – updates progress and message during processing
4. `JobManager.save_results()` – stores full pipeline output
5. `JobManager.get_results()` – retrieves stored results by job ID

### OpenAI Rate Limiter
**File:** `backend/app/services/openai_rate_limiter.py`

Token bucket rate limiter for OpenAI API calls:
- `OPENAI_REQUESTS_PER_MINUTE` (default: 15)
- `OPENAI_REQUESTS_PER_SECOND` (default: 2)
- Exponential backoff with `OPENAI_MAX_RETRIES` (default: 3)

### OpenAI Usage Monitor
**File:** `backend/app/services/openai_monitor.py`

Tracks token usage, request counts, and estimated costs per pipeline run. Logs detailed stats when `ENABLE_OPENAI_MONITORING=true` and `LOG_DETAILED_STATS=true`.

---

## API Endpoints

**Base URL:** `/api/v1`

### `POST /amazon-sales-intelligence` (Synchronous)
Runs the complete 4-agent pipeline. Accepts `multipart/form-data`:

| Field | Type | Required | Description |
|---|---|---|---|
| `asin_or_url` | string | ✅ | Amazon ASIN or full product URL |
| `marketplace` | string | default: `US` | Marketplace code |
| `main_keyword` | string | optional | Override for auto-detected main keyword |
| `revenue_csv` | file | optional | Helium 10 Cerebro export (top revenue competitors) |
| `design_csv` | file | optional | Helium 10 Cerebro export (top design competitors) |

### `POST /start-analysis` (Asynchronous – recommended for production)
Same parameters as above. Returns immediately with a `job_id`. Use status polling to track progress.

### `GET /job-status/{job_id}`
Poll for job progress. Returns:
```json
{
  "status": "processing",
  "progress": 45,
  "message": "Running keyword categorization...",
  "result": null
}
```

When `status == "complete"`, `result` contains the full pipeline output.

### `POST /upload/csv`
Standalone CSV upload and parsing endpoint. Returns parsed row data as JSON.

### `GET /`
Health check — returns `{"message": "Welcome to the Amazon Sales Agent API"}`.

---

## Frontend Application

**Directory:** `frontend/`

A **Next.js** (React) dashboard application providing a web UI for the platform.

**Framework:** Next.js with App Router, TypeScript, Tailwind CSS, shadcn/ui components

**Dashboard pages:**
- `/dashboard` – Overview with key metrics (total researches, keywords found, avg optimization score, competitors analyzed)
- `/dashboard/research` – New research form (ASIN/URL input, marketplace selection, keyword, CSV upload)
- `/dashboard/upload` – File upload for CSV data
- `/dashboard/results` – Research results display
- `/dashboard/history` – Past analysis history
- `/dashboard/reports` – Generated reports

**Frontend–Backend communication:**
- `lib/api.ts` – Synchronous API client using `fetch`
- `lib/api-client-background.ts` – Async API client for background job polling
- `lib/config.ts` – Backend URL configuration (via `NEXT_PUBLIC_API_URL` env var)

**Deployed at:** `https://amazon-sales-agent.vercel.app` (Vercel)

---

## External APIs and Services

| Service | Purpose | Configuration |
|---|---|---|
| **OpenAI API** | Powers all 4 AI agents and sub-agents | `OPENAI_API_KEY` |
| **SerpAPI** | Reliable Amazon product scraping | `SERPAPI_API_KEY` |
| **Upstash Redis** | Background job storage in production | `UPSTASH_REDIS_URL`, `UPSTASH_REDIS_TOKEN` |
| **Bright Data** | Residential proxy provider for scraping | `BRIGHT_DATA_HOST/USER/PASS` |
| **Smartproxy** | Alternative proxy provider | `SMARTPROXY_HOST/USER/PASS` |
| **Oxylabs** | Alternative proxy provider | `OXYLABS_HOST/USER/PASS` |

**OpenAI Model:** `gpt-5-mini-2025-08-07` (used by all agents)
**AI Framework:** `openai-agents>=0.2.9` (OpenAI Agents SDK)

---

## Configuration & Environment Variables

All settings are loaded from `.env` file or environment. See `backend/app/core/config.py`.

### Required
| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | OpenAI API key for all AI agents |

### Agent Behavior
| Variable | Default | Description |
|---|---|---|
| `OPENAI_MODEL` | `gpt-4` | OpenAI model name (each agent overrides this with `gpt-5-mini-2025-08-07` directly in code; this env var is not used by agents) |
| `USE_AI_AGENTS` | `true` | Enable/disable AI agents |
| `FALLBACK_TO_DIRECT` | `true` | Fall back to direct processing if agents fail |

### Rate Limiting
| Variable | Default | Description |
|---|---|---|
| `OPENAI_REQUESTS_PER_MINUTE` | `15` | OpenAI rate limit |
| `OPENAI_REQUESTS_PER_SECOND` | `2` | OpenAI burst limit |
| `OPENAI_MAX_RETRIES` | `3` | Max retries on rate limit errors |
| `OPENAI_BASE_RETRY_DELAY` | `1.0` | Base delay (seconds) for exponential backoff |

### Batch Processing
| Variable | Default | Description |
|---|---|---|
| `BATCH_SIZE` | `25` | Keywords per processing batch |
| `MAX_CONCURRENT_BATCHES` | `3` | Concurrent batch limit |
| `BATCH_TIMEOUT` | `120` | Seconds per batch |
| `KEYWORD_BATCH_SIZE` | `500` | Keywords per root extraction batch |

### Storage
| Variable | Default | Description |
|---|---|---|
| `UPSTASH_REDIS_URL` | — | Upstash Redis REST URL |
| `UPSTASH_REDIS_TOKEN` | — | Upstash Redis auth token |
| `USE_REDIS_FOR_JOBS` | `true` | Use Redis for job storage |
| `JOB_TTL_HOURS` | `24` | Hours before job data expires |

### Scraping
| Variable | Default | Description |
|---|---|---|
| `SCRAPER_PROXY` | — | Single proxy URL |
| `SCRAPER_PROXY_LIST` | — | Comma-separated proxy list |
| `BRIGHT_DATA_HOST/USER/PASS` | — | Bright Data proxy credentials |
| `SMARTPROXY_HOST/USER/PASS` | — | Smartproxy credentials |
| `OXYLABS_HOST/USER/PASS` | — | Oxylabs credentials |
| `SERPAPI_API_KEY` | — | SerpAPI key for SERP API scraper |

### CORS
| Variable | Default | Description |
|---|---|---|
| `CORS_ORIGINS` | `http://localhost:3000,...` | Comma-separated allowed origins |

---

## Technology Stack

### Backend
| Technology | Version | Role |
|---|---|---|
| Python | ≥3.13 | Runtime |
| FastAPI | ≥0.116.1 | Web framework + REST API |
| OpenAI Agents SDK | ≥0.2.9 | AI agent framework |
| Scrapy | ≥2.13.3 | Amazon web scraping |
| Playwright | ≥1.40.0 | Headless browser scraping |
| BeautifulSoup4 | ≥4.12.0 | HTML parsing |
| CloudScraper | ≥1.2.71 | Cloudflare bypass |
| Upstash Redis | ≥0.15.0 | Background job storage |
| Pydantic | (via FastAPI) | Data validation and schemas |
| uvicorn | (via FastAPI) | ASGI server |
| uv | — | Package manager |
| pytest | ≥8.4.1 | Testing |

### Frontend
| Technology | Role |
|---|---|
| Next.js (App Router) | React framework |
| TypeScript | Type-safe JavaScript |
| Tailwind CSS | Utility-first CSS |
| shadcn/ui | UI component library |
| Lucide React | Icons |

---

## Deployment

### Backend – Render
**File:** `backend/render.yaml`

The backend is configured for deployment on [Render](https://render.com):
- Start command: `uvicorn app.main:app --host 0.0.0.0 --port 8000 --timeout-keep-alive 18000`
- Dev command: same with `--reload`
- Long keep-alive timeout (18,000 seconds) to support lengthy pipeline runs.

Scripts:
- `backend/scripts/deploy.sh` – deployment helper
- `backend/scripts/health_check.sh` – endpoint health check

### Frontend – Vercel
The Next.js frontend is deployed to [Vercel](https://vercel.com). Set `NEXT_PUBLIC_API_URL` to the Render backend URL.

### Local Development

```bash
# Backend (requires Python 3.13+)
cd backend
uv sync
cp .env.example .env   # add OPENAI_API_KEY
uv run dev

# Frontend
cd frontend
npm install
cp .env.local.example .env.local   # set NEXT_PUBLIC_API_URL
npm run dev
```

---

## Input Data Format (Helium 10 Cerebro CSV)

The pipeline accepts **Helium 10 Cerebro** keyword research exports. Expected CSV columns:

| Column | Description |
|---|---|
| `Keyword Phrase` | The search keyword |
| `Search Volume` | Monthly Amazon search volume |
| `Relevancy` | Helium 10 relevancy score |
| `Title Density` | How many top results include the keyword in their title |
| `CPR` | Cerebro Product Rank (estimated giveaways needed to rank) |
| `Cerebro IQ Score` | Helium 10 opportunity score |
| `Competing Products` | Number of products competing for the keyword |
| `B0XXXXXXXXX` (columns) | Organic rank of each ASIN for the keyword |

Two files are accepted:
- **Revenue CSV** – keywords from top revenue-generating competitor ASINs
- **Design CSV** – keywords from top design-variant competitor ASINs
