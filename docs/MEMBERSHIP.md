# Point-in-Time S&P 500 Membership Methodology

Every company that was an S&P 500 member at any point during the covered window must be included—and its events counted **only while it was an active member**. Every membership interval and effective date in the output must originate from **primary sources** (SEC filings and S&P Dow Jones Indices announcements). Wikipedia may ONLY be used afterward to verify completeness and catch gaps.

## 1. Staged Execution Strategy

- **Phase 1 (Fast Trailing-12-Month Start, ~10–12 min)**:
  - Build survivorship-free PIT membership for the **trailing 12 months** (current constituents + all additions, removals, mergers, and spin-offs in the past 12 months) using the 4–5 most recent quarterly ETF `N-PORT-P` filings plus S&P DJI announcements from the last 12 months.
  - Set `membership_verified_back_to` to `T − 12 months` (recording `entered_before_verified_window = true` for continuing constituents). This immediately unblocks Phase 2 (Tier 1 XBRL + Milestone 1 Publish) without stalling on 2021–2024 delisted CIKs.
- **Background Historical Expansion (`membership-researcher` subagent, launched immediately at end of Phase 1)**:
  - Works continuously backward **one year at a time (`2025 → 2024 → 2023 → 2022 → 2021`)**, writing atomic staged JSON files (`work/staging_membership/year_{YYYY}.json`) that the main agent validates and integrates at batch boundaries (see `docs/ARCHITECTURE.md` §3).
  - If the background worker doesn't finish all years before Phase 4, the main agent completes remaining years at the start of Phase 4.

## 2. Primary-Source Discovery Method

### Step A: Dual-Fund Quarterly Snapshots via SEC Form `N-PORT-P`
1. Locate **two** major S&P 500 index-tracking ETFs dynamically on SEC EDGAR via `https://www.sec.gov/files/company_tickers_mf.json` and EDGAR series listings (e.g., iShares Core S&P 500 ETF as primary and Vanguard 500 Index Fund / VOO as secondary cross-validation; verify the exact series ID belongs to the S&P 500 index fund, not another fund in the same trust).
2. Download `N-PORT-P` primary XML filings (`primary_doc.xml`) covering the target window (quarterly snapshots).
3. Parse equity holdings (`assetCat = EC`, excluding cash, futures, repo, and collateral). Extract issuer name, CUSIP, ISIN, LEI, share balance, USD market value, portfolio weight, and **implied quarter-end price (`valUSD / balance`)** — these weights prioritize Phase 3 extraction, and the quarter-end prices + share balances support Acadian Eq. (1) and float-adjusted verification!
4. Cross-checking two S&P 500 funds immediately flags residual post-deletion holdings or tiny spin-off transients held briefly by one fund after an index change.

### Step B: Exact Effective Dates via S&P DJI Announcements & SEC Filings
Because `N-PORT-P` is quarterly and has up to a ~60-day SEC reporting lag:
1. **Launch S&P Dow Jones Indices press release ingestion early** (`press.spglobal.com` / official S&P DJI index-change announcements) covering both the post-snapshot gap (latest `N-PORT-P` → today) AND all historical index changes back to 2021.
2. Parse the exact S&P DJI effective date convention: *"effective prior to the opening of trading on date D"* means entrant `member_start_date = D` and departing company `member_end_date = D − 1 day`.
3. For any change not covered by an S&P DJI release (such as holding-company reorganizations or mergers), confirm exact dates from SEC EDGAR (`8-K` Item 2.01 / 5.01 / 8.01, `8-K12B` successor filings, `Form 25-NSE` delisting notices, and `Form 10` spin-off distributions via `https://efts.sec.gov/LATEST/search-index`).
4. Automate post-snapshot S&P DJI press release checking inside `--mode incremental` so ongoing runs detect newly announced index changes automatically without manual override files.

