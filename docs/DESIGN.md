# Good News AI — Design Doc

**Status:** Draft · **Last updated:** 2026-11-19

## 1. Overview

Good News AI is a news board that shows only stories most people would consider good news: rescues, scientific breakthroughs, environmental progress, community wins. Articles are collected automatically from public news feeds. An LLM classifies each one, and only stories that pass the filter appear on the site, each with a short summary and a one-line "why this is good news" explanation.

### Goals

- Pull articles from many news sources automatically and continuously.
- Classify each article as good or not good news, with a score, a category and a summary.
- Group duplicate coverage of the same story from different outlets.
- Serve a fast, paginated, filterable feed through a public REST API.
- Run in production with tests, CI, observability and a live deployment. This is also a portfolio project.

### Non-goals (for v1)

- Hosting full article text. We store the headline, snippet and link, and send readers to the original publisher.
- User accounts, comments or personalization (see §10).
- Native mobile apps.

## 2. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | **Python 3.12+** | Widely used for backend work; the author's strongest language; good LLM tooling |
| Packaging | `uv` | Fast dependency and virtualenv management with a lockfile |
| Web framework | **FastAPI** | Async, type-hinted, generates OpenAPI docs automatically, Pydantic validation |
| Database | **PostgreSQL 16** | The industry default; `pg_trgm` for title-similarity dedup; full-text search built in |
| DB access | **SQLAlchemy 2.0** (async, with `asyncpg`) | The standard Python ORM / query builder; typed models |
| Migrations | **Alembic** | The standard migration tool for SQLAlchemy |
| Queue / scheduler | **Redis** + **Celery** (+ Celery Beat) | The most widely used Python job queue: retries, scheduled tasks, routing |
| Feed fetching | `httpx` + `feedparser` | Async HTTP client; robust RSS/Atom parsing |
| Cache | Redis | Caching feed responses, rate limiting |
| LLM | **Anthropic Claude API** (official `anthropic` Python SDK) | Classification with structured output parsed directly into a Pydantic model (§5) |
| Frontend | **Next.js** (TypeScript) | Server-side rendering is good for SEO on a news site; consumes the FastAPI API |
| Local dev | Docker Compose | Postgres, Redis, API and worker started with one command |
| CI | GitHub Actions | Lint, test (with a real Postgres service), build |
| Deploy | Fly.io (or Railway / Render) | Cheap, supports Postgres, Redis and multiple processes |
| Code quality | `ruff` (lint + format) + `mypy` | Consistent style and type checking |
| Observability | `structlog` (JSON logs) + `prometheus-client` | Visibility into the pipeline and the API |

## 3. Architecture

```
            ┌────────────────────┐
            │ Scheduler (Beat)   │  every ~15 min, one job per source
            └─────────┬──────────┘
                      ▼
┌──────────┐   ┌──────────────┐   ┌──────────────┐
│ RSS / API│──►│ fetch_source │──►│ dedupe +     │── new article ──┐
│ sources  │   │ worker       │   │ store (raw)  │                 │
└──────────┘   └──────────────┘   └──────────────┘                 ▼
                                                        ┌────────────────────┐
                                                        │ classify_article   │
                                                        │ worker (Claude)    │
                                                        └─────────┬──────────┘
                                                                  ▼
                                                        ┌────────────────────┐
                                                        │ PostgreSQL         │
                                                        └─────────┬──────────┘
                                                                  ▼
                              ┌──────────┐   cache    ┌────────────────────┐
                              │ Next.js  │◄──────────►│ FastAPI REST API   │◄── Redis
                              │ frontend │            └────────────────────┘
                              └──────────┘
```

**Processes:**

1. **`api`** — FastAPI app served by `uvicorn`; reads from Postgres, with Redis caching.
2. **`worker`** — Celery worker running the `fetch_source` and `classify_article` tasks.
3. **`beat`** — Celery Beat, which enqueues scheduled fetches.

All three run from the same Python package (`app/`) and share its modules; they're just different entrypoints in Docker Compose.

## 4. Ingestion pipeline

1. The **scheduler** enqueues a `fetch_source` job for each enabled source at that source's interval.
2. **`fetch_source`**:
   - Fetches the feed with a conditional GET (`ETag` / `If-Modified-Since`) to avoid re-downloading unchanged feeds.
   - Parses the items: title, link, snippet, published time, image URL.
   - **Normalizes the URL**: strips tracking parameters (`utm_*`, `fbclid`), lowercases the host and removes the trailing slash.
   - Inserts with `ON CONFLICT (url_hash) DO NOTHING`, which is the first dedup layer.
   - Enqueues `classify_article` for each newly inserted row.
