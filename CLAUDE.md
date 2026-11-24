# Mission Control: Point-in-Time S&P 500 Net Equity Issuance

You are running an autonomous, time-bounded engineering and financial-data mission (~4 hours before hard stop). Your goal is to build a **point-in-time (bi-temporal)** dataset, chart, and coverage report of **S&P 500 net equity issuance** strictly from primary sources, starting with the trailing 12 months up to today and extending backward as far as time permits (targeting 2021–present to align with Federal Reserve Z.1 benchmarks).

## Non-Negotiable Invariants

1. **Primary Sources Only for Output Numbers**: Every value in `output/events.csv` must come from SEC filings (`10-K`, `10-Q`, `8-K`, `424B*`, `S-3`, `S-4`, `SC TO-I`, `N-PORT-P`, and their XBRL/XML data), company announcements, or S&P Dow Jones Indices announcements. Wikipedia, Yahoo Finance, Google Finance, and financial news may ONLY be used to verify/check results—never as the source of any output figure or membership record.
2. **True Point-in-Time (Bi-Temporal) Integrity**: Every event row must record both when the activity occurred (`event_date` / `period_start` / `period_end`) AND when the source document was published (`published_at`, using SEC `acceptanceDateTime` to the second where available). Never introduce look-ahead bias. If a later filing revises an earlier figure, **keep both rows** linked by `event_key`.
3. **Always-Shippable `output/`**: Produce a complete, validated `output/` directory (`events.csv`, `README.md`, chart files, `COVERAGE.md`) early (~45–60 minutes in) and **atomically refresh** it after every phase and major batch. Never defer output generation to the end of the run.
4. **Wall-Clock Awareness & Resumability**: Record `start_time_utc` in `work/run_meta.json` at minute 0. Keep all progress in `work/db/pit_equity.sqlite` and summarize state in `work/STATE.md` after each phase/batch. Whenever you receive `Continue.`, immediately read `work/STATE.md`, check elapsed minutes since `start_time_utc`, and resume without redoing completed work. At **210 minutes (3h30m)** elapsed, stop expanding scope/history and enter Phase 6 (Final Verification & Publish).
5. **SEC EDGAR Fair-Access Compliance**: Always send the `User-Agent` header from environment variable `$SEC_USER_AGENT` (fail loudly if unset). Enforce a strict global rate limit of **≤ 8 requests/second** across all scripts and subagents via a single shared disk cache and rate limiter. Never fetch the same EDGAR URL twice.
6. **Incremental / Continuous Design**: Architect your pipeline under `work/src/` with persistent watermarks (last processed `acceptanceDateTime` and accession number per CIK, plus daily EDGAR index support) so re-running the pipeline later picks up newly published filings automatically.
7. **Zero Double-Counting**: Never sum overlapping XBRL tags, never count secondary (selling stockholder) shares as primary issuance, never double-count dual-class issuers, and never let granular monthly/event rows and fallback quarterly XBRL rows both count toward the same period aggregate.

## Required Deliverables (in `output/`)

1. `output/events.csv` — One row per event (offering, monthly buyback, ASR, tender offer, M&A share issue, employee equity, warrant/conversion, or quarterly XBRL fallback where granular detail is unavailable), conforming strictly to `docs/SCHEMA_AND_RULES.md`.
2. `output/README.md` — Concise column dictionary, category definitions, point-in-time query instructions, and instructions for running the pipeline incrementally.
3. `output/net_equity_issuance.svg` and `output/net_equity_issuance.png` — Publication-grade chart of S&P 500 net equity issuance over the covered period by quarter (and monthly breakdown if clean), following `docs/VERIFICATION.md`.
4. `output/COVERAGE.md` — Detailed accounting of what was covered vs. not covered, S&P 500 membership methodology and verification, reconciliation checks (internal share-count check, Fed Z.1, Acadian), and an honest assessment of what may be wrong.

## Execution Plan

Read each spec doc right before entering its phase so your context stays sharp and focused.

