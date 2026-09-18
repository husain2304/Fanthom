# AGENTS.md — Guidelines for Fathom Sync Tool

## Project Overview
`fathom-sync` is a production-grade CLI tool and service that synchronizes meeting recordings, AI summaries, action items, and metadata from Fathom into Google Sheets and optionally Notion databases.

## Technology Stack & Standards
- **Python**: 3.11+ managed via `uv`.
- **CLI & Output**: `typer` + `rich` for rich console outputs, progress bars, and formatted tables.
- **HTTP Client & Resilience**: `httpx` with `tenacity` for retries, exponential backoff, and an adaptive token-bucket rate limiter tuned for Fathom API constraints (heavy request limits).
- **Data Models**: `pydantic` v2 for strict request/response parsing and validation.
- **Integrations**: `google-api-python-client` & `google-auth-oauthlib` for Google Sheets, `notion-client` for Notion.
- **Secret & State Storage**: `keyring` for OS keychain secret storage with `.env` fallback for headless environments; `~/.config/fathom-sync/` for non-secret config and checkpoints.
- **Testing**: `pytest`, `pytest-asyncio`, `respx` for mock Fathom REST & Webhook endpoints.

## Core Design Principles
1. **Idempotency**: Running backfill or incremental sync multiple times against the same target must produce zero duplicate rows.
2. **Error Isolation**: Individual malformed or incomplete meeting records must not abort a batch run; failures are logged and reported in a summary table.
3. **Resilience & Checkpointing**: Long-running backfills save state to `state.json` after every page to allow resume on interrupt.
4. **Rate Limit Respect**: Fathom heavy requests (`include_summary=true`) are capped at 30 req/min (and up to 5 req/min when throttled); the client dynamically throttles using `RateLimit-*` and `Retry-After` headers.
5. **No Secrets in Repo/Logs**: Never log API keys, OAuth tokens, full meeting transcripts, or customer summaries.
