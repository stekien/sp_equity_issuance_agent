# SEC EDGAR & Primary Source Extraction Playbook

## 1. SEC EDGAR Endpoints & Access Rules

- **Mandatory Header**: Every request to `sec.gov` must include `User-Agent: <value of $SEC_USER_AGENT>`.
- **Rate Limit**: SEC enforces a 10 req/sec hard cap. Operate at **≤ 8 req/sec globally** across all processes and subagents. Route all EDGAR fetches through your single cached HTTP client in `work/src/`.
- **Core URL Templates**:
  - Submissions index (all filings + exact `acceptanceDateTime`):
    `https://data.sec.gov/submissions/CIK{cik10}.json`
    *(Note: very active filers paginate older filings into `files[].name` under `filings.files` — fetch pagination JSONs if your date window reaches earlier than `filings.recent`!)*
  - XBRL CompanyFacts (all historical XBRL facts in one file):
    `https://data.sec.gov/api/xbrl/companyfacts/CIK{cik10}.json`
  - Filing archive documents & directory index:
    `https://www.sec.gov/Archives/edgar/data/{cik_int}/{accn_no_dashes}/{primary_document}`
    `https://www.sec.gov/Archives/edgar/data/{cik_int}/{accn_no_dashes}/index.json`
  - Full-Text Search (useful for locating specific 8-Ks, S&P additions, ASRs, offerings):
    `https://efts.sec.gov/LATEST/search-index`

## 2. Tier 1: Full-Universe XBRL Baseline (Phase 2)

Fetching `submissions` + `companyfacts` for ~550–650 historical S&P 500 CIKs takes only ~2–3 minutes of network time at 8 req/sec and gives an immediate, primary-sourced baseline across the entire universe back to 2021+.

### Exact Publication Timestamps
`companyfacts` only provides `filed` (YYYY-MM-DD). Always join each fact's `accn` against `submissions` (`accessionNumber` → `acceptanceDateTime`) so `published_at` has exact UTC second-level precision (`YYYY-MM-DDTHH:MM:SSZ`).