| Phase | Target Elapsed | Read First | Objective & Exit Criterion |
|---|---|---|---|
| **0. Setup** | 0:00–0:10 | `docs/ARCHITECTURE.md` | Verify `$SEC_USER_AGENT`, create `work/.venv` and install Python packages, record `start_time_utc`, create SQLite schema, build centralized cached rate-limited EDGAR client. |
| **1. PIT Membership** | 0:10–0:30 | `docs/MEMBERSHIP.md` | Build primary-source PIT S&P 500 membership table (trailing 12 months + back to 2021) using ETF `N-PORT-P` filings, S&P announcements, and company `8-K`s. Map all constituents (including departed/merged) to CIKs. Cross-check vs. Wikipedia. |
| **2. Tier 1 Baseline & Milestone 1 Publish** | 0:30–0:55 | `docs/SOURCES_EDGAR.md`, `docs/SCHEMA_AND_RULES.md` | Fetch EDGAR `submissions` + `companyfacts` for all historical S&P 500 members. Extract & de-cumulate trailing-12-month (and 2021+) XBRL equity flows with exact `acceptanceDateTime`. Run validator and **publish Milestone 1 `output/`**. |
| **3. Tier 2 Trailing-12M Deep Extraction** | 0:55–2:15 | `docs/SOURCES_EDGAR.md`, `docs/SCHEMA_AND_RULES.md` | Prioritize largest index weights / largest equity movers first. Parse 10-Q Item 2 & 10-K Item 5 **monthly buyback tables**, filtered `424B*` equity offerings (excluding bank debt/structured notes), `8-K` ASRs/offerings/M&A, `SC TO-I` tender offers, and `S-4` stock M&A for the trailing 12 months. Atomically refresh `output/` every ~75–100 companies. |
| **4. Historical Backfill (to 2021)** | 2:15–3:05 | `docs/SOURCES_EDGAR.md` | Extend Tier 2 monthly buyback tables and major offerings/M&A backwards year-by-year (prior year → 2021) while preserving Tier 1 full-universe XBRL coverage. Atomically refresh `output/` after each year completed. Stop immediately if elapsed time reaches 190 min. |
| **5. Tier 3 Anomaly Resolution** | 3:05–3:30 | `docs/VERIFICATION.md` | Run share-count balance check (`Δ Shares Outstanding` vs. net flows) and outlier scans. Dispatch `filing-analyst` subagents to resolve top 15–25 discrepancies, mega-deals, complex ASRs, multi-class edge cases, and revisions. Refresh `output/`. |
| **6. Final QA, Benchmarks & Publish** | 3:30–4:00 | `docs/VERIFICATION.md` | Stop all new ingestion. Run `qa-auditor` subagent. Execute Fed Z.1 (`NCBEILQ027S` / Table F.223) and Acadian comparisons, fix high-severity defects, render final charts, write final `output/COVERAGE.md` and `output/README.md`, and run final atomic publish + integrity check. |

## Subagent Delegation

- Use `.claude/agents/filing-analyst.md` ONLY for targeted reading of messy/ambiguous raw filings (e.g., complex ASRs, mixed primary/secondary prospectuses, spin-offs, large share-count discrepancies). Never use an LLM subagent for bulk parsing that deterministic Python scripts handle faster and more accurately.
- Use `.claude/agents/qa-auditor.md` in Phase 6 (and optionally after Milestone 1) for an independent adversarial audit of `output/` against raw SEC filings and external benchmarks.
- Ensure all subagents read cached local files from `work/cache/` or call your centralized rate-limited fetcher—never let multiple subagents make unthrottled requests to `sec.gov`.

## Resume Protocol

If your session starts or resumes with `Continue.`:
1. Check `work/run_meta.json` and current UTC time to calculate elapsed minutes.
2. Read `work/STATE.md` and query `work/db/pit_equity.sqlite` for exact progress counts.
3. If elapsed time ≥ 210 minutes, jump directly to Phase 6.
4. Otherwise, continue cleanly from the next unfinished batch/phase and update `work/STATE.md` upon completion.