3. **Story clustering** is the second dedup layer. Before classifying, compare the new title against articles from the last 48 hours using `pg_trgm` similarity. Above a threshold (start at 0.6 and tune it), attach the article to the existing `story` and skip classification. This catches the same story covered by 20 outlets.
4. **Politeness:** per-source rate limits, a custom User-Agent, and respect for `robots.txt` and feed terms of use.

**Initial sources (about 10):** major wire and general outlets plus dedicated science and environment feeds. The list lives in a seed migration so it's versioned.

## 5. Classification (the core)

### Approach

Each new article (title + snippet + source) goes to Claude with a fixed rubric. The response is constrained to a JSON schema with **structured outputs**. In Python, the SDK's `client.messages.parse()` validates the response straight into a Pydantic `Classification` model, so the worker never has to parse free-form text.

### Output schema

```json
{
  "is_good_news": true,
  "score": 0.87,
  "category": "animals",
  "summary": "Firefighters in Ohio rescued a kitten stuck 40 feet up an oak tree.",
  "reason": "A positive, low-stakes rescue story with a happy outcome."
}
```

- `score`: 0–1 confidence that a typical reader would consider this good news.
- `category`: enum — `animals`, `science`, `health`, `environment`, `community`, `technology`, `economy`, `culture`, `other`.
- An article is **published** when `is_good_news && score >= PUBLISH_THRESHOLD` (start at 0.7).
- Articles in a borderline band (0.5–0.7) are kept with `status = needs_review` for a future admin queue.

### Rubric (kept in the system prompt)

- **Good news:** a positive outcome for people, animals or the planet; progress, recovery, kindness, discovery.
- **Not good news:** tragedy, violence, disaster, or politics framed as conflict, even when the article contains a silver lining ("survives brutal attack").
- **Watch for:** positive facts in neutral language ("unemployment falls to record low" is good news); clickbait; sponsored or promotional content (not good news).

### Model and API settings

- **Model:** `claude-opus-5-5` (current default), with `output_config.effort: "low"`, since classification doesn't need deep reasoning. Once an eval set exists (below), measure whether a smaller model such as `claude-haiku-5-5` matches its accuracy at lower cost, and decide then.
- **Refusal handling:** check `stop_reason` before reading content. Enable server-side fallbacks (`fallbacks: "default"`, beta `server-side-fallback-2026-07-01`).
- **Prompt caching:** the system prompt and rubric are identical on every call, so cache them.
- **Batch API (later):** classification isn't latency-sensitive. The Message Batches API costs 50% less and fits a "classify everything fetched in the last N minutes" job.
- **Model and prompt version tracking:** store `model` and `prompt_version` on every classification so results can be re-run and compared.

### Evaluation

Hand-label about 200 real headlines as good or not good, including tricky cases. Keep them in `testdata/eval/`. A `make eval` command runs the classifier on them and reports precision and recall. **Precision matters most**, because one bad story on a good-news site hurts more than missing a good one.

### Failure handling

- Celery task retries with exponential backoff (`autoretry_for`, `retry_backoff`) on 429 and 5xx errors.
- After N failed attempts, the job moves to the dead-letter queue and the article is marked `status = failed`.
- A per-minute token-budget guard prevents runaway spend.

## 6. Data model

```sql
sources (
  id            bigserial PK,
  name          text NOT NULL,
  feed_url      text UNIQUE NOT NULL,
  homepage_url  text,
  enabled       boolean DEFAULT true,
  fetch_interval_minutes int DEFAULT 15,
  etag          text,
  last_modified text,
  last_fetched_at timestamptz
)

stories (                       -- cluster of articles about the same event
  id            bigserial PK,
  primary_article_id bigint,    -- the one we display
  created_at    timestamptz DEFAULT now()
)

articles (
  id            bigserial PK,
  source_id     bigint REFERENCES sources,
  story_id      bigint REFERENCES stories,
  url           text NOT NULL,
  url_hash      bytea UNIQUE NOT NULL,
  title         text NOT NULL,
  snippet       text,
  image_url     text,
  published_at  timestamptz,
  fetched_at    timestamptz DEFAULT now(),
  status        text NOT NULL,  -- pending | published | rejected | needs_review | failed
  search_vector tsvector        -- generated from title + snippet
)

classifications (
  article_id    bigint PK REFERENCES articles,
  is_good_news  boolean,
  score         real,
  category      text,
  summary       text,
  reason        text,
  model         text,
  prompt_version text,
  input_tokens  int,
  output_tokens int,
  created_at    timestamptz DEFAULT now()
)
```

