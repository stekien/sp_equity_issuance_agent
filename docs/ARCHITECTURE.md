# System Architecture, State Management, Time Budgeting & Incremental Design

## 1. Runtime Workspace Layout

Keep all scripts, caches, databases, logs, and staging directories under `work/` so the repository root stays clean and `output/` holds only validated deliverables:

- `work/.venv/` — Local **Python 3.14** virtual environment.
- `work/run_meta.json` — Run clock & scaled gates (`start_time_utc`, `run_budget_minutes`, `historical_cutoff_min`, `phase6_cutoff_min`, `freeze_cutoff_min`, `run_date_utc`, `python_version`).
- `work/STATE.md` — Compact checkpoint updated after every phase and batch (current phase, elapsed minutes, timing table excerpt, membership verified window, company/filing coverage counts, open QA items, exact next step).
- `work/logs/` — Detailed logs, parser diagnostics, and audit transcripts (never dump > 30 lines of verbose logs to terminal stdout).
- `work/staging_membership/` — Atomic per-year JSON handoff files (`year_2025.json`, `year_2024.json`, …, `year_2021.json`) written by the background `membership-researcher` subagent.
- `work/staging_benchmarks/` — Structured verification briefs written by `benchmark-researcher` and `market-verifier` subagents.
- `work/db/pit_equity.sqlite` — Single-writer SQLite database (WAL mode, commit after every small chunk of ~25 CIKs or every sub-step; **only the main agent writes to SQLite**):
  - `constituents` — PIT S&P 500 membership intervals, CIKs, share classes, SIC/`is_financial`, N-PORT weights & prices, primary-source citations, and `membership_verified_back_to`.
  - `filings` — Discovered SEC filings (`accession_number`, `form`, `filing_date`, `acceptance_datetime`, `primary_doc`, `parse_status`, `tier`).
  - `raw_extractions` — Raw parsed monthly buyback tables, classified `424B` covers, `8-K`/`SC TO-I` candidates, and Tier 3 analyst extracts stored **separately** from final inclusion/acceptance decisions so rule adjustments re-run in seconds without re-parsing HTML!
  - `events` — Normalized bi-temporal equity events matching `docs/SCHEMA_AND_RULES.md`.
  - `shares_outstanding` — Balance-sheet, cover-page, and dimensional per-class share counts + stock-split evidence for share-count reconciliation and Acadian Eq. (1) computation.
  - `watermarks` — Per-CIK `last_acceptance_datetime`, `last_accession_number`, `daily_index_last_date`, and `sp_release_last_date` for incremental operation.
  - `phase_timings` — Structured telemetry (`phase`, `step_or_batch`, `start_utc`, `end_utc`, `duration_sec`, `items_processed`, `notes`).
  - `qa_checks` — Automated validator results, publish-to-publish regression diffs, share-count reconciliation metrics, and audit findings.
- `work/cache/` — Persistent **gzip-compressed** disk cache (`gzip` stdlib — cuts HTML/XML/JSON size ~80–85% so multi-year runs never fill a Linux VM disk!). Fetch primary filing documents only (never bulk-download binary/image exhibits). Check `shutil.disk_usage` before each historical HTML wave; if free disk drops below **5 GB**, stop downloading raw HTML archives (continue tiny Tier 1 XBRL `companyfacts` only) and log cleanly in `COVERAGE.md`. Never fetch the same URL twice.
- `work/src/` — Modular Python package (EDGAR client, membership builder/integrator, XBRL/iXBRL extractor, golden-tested HTML/filing parsers, post-processing DAG, PIT query library, multi-methodology & benchmark analyzer, chart renderer, validator, atomic publisher, incremental CLI).
- `work/staging_output/` — Staging directory validated before every atomic swap into `output/`.
- `output/` — Final deliverables (`events.csv`, `README.md`, `net_equity_issuance.{svg,png}`, `acadian_comparison.{svg,png}`, `COVERAGE.md`).

## 2. Phase 0 Setup & Python 3.14 Resilience (Minutes 0–8)

1. Verify `$SEC_USER_AGENT` is set and non-empty. If missing, fail fast and report clearly.
2. Read `RUN_BUDGET_MINUTES` (default `240`) and compute scaled time gates in `work/run_meta.json`:
   - `phase6_cutoff_min = max(budget - 30, round(budget * 0.78))` → `210`m for 240m; `55`m for 70m; `23`m for 30m.
   - `freeze_cutoff_min = max(budget - 15, round(budget * 0.93))` → `225`m for 240m; `65`m for 70m; `28`m for 30m.
   - `historical_cutoff_min = min(round(budget * 0.79), phase6_cutoff_min - 2)` → `190`m for 240m; `53`m for 70m; `21`m for 30m.
