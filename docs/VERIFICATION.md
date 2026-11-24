# Verification, External Benchmarking, Charting, and Coverage Reporting

## 1. Automated Integrity Checks (Run Before Every Atomic Publish)

Every staging build in `work/staging_output/` must pass automated validation:
1. **Schema & Types**: Exact required columns, valid ISO dates/timestamps, valid category/direction/method enums, non-null required fields.
2. **Temporal Sanity**: Flag any row where `published_at < period_end` (allowable only for pre-announced transactions such as priced offerings before closing date or announced tender offers—must be explained in `notes`). Flag any look-ahead leakage (e.g., a Q1 row carrying a later 10-K timestamp without being marked as a revision).
3. **Zero Double-Counting**:
   - Zero duplicate `(event_key, published_at, source_accession_number)` rows.
   - Among rows with `include_in_net_aggregate = true`, verify no overlapping `(cik, category_group, period)` pairs exist (e.g., monthly buybacks AND quarterly XBRL fallback both `true`, or superseded revision still `true`).
   - Verify no dual-class CIK has duplicated XBRL totals across two tickers.
4. **Unit & Magnitude Outlier Scan**:
   - Flag any single company-month or single event exceeding $30B (possible 1,000x unit-scaling bug from `"in thousands"` HTML tables, or debt/structured-note `424B2` leakage!) and inspect every single flagged row before publishing.
   - Flag any `avg_price_usd` below $1 or above $10,000 (except genuinely high-priced stocks verified against market prices) as a likely unit mismatch between shares and dollars.

## 2. Internal Balance-Sheet Share-Count Reconciliation (Phase 5)

For each covered company-quarter (using `CommonStockSharesOutstanding` / `EntityCommonStockSharesOutstanding` from `companyfacts`, adjusted for any stock splits in that quarter):
- Compute `observed_delta_shares = shares_outstanding(t) - shares_outstanding(t-1)`.
- Compute `explained_net_shares = sum(shares_issued) - sum(shares_repurchased)` over the same quarter.
- Compute `residual_shares = observed_delta_shares - explained_net_shares` and approximate dollar residual (`residual_shares * avg_price`).
- Rank the **top 20 largest unexplained residuals** by estimated dollar magnitude and dispatch `.claude/agents/filing-analyst.md` to diagnose them (common root causes: missed M&A share issue, missed ASR delivery, unrecorded RSU vesting/tax withholding, unadjusted stock split, or convertible note conversion).
- Summarize reconciliation coverage and residual distribution honestly in `output/COVERAGE.md`.

## 3. External Benchmark Comparisons (Phase 6)

### A. Federal Reserve Z.1 Financial Accounts (Nonfinancial Corporate Net Equity Issuance)
1. Fetch the Federal Reserve's quarterly nonfinancial corporate business net equity issuance series directly from official public endpoints:
   - FRED series `NCBEILQ027S` (CSV endpoint: `https://fred.stlouisfed.org/graph/fredgraph.csv?id=NCBEILQ027S`) and/or Federal Reserve Board Z.1 release data (Table F.223 / L.223).
2. **CRITICAL UNIT CHECK**: Verify the exact units and annualization of the downloaded series! FRED `NCBEILQ027S` is typically reported in **Millions of Dollars, Seasonally Adjusted Annual Rate (SAAR)**. To compare against quarterly unadjusted dollar totals in billions, divide SAAR millions by `4,000` (check series metadata/header carefully before converting—never guess!).
3. Compare **quarterly from 2021 to latest available 2026 quarter**:
   - Plot/tabulate S&P 500 **Total** net issuance, S&P 500 **Nonfinancial** net issuance (`is_financial = false`), and **Fed Z.1 Nonfinancial** net issuance side by side.
   - Compute correlation, sign agreement by quarter, and ratio of S&P 500 Nonfinancial to Fed Z.1.
   - Explain structural differences (Fed Z.1 covers all US nonfinancial corporations including private/non-S&P-500 firms and includes cash M&A retirements, whereas S&P 500 covers ~500 large public firms).

### B. Acadian Asset Management Benchmark (November 2026)
1. Fetch and read the benchmark article referenced in the exercise:
   `https://www.acadian-asset.com/investment-insights/owenomics/the-smart-money-is-selling-equity`
