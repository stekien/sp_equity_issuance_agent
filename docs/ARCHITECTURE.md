# System Architecture, State Management, and Time Budgeting

## 1. Runtime Workspace Layout

Keep all scripts, caches, databases, and staging artifacts under `work/` so the repository root stays clean and `output/` holds only final deliverables:

- `work/.venv/` — Local Python virtual environment.
- `work/run_meta.json` — Run metadata: `start_time_utc`, `target_stop_minutes` (240), `phase6_cutoff_minutes` (210), `run_date_utc`, software versions.
- `work/STATE.md` — Human- and agent-readable checkpoint updated after every phase and batch (current phase, elapsed time, companies/filings completed, open anomalies, next exact step).
- `work/db/pit_equity.sqlite` — Authoritative relational store (WAL mode enabled) containing:
  - `constituents` — Historical S&P 500 membership intervals, CIKs, share classes, GICS/financial flag, index weights, and primary-source citations.
  - `filings` — Discovered SEC filings per CIK (`accession_number`, `form`, `filing_date`, `acceptance_datetime`, `primary_doc`, `parse_status`, `tier`).
  - `events` — Normalized bi-temporal equity events matching `docs/SCHEMA_AND_RULES.md`.
  - `shares_outstanding` — Quarterly/filing-level shares outstanding and stock split events for balance-sheet reconciliation.
  - `watermarks` — Per-CIK `last_acceptance_datetime`, `last_accession_number`, and `daily_index_last_date` for incremental re-runs.
  - `qa_checks` — Stored results of automated validation, share-count reconciliation, and benchmark comparisons.
- `work/cache/` — Local disk cache for all HTTP responses (EDGAR JSON/HTML/XML, FRED/Fed data, announcements), keyed cleanly by CIK/accession or URL hash. Never re-download an already-cached resource.
- `work/src/` — Modular Python package built by the agent (EDGAR client, membership builder, XBRL extractor, HTML/filing parsers, reconciler, publisher, chart generator, incremental CLI runner).
- `work/staging_output/` — Staging directory used for atomic `output/` validation before replacing files in `output/`.
- `output/` — Final deliverables: `events.csv`, `README.md`, `net_equity_issuance.svg`, `net_equity_issuance.png`, `COVERAGE.md`.

## 2. Phase 0 Environment Setup (Minutes 0–10)

1. Verify `$SEC_USER_AGENT` is set and non-empty. If missing, stop immediately and report that SEC EDGAR access requires `SEC_USER_AGENT`.
2. Record `start_time_utc` and today's UTC date (`as_of_date`) in `work/run_meta.json`.
3. Create `work/.venv` and install lightweight, reliable libraries (e.g., `httpx` or `requests`, `lxml`, `beautifulsoup4`, `pandas`, `matplotlib`, `numpy`). Prefer `uv pip install` if `uv` is available, otherwise `pip install`.
4. Initialize `work/db/pit_equity.sqlite` and build a single centralized EDGAR HTTP client module with:
   - Mandatory `User-Agent` header from `$SEC_USER_AGENT` and `Accept-Encoding: gzip, deflate`.
   - Persistent disk cache under `work/cache/`.
   - Strict global rate limiting at **≤ 8 requests/second** (using a file-locked or SQLite timestamp token bucket if multiple processes exist, or running network fetches strictly through a single process while parsing locally in parallel).
   - Exponential backoff on HTTP `429` and `5xx` (with jitter, max 5 retries); log persistent `404`/`403` without crashing the batch.

## 3. Time-Budget Discipline & Hard-Stop Safety

Because the run ends abruptly at **240 minutes**, enforce strict time gates:

- Check elapsed minutes before starting every new phase or batch.
- **Milestone 1 Gate (~55 min)**: You MUST have published a complete, validated `output/` directory (all 4 required deliverables) from Tier 1 full-universe XBRL + PIT membership before investing deep time in Tier 2 HTML parsing.
- **Batch Refresh Rule**: During Phases 3 and 4, work in batches (e.g., 75–100 companies per batch in Phase 3; one historical year per batch in Phase 4). After each batch, commit to SQLite, run validation checks, atomically refresh `output/`, and update `work/STATE.md`.
- **Historical Cutoff Gate (190 min)**: Stop Phase 4 historical backfill if elapsed time reaches 190 minutes, even if earlier years remain incomplete; document the exact cutoff cleanly in `COVERAGE.md`.
- **Phase 6 Hard Gate (210 min / 3h30m)**: Immediately stop all new ingestion and enter Phase 6 (adversarial QA, external benchmarking, chart polish, final `COVERAGE.md`, final atomic publish).

## 4. Atomic Publishing Protocol

Never write partial or unvalidated files directly into `output/`:
1. Generate `events.csv`, `README.md`, `net_equity_issuance.svg`, `net_equity_issuance.png`, and `COVERAGE.md` into `work/staging_output/`.
2. Run your automated validator against `work/staging_output/` (schema columns, non-empty checks, zero duplicate keys, zero double-counting, valid timestamps, image files non-empty and readable).
3. Only after validation passes, atomically move/replace files from `work/staging_output/` into `output/`.

## 5. Continuous / Incremental Operation Design

Design `work/src/` so this system is not a one-off script, but a maintainable point-in-time pipeline that can keep running daily or hourly:
- Provide a clean CLI entry point under `work/src/` supporting `--mode full` and `--mode incremental`.
- In `--mode incremental`:
  1. Poll SEC EDGAR daily index files (`https://www.sec.gov/Archives/edgar/daily-index/{YYYY}/QTR{Q}/master.{YYYYMMDD}.idx`) or each active constituent's `submissions/CIK{cik10}.json` for filings with `acceptanceDateTime` newer than the stored watermark in `watermarks`.
  2. Check for new ETF `N-PORT-P` filings and S&P index announcements to update `constituents`.
  3. Process only newly filed accession numbers, append new rows (preserving prior revisions with `is_revision = true` when a comparative/amended figure changes), advance watermarks only after SQLite commit succeeds, and re-publish `output/` atomically.
- Document exact commands and operational notes for continuous runs in `output/README.md`.
