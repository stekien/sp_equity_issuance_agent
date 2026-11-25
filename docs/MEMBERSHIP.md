# Point-in-Time S&P 500 Membership Methodology

Every company that was an S&P 500 member at any point during the covered window must be included—and its events counted **only while it was an active member**. Every membership interval and effective date in the output must originate from **primary sources** (SEC filings and S&P Dow Jones Indices announcements). Wikipedia may ONLY be used afterward to verify completeness and catch gaps.

## 1. Staged Execution Strategy

- **Phase 1 (Fast Trailing-12-Month Start, ~10–12 min)**:
  - Build survivorship-free PIT membership for the **trailing 12 months** (current constituents + all additions, removals, mergers, and spin-offs in the past 12 months) using the 4–5 most recent quarterly ETF `N-PORT-P` filings plus S&P DJI announcements from the last 12 months.
  - Set `membership_verified_back_to` to `T − 12 months` (recording `entered_before_verified_window = true` for continuing constituents). This immediately unblocks Phase 2 (Tier 1 XBRL + Milestone 1 Publish) without stalling on 2021–2024 delisted CIKs.
- **Background Historical Expansion (`membership-researcher` subagent, launched immediately at end of Phase 1)**:
  - Works continuously backward **one year at a time (`2025 → 2024 → 2023 → 2022 → 2021`)**, writing atomic staged JSON files (`work/staging_membership/year_{YYYY}.json`) matching this exact schema contract so the integration gate never rejects on missing fields:

```json
{
  "year": 2025,
  "verified_quarter_end_counts": {"2025-03-31": 503, "2025-06-30": 503, "2025-09-30": 503, "2025-12-31": 503},
  "constituents": [{"cik": "10-digit", "member_start_date": "YYYY-MM-DD", "member_end_date": "YYYY-MM-DD", "membership_announced_at": "YYYY-MM-DD", "membership_source": "sp_dji_release|sec_filing|nport_fallback", "source_url": "..."}],
  "wikipedia_crosscheck": {"matched_dates": 0, "total_dates": 0, "discrepancies": []}
}
```

  - If the background worker doesn't finish all years before Phase 4 (or any year fails validation), the main agent immediately re-tasks it or completes remaining years at the start of Phase 4.

## 2. Primary-Source Discovery Method

### Step A: Dual-Fund Quarterly Snapshots via SEC Form `N-PORT-P`
1. Locate **two** major S&P 500 index-tracking ETFs dynamically on SEC EDGAR via `https://www.sec.gov/files/company_tickers_mf.json` and EDGAR series listings (e.g., iShares Core S&P 500 ETF as primary and Vanguard 500 Index Fund / VOO as secondary cross-validation; verify the exact series ID belongs to the S&P 500 index fund, not another fund in the same trust).
2. Download `N-PORT-P` primary XML filings (`primary_doc.xml`) covering the target window (all ~23 quarterly snapshots 2021–present take < 15 seconds to fetch, so downloading the full quarterly set early lets the main agent compute authoritative quarter-end counts deterministically!).
3. Parse equity holdings (`assetCat = EC`, excluding cash, futures, repo, and collateral). **Note: `N-PORT-P` XML carries issuer name, CUSIP, ISIN, and LEI — it does NOT carry tickers!** Extract share balance, USD market value, portfolio weight, and **implied quarter-end price (`valUSD / balance`)** — these weights prioritize Phase 3 extraction, and the quarter-end prices + share balances support Acadian Eq. (1) and float-adjusted verification!
4. Cross-checking two S&P 500 funds immediately flags residual post-deletion holdings or tiny spin-off transients held briefly by one fund after an index change.

### Step B: Exact Effective Dates & `membership_announced_at` via S&P DJI Announcements & SEC Filings
Because `N-PORT-P` is quarterly and has up to a ~60-day SEC reporting lag:
1. **Launch S&P Dow Jones Indices press release ingestion early** (`press.spglobal.com` / official S&P DJI index-change announcements) covering both the post-snapshot gap (latest `N-PORT-P` → today) AND all historical index changes back to 2021. Search broadly across S&P 500 announcement titles (`"S&P 500"`, `"Set to Join"`, `"to Replace"`, `"Index Changes"`, `"Changes to the S&P"` — **do NOT filter only for the word `"join"`**, which misses replacements and spin-off removals!).
2. Parse both the **effective date** (`"effective prior to the opening of trading on date D"` → entrant `member_start_date = D`, departing company `member_end_date = D − 1 day`) AND the **announcement publication date (`membership_announced_at`)** right from the press release header, recording `membership_source = sp_dji_release` (or `sec_filing` / `nport_fallback`) in `constituents`.
3. **Real-Market Point-in-Time Rule**: Always gate economic membership inclusion on `[member_start_date, member_end_date]` (matching live financial market membership to the exact trading day) and enforce the audit check `membership_announced_at <= effective_date` on announced changes — **never** use `N-PORT-P`'s 60-day-later filing date to hide a real S&P 500 member inside `pit.as_of(D)`! Report in `COVERAGE.md` the exact % of boundaries backed by pre-effective S&P DJI/SEC announcements vs. `N-PORT-P` fallback.
4. For any change not covered by an S&P DJI release (such as holding-company reorganizations or mergers), confirm exact dates and filing timestamps from SEC EDGAR (`8-K` Item 2.01 / 5.01 / 8.01, `8-K12B` successor filings, `Form 25-NSE` delisting notices, and `Form 10` spin-off distributions via `https://efts.sec.gov/LATEST/search-index`).
5. Automate post-snapshot S&P DJI press release checking inside `--mode incremental` so ongoing runs detect newly announced index changes before they take effect without manual override files.

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

