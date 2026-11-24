# Point-in-Time S&P 500 Net Equity Issuance — Agent Instructions

This repository contains instructions and Claude Code configuration for autonomously building a **point-in-time (bi-temporal)** dataset and analysis of S&P 500 net equity issuance from primary sources over a 4-hour run.

## Run Configuration

- **Agent and model**: Claude Code with **Claude Opus 5.5**
- **Pre-installed software**:
  - `python3` (3.11 or newer) with `venv` and `pip` (optionally `uv` for faster package installs)
  - `git`
  - `sqlite3`
  - `curl` and `ca-certificates`
  - *(The agent will create a local Python virtual environment under `work/.venv` and install required Python packages such as HTTP, HTML/XML parsing, data analysis, and plotting libraries during Phase 0.)*
- **Environment variables**: `SEC_USER_AGENT` (provided by the test harness per SEC EDGAR fair-access rules).

## Repository Layout

| Path | Purpose |
|---|---|
| `CLAUDE.md` | Mission control: core invariants, phase schedule, checkpoint/resume loop, and subagent delegation rules. |
| `docs/ARCHITECTURE.md` | Runtime directory structure, 4-hour time budget, SQLite state ledger, atomic publishing protocol, and incremental re-run design. |
| `docs/MEMBERSHIP.md` | Primary-source methodology for point-in-time S&P 500 membership (additions, removals, mergers, spin-offs, multi-class shares). |
| `docs/SOURCES_EDGAR.md` | SEC EDGAR rate-limiting, caching, Tier 1 XBRL extraction rules, Tier 2 HTML/filing parsers, and known EDGAR traps. |
| `docs/SCHEMA_AND_RULES.md` | Event taxonomy, bi-temporal CSV schema, revision handling, split conventions, valuation rules, and provenance standards. |
| `docs/VERIFICATION.md` | Internal share-count reconciliation, Fed Z.1 & Acadian external checks, charting spec, and `COVERAGE.md` requirements. |
| `.claude/settings.json` | Portable Claude Code workspace permissions. |
| `.claude/agents/filing-analyst.md` | Subagent for inspecting ambiguous or high-value raw SEC filings. |
| `.claude/agents/qa-auditor.md` | Independent adversarial QA subagent run prior to final publication. |

## Design Highlights

1. **Always-Shippable Incremental Output**: The agent publishes a complete, schema-validated `output/` set early (after building the full-universe XBRL baseline) and atomically refreshes `output/` after every milestone. Even if stopped abruptly at 4:00, the repository holds a clean, consistent deliverable.
2. **Primary-Source Membership**: Uses SEC `N-PORT-P` holdings filings of S&P 500 index ETFs combined with S&P Dow Jones Indices announcements and company `8-K` filings to construct historical membership, using Wikipedia solely as an independent verification check.
3. **Three-Tier Extraction**: Combines breadth-first SEC XBRL `companyfacts` + `submissions` across all historical S&P 500 members with depth-first deterministic parsing of monthly buyback tables, equity prospectuses, `8-K` filings, and tender offers, plus targeted subagent analysis for complex deals and anomalies.
4. **True Bi-Temporal Provenance**: Separates economic event/period dates from exact SEC `acceptanceDateTime` publication timestamps, preserves all revisions without overwriting, and enforces internal share-count reconciliation alongside Federal Reserve Z.1 and Acadian cross-checks.
5. **Continuous Operation**: Stores filing watermarks per CIK and supports incremental polling of new SEC EDGAR filings on subsequent runs.