**Indexes:** `articles (status, published_at DESC)` for the feed; `classifications (category)`; GIN on `search_vector`; GIN `gin_trgm_ops` on `title` for clustering.

## 7. API (v1)

All responses are JSON. Lists use **cursor pagination** (`?cursor=` plus `next_cursor` in the response) rather than offset, so pages stay stable while new stories arrive.

| Method | Path | Description |
|---|---|---|
| GET | `/v1/articles` | Published feed. Query: `category`, `cursor`, `limit` (≤50) |
| GET | `/v1/articles/{id}` | One article with its classification and related coverage |
| GET | `/v1/categories` | Categories with article counts |
| GET | `/v1/search?q=` | Full-text search over published articles |
| GET | `/healthz` | Liveness |
| GET | `/readyz` | Readiness (DB + Redis reachable) |
| GET | `/metrics` | Prometheus (internal only) |

**Caching:** feed and category responses are cached in Redis for 60 seconds, and the cache is invalidated when new articles are published. Responses include `ETag` / `Cache-Control` headers.

**Rate limiting:** a per-IP token bucket in Redis; returns 429 with `Retry-After`.

## 8. Project layout

```
good-news-ai/
├── app/
│   ├── main.py         # FastAPI app entrypoint
│   ├── config.py       # env-based settings (pydantic-settings)
│   ├── db/             # SQLAlchemy models, session setup
│   ├── api/            # routers, schemas, middleware (logging, rate limit, CORS)
│   ├── ingest/         # feed fetching, parsing, URL normalization, clustering
│   ├── classify/       # Claude client, prompt, Pydantic schema, eval harness
│   ├── tasks/          # Celery app, tasks, Beat schedule
│   └── cache.py        # Redis helpers
├── migrations/         # Alembic
├── tests/
├── web/                # Next.js frontend
├── testdata/eval/      # labeled headlines for classifier eval
├── docs/DESIGN.md
├── pyproject.toml      # uv-managed
├── docker-compose.yml
├── Makefile
└── .github/workflows/ci.yml
```

## 9. Testing and quality

- **Framework:** `pytest` (+ `pytest-asyncio`), FastAPI's test client for endpoints.
- **Unit tests:** URL normalization, feed parsing (using saved feed fixtures), threshold logic, API routes.
- **Integration tests:** run against a real Postgres (Docker Compose locally, a service container in CI). No DB mocking.
- **Classifier:** in unit tests, the Claude client sits behind a small protocol/class and is faked. The real model is measured with `make eval` (§5).
- **Load test:** k6 against `/v1/articles`, with before/after numbers for caching recorded in the README.
- **CI:** `ruff check`, `ruff format --check`, `mypy`, `pytest`, frontend build.

## 10. Milestones

**M1 — Pipeline skeleton**
Docker Compose (Postgres + Redis), migrations, sources seed, `fetch_source` worker, articles stored with URL dedup.

**M2 — Classification**
Claude classification worker, structured output, thresholds, eval set and `make eval`.

**M3 — API**
Feed, article, categories and search endpoints; cursor pagination; Redis caching; rate limiting; health checks.

**M4 — Frontend + deploy**
Next.js feed with category filters; deployed to Fly.io; CI on every push.

**M5 — Polish (resume-ready)**
Story clustering, Prometheus metrics, a k6 load test with results, a README with an architecture diagram and design tradeoffs.

**Later ideas:** user accounts and saved stories, a daily email digest (another scheduled job), upvotes feeding into ranking, an admin review queue for `needs_review`, Batch API for classification, RSS output of the good-news feed itself.

## 11. Open questions

- Which news sources allow this use? Check each feed's terms of use.
- Publish threshold: tune with the eval set.
- Should stories be ranked by recency only, or also by score and how many outlets covered them?
- Hosting budget for LLM calls: estimate after measuring average tokens per article.