2. Extract Acadian's exact methodology, time window (12 months to 12 November 2026), headline total, and ex-SpaceX figures directly from the article.
3. Construct a quantitative **reconciliation bridge** in `COVERAGE.md` comparing your S&P 500 trailing-12-month net issuance to Acadian's total US market figure, explicitly accounting for:
   - Universe difference (S&P 500 vs. total US market, including mega-IPOs outside the S&P 500 such as SpaceX / Anthropic, depending on whether either entered the S&P 500 during the window).
   - Cash M&A retirements vs. buybacks.
   - Valuation differences (e.g., SBC/RSU treatment, net share settlement).
   - Cutoff date differences (12 Nov 2026 vs. today's run date).

### C. Spot-Check Audit (via `qa-auditor` Subagent)
Before finalizing Phase 6, invoke `.claude/agents/qa-auditor.md` to independently verify at least **10 diverse high-impact records** directly against raw SEC filings (and cross-check sanity against public financial portals/news as allowed for checking):
- Top 3 largest buyback quarters/months
- Top 3 largest equity offerings or M&A share issuances
- At least 1 ASR and/or tender offer
- At least 1 multi-share-class company
- At least 1 company that entered or exited the S&P 500 during the period
- At least 1 revised/restated observation (`is_revision = true`)

## 4. Chart Specification (`output/net_equity_issuance.svg` & `.png`)

Generate both vector (`output/net_equity_issuance.svg`) and high-DPI raster (`output/net_equity_issuance.png`, 300 DPI) versions of a clean, publication-grade chart:
- **Primary view**: Quarterly (or monthly if cleanly covered without noisy quarterly-fallback spikes—if mixed, aggregate to **calendar quarters** for visual comparability with Fed Z.1, and optionally add a second panel or subplot showing the trailing-12-month monthly series!).
- **Recommended clean two-panel layout** (12" × 7.5" or similar clean aspect ratio, light background, crisp sans-serif typography, minimal chartjunk, clear `$B` USD y-axis with prominent `y = 0` baseline):
  - **Top panel (Main chart)**: S&P 500 Net Equity Issuance by quarter over the covered period (distinct colors for negative net issuance / net buybacks vs. positive net issuance, clearly highlighting the 2026 turn if present, with Gross Issuance and Gross Buybacks shown cleanly or stacked behind a bold Net Issuance line/bar).
  - **Bottom panel (Benchmark comparison)**: S&P 500 Nonfinancial Net Issuance vs. Federal Reserve Z.1 US Nonfinancial Corporate Net Equity Issuance (converted to matching quarterly `$B` units!), illustrating how the two series tell the same directional story while differing in universe breadth.
- Include clear title, informative subtitle (exact period covered, S&P 500 PIT scope, number of companies), direct legend, zero-line label, and bottom source footnote (`Source: Primary SEC EDGAR filings (10-K, 10-Q, 8-K, 424B, SC TO-I, N-PORT-P); Federal Reserve Z.1 (NCBEILQ027S). Point-in-time as of <run date>.`).
- **Visually inspect** the rendered PNG (or verify layout dimensions, non-overlapping labels, legible axes) before final publish!

## 5. Required Structure of `output/COVERAGE.md`

Make `output/COVERAGE.md` transparent, quantitative, and ruthlessly honest:
1. **Executive Summary**: Date range covered, total unique S&P 500 historical members identified vs. covered by Tier 1 / Tier 2 / Tier 3, total event rows, headline trailing-12-month and multi-year Gross Issuance, Buybacks, and Net Issuance.
2. **S&P 500 Membership Methodology & Verification**: Primary sources used (`N-PORT-P` funds/periods, S&P announcements, `8-K`s), total additions/removals/mergers/spin-offs/multi-class firms handled, and exact results of the Wikipedia cross-check (matches and any discrepancies).
3. **Coverage Matrix & What Was Not Covered**:
   - Table by year/quarter showing company count, % of index weight covered at Tier 1 (XBRL) vs. Tier 2 (monthly buyback tables & offerings/8-Ks), and exact list/count of companies, filings, or historical periods NOT reached before the 4-hour cutoff (and why).
4. **Verification & Reconciliation Results**:
   - Automated integrity check results (pass/fail counts, unit-outlier resolutions).
   - Internal share-count balance-check statistics and top discrepancies investigated.
   - Federal Reserve Z.1 quarterly comparison table & correlation/magnitude commentary.
   - Acadian (Nov 2026) methodology & reconciliation bridge.
   - `qa-auditor` spot-check summary (records checked, defects found and fixed).
5. **What May Be Wrong & Known Limitations**:
   - Explicitly list every known weakness, edge case, unverified membership boundary, employee-SBC valuation assumption, partial-quarter membership edge case, or noisy filing parser area. Evaluators value calibrated honesty far more than false perfection.
6. **Incremental / Continuous Operation**:
   - Brief note confirming watermark status and how subsequent runs pick up new SEC filings.
