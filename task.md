# Task Checklist: Production-Quality Fathom → Google Sheets/Notion Sync Tool

- [ ] **Phase 0: Planning & Clarification**
  - [x] Initial workspace inspection and tool availability verification (`uv`, Python 3.11)
  - [x] Research Fathom REST API specifications, rate limits, webhook signatures, and data contracts
  - [x] Create `AGENTS.md` and `task.md`
  - [ ] Gather user preferences (Timezone, recording count, destinations, webhook vs poller, topic taxonomy)
  - [ ] Write comprehensive `implementation_plan.md` and obtain approval

- [ ] **Phase 1: Project Setup & Core Foundation**
  - [ ] Initialize Python package structure (`pyproject.toml`, `uv.lock`) with dependencies (`httpx`, `pydantic`, `typer`, `rich`, `google-api-python-client`, `google-auth-oauthlib`, `notion-client`, `keyring`, `tenacity`, `uvicorn`, `fastapi`, `tomlkit`, `respx`, `pytest`, `pytest-asyncio`)
  - [ ] Implement `fathom_sync/config.py` (configuration schema, TOML reader/writer, `keyring` secure store, `--use-env` fallback)
  - [ ] Implement `fathom_sync/state.py` (atomic checkpointing, `state.json`, seen-set, classification cache)
  - [ ] Implement `topics.toml` and `fathom_sync/classify.py` (two-tier classification: rules regex/keywords + optional LLM fallback)

- [ ] **Phase 2: Fathom API Client & Resilience Engine**
  - [ ] Implement `fathom_sync/fathom/models.py` (Pydantic schema for meetings, summaries, action items, invitees, webhooks)
  - [ ] Implement `fathom_sync/fathom/client.py` (Thin `httpx` client, adaptive token bucket rate limiter, `RateLimit-*` header tracking, 429 backoff with `Retry-After` + jitter, cursor pagination)
  - [ ] Implement `fathom_sync/fathom/webhooks.py` (Standard Webhooks HMAC-SHA256 signature verification over `{id}.{timestamp}.{raw_body}`, 5-min timestamp window)

- [ ] **Phase 3: Destinations Architecture (Google Sheets & Notion)**
  - [ ] Implement `fathom_sync/destinations/base.py` (`Destination` protocol for pluggable targets)
  - [ ] Implement `fathom_sync/destinations/sheets.py`:
    - [ ] Tab `Recordings` schema (20+ columns, frozen headers, filter view, date formatting, CLIP text wrapping, hyperlinked share URLs)
    - [ ] 45k cell character truncation guard for summaries
    - [ ] Tab `By Topic` pivot/summary formulas (recordings count, total duration min per topic per month)
    - [ ] Tab `Sync Log` run history tracker
    - [ ] Idempotent batch upsert via `spreadsheets.values.batchUpdate` and formatting via `spreadsheets.batchUpdate`
    - [ ] Automated spreadsheet scaffolding / creation helper
  - [ ] Implement `fathom_sync/destinations/notion.py`:
    - [ ] Notion database schema builder (`fathom-sync notion scaffold`)
    - [ ] Markdown to Notion blocks converter (headings, bullet lists, paragraphs, to_do action items)
    - [ ] 2,000 char per rich_text limit handling and 100 blocks per request chunking
    - [ ] Idempotent page upsert based on Recording ID

- [ ] **Phase 4: CLI Commands & First-Run Flow**
  - [ ] Implement `fathom-sync init`:
    - [ ] Interactive Fathom API key setup with live validation (`GET /meetings?limit=1`) and user display
    - [ ] Google Sheets OAuth InstalledAppFlow with browser consent, auto token refresh, and spreadsheet creation/linking
    - [ ] Optional Notion token & database ID setup
    - [ ] `GET /meeting_types` integration to seed initial `topics.toml`
  - [ ] Implement `fathom-sync doctor`:
    - [ ] Validate Fathom credentials, Google OAuth tokens, Drive/Sheets scopes, Notion permissions, and local state
  - [ ] Implement `fathom-sync backfill`:
    - [ ] Resilient pagination, checkpoint saving after every page, `--since`, `--limit`, `--dry-run`, `--slow`, `--classify-llm`
    - [ ] Rich live progress bar, ETA, and skipped record isolation report

- [ ] **Phase 5: Ongoing Pipeline (Incremental Sync, Webhook Server, CI/CD)**
  - [ ] Implement `fathom-sync sync`:
    - [ ] Incremental poller using `created_after` with 24-hour overlap window and watermark advancement
  - [ ] Implement `fathom-sync serve`:
    - [ ] FastAPI webhook receiver endpoint (`POST /webhooks/fathom`), HMAC verification, webhook-id deduplication, background task async write
  - [ ] Implement `fathom-sync webhook register`:
    - [ ] Register public URL with Fathom API, save returned secret and webhook ID
  - [ ] Generate `.github/workflows/sync.yml` for automated scheduled runs
  - [ ] Document crontab and launchd scheduling examples

- [ ] **Phase 6: Testing & Verification**
  - [ ] Unit & integration tests using `pytest` and `respx`:
    - [ ] Pagination & cursor handling
    - [ ] 429 rate limit + Retry-After + jitter backoff
    - [ ] Checkpoint & resume on crash
    - [ ] Idempotent upsert (verifying 0 duplicate rows on rerun)
    - [ ] Long cell content truncation at 45k chars
    - [ ] Webhook signature verification (valid, tampered, expired timestamp)
    - [ ] Notion markdown to blocks conversion & chunking
  - [ ] Live verification:
    - [ ] Run `fathom-sync doctor`
    - [ ] Run `fathom-sync backfill --limit 30` against real account
    - [ ] Inspect Google Sheet via browser, check clickable links, CLIP wrapping, formatting, and capture screenshot artifacts
    - [ ] Re-run backfill to verify 0 duplicate rows
    - [ ] Run full `pytest` suite
  - [ ] Generate comprehensive `walkthrough.md` and user documentation `README.md`
