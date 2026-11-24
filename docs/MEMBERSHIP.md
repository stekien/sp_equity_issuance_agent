# Point-in-Time S&P 500 Membership Methodology

Every company that was an S&P 500 member at any point during the covered window must be included—and its events counted **only while it was an active member**. All membership data in the output must come from **primary sources** (SEC filings, S&P Dow Jones Indices announcements, and company announcements). Wikipedia and financial portals may ONLY be used afterward to verify completeness and catch gaps.

## 1. Primary-Source Discovery Strategy

### Step A: Authoritative Quarterly Snapshots via SEC Form `N-PORT-P`
Major S&P 500 index-tracking ETFs (such as Vanguard S&P 500 ETF and iShares Core S&P 500 ETF) are SEC-registered investment companies that file **Form `N-PORT-P`** on SEC EDGAR containing complete XML schedules of portfolio holdings (issuer name, CUSIP, LEI, shares, market value, portfolio weight) as of each reporting period.
1. Locate the trust/series CIKs on SEC EDGAR using EDGAR entity search or `https://www.sec.gov/files/company_tickers.json` / `https://efts.sec.gov/LATEST/search-index` (verify you select the specific S&P 500 index fund series, not an unrelated fund in the same trust).
2. Download the `N-PORT-P` primary XML documents covering the trailing 12 months and going back to 2021 (typically 4 filings per year, plus the most recent filing available).
3. Parse equity holdings (common stock holdings, excluding cash/futures/repo/money-market collateral) to build quarterly constituent snapshots.
4. Capture each constituent's **portfolio weight** from the latest snapshot—this gives a free, primary-sourced materiality ranking for prioritizing Phase 3 extraction (largest weights first!).

*Note on lag*: `N-PORT-P` filings have up to a ~60-day SEC reporting lag. Bridge the gap from the latest `N-PORT-P` report date to today using Step B below.

### Step B: Exact Effective Dates via S&P Announcements & Company `8-K` Filings
Comparing consecutive `N-PORT-P` snapshots immediately reveals every **addition** and **removal** (as well as mergers, ticker changes, and spin-offs) between snapshot dates, and S&P announcements cover any changes since the most recent `N-PORT-P`.
1. For each detected change between snapshots—and for the window from the latest `N-PORT-P` up to today—find the **exact effective date** and **announcement publication timestamp** from primary sources:
   - **S&P Dow Jones Indices press releases** (official S&P Global newsroom / syndicated primary press releases).
   - **SEC EDGAR Full-Text Search** (`https://efts.sec.gov/LATEST/search-index`) searching `8-K` filings and press release exhibits (`EX-99.1`) mentioning S&P 500 addition/inclusion, merger closing (`Item 2.01` / `Item 5.01`), or spin-off distribution (`Item 8.01` / `Form 10`).
2. Record the primary source URL / accession number and exact effective date (`member_start_date`, `member_end_date`) in the `constituents` table.
3. If an exact effective date between two `N-PORT-P` snapshots cannot be confirmed from a primary source within your time budget, bound it transparently using the adjacent `N-PORT-P` snapshot dates and flag it explicitly in `COVERAGE.md` as an approximate boundary—never silently copy an unverified date from Wikipedia.

### Step C: CIK Resolution (Including Departed & Acquired Companies)
SEC's `company_tickers.json` only lists currently active filers. Departed, acquired, merged, or renamed historical S&P 500 members will often be missing from `company_tickers.json`!
- Resolve active constituents via `company_tickers.json` / `company_tickers_exchange.json`.
- Resolve departed or renamed constituents using SEC EDGAR entity search / `efts.sec.gov` by issuer name, historical ticker, CUSIP, or LEI from `N-PORT-P`.
- Verify every mapped CIK by inspecting `submissions/CIK{cik10}.json` (confirm company name, former names, SIC code, and filing dates match the membership period).
- Record each issuer's SIC code from `submissions` so you can cleanly classify **Financials** (typically SIC 6000–6799, verified for edge cases like REITs/fintechs) vs. **Nonfinancials** when comparing against Federal Reserve Z.1 nonfinancial corporate figures.

## 2. Special Corporate Structures

### Multi-Share-Class Companies
Some S&P 500 issuers have multiple share classes (either both classes included in the index or only one class included).
- Detect multi-class constituents dynamically whenever two `N-PORT-P` holdings map to the same CIK or parent issuer.
- **Zero double-counting rule**: Company-level SEC XBRL filings (`companyfacts`) report consolidated company-wide common stock buybacks and issuances across all common share classes. Never pull and sum the same CIK's XBRL totals twice for dual-ticker companies!
- Store events at the **issuer (CIK) level** with `share_class` populated as specifically as the primary filing allows (e.g., `Class A`, `Class C`, or `All Common Classes` when the filing only reports a consolidated total). Document multi-class treatment explicitly in `output/README.md` and `COVERAGE.md`.

### Mergers, Acquisitions, and Spin-Offs
- **Acquirer in S&P 500 issues stock**: Record the shares issued in the acquisition under `issuance_acquisition` for the acquirer (while it is a member).
- **Target in S&P 500 acquired for cash (or taken private)**: Membership ends on the effective delisting/closing date. Optionally record the equity retirement under `retirement_cash_merger` (clearly tagged and excluded from or shown separately alongside standard issuance/buybacks, per `docs/SCHEMA_AND_RULES.md`) so the Fed Z.1 / Acadian comparison can quantify cash M&A retirements cleanly.
- **Target in S&P 500 acquired in a stock-for-stock deal by another S&P 500 member**: Membership of target ends on closing; acquirer's share issuance is recorded on the acquirer. Note the stock-for-stock offset clearly in `COVERAGE.md` when reconciling against aggregate market-wide net issuance (since stock-for-stock M&A within the index is economically net zero in aggregate!).
- **Spin-offs**: A spun-off subsidiary distributed to existing S&P 500 shareholders raises $0 of new equity capital (it is a pro-rata distribution, not a capital-raising issuance). Do NOT count a pro-rata spin-off distribution as positive equity issuance! Track the new entity's S&P 500 membership start/end dates accurately if it enters (or briefly enters and exits) the index.

## 3. Membership Boundary Rule for Period Events

- **Transaction-level events** (`granularity = transaction`): Include if and only if `member_start_date <= event_date <= member_end_date`.
- **Monthly / Quarterly period events** (`granularity = monthly` or `quarterly`):
  - Include if the entire period `[period_start, period_end]` falls within active membership.
  - For a **partial-period** entry or exit (company joined or left mid-month or mid-quarter): prefer transaction/monthly data where available; if only a quarterly figure exists, apply a clear, deterministic rule (e.g., include if active on `period_end` OR prorate/flag clearly with `notes`) and document the exact rule and materiality impact in `COVERAGE.md`.

## 4. Independent Verification Check (Wikipedia Cross-Check)

After constructing the primary-source membership table:
1. Fetch Wikipedia's S&P 500 constituent list and historical changes table strictly as a **verification check**.
2. Compare: total current constituent count (~500 companies / ~503 listings), every addition/removal over the covered period, and exact effective dates.
3. If Wikipedia reveals a constituent or change you missed, do **NOT** copy it from Wikipedia—instead, go back to SEC EDGAR (`N-PORT-P` / `8-K` / S&P announcements) to verify and ingest it from a primary source!
4. Report the exact reconciliation statistics (matches, resolved gaps, any remaining unexplained differences) in `output/COVERAGE.md`.