### XBRL Tag Precedence & Anti-Double-Counting
A single 10-Q/10-K frequently tags the same economic activity under multiple `us-gaap` concepts (e.g., cash-flow statement AND stockholders' equity rollforward). **Never sum multiple concepts for the same category within the same filing and period.** Apply strict ordered precedence per category:

- **Common Stock Buybacks (USD)**:
  1. `PaymentsForRepurchaseOfCommonStock` (Cash flow — cleanest cash outlay for common buybacks)
  2. `StockRepurchasedDuringPeriodValue` or `StockRepurchasedAndRetiredDuringPeriodValue`
  3. `TreasuryStockValueAcquiredCostMethod`
  4. `PaymentsForRepurchaseOfEquity` (ONLY if verified that preferred stock repurchases are zero or excluded)
- **Common Stock Buybacks (Shares)**:
  1. `StockRepurchasedDuringPeriodShares` or `StockRepurchasedAndRetiredDuringPeriodShares`
  2. `TreasuryStockSharesAcquired`
- **Underwritten / New Common Stock Issuance (USD & Shares)**:
  - USD: `ProceedsFromIssuanceOfCommonStock`, `StockIssuedDuringPeriodValueNewIssues`
  - Shares: `StockIssuedDuringPeriodSharesNewIssues`
  - *(Verify against 10-Q/10-K/424B in Tier 2 to classify underwritten vs. ATM and exclude any preferred/debt proceeds.)*
- **Acquisition Share Issuance (USD & Shares)**:
  - USD: `StockIssuedDuringPeriodValueAcquisitions`
  - Shares: `StockIssuedDuringPeriodSharesAcquisitions`
- **Employee Equity Plans — Options, RSUs/PSUs, ESPP (USD & Shares)**:
  - Cash proceeds tags: `ProceedsFromStockOptionsExercised`, `ProceedsFromIssuanceOfSharesUnderIncentiveAndShareBasedCompensationPlansIncludingStockOptions`, `ProceedsFromStockPlans`
  - Equity issuance value/share tags: `StockIssuedDuringPeriodValueShareBasedCompensation`, `StockIssuedDuringPeriodSharesShareBasedCompensation`, `StockIssuedDuringPeriodSharesStockOptionsExercised`, `StockIssuedDuringPeriodSharesEmployeeStockPurchasePlans`, `StockIssuedDuringPeriodSharesRestrictedStockAwardNetOfForfeitures`
  - **RSU Tax Withholding (`PaymentsRelatedToTaxWithholdingForShareBasedCompensation`)**: Track separately under `tax_withholding_net_settlement` (see `docs/SCHEMA_AND_RULES.md`) so net-share settlement of vested RSUs is neither missed nor silently conflated with open-market buybacks.
- **Warrant Exercises & Conversions (USD & Shares)**:
  - `ProceedsFromWarrantExercises`, `StockIssuedDuringPeriodSharesWarrantsExercised`, `StockIssuedDuringPeriodValueWarrantsExercised`
  - `StockIssuedDuringPeriodSharesConversionOfConvertibleSecurities`, `StockIssuedDuringPeriodValueConversionOfConvertibleSecurities`
- **Shares Outstanding (for Reconciliation Check)**:
  - `EntityCommonStockSharesOutstanding` (dei) and `CommonStockSharesOutstanding` (us-gaap).

### Critical XBRL Pitfalls You Must Handle
1. **YTD Cumulative Cash-Flow De-cumulation**:
   - Cash-flow XBRL facts on Q2 and Q3 10-Qs are almost always **cumulative YTD** (6-month or 9-month duration), and 10-K facts are **12-month FY**.
   - Classify each fact's duration in days (`(end - start).days`): ~75–105d = standalone quarter; ~165–200d = 6M YTD; ~255–290d = 9M YTD; ~350–380d = FY.
   - Prefer an explicit standalone-quarter fact when present in the same filing. Otherwise derive standalone quarters strictly within the same fiscal year: `Q2 = 6M - Q1`, `Q3 = 9M - 6M`, `Q4 = FY - 9M`.
   - **Point-in-time rule for derived quarters**: A derived Q4 row uses `published_at` equal to the **10-K's `acceptanceDateTime`** (the earliest moment both FY and 9M were known), sets `extraction_method = calculated`, and records both accession numbers and the exact subtraction formula in `calculation_formula`.
2. **Non-Calendar Fiscal Years & 52/53-Week Calendars**:
   - Many large S&P 500 firms end fiscal years in January, June, or September, or use 52/53-week retail calendars. Always store exact calendar `period_start` and `period_end` dates and aggregate into calendar quarters/months consistently (documenting your date-assignment rule, e.g., assigning by `period_end` or midpoint).
3. **Comparative Columns vs. Genuine Revisions**:
   - A 2026 Q1 10-Q reports both Q1 2026 and comparative Q1 2025. Compare the comparative Q1 2025 value against the value originally reported in the 2025 Q1 10-Q:
     - If identical (within rounding tolerance, e.g., <0.1% and <$1M), do NOT emit a duplicate row.
     - If materially revised (restatement, reclassification, or post-split share restatement), **keep both rows** linked by `event_key`, set `is_revision = true` and `supersedes_record_id`, and note whether the change is a substantive revision or a post-split restatement!
4. **Sign Conventions in XBRL**:
   - `PaymentsForRepurchaseOfCommonStock` is usually reported as a positive cash outflow (though some filers occasionally tag it negative). Inspect sign polarity carefully before normalizing into `amount_usd > 0` and `net_amount_usd < 0`.

## 3. Tier 2: High-Resolution Primary Filing Extraction (Phases 3 & 4)

Prioritize companies by **materiality** (latest `N-PORT-P` index weight + Tier 1 total gross equity volume) so the largest buybackers and issuers are upgraded to full granularity first. Process the **trailing 12 months first (Phase 3)**, then backfill backwards year-by-year toward 2021 (Phase 4).

### A. Monthly Buyback Tables (10-Q Part II Item 2 & 10-K Part II Item 5)
SEC regulations require issuers to report monthly common stock repurchases in a standardized table ("Issuer Purchases of Equity Securities") inside every 10-Q (Item 2) and 10-K (Item 5).
1. Locate the table in the filing HTML by searching for Item 2 / Item 5 headings and table headers containing `"Total Number of Shares Purchased"` and `"Average Price Paid per Share"`.
2. Extract each individual **monthly** row (skipping the "Total" summary row!): month date range (`period_start`, `period_end`), shares purchased, and average price paid per share.
3. **Important footnote checks**:
   - Check whether the table includes **employee tax-withholding shares** (often explained in a footnote like *"Includes X shares withheld to satisfy tax obligations"* or excluded in the "Publicly Announced Plans" column). Use the appropriate column/note consistently and document your rule.
   - Check share-count scaling units in the table header (`in thousands`, `in millions`, or exact shares) — a 1,000x unit bug is one of the easiest ways to corrupt aggregates! Always sanity-check `sum(monthly shares * avg_price)` against Tier 1 XBRL quarterly buyback dollars (`PaymentsForRepurchaseOfCommonStock`). If the ratio is near `0.001` or `1000`, re-check table unit headers!
4. Compute `amount_usd = shares * avg_price_usd` (`extraction_method = calculated`, `valuation_basis = shares_times_avg_price`).
5. When clean monthly buyback rows are extracted for a quarter (and reconcile reasonably with XBRL), mark the monthly rows as `include_in_net_aggregate = true` and flip the corresponding Tier 1 quarterly XBRL fallback row to `include_in_net_aggregate = false` (never double-count!).

### B. Underwritten & At-the-Market (ATM) Equity Offerings (`424B*`, `8-K`, `S-3`)
1. **CRITICAL FILTER — Bank Structured-Note / Debt `424B2` Spam**:
   - Large financial institutions file thousands of `424B2` pricing supplements for structured notes, medium-term notes, and debt securities.
   - **Never** treat `424B2` filings as common equity by default! Pre-filter all `424B*` filings by checking `primaryDocDescription` and scanning the cover page (first ~15 KB of HTML) for common stock offering language (`"Shares of Common Stock"`, `"At-the-Market"`, `"Equity Distribution Agreement"`, `"Public Offering"`, `"Underwriting Agreement"`) while explicitly rejecting debt/note/preferred-only offerings (`"Senior Notes"`, `"Medium-Term Notes"`, `"Structured Notes"`, `"Fixed Rate Notes"`, `"Depositary Shares"`, `"Preferred Stock"` without common stock).
2. **CRITICAL FILTER — Secondary vs. Primary Shares**:
   - Secondary offerings (shares sold by existing selling stockholders) raise **$0** for the company and create **0** new shares. Do NOT record secondary-only offerings as equity issuance!
   - For mixed offerings (part primary, part secondary), extract **only the primary shares** issued by the company and note the excluded secondary portion in `notes`.
3. Capture offering pricing/announcement date, closing date, shares issued (including overallotment/greenshoe exercise if announced in a subsequent `8-K` or `424B`), offering price, gross/net proceeds, and exact `acceptanceDateTime`.

### C. Accelerated Share Repurchases (ASRs) & Tender Offers (`8-K`, `SC TO-I`, Footnotes)
- **ASRs**: Search `8-K` filings and 10-Q/10-K equity footnotes for `"accelerated share repurchase"` or `"ASR"`. Record the upfront cash payment and initial shares delivered when executed/disclosed, and record final share settlement adjustments when settled—**never count the upfront cash twice** at settlement!
- **Tender Offers**: Check `submissions` for `SC TO-I` and final `SC TO-I/A` amendments (and accompanying `8-K` final results press releases). Extract final shares accepted for purchase and final purchase price from the final results filing; preserve preliminary announcements separately only if no final result was filed yet.

### D. Acquisitions (`S-4`, `8-K` Item 2.01/3.02), Conversions, Warrants & Employee Equity
- Cross-check large XBRL spikes in `StockIssuedDuringPeriodSharesAcquisitions`, `StockIssuedDuringPeriodValueNewIssues`, or `StockIssuedDuringPeriodSharesConversionOfConvertibleSecurities` against `8-K` (Items 2.01 / 3.02) and `S-4` / 10-Q footnotes to obtain exact transaction dates, share counts, and fair value at closing.
- For **Employee Equity Plans**, reconcile Tier 1 XBRL proceeds vs. share-based issuance counts and apply the consistent valuation rule defined in `docs/SCHEMA_AND_RULES.md`.

## 4. Tier 3: Subagent Inspection (Phase 5)

Whenever deterministic parsers encounter high-dollar anomalies (e.g., unexplained quarterly share-count discrepancy > $500M, unit ambiguity, complex ASR collar, spin-off, or mixed primary/secondary prospectus), dispatch `.claude/agents/filing-analyst.md` on the **already-cached** filing in `work/cache/` to extract verified structured facts.
