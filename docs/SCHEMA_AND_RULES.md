# Bi-Temporal Schema, Taxonomy, and Accounting Rules

## 1. Scope & Security Definition

- **Universe**: Every company that was an S&P 500 member at any point during the covered period, counted **only while it was an active member**.
- **Security Type**: **Common stock only** (`security_type = common_stock`) in headline S&P 500 net equity issuance. Preferred stock, straight debt, and unconverted convertible bonds are excluded (convertibles enter ONLY upon conversion into common stock under `issuance_conversion`).
- Supplemental flows (`tax_withholding_net_settlement`, `retirement_cash_merger`, and SBC vest-date fair value `sub_id = sbc_fair_value`) are recorded in `events.csv` with explicit categories and `include_in_net_aggregate = false` in the cash-flow headline so users and benchmarks can toggle them cleanly without ambiguity.

## 2. Controlled Category Taxonomy

Apply these exact `category` values consistently across every row:

### Buybacks (`direction = buyback`, `net_amount_usd < 0`)
- `buyback_open_market` — Open-market common stock repurchases (primarily monthly `10-Q`/`10-K` Item 2/5 tables).
- `buyback_asr` — Accelerated share repurchase agreements (upfront execution and linked final settlement share true-up).
- `buyback_tender_offer` — Issuer self-tender offers and split-off exchange offers (`SC TO-I` / `SC TO-I/A` / `8-K`, noting cash tender vs. split-off exchange in `notes`).
- `buyback_unspecified` — Aggregate common repurchases from XBRL cash-flow statements where monthly/method breakdown could not be extracted (minimize via Tier 2/3!).
- `tax_withholding_net_settlement` — Shares withheld upon RSU/PSU vesting for payroll taxes (supplemental; kept distinct from discretionary buybacks).

### Issuance (`direction = issuance`, `net_amount_usd > 0`)
- `issuance_underwritten_offering` — Primary underwritten follow-on offerings, public offerings, or direct placements of common stock (secondary selling-stockholder shares strictly excluded).
- `issuance_atm_offering` — Actual primary common shares sold under At-the-Market equity distribution programs (never count mere ATM program shelf capacity!).
- `issuance_acquisition` — Common shares issued as consideration in mergers and acquisitions.
- `issuance_employee_equity` — Common shares issued under employee equity plans (options, RSUs/PSUs, ESPP, executive equity awards).
- `issuance_warrant_exercise` — Common shares issued upon warrant exercises.
- `issuance_conversion` — Common shares issued upon conversion of convertible debt or convertible preferred stock.
- `issuance_other_common` — Residual primary common stock issuances only after checking all specific categories above (DRIP, debt-for-equity exchanges, legal settlements).

### Supplemental Corporate Action
- `retirement_cash_merger` — Common equity retired when an S&P 500 constituent is acquired for cash or taken private (`direction = retirement`, `net_amount_usd < 0`, `include_in_net_aggregate = false` in cash-flow headline; quantified separately for Fed Z.1 & Acadian bridging).

## 3. Consistent Valuation Policy & Multiple Methodology Views

1. **Headline Event Cash-Flow Basis (`NI_cash`)**:
   - **Buybacks**: Cash paid (`cash_paid`) or `shares * avg_price_usd` (`shares_times_avg_price`).
   - **Offerings (`underwritten`, `atm`)**: Gross offering proceeds (`gross_proceeds`, with `shares * offer_price` recorded in `calculation_formula` when computed; `net_proceeds` if only net proceeds are disclosed).
   - **Acquisitions & Conversions**: Closing/conversion-date fair value (`fair_value_at_issuance`) from the primary filing.
   - **Employee Equity Plans (Headline)**: Actual cash proceeds (`cash_proceeds` from option exercises & ESPP).
2. **Supplemental Event Fair-Value Basis (`NI_sbc_fv`)**:
   - Also record filing-disclosed vest-date fair value of employee equity awards (`sub_id = sbc_fair_value`, `valuation_basis = vest_date_fair_value`, `include_in_net_aggregate = false`) and `tax_withholding_net_settlement` / `retirement_cash_merger` so both cash-flow net issuance AND full economic/SBC-adjusted net issuance can be computed directly from `events.csv`!
3. **Market-Value Share-Count & Float-Adjusted Measures (for Acadian Verification)**:
   - Computed directly from primary SEC share counts + `N-PORT-P` prices/holdings per `docs/VERIFICATION.md` §2.

## 4. True Point-in-Time (Bi-Temporal) Revision & Lifecycle Chains

