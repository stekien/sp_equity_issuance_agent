# Verification, All Acadian & Fed Methodologies, Trusted Check Sources & Charting

## 1. Trusted Source Boundary (Primary Sources vs. Check-Only Websites)

Respect this boundary strictly:

- **Primary Sources (Allowed as Sources of Output Numbers & Membership)**:
  - SEC EDGAR (`data.sec.gov`, `www.sec.gov`, `efts.sec.gov`)
  - S&P Dow Jones Indices press releases (`press.spglobal.com`, `spglobal.com/spdji`)
  - Official Company Investor Relations press releases & wire releases (Business Wire, PR Newswire, GlobeNewswire)
- **Authoritative Macro Benchmarks (For Benchmark Verification & Charts)**:
  - Federal Reserve Board Z.1 Financial Accounts (`https://www.federalreserve.gov/releases/z1/` — download `z1_csv_files.zip` directly via the main Python HTTP client; Table F.5.1.1 / S11.1, series `FU103164105.Q` NSA quarterly transactions and `FA103164105.Q` SAAR transactions. **WARNING**: FRED `NCBEILQ027S` is the *level* series `FL103164105.Q` (~$80T market value), NOT the net issuance transactions flow — NEVER use `NCBEILQ027S` as a flow!)
  - Federal Reserve Board Table 1.46 (New Security Issues of U.S. Corporations — gross stock and corporate bond issues, used by Baker & Wurgler and Acadian Figure 3)
  - FRED transactions mirror (secondary fallback only if `z1_csv_files.zip` is unreachable: verify series metadata explicitly states *"Transactions"* / *"Flow"*, e.g. `NCBCEBQ027S`, never *"Level"*)
  - Acadian Asset Management (`https://www.acadian-asset.com/investment-insights/owenomics/the-smart-money-is-selling-equity` and linked Owenomics methodology notes)
  - S&P Dow Jones Indices Quarterly Buybacks press releases (official published S&P 500 quarterly aggregate buyback totals — an exceptionally strong aggregate sanity check!)
- **External Verification Websites (Strictly CHECK-ONLY — NEVER use as the source of any output number!)**:
  - **Yahoo Finance** (`finance.yahoo.com`) — check stock splits, historical share counts, market caps, recent offerings/buybacks, and mega-IPO tickers/dates.
  - **Google Finance** (`google.com/finance`) — check current prices, market caps, corporate event summaries, and multi-class share listings.
  - **Wikipedia** (`en.wikipedia.org/wiki/List_of_S%26P_500_companies`) — check S&P 500 constituents and historical additions/removals.
  - **StockAnalysis** (`stockanalysis.com`) & **Macrotrends** (`macrotrends.net`) — check multi-year share-count histories, buyback histories, and IPO listings.
  - **Renaissance Capital** (`renaissancecapital.com`) & **Nasdaq / NYSE IPO & Corporate Actions Calendars** (`nasdaq.com`) — verify major U.S. IPO dates, proceeds raised vs. initial market cap (including SpaceX, Anthropic, and other 2025–2026 mega-IPOs).
  - **Financial News** (Reuters, Morningstar, WSJ, FT) — verify context around mega-deals, ASRs, M&A closings, and major IPOs.

**Golden Rule of Web Verification**: Whenever `market-verifier` or `benchmark-researcher` discovers a discrepancy via a check-only website (e.g., a missed IPO, wrong split ratio, missed buyback quarter, or missing M&A deal), do **NOT** copy the web number into `events.csv`. Instead, locate the corresponding **primary SEC filing or official announcement** and ingest the primary-sourced fact!

## 2. Computing & Reconciling ALL Acadian & Fed Z.1 Methodologies

Read the Acadian article (`the-smart-money-is-selling-equity`) in full via `benchmark-researcher` and compute **every single methodology version** described in the article for the S&P 500 so all definitions can be compared like-for-like:

```text
NI_cash(q) = sum_{r active} net_amount_usd(r) * overlap_days(r ∩ membership_interval, q) / total_days(r)
NI_sc(q)   = sum_{c, t in q} (Shares_c(t) - Shares_c(t-1)) * Price_c(t)   # splits excluded; sum ALL common classes!
Fed Z.1 $B = FU103164105.Q / 1,000 == FA103164105.Q / 4,000
```

1. **Version 1 — Bottom-Up Cash-Flow Event Net Issuance (`NI_cash`, Headline from `events.csv`)**:
   - Sum of primary offerings, M&A share fair value, conversions, warrants, and employee-plan cash proceeds minus buybacks (open-market, ASR, tender/exchange), **strictly allocated to calendar quarters by day-weighting (`overlap_days(r ∩ membership_interval, q) / total_days(r)`)** rather than bucketing non-calendar fiscal quarters solely by `period_end` (plus monthly trailing-12-month detail).
   - Also compute event-level sensitivity variants: (a) with RSU `tax_withholding_net_settlement`, (b) with `issuance_employee_equity` valued at vest-date fair value (`NI_sbc_fv`), (c) with `retirement_cash_merger`, and (d) excluding non-cash split-off exchanges.
