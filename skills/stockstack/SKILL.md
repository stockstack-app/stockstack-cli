---
name: stockstack
description: "Retrieve and analyze Japanese stock fundamental data using the StockStack CLI. Covers company search, financial statements (IS/BS/CF), financial ratios (ROE/PER etc.), disclosure content (annual report sections), and watchlists via CLI commands. Use this skill when a user asks about Japanese stock tickers, earnings, financial data, securities reports, or shareholder information, or when they mention a securities code (4-5 digit number)."
allowed-tools: Bash(stockstack *), Bash(jq *), Bash(for *), Bash(echo *)
---

# StockStack CLI

StockStack is a CLI tool for Japanese stock fundamental analysis. It retrieves company financials, disclosure content, and analysis metrics as JSON.

**Why use the CLI**: The `stockstack` CLI handles API authentication, rate limiting, and error handling internally — there is no need to call the API directly. Always use CLI commands for data retrieval.

**Detailed usage**: Run `stockstack --help` or `stockstack <command> --help` for full option details and available flags.

## Setup

If authentication is not configured (commands return auth errors):

```bash
stockstack auth login    # Open browser for OAuth login
stockstack auth status   # Check plan and remaining quota
stockstack auth logout   # Clear stored tokens
```

## Command Reference

### Account Information

```bash
stockstack account plan                            # Plan details, rate limit, daily quota
```

### Company Search

```bash
stockstack companies list --q Toyota               # Keyword search
stockstack companies list --market prime --limit 10 # Filter by market segment
```

### Financial Statements

Financial commands share common options: `--from-year`, `--to-year`, `--basis`

```bash
stockstack financial income-statement 7203          # Income statement
stockstack financial balance-sheet 7203             # Balance sheet
stockstack financial cash-flow 7203                 # Cash flow statement
stockstack financial equity 7203                    # Statement of changes in equity
stockstack financial summary 7203                   # IS/BS/CF key items summary
stockstack financial reporting-bases 7203           # Available accounting standards
```

### Financial Analysis

```bash
stockstack analysis ratios 7203                     # ROE, ROA, equity ratio, etc.
stockstack analysis valuation 7203                  # PER, PBR, dividend yield, etc.
stockstack analysis indicators 7203                 # Revenue growth, operating margin (normalized)
```

### Corporate Structure

```bash
stockstack corporate segments 7203                  # Segment list (overview)
stockstack corporate segment 7203 <segment_key>    # Segment detail (performance data)
stockstack corporate shareholders 7203              # Major shareholders
```

### Filings & Disclosure

```bash
stockstack filings list 7203                                    # List annual reports
stockstack disclosure sections <filingId>                       # Section list (with block index)
stockstack disclosure block <filingId> <blockKey>               # Block content
stockstack disclosure block <filingId> <blockKey> --content-format text  # Plain text format
```

**Workflow**: `filings list` → `disclosure sections` → `disclosure block` (3 steps).
Use `sections` to see available blocks (key + label) per section, then pass the desired `blockKey` to `block`.

Content format: `--content-format markdown` (default) or `--content-format text`

### Watchlist

```bash
stockstack watchlist list
stockstack watchlist add 7203
stockstack watchlist remove 7203
```

## JSON Response Structures

All commands output JSON to stdout. Use `jq` for filtering. The `jq` patterns below are examples to illustrate the response structure — adapt them as needed for the task at hand.

### Financial Statements (IS/BS/CF)

Row-oriented: each period contains its own `lineItems`.

```json
{
  "consolidation": "consolidated",
  "appliedFromYear": 2020,
  "appliedToYear": 2024,
  "periods": [
    {
      "fiscalYear": 2024,
      "accountingStandard": "JGAAP",
      "lineItems": [
        { "key": "NetSales", "label": "売上高", "unit": "JPY", "value": 1500000000 }
      ]
    }
  ]
}
```

```bash
# List all line item keys and labels (latest period)
stockstack financial income-statement 7203 | \
  jq '.periods[0].lineItems[] | {key, label}'

# Find a specific item by label
stockstack financial income-statement 7203 | \
  jq '.periods[0].lineItems[] | select(.label | contains("売上"))'

# Extract values across all periods for a specific key
stockstack financial income-statement 7203 | \
  jq '[.periods[] | {fiscalYear, value: (.lineItems[] | select(.key == "NetSales") | .value)}]'
```

### Equity Statement (Statement of Changes in Equity)

Different structure from IS/BS/CF: each period contains `changes`, and each change has values `byComponent`.