3. Create `work/.venv` with Python 3.14 (`python3 -m venv work/.venv`). Install packages using binary wheels first (`uv pip install` or `pip install --prefer-binary requests httpx beautifulsoup4 lxml pandas numpy matplotlib`). If any C-extension package (`lxml`) lacks a Python 3.14 wheel and fails to build, immediately skip it and use Python's stdlib (`xml.etree.ElementTree`, `html.parser`) rather than burning minutes on compiler errors.
4. Build the centralized HTTP client in `work/src/<pkg>/edgar_http.py` (**never name a module after a Python stdlib package such as `http.py`, `json.py`, or `csv.py`!**) with:
   - **Host-scoped User-Agent**: Send `$SEC_USER_AGENT` **only** to `*.sec.gov` hosts; send a clean neutral descriptive User-Agent (no personal email, never spoofing a browser) to non-SEC hosts.
   - **Cross-process global rate limiter**: Enforce **≤ 8 requests/second** across ALL processes and subagents targeting `sec.gov` using an `fcntl`-locked timestamp token bucket file under `work/cache/.edgar_ratelimit.lock`.
   - **Explicit timeouts & backoff**: 10s connect / 25s read timeout on every request; exponential backoff with jitter on HTTP `429`/`5xx` (max 3–5 retries); fail-fast circuit breaker on persistent `403`/`404`.
   - **Mandatory Clock Print & Module Invocation**: Invoke Python modules cleanly as `work/.venv/bin/python -u -m <pkg>.<mod>` (never compound `cd ... && ...` or `sleep`). Every CLI module MUST print `elapsed=X.X / <budget> min (phase6=<P>, freeze=<F>)` from `work/run_meta.json` at exit so routing decisions always read the real clock! Log progress every ~25 CIKs and record a `phase_timings` row in the **main** SQLite DB for every phase (0–6). Avoid shell-specific word splitting (`for x in $var`) in batch commands.
5. Download official **Federal Reserve Board Z.1 (`z1_csv_files.zip`)** directly from the main agent's Python client during Phase 0/1 (takes ~1–2 seconds) so macro benchmarks never depend on a background agent shell fetch.

## 3. Staged Background Handoff & Fast Integration Checks

To avoid SQLite write locks (`database is locked`) and brief-vs-gate mismatches:
- Background subagents (`membership-researcher`, `benchmark-researcher`, `market-verifier`) **never write directly to `pit_equity.sqlite`**. They write atomic JSON files (write to `.tmp`, then `os.replace`) into `work/staging_membership/` or `work/staging_benchmarks/`.
- **Exact Brief-to-Gate Alignment**: Every subagent brief MUST require the exact JSON schema validated by the integration gate (see `docs/MEMBERSHIP.md` §1 for the exact `year_{YYYY}.json` contract — never mark a required gate field optional in the subagent brief!). Even better: in Phase 1, downloading all ~23 quarterly `N-PORT-P` XML files (2021–2026) takes < 15 seconds, so the main agent can compute authoritative quarter-end `N-PORT-P` snapshot counts directly while `membership-researcher` resolves exact S&P DJI effective dates, `membership_announced_at`, and departed CIKs!
- At every main-agent batch boundary (~every 15–20 minutes, or immediately when a background agent completes), check `work/staging_membership/` for new `year_{YYYY}.json` files and run a **fast automated integration check**:
  1. Every quarter-end in year `YYYY` has ~500 active companies (495–508 listings allowing dual share classes).
  2. Zero overlapping or inverted `[member_start_date, member_end_date]` intervals per CIK.
  3. 100% of CIKs are valid 10-digit SEC CIKs verified against `submissions`, with primary-source URL/accession citations and `membership_announced_at <= effective_date` where announced.
  4. Wikipedia historical cross-check delta is recorded.
- Only when the integration check passes does the main agent atomically merge year `YYYY` into `constituents`, extend `membership_verified_back_to`, and enqueue newly discovered departed CIKs for Tier 1/Tier 2 extraction. Immediately re-task finished background agents if any year failed validation or remains incomplete!
- **Fallback**: If the background `membership-researcher` stops early or misses any year, the main agent completes remaining historical years immediately at the start of Phase 4 (never skip Phase 4!).