## 3. Multi-Share-Class Companies, M&A, Spin-Offs & Mega-IPOs

- **Multi-Share-Class Issuers**:
  - Detect whenever multiple `N-PORT-P` holdings map to the same CIK/parent issuer. Store consolidated company-level XBRL flows ONCE at the CIK level (never sum twice across dual tickers!).
  - **Single-class index inclusion**: Even when only one common class of an S&P 500 company is listed in the index (while another super-voting/high-price common class is not), common buybacks and issuances across **all common classes** of that member company still belong in its flows.
  - **Never blend unequal-price share classes**: Whenever a filing reports per-class buybacks/issuances (e.g., Class A vs. Class B/C), emit **one row per share class** (`share_class = Class A`, `Class B`, `Class C`). If a filing reports only a consolidated dollar total across classes with unequal economic ratios/prices, record `amount_usd` under `share_class = All Common Classes` and leave `shares` and `avg_price_usd` **blank** with a note—never sum raw share counts of unequal-price classes! For share-count reconciliation and Acadian Eq. (1), value all common classes on a common economic basis (`Σ shares_class * price_class`) so dual-class tech issuers are never omitted.
  - **Intra-common class conversions**: Converting one common share class into another common share class of the same issuer (e.g., Class B → Class A exchange) raises $0 and retires $0 of net common equity—NEVER record it as `issuance_conversion` or `buyback_tender_offer`.
- **Stock-for-Stock M&A**: Record acquirer's issued shares under `issuance_acquisition` (while a member). End target's membership on closing date. In `COVERAGE.md`, explicitly quantify intra-index stock-for-stock offsets when bridging to market-wide aggregates.
- **Cash M&A / Take-Privates**: End target's membership on closing date. Extract exact deal consideration (cash vs. stock split and per-share cash price) from the target's merger `8-K` (Item 2.01) or `DEFM14A` proxy statement (falling back to last `N-PORT-P` value only if unavailable, flagged clearly) and record cash retirements under supplemental `retirement_cash_merger` for transparent Fed Z.1 / Acadian bridging.
- **Pro-Rata Spin-Offs**: A pro-rata spin-off distribution to existing shareholders raises $0 of new equity capital—NEVER record a pro-rata spin-off as positive equity issuance! (By contrast, a **split-off exchange offer** where parent shareholders tender parent shares to receive subsidiary shares IS a retirement of parent common stock—record under `buyback_tender_offer` with clear notes and show aggregates both with and without split-off exchanges.)
- **Explicit Mega-IPO S&P 500 Status Verification**: Never assume whether major recent U.S. IPOs driving macro benchmarks (such as SpaceX, Anthropic, or other top-tier IPOs during the run window) are inside or outside the S&P 500. Explicitly verify each mega-IPO's index membership status across `N-PORT-P`, S&P DJI press releases through today, and Wikipedia; if excluded, document the primary-source confirmation in `COVERAGE.md` and annotate it on the benchmark charts; if added, count only flows from `member_start_date` onward.

## 4. Membership Inclusion Rule (Including Straddling Quarters) & Verification Check

- **Transactions**: Included iff `member_start_date <= event_date <= member_end_date`.
- **Periodic Rows (monthly/quarterly) & Straddling Entry/Exit Quarters**:
  - If `[period_start, period_end]` lies fully inside `[member_start_date, member_end_date]`, include 100%; if fully outside, exclude (`sp500_member_on_event_date = false`).
  - **Large straddling quarters (> $250M)**: Any quarterly row exceeding $250M whose `[period_start, period_end]` straddles `member_start_date` or `member_end_date` MUST be flagged for **exact monthly or transaction dating** (10-Q Item 2 monthly buyback table or `8-K`/`424B`/M&A footnote date) so pre-entry deals are 100% excluded and pre-exit deals are 100% included.
  - **Routine continuous straddling quarters (without daily/monthly dates)**: Preserve the **exact filing-reported `amount_usd`** in `events.csv` (maintaining 1-to-1 auditability against the SEC filing!) and allocate only active-member overlap days in quarterly aggregation (`overlap_days(period ∩ membership_interval, quarter) / total_days(period)`), noting the straddling proration explicitly in `notes` and `COVERAGE.md`.
- **Mandatory Wikipedia Cross-Check (Verification Only)**: Always run the Wikipedia check for **both current constituents (Phase 1, verified directly by the main agent)** and historical changes (each integrated year). If Wikipedia surfaces a missed constituent or date, **never copy from Wikipedia**—go back to `N-PORT-P`, S&P DJI releases, or SEC `8-K`/`Form 25` to confirm and ingest from a primary source. Log exact match rate (e.g., `X/500` current CIKs, `Y/Z` change dates) in `COVERAGE.md`.
