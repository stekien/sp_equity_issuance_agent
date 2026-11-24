# Bi-Temporal Schema, Taxonomy, and Accounting Rules

## 1. Scope & Security Definition

- **Universe**: Every company that was a member of the S&P 500 at any point during the covered period, counted **only while it was an active member**.
- **Security Type**: **Common stock only** for the core S&P 500 net equity issuance series (`security_type = common_stock`). Preferred stock, straight debt, and unconverted convertible bonds are excluded (convertible debt or preferred stock enters ONLY when converted into common stock, under `issuance_conversion`).
- If you separately record cash M&A retirements of departing S&P 500 members (`retirement_cash_merger`) or RSU net-share tax withholdings (`tax_withholding_net_settlement`) for analytical completeness and benchmark bridging, tag them cleanly with their dedicated categories and document clearly in `output/README.md` and `COVERAGE.md` how they are treated in headline vs. supplemental aggregates.

## 2. Controlled Category Taxonomy

Apply these exact `category` values consistently across every row:

### Buybacks (`direction = buyback`, `net_amount_usd < 0`)
- `buyback_open_market` — Open-market common stock repurchases (including monthly 10-Q/10-K Item 2/5 tables when excluding ASRs/tenders).
- `buyback_asr` — Accelerated share repurchase agreements (upfront execution and final share true-up).
- `buyback_tender_offer` — Issuer self-tender offers (`SC TO-I` / `8-K`).
- `buyback_unspecified` — Aggregate common stock repurchases from XBRL cash-flow / equity statements where method breakdown was not yet parsed or not separately disclosed.
- `tax_withholding_net_settlement` — Shares withheld by the company upon RSU/PSU vesting to satisfy employee payroll tax obligations (`PaymentsRelatedToTaxWithholdingForShareBasedCompensation`). Keep distinct from discretionary buybacks!

### Issuance (`direction = issuance`, `net_amount_usd > 0`)
- `issuance_underwritten_offering` — Primary underwritten follow-on offerings, public offerings, or direct placements of common stock (excluding secondary selling-stockholder shares).
- `issuance_atm_offering` — Primary shares sold under At-the-Market equity distribution programs.
- `issuance_acquisition` — Common shares issued as consideration in mergers and acquisitions.
- `issuance_employee_equity` — Common shares issued under employee equity plans (stock option exercises, RSU/PSU vesting/settlement, ESPP).
- `issuance_warrant_exercise` — Common shares issued upon warrant exercises.
- `issuance_conversion` — Common shares issued upon conversion of convertible debt or convertible preferred stock.
- `issuance_other_common` — Other primary common equity issuances (e.g., DRIP, debt-for-equity exchange, legal settlement shares).

### Supplemental Corporate Action (Optional, Clearly Separated)
- `retirement_cash_merger` — Equity retired when an S&P 500 constituent is acquired for cash / taken private. Set `direction = retirement`, `net_amount_usd < 0`, and document separately in `COVERAGE.md` when bridging to Fed Z.1 / Acadian (which include cash M&A retirements).

## 3. Consistent Valuation Rules

Choose a clear, economically consistent valuation hierarchy and record `valuation_basis` on every row:
1. **Buybacks**: Actual cash paid (`cash_paid`) or `shares * avg_price_usd` (`shares_times_avg_price`). Exclude excise tax if separately broken out (or note if included as reported).
2. **Offerings (`underwritten`, `atm`)**: Gross offering proceeds (`gross_proceeds`) if stated, or `shares * offering_price` (`shares_times_offer_price`); if only net proceeds after underwriting discount are reported, record `net_proceeds` and note it.
3. **Acquisitions & Conversions**: Fair value of common shares issued at closing/conversion (`fair_value_at_issuance`) as stated in the filing (`StockIssuedDuringPeriodValueAcquisitions` / M&A footnote).
4. **Employee Equity Plans (`issuance_employee_equity`) — CRITICAL CONSISTENCY RULE**:
   - Cash proceeds from option exercises and ESPP (`cash_proceeds`) capture actual cash inflows, whereas RSUs/PSUs issue real shares with **$0 cash proceeds**.
   - Be explicit and consistent! **Recommended default** (matching economic market value and Fed/Acadian equity flow concepts while remaining primary-source auditable):
     - Record **cash proceeds** (`cash_proceeds`) where disclosed for options/ESPP, AND/OR value RSU/SBC share issuances using filing-disclosed fair value (`fair_value_at_issuance` / `StockIssuedDuringPeriodValueShareBasedCompensation` or `shares * filing_weighted_avg_fair_value`).
     - Whichever convention you apply across the universe, apply it **uniformly** across all companies, never double-count options + RSUs, and document the exact treatment and its dollar impact in `output/README.md` and `COVERAGE.md` (including showing both cash-flow-basis net issuance and SBC-fair-value-adjusted net issuance in `COVERAGE.md`!).

