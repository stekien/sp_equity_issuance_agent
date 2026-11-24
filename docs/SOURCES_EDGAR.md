# SEC EDGAR & Primary Source Extraction Playbook

## 1. EDGAR Endpoints & Access Discipline

- **Host-Scoped Header**: Send `User-Agent: <$SEC_USER_AGENT>` **only** to `*.sec.gov` hosts.
- **Cross-Process Rate Limit**: ≤ 8 requests/sec globally via `work/src/`'s `fcntl`-locked token bucket and disk cache. Never fetch any URL twice.
- **Core URL Templates**:
  - Submissions index (and `filings.files[]` pagination for high-volume filers): `https://data.sec.gov/submissions/CIK{cik10}.json`
  - XBRL CompanyFacts (historical non-dimensional facts): `https://data.sec.gov/api/xbrl/companyfacts/CIK{cik10}.json`
  - Filing archives & directory index (for HTML tables, iXBRL instance documents, exhibits): `https://www.sec.gov/Archives/edgar/data/{cik_int}/{accn_no_dashes}/{doc}` and `.../index.json`
  - SEC Financial Statement and Notes Bulk Data Sets (quarterly zip archives containing **dimensional** XBRL facts including `StatementEquityComponentsAxis` and `ClassOfStockDomain`): `https://www.sec.gov/dera/data/financial-statement-and-notes-data-sets`
  - Full-Text Search (locating `8-K`s, ASRs, ATM programs, S&P additions, M&A closings): `https://efts.sec.gov/LATEST/search-index`

## 2. Tier 1: Full-Universe XBRL & Dimensional Equity-Statement Extraction (Phase 2)

### Exact Publication Timestamps
`companyfacts` carries only `filed` (date). Always join each fact's `accn` against `submissions` (`accessionNumber` → `acceptanceDateTime`, verified true UTC) so `published_at` has second-level UTC precision (`YYYY-MM-DDTHH:MM:SSZ`).

### CRITICAL LESSON: Never Blindly Trust Non-Dimensional Total-Equity XBRL Facts!
In `companyfacts`, non-dimensional `StockIssuedDuringPeriodValue*` facts frequently represent the **Total Equity** column of the Statement of Stockholders' Equity—silently sweeping in **preferred stock issuances**, **mandatory convertible preferreds**, **noncontrolling interests (NCI)**, and **subsidiary equity transactions**!
Apply **both** structural defenses from Phase 2 onward:
1. **Dimensional Equity-Statement Filtering (Preferred Method for Large Flows & Issuers with Preferred/NCI)**:
   - For filings with large issuance flows (> $250M) or any issuer reporting preferred stock / noncontrolling interest lines, inspect the **iXBRL instance** (or SEC quarterly Financial Statement & Notes bulk dataset) along `StatementEquityComponentsAxis` and `ClassOfStockDomain`, summing ONLY **Common Stock + Additional Paid-in Capital (parent common)** columns and excluding `PreferredStockMember` and `NoncontrollingInterestMember`.
   - Dimensional parsing also recovers large flows tagged ONLY across separate par-value + APIC columns or per-class share counts (e.g., dual-class tech issuers that omit consolidated `CommonStockSharesOutstanding`).
2. **Automated Cross-Tag Sanity Filters (Always Active)**:
   - **Preferred-stock filter**: Flag/drop any `NewIssues`, `Conversion`, or `Acquisitions` USD fact that matches (within ~5%) same-period preferred stock issuance proceeds (`ProceedsFromIssuanceOfPreferredStockAndPreferenceStock` / `StockIssuedDuringPeriodValuePreferredStock`) unless primary filing confirms common stock.
   - **NCI subtraction**: Compare `StockIssuedDuringPeriodValueAcquisitions` against `MinorityInterestIncreaseFromBusinessCombination` in the same period; subtract confirmed NCI component (or drop if ≥ 95% NCI) and verify large deals against the filing.