```json
{
  "consolidation": "consolidated",
  "appliedFromYear": 2020,
  "appliedToYear": 2024,
  "periods": [
    {
      "fiscalYear": 2024,
      "accountingStandard": "JGAAP",
      "changes": [
        {
          "key": "IssuanceOfNewShares",
          "label": "新株の発行",
          "unit": "JPY",
          "byComponent": [
            { "memberName": "ShareholdersEquity", "label": "株主資本", "value": 1000000 },
            { "memberName": "TotalNetAssets", "label": "純資産合計", "value": 1000000 }
          ]
        }
      ]
    }
  ]
}
```

```bash
# List all change items in the latest period
stockstack financial equity 7203 | \
  jq '.periods[0].changes[] | {key, label}'

# Get values for a specific component across changes
stockstack financial equity 7203 | \
  jq '.periods[0].changes[] | {label, total: (.byComponent[] | select(.memberName == "TotalNetAssets") | .value)}'
```

### Financial Summary

Each period groups items by statement type (`income`, `balance`, `cashFlow`).

```json
{
  "periods": [
    {
      "fiscalYear": 2024,
      "accountingStandard": "JGAAP",
      "income": [{ "key": "...", "label": "...", "unit": "JPY", "value": 123 }],
      "balance": [{ "key": "...", "label": "...", "unit": "JPY", "value": 456 }],
      "cashFlow": [{ "key": "...", "label": "...", "unit": "JPY", "value": 789 }]
    }
  ]
}
```

### Financial Ratios

Same row-oriented structure with optional `abbreviation` field.

```bash
# Extract ROE across periods
stockstack analysis ratios 7203 | \
  jq '[.periods[] | {fiscalYear, roe: (.lineItems[] | select(.abbreviation == "ROE") | .value)}]'
```

### Valuation Metrics

Row-oriented with `reportingBasis` per period. Market metrics (PER, PBR, etc.) are included in `lineItems`.

```bash
# Get PER and PBR from latest period
stockstack analysis valuation 7203 | \
  jq '.periods[0].lineItems[] | select(.label | test("PER|PBR")) | {label, value}'
```

### Financial Indicators (Normalized)

Columnar format: `items[].values[]` parallel to `periods[]`.

```json
{
  "periods": [
    { "periodEnd": "2025-03-31", "fiscalYear": "2024年", "accountingStandard": "JGAAP", "reportingBasis": "consolidated" }
  ],
  "items": [
    { "key": "revenue", "label": "売上高", "labelEn": "Revenue", "section": "income", "unit": "JPY", "values": [1500000000] }
  ]
}
```

```bash
# List all normalized items
stockstack analysis indicators 7203 | jq '.items[] | {key, label, section}'

# Extract a specific item's time series
stockstack analysis indicators 7203 | \
  jq '{periods: [.periods[].fiscalYear], values: (.items[] | select(.key == "revenue") | .values)}'
```

### Major Shareholders

```bash
# Top 5 shareholders by holding ratio (latest period)
stockstack corporate shareholders 7203 | \
  jq '.periods[0].shareholders[:5][] | {name, holdingRatio}'
```

### Reporting Bases

Returns an array of available reporting bases.

```bash
stockstack financial reporting-bases 7203
# → [{"basis":"consolidated_ifrs","consolidation":"consolidated","accountingStandard":"IFRS","fromYear":2017,"toYear":2025,"isDefault":true}, ...]
```

## Accounting Standards

Japanese companies report under **JGAAP**, **IFRS**, or **US-GAAP**. Standards vary by company and may change over time. Check first, then retrieve data:

```bash
# Check available standards (e.g., Toyota: IFRS 2017-2025, US-GAAP 2012-2020)
stockstack financial reporting-bases 7203

# Retrieve data with a specific standard
stockstack financial income-statement 7203 --basis consolidated_ifrs --from-year 2022
```

`--basis` values: `consolidated_jgaap` (default), `consolidated_ifrs`, `consolidated_us_gaap`, `non_consolidated_jgaap`

## Advanced Analysis Patterns

For patterns beyond basic commands, see the reference documents:

- **Financial analysis patterns**: Period comparison, standard switching, ROE/PER time series, segment analysis, multi-ticker comparison, jq cheatsheet → [references/financial-patterns.md](references/financial-patterns.md)
- **Disclosure analysis patterns**: Filing search, section content retrieval, text analysis, year-over-year comparison, peer comparison → [references/disclosure-patterns.md](references/disclosure-patterns.md)

## Error Handling

- **Auth error**: Run `stockstack auth login` to re-authenticate
- **Rate limit**: Wait and retry (burst: 100 req/min, daily quota: check with `stockstack account plan`)
- **Ticker not found**: Verify the securities code (4-5 digits). Search with `stockstack companies list --q <name>`