## 4. Point-in-Time Revisions & Anti-Double-Counting Rules

1. **Canonical `event_key`**: Construct a deterministic key identifying the underlying economic event:
   `{cik}|{category}|{share_class}|{granularity}|{period_start}|{period_end}|{sub_id}`
   (where `sub_id` distinguishes multiple discrete offerings/ASRs on the same date, or `base` for periodic aggregates).
2. **Never Overwrite Revisions**: If a later filing (e.g., `10-Q/A`, `10-K`, or comparative prior-period column) reports a different number for the same `event_key`, insert a **new row** with:
   - New unique `record_id`
   - The new filing's `published_at` and `source_accession_number`
   - `is_revision = true` and `supersedes_record_id = <prior record_id>`
   - **Keep the original row intact.**
3. **Point-in-Time Query Semantics**: For any historical date `D`, a client reconstructs what was known on `D` by filtering `published_at <= D` and selecting the row with the latest `published_at` for each `event_key` (and `include_in_net_aggregate = true` at that granularity).
4. **Granularity Superseding (Monthly vs. Quarterly Fallback)**:
   - When Tier 2 extracts monthly buyback rows (or specific transaction rows) covering a quarter, set `include_in_net_aggregate = true` on the granular rows and `include_in_net_aggregate = false` on the overlapping Tier 1 `quarterly` XBRL fallback row (with `notes` referencing the granular replacement).
   - Never allow both a quarterly XBRL total and its constituent monthly rows to have `include_in_net_aggregate = true` simultaneously!

## 5. Stock Split Policy

- **Preserve raw as-reported share counts** (`split_adjusted = false`) from each filing as it was published on `published_at`, so point-in-time queries reflect exact historical reported figures.
- If a later comparative filing restates historical shares on a post-split basis, mark that revision row `split_adjusted = true` and explain the split ratio and effective date in `split_adjustment_note` (e.g., `10-for-1 split effective 2024-06-10; restated in FY2024 10-K`).
- For untouched raw rows, explicitly populate `split_adjustment_note` as `unadjusted as reported on filing date` (never leave split handling undocumented!).

## 6. Exact CSV Column Specification (`output/events.csv`)

UTF-8 encoded, comma-delimited, standard RFC 4180 quoting, header row required, sorted deterministically by `event_date`, `cik`, `category`, `published_at`:

| Column | Type / Format | Required | Description |
|---|---|---|---|
| `record_id` | string | Yes | Unique deterministic identifier for this observation row. |
| `event_key` | string | Yes | Canonical identifier grouping all revisions of the same underlying event. |
| `cik` | string (10-digit) | Yes | Zero-padded SEC CIK of the issuer. |
| `ticker` | string | Yes | Primary ticker symbol as of `event_date` (blank only if unlisted/post-delisting edge case). |
| `company_name` | string | Yes | Issuer legal/filing name. |
| `share_class` | string | Yes | Share class (e.g., `Common`, `Class A`, `Class B`, `All Common Classes`). |
| `security_type` | string | Yes | `common_stock` (default). |
| `is_financial` | boolean (`true`/`false`) | Yes | Whether issuer is in the financial sector (for Fed Z.1 nonfinancial comparison). |
| `sp500_member_on_event_date` | boolean (`true`/`false`) | Yes | True if issuer was an active S&P 500 member during the event/period. |
| `category` | enum string | Yes | Controlled category from Section 2. |
| `direction` | enum string | Yes | `issuance`, `buyback`, or `retirement`. |
| `granularity` | enum string | Yes | `transaction`, `monthly`, `quarterly`, or `annual`. |
| `event_date` | `YYYY-MM-DD` | Yes | Economic transaction/pricing/closing date, or `period_end` for periodic rows. |
| `period_start` | `YYYY-MM-DD` | Conditional | Start date of activity period (required for `monthly`/`quarterly`/`annual`; equals `event_date` for single-day transactions). |
| `period_end` | `YYYY-MM-DD` | Conditional | End date of activity period. |
| `shares` | decimal | Conditional | Positive share count (blank only if source disclosed dollars without shares). |
| `avg_price_usd` | decimal | Conditional | Per-share price in USD (disclosed or implied `amount_usd / shares`). |
| `amount_usd` | decimal | Yes | Positive USD dollar amount of the event. |
| `net_amount_usd` | decimal | Yes | Signed USD amount: `+amount_usd` for issuance, `-amount_usd` for buybacks/retirements. |
| `valuation_basis` | string | Yes | How `amount_usd` is measured (`cash_paid`, `shares_times_avg_price`, `gross_proceeds`, `net_proceeds`, `fair_value_at_issuance`, `cash_proceeds`). |
| `split_adjusted` | boolean (`true`/`false`) | Yes | Whether `shares` and `avg_price_usd` reflect a post-event stock split adjustment. |
| `split_adjustment_note` | string | Yes | Explicit statement of split treatment (ratio/date if adjusted, or `unadjusted as reported on filing date`). |
| `published_at` | ISO-8601 UTC | Yes | Earliest publication timestamp of this source (`YYYY-MM-DDTHH:MM:SSZ` from SEC `acceptanceDateTime`, or `YYYY-MM-DD` if time unavailable). |
| `published_at_precision` | enum string | Yes | `second`, `minute`, or `date`. |
| `source_type` | enum string | Yes | `sec_10q`, `sec_10k`, `sec_8k`, `sec_424b`, `sec_s4`, `sec_sc_to_i`, `sec_other`, `company_announcement`, `sp_announcement`. |
| `source_accession_number` | string | Conditional | SEC accession number (`0000000000-00-000000`), blank only for non-SEC primary announcements. |
| `source_url` | URL | Yes | Direct URL to the primary filing or announcement. |
| `source_location` | string | Yes | Exact XBRL concept+context, HTML table/item, or footnote/section reference. |
| `extraction_method` | enum string | Yes | `direct` (taken verbatim from source) or `calculated` (derived via arithmetic). |
| `calculation_formula` | string | Conditional | Exact formula and inputs when `extraction_method = calculated` (blank when `direct`). |
| `is_revision` | boolean (`true`/`false`) | Yes | True if this row revises an earlier `published_at` observation of the same `event_key`. |
| `supersedes_record_id` | string | Conditional | Prior `record_id` superseded by this revision. |
| `include_in_net_aggregate` | boolean (`true`/`false`) | Yes | True for the active non-overlapping latest-revision rows used in headline S&P 500 net issuance totals (false for superseded revisions, replaced quarterly fallbacks, or out-of-scope supplemental rows). |
| `notes` | string | Optional | Concise explanatory notes (e.g., secondary shares excluded, partial-quarter membership, ASR initial delivery). |

## 7. Required `output/README.md`

Write a clear, concise `output/README.md` covering:
1. Quick summary of files in `output/`.
2. Complete column dictionary for `events.csv`.
3. Category definitions, sign conventions, valuation policy (especially employee equity & tax withholdings), and split-adjustment policy.
4. How to run a **point-in-time query** for any historical `as_of_date` (filtering `published_at <= as_of_date` and taking the latest `published_at` per `event_key`).
5. How to run the incremental pipeline in `work/src/` to pick up newly filed SEC documents going forward.