1. **Canonical `event_key` & Unique Sub-Transaction IDs**:
   `{cik}|{category}|{share_class}|{granularity}|{period_start}|{period_end}|{sub_id}`
   *(For multi-filing corporate transactions—e.g., an offering priced in a `424B5`, expanded by greenshoe in an `8-K`, and confirmed in a `10-Q`; an ASR announced in an `8-K` and settled 3 months later; or a tender offer announced in `SC TO-I` and finalized in `SC TO-I/A`—link all lifecycle observations under a **shared `event_key`** so an `as_of_date` query never drops the event between announcement and quarterly confirmation! Conversely, when TWO distinct deals occur for the same issuer in the same quarter—such as a direct placement AND an ATM program—assign **distinct `sub_id`s** and `granularity = transaction` so one never overwrites or supersedes the other!)*
2. **Never Overwrite Prior Observations & Zero Look-Ahead Across Stages**:
   - Every newly filed update, settlement, greenshoe, or restatement creates a **new row** with its own `record_id`, the new filing's `published_at` (`acceptanceDateTime` UTC), `is_revision = true`, and `supersedes_record_id = <prior record_id>`.
   - **STRICT ANTI-LOOK-AHEAD RULE**: Every lifecycle stage MUST carry **ONLY the figure disclosed in THAT specific filing on its own `published_at`** — NEVER stamp an earlier `424B` shelf or `8-K` announcement with a dollar amount that only became known in a later `10-Q`/`10-K`, and NEVER replace a `424B`/`8-K` `gross_proceeds` figure with a `10-Q` `net_proceeds` figure without updating `valuation_basis = net_proceeds` (prefer preserving gross proceeds when the `10-Q` only states net!).
   - Populate the first-class **`superseded_at`** column on the prior row with the newer row's `published_at` (leaving `superseded_at` blank on active rows) and set `include_in_net_aggregate = false` on the superseded row while keeping the row intact in `events.csv`.
3. **Exact Point-in-Time `as_of(D)` Operator**:
   - Implement a reusable library (`work/src/<pkg>/pit.py`) computing `as_of(D)` for any timestamp `D` cleanly using the first-class `superseded_at` column (zero regex on `notes`!):

```text
A(D) = { r : in_scope(r) and published_at(r) <= D and (superseded_at(r) == "" or superseded_at(r) > D) }
Headline_as_of(D) = sum_{r in A(D)} net_amount_usd(r) * member_overlap_fraction(r)
Check: A(max published_at) == { r : include_in_net_aggregate == 1 }
```

   - Enforce `A(max published_at) == headline` as a **blocking validator check**, AND verify `as_of(D)` at ≥ 3 intermediate historical dates **inside the trailing-12-month headline window** (e.g., right after a major `424B5`/`8-K` offering or ASR before the `10-Q` was filed) to prove live point-in-time reconstruction works where it matters most!
