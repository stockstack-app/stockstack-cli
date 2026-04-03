# Financial Analysis Patterns

Example patterns for retrieving and analyzing financial data with the StockStack CLI. These are starting points — adapt the `jq` filters to suit your specific analysis needs.

## Period Comparison

### Multi-Year Income Statement

```bash
# Income statement for 5 years
stockstack financial income-statement 7203 --from-year 2020 --to-year 2024

# Revenue trend across periods
stockstack financial income-statement 7203 --from-year 2020 | \
  jq '[.periods[] | {fiscalYear, revenue: (.lineItems[] | select(.label | contains("売上")) | .value)}]'
```

### Balance Sheet Changes Over Time

```bash
# Total assets and net assets trend
stockstack financial balance-sheet 7203 --from-year 2020 | \
  jq '[.periods[] | {fiscalYear, items: [.lineItems[] | select(.label | test("総資産|純資産|自己資本")) | {label, value}]}]'
```

### Financial Summary for Quick Overview

Retrieves key IS/BS/CF items in one call — more efficient than fetching each statement separately.

```bash
stockstack financial summary 7203 --from-year 2022

# Extract income items from all periods
stockstack financial summary 7203 --from-year 2022 | \
  jq '[.periods[] | {fiscalYear, income: [.income[] | {label, value}]}]'
```

## Accounting Standard Switching

Standards differ by company and may change over time. Always check first.

```bash
# Check available standards
stockstack financial reporting-bases 7203
# → [{"basis":"consolidated_us_gaap",...,"fromYear":2012,"toYear":2020},
#    {"basis":"consolidated_ifrs",...,"fromYear":2017,"toYear":2025,"isDefault":true}]

# IFRS income statement
stockstack financial income-statement 7203 --basis consolidated_ifrs

# US-GAAP era data
stockstack financial income-statement 7203 --basis consolidated_us_gaap --from-year 2015 --to-year 2020
```

### Comparing Across Standard Changes

```bash
# Before and after IFRS adoption (2017) — compare overlapping year
stockstack financial income-statement 7203 --basis consolidated_us_gaap --from-year 2016 --to-year 2017
stockstack financial income-statement 7203 --basis consolidated_ifrs --from-year 2017 --to-year 2018
```

## Financial Ratio Analysis

### ROE/ROA Trend

```bash
# All ratios
stockstack analysis ratios 7203 --from-year 2020

# ROE trend
stockstack analysis ratios 7203 --from-year 2020 | \
  jq '[.periods[] | {fiscalYear, roe: (.lineItems[] | select(.abbreviation == "ROE") | .value)}]'

# Multiple ratios in one query (abbreviation can be null, so use // "" for null-safety)
stockstack analysis ratios 7203 --from-year 2020 | \
  jq '[.periods[] | {fiscalYear, ratios: [.lineItems[] | select((.abbreviation // "") | test("ROE|ROA")) | {(.abbreviation): .value}] | add}]'
```

### Valuation Metrics

PER, PBR, dividend yield, and market metrics are included in `lineItems`.

```bash
# All valuation metrics
stockstack analysis valuation 7203 --from-year 2020

# PER and PBR from the latest period
stockstack analysis valuation 7203 | \
  jq '.periods[0].lineItems[] | select(.label | test("PER|PBR")) | {label, value}'

# Dividend yield trend
stockstack analysis valuation 7203 --from-year 2020 | \
  jq '[.periods[] | {fiscalYear, dividendYield: (.lineItems[] | select(.label | contains("配当利回り")) | .value)}]'
```

### Normalized Financial Indicators

Columnar format — `items[].values[]` parallel to `periods[]`. Includes `section` for categorization.

```bash
# All items with their sections
stockstack analysis indicators 7203 | jq '.items[] | {key, label, section}'

# Revenue time series
stockstack analysis indicators 7203 | \
  jq '{periods: [.periods[].fiscalYear], revenue: (.items[] | select(.key == "revenue") | .values)}'

# All ratio-type items
stockstack analysis indicators 7203 | \
  jq '[.items[] | select(.section == "ratio") | {key, label, values}]'
```

## Equity Statement Analysis

The equity statement has a unique structure: `periods[].changes[].byComponent[]`.

```bash
# List all change items (e.g., dividends, net income, new shares)
stockstack financial equity 7203 | \
  jq '.periods[0].changes[] | {key, label}'

# Extract net assets total for each change item
stockstack financial equity 7203 | \
  jq '.periods[0].changes[] | {label, total: (.byComponent[] | select(.memberName == "TotalNetAssets") | .value)}'

# Compare equity changes across periods
stockstack financial equity 7203 --from-year 2022 | \
  jq '[.periods[] | {fiscalYear, changes: [.changes[] | {label, total: (.byComponent[] | select(.memberName == "TotalNetAssets") | .value)}]}]'
```

## Segment Analysis

```bash
# Segment list (overview)
stockstack corporate segments 7203 --from-year 2022

# Available segment keys
stockstack corporate segments 7203 | jq '.segments[] | {key, label}'

# Segment detail with performance data
stockstack corporate segment 7203 JapanSegment --from-year 2022

# Extract revenue from a specific segment
stockstack corporate segment 7203 JapanSegment | \
  jq '[.periods[] | {fiscalYear, revenue: (.lineItems[] | select(.label | contains("売上")) | .value)}]'
```

## Consolidated vs Non-Consolidated

```bash
# Consolidated (default)
stockstack financial income-statement 7203

# Non-consolidated
stockstack financial income-statement 7203 --basis non_consolidated_jgaap
```

## Multi-Ticker Comparison

```bash
# Compare ROE across competitors (Toyota, Honda, Nissan)
for ticker in 7203 7267 7201; do
  echo "=== $ticker ==="
  stockstack analysis ratios $ticker --from-year 2023 | \
    jq '[.periods[] | {fiscalYear, roe: (.lineItems[] | select(.abbreviation == "ROE") | .value)}]'
done
```

## jq Cheatsheet (Financial Data)

Periods are ordered newest-first (`.periods[0]` = latest).

```bash
# List all line item keys and labels (latest period)
jq '.periods[0].lineItems[] | {key, label}'

# Search items by label keyword
jq '.periods[0].lineItems[] | select(.label | contains("売上"))'

# Latest period values only
jq '.periods[0].lineItems[] | {label, value}'

# Non-null values only
jq '.periods[0].lineItems[] | select(.value != null) | {label, value}'

# Time series for a specific key
jq '[.periods[] | {year: .fiscalYear, val: (.lineItems[] | select(.key == "NetSales") | .value)}]'

# Financial summary: flatten all statement types (latest period)
jq '.periods[0] | (.income + .balance + .cashFlow)[] | {label, value}'

# Normalized indicators: pair periods with values
jq '[range(.periods | length) as $i | {year: .periods[$i].fiscalYear, value: .items[0].values[$i]}]'
```
