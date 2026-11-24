# Point-in-Time S&P 500 Net Equity Issuance — Agent Instructions

This repository contains instructions and Claude Code configuration for autonomously building a **point-in-time (bi-temporal)** dataset, multi-methodology benchmark suite, publication charts, and coverage report of S&P 500 net equity issuance strictly from primary sources.

## Run Configuration

- **Agent and model**: Claude Code with **Claude Opus 5.5 (high effort)**
- **Pre-installed software**:
  - `python3` (**Python 3.14**) with `python3-venv` and `pip` (optionally `uv` for faster installs)
  - `git`
  - `sqlite3`
  - `curl` and `ca-certificates`
  - *(The agent creates a local virtual environment under `work/.venv` and installs required Python packages using binary wheels, with pure-Python stdlib fallbacks if any C-extension wheel is unavailable on Python 3.14.)*
- **Environment variables**:
  - `SEC_USER_AGENT` (required; provided by the harness per SEC EDGAR fair-access policy — sent **only** to `sec.gov` hosts).
  - `RUN_BUDGET_MINUTES` (optional; defaults to `240` for a 4-hour run; can be overridden for shorter smoke tests).

## Repository Layout

| Path | Purpose |
|---|---|
| `CLAUDE.md` | Mission control: invariants, staged phases, `RUN_BUDGET_MINUTES` scaling, `Continue.` resume protocol, and subagent orchestration. |
| `docs/ARCHITECTURE.md` | Runtime layout, timing telemetry, cross-process EDGAR rate limiter, staged membership handoff, idempotent post-processing DAG, atomic publishing, and continuous `--mode incremental` design. |
| `docs/MEMBERSHIP.md` | Primary-source point-in-time S&P 500 membership methodology (dual ETF `N-PORT-P` + S&P DJI releases + SEC successions + edge cases + Wikipedia verification). |
| `docs/SOURCES_EDGAR.md` | Tier 1 XBRL/iXBRL dimensional rules, Tier 2 parsers for every buyback & issuance category, golden test corpus, and known SEC filing pitfalls. |
| `docs/SCHEMA_AND_RULES.md` | Bi-temporal CSV schema (`events.csv`), transaction lifecycle revision chains, complete buyback/issuance taxonomy, valuation variants, and split rules. |
| `docs/VERIFICATION.md` | Trusted verification sources, all Acadian methodology versions, Federal Reserve Z.1 & Table 1.46 benchmarks, editorial chart spec with mega-IPO annotations, and `COVERAGE.md` template. |
| `.claude/settings.json` | Portable Claude Code workspace permissions. |
| `.claude/agents/membership-researcher.md` | Background subagent expanding historical PIT membership year-by-year via staged JSON handoff. |
| `.claude/agents/filing-analyst.md` | Specialist subagent reading cached primary SEC filings for complex transactions and anomalies. |
| `.claude/agents/market-verifier.md` | Independent check-only subagent cross-verifying numbers against Yahoo Finance, Google Finance, S&P DJI buyback reports, and market portals. |
| `.claude/agents/benchmark-researcher.md` | Independent check-only subagent extracting all Acadian methodology versions, Fed Z.1/FRB Table 1.46, Wikipedia membership checks, and mega-IPO details (SpaceX, Anthropic, etc.). |
| `.claude/agents/qa-auditor.md` | Adversarial QA auditor enforcing structural, semantic, bi-temporal, and benchmark integrity across early and final checkpoints. |

## Design Highlights

1. **Staged, Always-Shippable Architecture**: Builds trailing-12-month primary-source membership first to unlock Milestone 1 `output/` early, while a background `membership-researcher` expands survivorship-free membership backward to 2021 via atomic staged JSON handoffs and fast integration checks.
2. **True Bi-Temporal Event Chains & Full Category Coverage**: Tracks offering, ASR, tender offer, ATM, M&A, employee-equity, warrant, and conversion events from initial announcement (`8-K`/`424B`/`SC TO-I`) through closing/settlement and `10-Q`/`10-K` confirmation under linked `event_key` revision chains, so any historical `as_of_date` reconstructs exactly what was known then.
3. **All Acadian & Fed Z.1 Methodologies Computed and Reconciled**: Produces and compares bottom-up cash-flow event net issuance, SBC-fair-value/cash-M&A variants, Acadian Eq. (1) market-cap/share-count net issuance (total and `% of market cap`, plus Magnificent Seven), Acadian float-adjusted net issuance, Baker-Wurgler equity share `E/(E+D)`, and Federal Reserve Z.1 (`FU103164105.Q`) — with major IPO events (including SpaceX and Anthropic) directly annotated on the graphics.
4. **Continuous Incremental Operation**: Designed from Phase 0 around per-CIK watermarks, EDGAR daily-index polling, automated S&P DJI release checks, and an end-to-end incremental rollback test.