4. **Granularity Superseding (Zero Double-Counting)**:
   - When monthly buybacks or transaction-level offerings/ASRs/M&A supersede an overlapping Tier 1 quarterly XBRL fallback row, populate `superseded_at` (with the replacing row's `published_at`) and set `include_in_net_aggregate = false` on the fallback row (noting the replacement in `notes`) so summing `include_in_net_aggregate == true` never double-counts.

## 5. Evidence-Based Stock Split Policy

- Preserve **raw as-reported share counts** (`split_adjusted = false`, `split_adjustment_note = unadjusted as reported on filing date`) on initial filings.
- Mark `split_adjusted = true` **only with concrete primary-filing evidence** (an observed split event or explicit comparative restatement where share counts jump by a clean split ratio while dollar values stay unchanged—never guess splits solely from stock price drift!). Record exact split ratio, effective date, and restating filing in `split_adjustment_note`.

## 6. Exact CSV Column Specification (`output/events.csv`)

UTF-8, RFC 4180 CSV, sorted deterministically by `event_date`, `cik`, `category`, `published_at`. **Serialize all boolean columns strictly as lowercase `true` or `false`** (never `True`/`False` or `1`/`0`). Reject off-enum values in the loader before publishing!

| Column | Type / Format | Required | Description |
|---|---|---|---|
| `record_id` | string | Yes | Unique deterministic ID for this observation row. |
| `event_key` | string | Yes | Canonical key grouping all lifecycle updates/revisions of the same economic event. |
| `cik` | string (10-digit) | Yes | Zero-padded SEC CIK of the issuer. |
| `ticker` | string | Yes | Ticker as of `event_date` — **blocking non-empty check**: backfill departed/predecessor CIKs from `dei:TradingSymbol` / `submissions.tickers` (blank allowed ONLY for genuinely unlisted entities with note). |
| `company_name` | string | Yes | Issuer legal/filing name. |
| `share_class` | string | Yes | Share class (`Common`, `Class A`, `Class B`, `All Common Classes`). |
| `security_type` | string | Yes | `common_stock`. |
| `is_financial` | boolean | Yes | Financial-sector flag (SIC 6000–6799) for Fed Z.1 nonfinancial comparison. |
| `sp500_member_on_event_date` | boolean | Yes | True iff `member_overlap_fraction > 0` (active on transaction date or periodic row overlaps active membership). |
| `member_overlap_fraction` | decimal (0.0–1.0) | Yes | `1.0` for active transactions & full-member periods, `0.0` outside membership, `overlap_days / total_days` for routine straddling periodic rows (preserves raw SEC `amount_usd` while allowing exact `net_amount_usd * member_overlap_fraction` aggregation). |
| `category` | enum string | Yes | Controlled category from Section 2. |
| `direction` | enum string | Yes | `issuance`, `buyback`, or `retirement`. |
| `event_status` | enum string | Yes | `completed` (settled/executed/reported), `announced_or_priced` (priced offering / launched ASR or tender prior to closing/10-Q), or `announced_unsettled` (forward-sale agreement prior to share delivery; `include_in_net_aggregate = false` until settled). |
| `granularity` | enum string | Yes | `transaction`, `monthly`, `quarterly`, or `annual`. |
| `event_date` | `YYYY-MM-DD` | Yes | Transaction/pricing/closing date, or `period_end` for periodic rows. |
| `period_start` | `YYYY-MM-DD` | Conditional | Start of activity period (equals `event_date` for single-day transactions). |
| `period_end` | `YYYY-MM-DD` | Conditional | End of activity period. |
| `shares` | decimal | Conditional | Positive share count (blank only if source omitted shares or blended multi-class counts). |
| `avg_price_usd` | decimal | Conditional | Per-share price in USD (disclosed or implied `amount_usd / shares`). |
| `amount_usd` | decimal | Yes | Positive USD dollar amount. |
| `net_amount_usd` | decimal | Yes | Signed USD amount (`+amount_usd` for issuance, `-amount_usd` for buybacks/retirements). |
| `valuation_basis` | string | Yes | `cash_paid`, `shares_times_avg_price`, `gross_proceeds`, `net_proceeds`, `fair_value_at_issuance`, `cash_proceeds`, `vest_date_fair_value`. |
| `split_adjusted` | boolean | Yes | Evidence-based flag indicating whether shares/prices reflect a post-event split adjustment. |
| `split_adjustment_note` | string | Yes | Explicit split note (`unadjusted as reported on filing date` or exact ratio/date/filing). |
| `published_at` | ISO-8601 UTC | Yes | Publication timestamp (`YYYY-MM-DDTHH:MM:SSZ` from SEC `acceptanceDateTime`, or `YYYY-MM-DD` if time unavailable). |
| `published_at_precision` | enum string | Yes | `second`, `minute`, or `date`. |
| `source_type` | enum string | Yes | `sec_10q`, `sec_10k`, `sec_8k`, `sec_424b`, `sec_s4`, `sec_sc_to_i`, `sec_other`, `company_announcement`, `sp_announcement`. |
| `source_accession_number` | string | Conditional | SEC accession number (`0000000000-00-000000`), blank only for non-SEC announcements. |
| `source_url` | URL | Yes | Direct URL to the primary filing or announcement. |
| `source_location` | string | Yes | Exact XBRL concept+context, iXBRL dimension, HTML table/item, or footnote reference. |
| `extraction_method` | enum string | Yes | `direct` or `calculated`. |
| `calculation_formula` | string | Conditional | Exact arithmetic formula and source inputs when `calculated`. |
| `is_revision` | boolean | Yes | True if this row updates/revises an earlier `published_at` observation of `event_key`. |
| `supersedes_record_id` | string | Conditional | Prior `record_id` superseded by this revision. |
| `superseded_at` | ISO-8601 UTC | Conditional | Blank if active; set to the replacing/revising row's `published_at` once superseded (enables regex-free `as_of(D)` queries). |
| `include_in_net_aggregate` | boolean | Yes | True for latest non-overlapping in-scope rows forming the headline `NI_cash` series. |
| `notes` | string | Optional | Concise provenance, superseding context, and methodological notes (tag any recent pre-10-Q/10-K estimated closing valuation with `[provisional_estimate]` so `--mode incremental` automatically replaces it with a linked revision once final filing fair value arrives). |

## 7. Required `output/README.md`

Keep `output/README.md` crisp and operational:
1. Summary of all files in `output/`.
2. Complete column dictionary for `events.csv`.
3. Category definitions, sign conventions, valuation variants (`NI_cash`, `NI_sbc_fv`, Acadian `NI_sc`, float-adjusted, Baker-Wurgler), and split policy.
4. Concrete **Point-in-Time Query Guide** showing how to query `events.csv` (and `pit.py`) for any historical `as_of_date`, with a concrete worked example of an event whose value/status evolved from announcement (`424B`/`8-K`) to `10-Q` confirmation.
5. Exact **Continuous / Incremental Run Guide** (`--mode incremental`, automated S&P DJI + EDGAR daily index polling, recommended cron schedule, and rollback test summary).