### Ordered XBRL Tag Precedence & Concept-Twin Guard
Never sum overlapping Cash Flow and Stockholders' Equity concepts for the same flow in the same period! Apply strict precedence + overlap-aware selection:
- **Buybacks (USD)**: `PaymentsForRepurchaseOfCommonStock` ≻ `StockRepurchasedAndRetiredDuringPeriodValue` ≻ `StockRepurchasedDuringPeriodValue` ≻ `TreasuryStockValueAcquiredCostMethod` (reject cumulative multi-year program totals tagged as period facts—verify against annual cash flow!).
- **Buybacks (Shares)**: `StockRepurchasedAndRetiredDuringPeriodShares` ≻ `StockRepurchasedDuringPeriodShares` ≻ `TreasuryStockSharesAcquired`.
- **New Issues / Offerings**: `ProceedsFromIssuanceOfCommonStock` vs. `StockIssuedDuringPeriodValueNewIssues` (never count both for the same period; check whether `NewIssues` duplicates `ProceedsFromStockOptionsExercised` by comparing exact/near-matching dollars or shares and verifying against the filing rather than using an arbitrary multiplier!).
- **Acquisitions**: `StockIssuedDuringPeriodValueAcquisitions` / `StockIssuedDuringPeriodSharesAcquisitions`.
- **Employee Equity Plans**:
  - Cash proceeds basis: `ProceedsFromStockOptionsExercised`, `ProceedsFromIssuanceOfSharesUnderIncentiveAndShareBasedCompensationPlansIncludingStockOptions`, `ProceedsFromStockPlans`.
  - Market/vest-date fair value basis (for SBC-fair-value comparison variant): `ShareBasedCompensationArrangementByShareBasedPaymentAwardEquityInstrumentsOtherThanOptionsVestedInPeriodTotalFairValue` (RSU/PSU vest-date fair value) + option exercise intrinsic/proceeds value (prefer vest-date fair value over accounting SBC expense!).
  - Tax withholding (always tracked separately under `tax_withholding_net_settlement`): `PaymentsRelatedToTaxWithholdingForShareBasedCompensation`.
- **Warrants & Conversions**: `ProceedsFromWarrantExercises`, `StockIssuedDuringPeriodSharesWarrantsExercised`, `StockIssuedDuringPeriodValueWarrantsExercised`, `StockIssuedDuringPeriodSharesConversionOfConvertibleSecurities`, `StockIssuedDuringPeriodValueConversionOfConvertibleSecurities`.
- **Debt Issuance (for Acadian Figure 3 Baker-Wurgler Equity Share `E/(E+D)`)**: `ProceedsFromIssuanceOfLongTermDebt`, `ProceedsFromIssuanceOfDebt`, `ProceedsFromIssuanceOfSeniorLongTermDebt`.
- **Shares Outstanding (for Share-Count Reconciliation & Acadian Eq. 1)**: Collect `CommonStockSharesOutstanding` (us-gaap), `EntityCommonStockSharesOutstanding` (dei cover page), AND **sum all common share classes** from `10-Q`/`10-K` cover pages or iXBRL `ClassOfStockDomain` for dual-class/multi-class issuers (and cover-page exact counts when balance-sheet counts are rounded to 0.1B) so NO major constituent (especially any Magnificent Seven member!) is dropped from share-count reconciliation or Acadian Eq. (1)!

### Context Validation, YTD De-cumulation & Blocking FY Reconciliation
1. **Strict Context Validation**:
   - Accept a direct ~3-month fact `[a, b]` ONLY when `b` and `a − 1` align (±3 days) with actual report dates of the issuer's `10-Q`/`10-K`/`10-QT` filings (or `a` equals fiscal-year start; explicitly support `10-QT` transition quarters and 12–17 week retail quarters!).
   - Reject 3-month contexts reported inside a `10-K` unless they correspond to the fiscal-year-end quarter (prevents mis-tagged full-year figures on Q3/Q4 contexts!).
   - Reject a 3-month fact whose value equals the same filing's YTD value when an earlier quarter of that fiscal year was non-zero.