### Step C: Hardened CIK Resolution (Critical Edge Cases)
Never rely naively on `company_tickers.json` (which omits departed/acquired companies) or raw metadata joins:
1. **Never join on sentinel values**: `N-PORT-P` frequently reports `LEI = "N/A"` across dozens of issuers and `CUSIP = "000000000"` for foreign-domiciled constituents. Explicitly reject sentinel/blank LEIs and zero CUSIPs before matching!
2. **Normalize issuer names carefully**: Strip legal suffixes (`INC`, `CORP`, `CO`, `LTD`, `PLC`, `HLDGS`), state/incorporation tags (`/DE/`, `/MA`), punctuation/apostrophes, and reversed EDGAR naming conventions (`"LAST FIRST INC"` vs. `"FIRST LAST INC"`).
3. **Verify every mapped CIK against `submissions/CIK{cik10}.json`**: Confirm legal name / `formerNames`, active periodic-report history (`10-K`/`10-Q`) overlapping the membership window, and SIC code. Watch out for EDGAR name collisions with inactive shells, operating subsidiaries, or limited partnerships!
4. **Holding-Company Successions & Reincorporations**: When an S&P 500 company reincorporates or restructures under a new holding-company CIK (`8-K12B` / `Form 25`), link the predecessor and successor CIKs as a **continuous index seat** (ending predecessor on succession date − 1, starting successor on succession date) — NEVER double-count overlapping pre-succession filings across both CIKs!
5. **Debt-Only `Form 25` Trap**: Verify that a `Form 25` applies to common stock (not solely a redeemed bond/debt listing) before treating it as an equity delisting.
6. **Residual ETF Positions & Spin-Off Transients**: An ETF may briefly hold shares of a removed stock or a newly spun-off subsidiary bound for the S&P MidCap 400 / SmallCap 600. Verify every brief appearance against S&P DJI announcements so non-S&P-500 spin-offs are not erroneously counted as members.
7. **FDIC-Regulated Bank Filers**: A small number of state-chartered banks file periodic reports with the FDIC rather than SEC EDGAR. Identify any such historical constituents transparently, note their absence from EDGAR `companyfacts`, and disclose them explicitly in `COVERAGE.md`.
8. **Financial vs. Nonfinancial Classification**: Record each CIK's SIC code from `submissions` and set `is_financial = true` for financials (SIC 6000–6799, noting REITs/fintech boundary policy clearly) so S&P 500 Nonfinancial totals align cleanly with Federal Reserve Z.1.

## 3. Multi-Share-Class Companies, M&A & Spin-Offs

- **Multi-Share-Class Issuers**: Detect whenever multiple `N-PORT-P` holdings map to the same CIK/parent issuer. Store consolidated company-level XBRL flows ONCE at the CIK level (never sum twice across dual tickers!), and populate `share_class` specifically whenever the filing breaks out classes (e.g., per-class buyback tables or per-class cover-page share counts).
- **Stock-for-Stock M&A**: Record acquirer's issued shares under `issuance_acquisition` (while a member). End target's membership on closing date. In `COVERAGE.md`, explicitly quantify intra-index stock-for-stock offsets when bridging to market-wide aggregates.
- **Cash M&A / Take-Privates**: End target's membership on closing date. Extract exact deal consideration (cash vs. stock split and per-share cash price) from the target's merger `8-K` (Item 2.01) or `DEFM14A` proxy statement (falling back to last `N-PORT-P` value only if unavailable, flagged clearly) and record cash retirements under supplemental `retirement_cash_merger` for transparent Fed Z.1 / Acadian bridging.
- **Pro-Rata Spin-Offs**: A pro-rata spin-off distribution to existing shareholders raises $0 of new equity capital—NEVER record a pro-rata spin-off as positive equity issuance! (By contrast, a **split-off exchange offer** where parent shareholders tender parent shares to receive subsidiary shares IS a retirement of parent common stock—record under `buyback_tender_offer` with clear notes and show aggregates both with and without split-off exchanges.)

## 4. Membership Inclusion Rule & Verification Check

- **Transactions**: Included iff `member_start_date <= event_date <= member_end_date`.
- **Periodic Rows (monthly/quarterly)**: Included iff active on `period_end` (preferring monthly/transaction granularity around entry/exit dates to minimize partial-quarter edge effects; document exact partial-period impact in `COVERAGE.md`).
- **Mandatory Wikipedia Cross-Check (Verification Only)**: After integrating each membership window, compare against Wikipedia's S&P 500 constituent list and historical changes table. If Wikipedia surfaces a missed constituent or date, **never copy from Wikipedia**—go back to `N-PORT-P`, S&P DJI releases, or SEC `8-K`/`Form 25` to confirm and ingest from a primary source. Log exact match rate (e.g., `X/500` current CIKs, `Y/Z` change dates) in `COVERAGE.md`.