## 4. Explicit, Idempotent Post-Processing DAG

Do **not** stack ad-hoc SQL updates that mutate flags in unpredictable order. Implement a single deterministic, idempotent post-processing pipeline that rebuilds final `events` flags from `raw_extractions` + Tier 1 + Tier 3 in strict declared order:
1. **Structural & Filing-Verified Exclusions** (preferred-stock leakage, NCI/subsidiary columns, invalid contexts, preliminary/unpriced prospectuses, reviewed exclusions with documented filing reasons).
2. **Tier 3 Analyst Overrides** (verified primary-filing extracts supersede overlapping Tier 1/2 rows cleanly, recording `[replaced_by_tier3:<id>]`).
3. **Monthly Buyback Superseding** (accepted monthly 10-Q/10-K Item 2/5 rows supersede their covering Tier 1 quarterly XBRL fallback row).
4. **Transaction Lifecycle Linking & Prospectus Activation** (link `424B`/`8-K`/`SC TO-I` announcements to closing/settlement/`10-Q` confirmations under shared `event_key`; ensure a transaction row and a periodic row never both count for the same economic flow).
5. **Cross-Category Twin / Duplicate Guard** (prevent the same economic flow from counting twice across Cash Flow vs. Equity Statement tags or across an excluded/replaced anchor row and its twin).
6. **Point-in-Time Revision Chain Wiring** (populate `is_revision`, `supersedes_record_id`, and first-class `superseded_at` deterministically).
7. **Membership Gating** (set `sp500_member_on_event_date` and gate `include_in_net_aggregate` strictly to active membership intervals and `>= membership_verified_back_to` for headline charts/totals).
8. **Blocking Validator + Regression Snapshot Diff** (enforce schema, enums, FY reconciliation, **zero overlapping included rows for the same `(cik, category, share_class)`**, and `as_of(latest) == headline` from Milestone 1 onward; compare company-year totals against the previous publish and flag any swing > $500M in `work/logs/regression_diff.md` so accidental rule regressions are caught immediately!).

## 5. Atomic Publishing Protocol

1. Generate all deliverables directly into `work/staging_output/` (`events.csv`, `README.md`, `net_equity_issuance.{svg,png}`, `acadian_comparison.{svg,png}`, `COVERAGE.md` — keep all charts at the root of `output/`, not buried in subdirectories).
2. Run the full blocking validation suite (schema, enums, timestamps, FY reconciliation, zero overlap, PIT `as_of(latest) == headline`, chart non-empty & legible).
3. Only after every blocking check passes, atomically move files from `work/staging_output/` into `output/` via `os.replace`, and record the publish + timing row in `phase_timings` and `work/STATE.md`.

## 6. Continuous / Incremental Operation (`--mode incremental`)

Build incremental operation as a first-class workflow from Phase 0, NOT a late add-on:
- Provide a clean CLI (`python -m <pkg>.run --mode full | --mode incremental`) supporting:
  1. **New SEC Filings**: Poll SEC EDGAR daily index (`https://www.sec.gov/Archives/edgar/daily-index/{YYYY}/QTR{Q}/master.{YYYYMMDD}.idx`) and/or refresh `submissions/CIK{cik10}.json` for active constituents, selecting only filings with `acceptanceDateTime` newer than each CIK's watermark.
  2. **New S&P 500 Membership Changes**: Poll S&P Dow Jones Indices index-change press releases AND check for newer ETF `N-PORT-P` filings automatically so new additions, removals, mergers, and spin-offs are detected without manual override edits.
  3. **Incremental Ingestion & Revision Chaining**: Parse newly arrived `8-K`, `424B*`, `SC TO-I`, `S-4`, `10-Q`, `10-K` filings into `raw_extractions`, run the idempotent post-processing DAG (automatically appending revision rows when new filings update prior announcements/comparatives), atomically publish `output/`, and advance `watermarks` **only after** atomic publish succeeds.
- **Mandatory Phase 6 Verification**: In Phase 6, copy `work/db/pit_equity.sqlite` to a temporary scratch path, roll back watermarks by 7 days (deleting those last 7 days of filings from the scratch copy), run `--mode incremental`, and confirm it rediscovers the filings from cache/EDGAR and reproduces identical counts and totals! Document exact continuous-run usage (and example cron/scheduled cadence) in `output/README.md` and `COVERAGE.md`.