2. **Version 2 — Acadian Equation (1) Total Market-Cap / Share-Count Net Issuance (`NI_sc`)**:
   - Formula from Acadian Eq. (1): `Net_Issuance(t) = Market_Cap(t) - Market_Cap(t-1) * P(t) / P(t-1)` = `Δ(split-adjusted shares outstanding) * P(t)`.
   - Compute both in **USD billions** AND as **% of total S&P 500 market cap** (trailing 12 months, matching Acadian Figure 1 & Table 1) across the **entire verified window** (ingest `N-PORT-P` AND pre-2020 `N-Q`/`N-CSR` quarter-end prices from `membership-researcher`'s year files so `NI_sc` and `NI_float` cover the full verified history, not only 2020Q4+!).
   - Use SEC balance-sheet (`CommonStockSharesOutstanding`), cover-page (`EntityCommonStockSharesOutstanding`), AND dimensional/per-class iXBRL share counts combined with `N-PORT-P` quarter-end prices — **explicitly solve per-class and rounded-share-count issuers so ALL Magnificent Seven members (including dual-class and large semiconductor issuers!) are included cleanly without omissions!**
   - Report the **Magnificent Seven** sub-cut explicitly alongside total S&P 500 to verify directly against Acadian Table 1.
   - **Acadian IPO Valuation Bridge**: From each mega-IPO's primary `424B4` cover page (SpaceX, Anthropic), extract **BOTH** (a) **Primary Cash Proceeds Raised** (`new primary shares × IPO price`) AND (b) **Acadian Eq. (1) Market-Cap Issuance** (`total post-IPO shares outstanding × IPO/first-day price`) so the quantitative bridge from S&P 500 cash basis → S&P 500 Eq. (1) → Acadian all-U.S. headline is 100% transparent!
3. **Version 3 — Acadian Figure 2 Float-Adjusted Net Issuance Analog (`NI_float`)**:
   - Apply Equation (1) using **float-adjusted shares** inferred from S&P 500 index-tracking ETF `N-PORT-P` holdings across all historical snapshots. **CRITICAL NORMALIZATION**: NEVER scale using one total-market-cap / total-ETF-value ratio (outlier share counts distort the ratio by 100×!). Always normalize using the **median of per-issuer `total_shares / ETF_shares` ratios** (or `balance_i(t) / ETF_shares_outstanding(t)`) to strip out ETF creations/redemptions cleanly, and enforce a **blocking sanity bound (`|quarterly NI_float| < $1,000B`)**!
4. **Version 4 — Acadian Figure 3 Baker & Wurgler (2000) Equity Share (`E / (E + D)`)**:
   - Compute trailing-12-month `E / (E + D)` for S&P 500 constituents from primary SEC XBRL gross equity issuance (`E`, dollars raised) and gross corporate debt issuance (`D`, from `ProceedsFromIssuanceOfLongTermDebt` / `ProceedsFromIssuanceOfDebt`).
   - Compare against **Federal Reserve Board Table 1.46** macro `E / (E + D)` — **specifically use corporate bonds SOLD IN THE UNITED STATES (Table 1.46 line 3) as `D`** (matches Acadian Figure 3 to ~0.04pp RMSE; total bonds line 2 is biased ~0.33pp low — keep total-bond variant as supplemental if desired) — and 12-month changes in `E` and `D`!
5. **Version 5 — Federal Reserve Z.1 Benchmark (`FU103164105.Q`)**:
   - Pull `FU103164105.Q` (NSA quarterly transactions `$M`, divide by `1,000` for `$B`) and verify `FA103164105.Q / 4 == FU103164105.Q` directly from Federal Reserve Board `z1_csv_files.zip` (Table F.5.1.1 / S11.1 — downloaded by the main agent's Python HTTP client if a background subagent hits a network permission block).
   - Compare quarterly across the **entire survivorship-free verified window** (not capped at 2021!) against **S&P 500 Nonfinancials** (`is_financial = false`) and **Total S&P 500**, reporting sign agreement, correlation, and cumulative ratio.
6. **S&P DJI Quarterly Buybacks Cross-Check**:
   - Compare our S&P 500 quarterly gross buybacks against official S&P Dow Jones Indices quarterly S&P 500 buyback press releases (via `market-verifier`) — one of the sharpest direct sanity checks available!

## 3. Automated Validation, Identities & Multi-Stage Audits

### Blocking Structural & Semantic Checks (Every Publish)
1. Schema, strict controlled enums (reject off-enum subagent values), lowercase `true`/`false` booleans, **non-empty `ticker`** (backfilled from `dei:TradingSymbol` / `submissions.tickers`), required columns, ISO UTC timestamps, `source_accession_number`, `source_url`, `source_location`, `split_adjustment_note`.
2. **Fiscal-Year Reconciliation (`Σ quarters == FY`)** per issuer and concept family (blocking on over-counting!), plus **granular-sum check** (granular transaction/monthly rows + any residual must equal the covering XBRL quarter within tolerance, never > 110%).
3. **Zero Double-Counting & Pass-Through Guard**: No overlapping included monthly/quarterly/annual/transaction rows for the same `(cik, category, share_class)`, no dual-class CIK duplication, **no cross-category equal-amount twins** (e.g., same flow in `issuance_acquisition` and `issuance_other_common`), no employee-plan twins, no preferred/mandatory-convertible/NCI leakage, AND **flag any equal-and-opposite issuance + buyback pair (> $500M within 1% in the same issuer-quarter)** to catch pass-through secondary structures!
4. **Transaction Membership Purity & Straddler Report**: Transaction-dated rows (`granularity = transaction`) get strict `0.0` or `1.0` `member_overlap_fraction` on `event_date` (never day-prorated; large acquisition issuances on an entry/formation date verified for pre-entry exclusion!), and emit a report of all remaining undated > $250M straddling periodic rows.
5. **Bi-Temporal Verification & Stage Provenance**: Blocking check that `pit.as_of(latest) == headline`, plus automated tests at ≥ 3 earlier `as_of_date` snapshots asserting **both** row transitions AND **amount provenance** (every stage's amount actually appears in its own `published_at` filing — zero look-ahead from a later `10-Q` onto an earlier `424B`/`8-K`!).
6. **Regression Snapshot Diff**: Compare company-year totals against the previous publish; log any delta > $500M so rule changes never silently regress earlier fixes.
7. **Outlier Guard**: Flag any single event > $30B or `avg_price_usd` outside `[$1, $20,000]` (except verified ultra-high-price shares) for mandatory review.

### Internal Share-Count Reconciliation (Phase 5)
- Compute `ΔS_obs − (S_issued − S_repurchased)` across all company-quarters; investigate the top 20–25 dollar residuals with `filing-analyst`.

### Multi-Stage Audits & External Web Verification
- **Early Audit (end of Phase 2 / Milestone 1)**: Fast `qa-auditor` pass to catch semantic XBRL bugs (preferred/NCI/context/twins) before bulk Tier 2 runs.
- **Web Verification Pass (Phase 5 — Mandatory, Never Skip!)**: Dispatch `market-verifier` (Yahoo Finance, Google Finance, S&P DJI buybacks, StockAnalysis, Nasdaq) to verify the top 25 S&P 500 movers, major splits, S&P 500 buyback totals, and mega-IPO details (SpaceX, Anthropic, etc.), and confirm every major IPO chart annotation directly against its **primary `424B4` prospectus cover page** (exact pricing date, shares offered, offering price, gross proceeds)!
- **Final Adversarial Audit (Phase 6)**: Full `qa-auditor` pass including ≥ 15 raw-filing spot-checks (top buybacks, top offerings, ASRs, tender/split-off offers, ATMs, M&A, multi-class, S&P additions/removals, and multi-filing revision chains). Must achieve `PASS` before final publish.

## 4. Editorial Chart Specifications & Mega-IPO Callouts

Adopt a clean, understated editorial aesthetic modeled on modern financial publications (warm off-white canvas `#fcfcfb`, dark charcoal title `#0b0b0b`, muted subtitle `#52514e`, subtle horizontal gridlines `#e1e0d9`, crisp zero baseline `#c3c2b7`, clean sans-serif typography, **programmatic label-collision checks**, rendered as both `.svg` and 300-DPI `.png`). **Mandatory Visual Inspection Gate**: Before final publish, visually inspect BOTH rendered PNG files using your image-reading tool to confirm zero overlapping labels, zero leader lines colliding with data series, clean whitespace placement for IPO callouts, and 100% numeric consistency with `events.csv` and `COVERAGE.md`.

### Chart 1: `output/net_equity_issuance.svg` & `.png`
- **Top Panel (Main S&P 500 Quarterly Series — Full Verified History)**:
  - Plot the **entire survivorship-free verified window** (back to `membership_verified_back_to` — do NOT cap at 2021 if earlier years are verified!). Never plot unverified/survivorship-biased years as solid bars in final Phase 6 charts. **Exclude in-progress incomplete quarters** (such as a partially elapsed current quarter where < 25% of companies have reported — e.g., Q4 in late November) from solid quarterly bars so the chart never terminates on a misleading tiny stub bar; footnote any late-filer lag on the latest completed quarter clearly.
  - Clean rounded-tip quarterly bars (blue `#2a78d6` / coral-red distinction or stacked Gross Issuance & Buybacks with bold Net Issuance series, `$bn per quarter`, prominent `y = 0` line, clear direct data labels on key peaks/troughs and latest quarter).
  - **Direct Mega-IPO Timeline Annotations**: Place subtle vertical dashed markers and crisp callout labels directly on the timeline for major U.S. market IPO events discovered from primary filings/benchmarks — **at minimum SpaceX IPO (Jun 2026) and Anthropic IPO (Nov 2026)**, plus any other top-tier mega-IPO in the window — clearly stating date, scale (e.g., initial market cap & proceeds raised verified directly from primary SEC `424B4` cover pages), and whether it is inside or outside the S&P 500!
- **Bottom Panel (Federal Reserve Z.1 Benchmark)**:
  - S&P 500 Nonfinancial Net Issuance vs. **Federal Reserve Z.1 (`FU103164105.Q`, Table F.5.1.1, not seasonally adjusted, `$bn` per quarter)** across the exact same verified window, with SpaceX / Anthropic timeline markers explaining the 2026 divergence.
- Crisp bottom source footnote citing exact primary SEC filings, S&P DJI releases, Fed Z.1 `FU103164105.Q`, and point-in-time run date.

### Chart 2: `output/acadian_comparison.svg` & `.png`
- **Top Panel (Multi-Methodology Rolling 12-Month Net Issuance)**:
  - Compares all methodologies side-by-side across the full verified window: (1) S&P 500 Cash-Flow Headline (`NI_cash`), (2) S&P 500 SBC-Fair-Value Event Basis (`NI_sbc_fv`), (3) S&P 500 Acadian Eq. (1) Share-Count × Price (`NI_sc` — **break line cleanly at `NaN` gaps**, never connect straight segments across missing quarters!), (4) S&P 500 Float-Adjusted Analog (`NI_float`), and (5) Acadian All-U.S. ex-SpaceX & incl. SpaceX markers — with **explicit SpaceX IPO and Anthropic IPO callouts** kept 100% consistent with `mega_ipos_primary.json` (citing primary `424B4` proceeds, any concurrent private placement disclosed on the cover, and IPO-price vs. Acadian first-day close market cap!).
- **Bottom Panel (Baker-Wurgler Equity Share `E / (E + D)`)**:
  - S&P 500 primary-filing `E / (E + D)` vs. Federal Reserve Board Table 1.46 macro `E / (E + D)` (using U.S.-sold corporate bonds, Table 1.46 line 3) across the full verified window, with SpaceX and Anthropic IPO timeline markers.

## 5. Required Structure of `output/COVERAGE.md`

1. **Executive Summary**: Date range covered, total unique PIT S&P 500 issuers identified vs. covered by tier, event & revision-chain counts, headline trailing-12-month & annual totals across all methodology versions.
2. **S&P 500 PIT Membership & Verification**: Primary sources (dual ETF `N-PORT-P` + S&P DJI releases + SEC successions), additions/removals/mergers/spin-offs/multi-class counts, FDIC-regulated bank disclosures, and exact Wikipedia cross-check match rate.
3. **Coverage Matrix & Explicit Gaps**: Breakdown by year/quarter and by **all** buyback & issuance categories (`open_market`, `asr`, `tender_offer`, `underwritten`, `atm`, `acquisition`, `employee_equity`, `warrant`, `conversion`, plus residual `unspecified`/`other_common` %), plus an explicit list of companies/periods NOT covered before cutoff.
4. **Verification, Web Cross-Checks & Multi-Methodology Reconciliation**:
   - Blocking validator results, FY reconciliation, and PIT `as_of(D)` multi-date test results.
   - Internal share-count identity statistics and top residuals investigated.
   - All Acadian versions comparison table (Cash-flow, SBC-FV, Eq. 1 `$B` & `% mkt cap`, Magnificent Seven, Float-adjusted, Baker-Wurgler `E/(E+D)`), Fed Z.1 (`FU103164105.Q`) comparison, and S&P DJI Quarterly Buybacks cross-check.
   - Summary of `market-verifier` (Yahoo Finance / Google Finance / IPO calendars) and `qa-auditor` spot-checks (records checked, defects found & fixed).
5. **Run Timing Telemetry & Continuous-Run Proof**: Summary table from `phase_timings` (wall-clock duration per phase/batch) + result of the Phase 6 7-day watermark-rollback `--mode incremental` test.
6. **Honest Assessment: What May Be Wrong & Known Limitations**: Candid list of residual data limitations, edge cases, parser weaknesses, and uncertainty bounds.