2. **Vintage-Consistent YTD De-cumulation & Sign Normalization (Take Absolute Value BEFORE Subtracting!)**:

```text
Q_k = |YTD_v(s, e_k)| - |YTD_v(s, e_{k-1})|   # same fiscal_year_start s, same filing vintage v!
published_at(Q_k) = max(published_at(YTD_k), published_at(YTD_{k-1}))
```

   - Always take absolute values **before** de-cumulating buyback YTD pairs (prevents sign-inconsistent pairs from summing instead of subtracting!).
   - Always pair YTD values from the **same filing vintage** (never subtract an earlier-vintage 9M YTD from a later spin-off/re-presented 10-K FY, which creates phantom revision spikes!). Only emit a revision when an amended filing (`10-Q/A`, `10-K/A`) or genuine comparative restatement changes the same period within consistent accounting scope.
   - Discard negative "new issue" values (accounting reversals with $0 cash proceeds, not equity issuance).
   - Every derived quarter (`Q2`, `Q3`, `Q4`) sets `extraction_method = calculated`, carries the `acceptanceDateTime` of the later filing needed to compute it, and records exact accession numbers and subtraction formula in `calculation_formula`.
3. **Blocking Fiscal-Year Identity (`Σ quarters == FY`)**:
   - For every issuer, fiscal year, and concept family, verify that the sum of included quarterly/monthly rows does not exceed the 10-K FY total (within tolerance) — make FY over-counting a **blocking validator check** from Milestone 1!
4. **Comparative Re-Presentation vs. Real Revisions**:
   - Never assume a comparative value of `0` in a later filing means the prior period was revised to zero (filers often re-label concepts across years!). Only record a revision when the comparative fact in the same concept/line genuinely changes or an amended filing (`10-Q/A`, `10-K/A`) restates it.

## 3. Tier 2: Comprehensive Category Extraction & Golden Test Corpus (Phases 3 & 4)

### Mandatory Golden Test Corpus & Raw-Parse Separation
Before running any full-universe Tier 2 batch:
1. Build a **golden test set of ~30–50 diverse cached filings** spanning tricky layouts (multi-row headers, `<br>` inside headers, zero-width spaces, per-class buyback columns, ASR sub-rows, `"in thousands"`/`"in millions"` units, date typos, preliminary `"subject to completion"` prospectuses, forward-sale agreements, secondary-only offerings, depositary/preferred covers, split-off `SC TO-I` exchanges, and ATM footnotes).
2. Store **raw parsed candidates in `raw_extractions`** separately from final inclusion/acceptance decisions so any rule improvement re-runs in seconds without re-parsing HTML!

### Dedicated Extraction for Every Required Buyback & Issuance Category
Do NOT allow large flows to sit lazily in `buyback_unspecified` or `issuance_other_common`! Target **all** categories systematically (prioritizing trailing 12 months first, ordered by materiality, then backfilling to 2021):

1. **Open-Market Buybacks (`buyback_open_market`) — Monthly Tables (`10-Q` Part II Item 2 & `10-K` Part II Item 5)**:
   - Match headers whitespace-insensitively (`"average ... price"`, `"shares ... purchased"`). Handle multi-row headers, sub-rows under period headers, per-class columns (`Σ shares_k * price_k`), and skip summary `"Total"` / `"Nine months ended"` rows.
   - Prefer column (c) `"purchased as part of publicly announced plans"` when populated (excluding tax withholding); fall back to column (a) with explicit note if column (c) is absent. Exclude unpriced ASR delivery rows from monthly cash sums so prepaid ASR cash isn't double-counted.
   - Infer share unit scale (`1`, `10³`, `10⁶`) by matching `Σ(shares * avg_price)` against the covering Tier 1 XBRL quarterly buyback total:

```text
scale = argmin_{s in {1, 1e3, 1e6}} |ln(sum(shares * avg_price) * s / XBRL_quarter_buyback)|
Accept iff ratio in [0.60, 1.60] (or [0.90, 1.10] when ASR rows are present)
and every implied monthly price in [$1, $20,000] (except verified ultra-high-price share classes)
```
2. **Accelerated Share Repurchases (`buyback_asr`) — `8-K` & `10-Q/10-K` Repurchase Footnotes**:
   - Scan `8-K` filings (`Item 8.01` / `1.01`) and `10-Q`/`10-K` repurchase footnotes for `"accelerated share repurchase"` or `"ASR"` (especially on every quarter where monthly tables didn't cover XBRL buybacks!).
   - Record upfront execution cash payment & initial share delivery at announcement/execution `published_at`, and link final settlement share adjustment as a revision row on the same `event_key` (**never double-count upfront cash at settlement!**).
3. **Tender Offers & Split-Off Exchanges (`buyback_tender_offer`) — `SC TO-I`, `SC TO-I/A`, `8-K`**:
   - Scan ALL `SC TO-I` and `SC TO-I/A` filings for active S&P 500 members. Filter out debt puts, option exchanges, and preferred-only tenders.
   - Distinguish **cash self-tenders / Dutch auctions** from **non-cash split-off exchange offers** (note clearly in `notes`). Link initial tender offer announcement (`SC TO-I`) → final results amendment (`SC TO-I/A` / `8-K`) under one `event_key`.
4. **Underwritten Offerings (`issuance_underwritten_offering`) — `424B1/3/4/5`, `8-K`, `S-3`**:
   - **Bank Structured-Note Filter**: Exclude high-volume bank `424B2`/`424B3` structured-note/debt supplements by per-filer volume + cover-page screening (`Senior Notes`, `Medium-Term Notes`, `Structured Notes`, `Depositary Shares`, `Preferred Stock` without common stock).
   - **Strict Cover-Page Filters**: Reject `"subject to completion"` / preliminary supplements (never priced or never closed!), reject **secondary-only** selling-stockholder offerings (extract ONLY primary shares in mixed deals!), and do NOT count **forward-sale agreements** until physical settlement actually issues shares (or link announcement → settlement explicitly with `include_in_net_aggregate = false` until settled!).
   - Link `424B5`/`424B4` pricing → `8-K` closing/greenshoe exercise → `10-Q`/`10-K` confirmation under a single `event_key`!
5. **At-the-Market Offerings (`issuance_atm_offering`) — `424B5` + `10-Q/10-K` Equity Footnotes**:
   - Note that an ATM `424B5` prospectus supplement only registers *program capacity*, NOT actual sales!
   - Systematically search `10-Q` and `10-K` Stockholders' Equity / Liquidity footnotes (especially for utilities, REITs, industrials, and large AI-capex issuers, plus every issuer with non-zero `ProceedsFromIssuanceOfCommonStock` or `issuance_other_common`) for `"at-the-market"`, `"ATM"`, `"equity distribution agreement"`, `"sales agreement"`, and `"continuous equity offering"` to extract **actual shares sold and gross/net proceeds** per quarter!
6. **Acquisition Share Issuance (`issuance_acquisition`) — `S-4`, `8-K` Item 2.01/3.02, M&A Footnotes**:
   - Extract exact closing date, shares issued, and closing-date fair value for all material stock-for-stock acquisitions (cross-checking against share-count reconciliation spikes to catch company-extension XBRL tags!).
7. **Employee Equity Plans (`issuance_employee_equity`), Warrants (`issuance_warrant_exercise`) & Conversions (`issuance_conversion`)**:
   - Extract employee equity cash proceeds + vest-date fair value (for SBC-adjusted views), and scan `8-K` Item 3.02 & `10-Q`/`10-K` equity/debt footnotes for material warrant exercises and convertible-debt / preferred conversions.
   - **Reclassification Gate**: Before Phase 6, audit and reclassify every remaining `issuance_other_common` or `buyback_unspecified` row exceeding **$500M** in the trailing 12 months (and top historical rows) via `filing-analyst` so vague fallback buckets remain minimal.
