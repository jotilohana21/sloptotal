# SlopTotal Architecture

## Layout

```
sloptotal/
├── app/          Backend (Python/FastAPI)
├── web/          Web UI (Jinja2 + vanilla JS), served by the backend
├── tests/        Unit tests and the tests/eval/ accuracy harness
└── scripts/      End-to-end smoke test, model drift check
```

The Chrome extension lives in its own repository,
[pablocaeg/sloptotal-extension](https://github.com/pablocaeg/sloptotal-extension),
and talks to the JSON API.

### Backend (`app/`)

FastAPI application with hardware-aware auto-configuration.

**Entry point:** `app/main.py` -- app factory, lifespan, CORS, static mounts, router includes (~120 lines).

**Routes:**
- `app/routes/web.py` -- HTML page routes (`/`, `/analyze`, `/report/{id}`, `/api/stream/{id}`, `/api/web/analyze`)
- `app/routes/api.py` -- JSON API routes (`/api/quick-score`, `/api/paragraph-score`, `/api/scan/snippets`, `/api/analyze`, `/api/engines`, etc.)
- `app/routes/queue.py` -- Queue management (`/api/queue/status`, `/api/queue/ticket/{id}`)

**Core:**
- `app/analyzer.py` -- Engine orchestration, scoring calibration, batch inference
- `app/engines/` -- 23 detection engines, each inheriting from `BaseEngine`
- `app/config.py` -- Engine weights, scoring thresholds, paths
- `app/schemas.py` -- Pydantic models for requests/responses
- `app/autoconfig.py` -- Hardware detection (CPU/GPU/RAM), profile selection (lite/standard/performance)
- `app/model_pool.py` -- Thread-safe model replica pools for hot engines
- `app/queue_manager.py` -- Request queuing with backpressure
- `app/database.py` -- SQLite async storage for reports
- `app/cache.py` -- Content hash-based caching
- `app/scraper.py` -- URL fetching (rejects private/metadata addresses on every redirect hop) and main-content extraction
- `app/site_fingerprints.py` -- AI app builder detection (Lovable, v0, Bolt, Base44, Replit, Same); see [docs/SITE_FINGERPRINTS.md](SITE_FINGERPRINTS.md)
- `app/documents.py` -- Text extraction from uploaded PDF / DOCX / TXT

### Web Frontend (`web/`)

Server-rendered Jinja2 templates with vanilla JavaScript.

- `web/templates/base.html` -- Layout, header, ticker, footer
- `web/templates/index.html` -- Homepage with dynamic engine grid
- `web/templates/report.html` -- Live report page
- `web/static/css/style.css` -- All styles (forensic lab aesthetic)
- `web/static/js/app.js` -- Tab switching, queue-aware form submission, ticker
- `web/static/js/report.js` -- SSE streaming, gauge animation, engine row updates

---

## Engine Architecture

Each engine inherits from `BaseEngine` and provides:
- `name`, `description` -- Display metadata
- `code` -- 2-letter short code (e.g. "FS" for Fakespot)
- `engine_type` -- Category: neural, statistical, linguistic, embedding, classifier
- `url` -- Link to model/paper
- `analyze(text)` -- Returns `EngineResult` with score, verdict, details
- `analyze_batch(texts)` -- (Optional) Batch inference for snippet scanning

### Concurrency Model

```
Incoming request
    ├── Snippet endpoint ──→ _snippet_executor (4 workers)
    ├── Quick-score      ──→ _snippet_executor (shared)
    └── Full analysis    ──→ _full_executor (N workers, N = usable cores)
```

### Scoring Calibration

The final score is produced by a logistic fit with non-negative weights trained on human-labelled examples. The fit lives in `app/calibration.py` and the fitted parameters in `app/calibration.json`.

Key properties of the current model:

- **Logistic fit with non-negative weights** — each signal contributes positively to the slop score; no signal can cancel another.
- **Short-text shrink** — documents shorter than a minimum token count are shrunk toward the prior to avoid overconfident scores on thin evidence.
- **Per-language offset and support** — languages with enough training data carry a per-language intercept offset; unsupported languages fall back to the language-agnostic baseline.
- **Confidence rule** — a score is marked high-confidence only when the input length and signal strength both exceed per-model thresholds.

Verdict bands (`SCORE_CLEAN` ... `SCORE_LIKELY_AI`) live only in `config.py`; the report page receives them from the template. See the README's Scoring section for a worked example and parameter details.

---

## Data Flow

### Full Analysis (Web)
```
Browser POST /api/web/analyze
    → QueueManager checks capacity
    → start_analysis() creates report in SQLite
    → 23 engines run in parallel via _full_executor
    → Each engine callback: insert result → recalculate score → push to asyncio.Queue
    → Browser receives report_id, opens /report/{id}
    → EventSource /api/stream/{id} yields SSE events as engines complete
    → Final SSE event: {done: true}
```

### Site check
```
Browser / client POST /api/scan/site {url}
    → fetch_page(): guarded fetch, returns HTML, headers, final URL
    → detect_builders(): host, header and HTML fingerprints per builder
    → optional quick text score of the extracted main content
    → {site: {builders, generator, verdict}, text: {...} | null}
```

### Snippet Scan (Chrome Extension)
```
Extension POST /api/scan/snippets [{id, text, url}, ...]
    → QueueManager checks capacity
    → 3 engines (BERT-RAID, E5, Fakespot) run batched inference
    → Each engine: single forward pass for all N texts
    → Fakespot-anchored calibration per snippet
    → Return [{id, score, indicator, confidence}, ...]
```
